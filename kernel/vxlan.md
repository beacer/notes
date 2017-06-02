
数据结构
========

#### VXLAN虚拟设备

VXLAN使用虚拟设备实现，相关信息保存在`vxlan_dev`中，并附属于`net_device`结构。

```C
/* Pseudo network device */
struct vxlan_dev {
	struct hlist_node hlist;        /* vni hash table */
	struct list_head  next;         /* vxlan's per namespace list */
	struct vxlan_sock __rcu *vn4_sock;      /* listening socket for IPv4 */
#if IS_ENABLED(CONFIG_IPV6)
	struct vxlan_sock __rcu *vn6_sock;      /* listening socket for IPv6 */
#endif
	struct net_device *dev;
	struct net        *net;         /* netns for packet i/o */
	struct vxlan_rdst default_dst;  /* default destination */
	u32               flags;        /* VXLAN_F_* in vxlan.h */

	struct timer_list age_timer;
	spinlock_t        hash_lock;
	unsigned int      addrcnt;
	struct gro_cells  gro_cells;

	struct vxlan_config     cfg;

	struct hlist_head fdb_head[FDB_HASH_SIZE];
};
```

* hlist
* vn4_sock
* vn6_sock

每个`vxlan_dev`隶属于一个*VNI*。同时，每个`vxlan_dev`都有对应的UDP socket，即`vxlan_sock`，可理解为RFC中的VTEP 。默认情况下，多个监听相同*vxlan端口*的`vxlan_dev`共享同一个`vxlan_sock`（共享一个VTEP）。所以，`vxlan_sock`维护了一个`vxlan_dev`的Hash表 ，Hash Key为VNI。`vxlan_dev.hlist`即该Hash的节点。

> 监听的vxlan端口即`vxlan_config.dst_port`字段，它表示本地监听的UDP vxlan端口。

* default_dst

对于目的MAC对端VTEP未知（Unknow MAC Destination）的数据需要使用默认Remote即`vxlan_dev.default_dst`发送组播数据。此外，多播和广播也会映射到underlay的多播，它会保存在`default_dst`中。`vxlan_dev`所在的VNI保存在`default_dst.remove_vni`。

* hash_lock
* fdb_head
* age_timer

`vxlan_dev`在发Inner Ethernet包的时候，先根据目的MAC搜索FDB，FDB是一个Hash表。同时为FDB维护了一个老化定时器，定期进行垃圾收集。`hash_lock`用户保护FDB Hash。

#### UDP Socket

每个vxlan设备都有对应的UDP Socket用于发送接收封装VXLAN报文的underlay UDP数据报，同时多个vxlan设备（每个vxlan 设备都有自己的VNI）可能会共享`vxlan_sock`。因此`vxlan_sock`维护一个以VNI为key的哈希表，表的节点是`vxlan_dev`。

> 除非打开"no_shared"参数，否则只要元组`<net-namespace, family, vxlan-port, flags>`相同就会共享。

```C
/* per UDP socket information */
struct vxlan_sock {
	struct hlist_node hlist;          // net-namespace sock_list Hash元节点
	struct socket    *sock;		// UDP Socket
	struct hlist_head vni_list[VNI_HASH_SIZE];  // 以VNI为Key的vxlan_dev哈希表。
	atomic_t          refcnt;
	u32               flags;
};
```

`flags`表示不同的vxlan vtep工作模式，包括，是否打开“data plane learning", 是否支持RSC，是否作为ARP代理实现ARP redution等。具体参考`net/vxlan.h`中`VXLAN_F_XXX`的定义。

#### VXLAN配置

Linux下vxlan设备可以通过iproute2套件即*ip(8)*命令进行配置，示例如下，

```
ip link add vxlan0 type vxlan id 42 group 239.1.1.1 dev eth1 dstport 4789
```

输入以下命令获取帮助，

```
ip link help vxlan
```

可参考ip(8)或者[vxlan配置](https://www.kernel.org/doc/Documentation/networking/vxlan.txt)。`iproute2`工具通过netlink操作vxlan设备，相关netlink参数被转换并保存在`vxlan_config`结构中。

```C
struct vxlan_config {
	union vxlan_addr        remote_ip;
	union vxlan_addr        saddr;
	__be32                  vni;
	int                     remote_ifindex;
	int                     mtu;
	__be16                  dst_port;
	u16                     port_min;
	u16                     port_max;
	u8                      tos;
	u8                      ttl;
	__be32                  label;
	u32                     flags;
	unsigned long           age_interval;
	unsigned int            addrmax;
	bool                    no_share;
};
```

其中几个重要的字段描述如下，

* `remote_ip`

  当转发表(FDB)中没有目的MAC的entry时，或发送广播、多播时，Outter IP使用该多播地址发送vxlan数据。
  该字段可通过`ip`命令的*group*或*remote*参数配置。
  
 > 创建设备时会为remote_ip创建一个zero_mac的FDB条目，发包总是查询FDB。

* `dst_port`

  运行vxlan协议的outer UDP目的端口，IANA指定的VXLAN Well Known端口是4789，Linux为了兼容旧设备默认
  使用8472，用户可用`ip`命令的*dstport*参数配置。

* `vni`

  VXLAN设备的VNI。

* `remote_ifindex`

  VXLAN设备的lower设备的ifindex。

* `port_min`及`port_max`

  VXLAN UDP源端口选择范围。

#### per-net数据结构

```C
struct vxlan_net {
	struct list_head  vxlan_list;
	struct hlist_head sock_list[PORT_HASH_SIZE];
	spinlock_t        sock_lock;
};
```

* `vxlan_list`: 所在network namespace中所有的vxlan虚拟设备，即`vxlan_dev`。
* `sock_list`: namespace中，维护所有的vxlan的UDP Sockets，即`vxlan_sock`。


#### 转发数据库

`vxlan_dev`维护了`vxlan_fdb`的Hash表，即`vxlan_dev.fdb_head`，Key是Remote MAC和VNI。`hlist`是该Hash的节点。`vxlan_fdb`为FDB表。


```C
/* Forwarding table entry */
struct vxlan_fdb {
        struct hlist_node hlist;        /* linked list of entries */ // vxlan_dev.fdb_head表节点
        struct rcu_head   rcu;                                                                                      
        unsigned long     updated;      /* jiffies */              // 最近修改时间                      
        unsigned long     used;                                          // 最近使用时间                    
        struct list_head  remotes;                                     // vxlan_rdst链表                        
        u8                eth_addr[ETH_ALEN];                        // 源MAC
        u16               state;        /* see ndm_state */                                                         
        __be32            vni;                                                 // 所在VNI
        u8                flags;        /* see ndm_flags */                                                         
};
```

* hlist

Hash表`vxlan_dev.fdb_head`节点。

* updated
* used
   上次使用、更新的时间，used用于垃圾收集等。

* eth_addr

目的（目标VMMAC地址。

* remotes
   VTEP信息保存于`vxlan_rdst`。`remotes`即`vxlan_dst`链表。对于多播/all_zero MAC，一个源MAC可能有多个remote，发送数据时会向每个remote发送一个skb的拷贝。
	1. all_zero MAC：remotes是多播（vxlan默认多播），或者多个Unicast。
	2. Multicast MAC：有多个VTEP加入该多播组。（可用`bridge fdb add/del/append/replace`配置）

* vni
所在VNI。

结构`vxlan_rdst`描述remote VTEP。包括对方的IP，UDP端口，VNI和通往对方的本地vxlan接口ifindex以及路由缓存。

```C
struct vxlan_rdst {
        union vxlan_addr         remote_ip;
        __be16                   remote_port;
        __be32                   remote_vni;
        u32                      remote_ifindex;
        struct list_head         list;
        struct rcu_head          rcu;
        struct dst_cache         dst_cache;
};
```

> fdb可以用*bridge*命令查看 `bridge fdb show dev vxlan0`

初始化，设备创建及打开
=============

#### 模块初始化

vxlan是一个内核模块，其初始化函数如下，

1. 创建随机数用于`vxlan_sock`的VNI（`vxlan_dev`）hash，引入随机因子防DoS攻击。
1. *per-net*初始化，主要是初始化`vxlan_net`数据结构。
1. 注册*netdevice*通告链，用于处理lower-layer设备，及UDP隧道状态变化。
1. 注册*rtnetlink*回调函数组。

```C
static int __init vxlan_init_module(void)
{
        int rc;

        get_random_bytes(&vxlan_salt, sizeof(vxlan_salt));

        rc = register_pernet_subsys(&vxlan_net_ops);
        ... ...

        rc = register_netdevice_notifier(&vxlan_notifier_block);
        ... ...

        rc = rtnl_link_register(&vxlan_link_ops);
        ... ...
}
```

我们看下vxlan相关netlink的函数，

```C
static struct rtnl_link_ops vxlan_link_ops __read_mostly = {
        .kind           = "vxlan",
        .maxtype        = IFLA_VXLAN_MAX,
        .policy         = vxlan_policy,
        .priv_size      = sizeof(struct vxlan_dev),
        .setup          = vxlan_setup,
        .validate       = vxlan_validate,
        .newlink        = vxlan_newlink,
        .changelink     = vxlan_changelink,
        .dellink        = vxlan_dellink,
        .get_size       = vxlan_get_size,
        .fill_info      = vxlan_fill_info,
        .get_link_net   = vxlan_get_link_net,
};
```

#### 设备创建

##### 通过netlink创建设备

当用户通过*ip(8)*命令创建vxlan设备时，Kernel收到`RTM_NEWLINK`类型的netlink消息（保存了vxlan相关参数）。

```bash
$ ip link add vxlan0 type vxlan id 42 group 239.1.1.1 dev eth0 dstport 4789
$ ip link -d show
... ...
3: vxlan0: <BROADCAST,MULTICAST> mtu 1450 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 16:35:89:64:de:21 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 42 group 239.1.1.1 dev eth0 srcport 0 0 dstport 4789 ageing 300 addrgenmode eui64
```

于是`rtnl_newlink`函数被调用，它完成以下工作，


```C
static int rtnl_newlink(struct sk_buff *skb, struct nlmsghdr *nlh,
                        struct netlink_ext_ack *extack)
```

- 解析netlink消息中的参数，例如ifname="vxlan0"，type="vxlan", id=42, ...；
- 根据kind，搜索`rtnl_link_ops`，这里kind是vxlan，所以会找到`vxlan_link_ops`；
- 调用ops->validate（此处是vxlan_validate）进行进一步（虚拟设备相关的）的参数检查；
- 如果设备存在在，改变其参数，会调用`ops->change_link`
- 如果设备不存在，则创建、注册设备`net_device`（alloc_netdev_mqs/register_netdevice）


> 创建Linux网络设备的两个核心函数是`alloc_netdev_mqs`和`register_netdevice`。前者负责分配、初始化每个Kenel网络设备（包括所有物理设备、虚拟设备）的“基类” `net_device`。它根据`ops.priv_size`大小分配设备类型相关的私有数据，私有数据附属于`net_device`内存之后（继承），随后可通过`netdev_priv`获取。

> 对于vxlan而言，私有数据即是`vxlan_dev`；此外，`allo3: vxlan0: <BROADCAST,MULTICAST> mtu 1450 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 16:35:89:64:de:21 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 42 group 239.1.1.1 dev eth0 srcport 0 0 dstport 4789 ageing 300 addrgenmode eui64c_netdev_mqs`还会调用设备类型相关的*setup*函数`vxlan_setup`。

> `register_netdevice`则负责将`net_device`注册到kernel中，随后就能通过统一的ifindex/ifname进行管理；通过注册的`dev->netdev_ops/ethtool_ops`进行收发包、参数设置。

整个过程，以及其中`net_device`的私有数据（`vxlan_dev`）分配，`validate`, `setup`回调函数被调用处简述如下，

```
rntl_newlink
     +
     |- 从netlink msg进行参数解析，保存在nlattr *tb[]中。
     |- 按kind找到rtnl_link_ops  # 即vxlan_link_ops
     |- 调用ops->validate         # 即vxlan_validate
     |- rtnl_create_link # 若设备不存在，创建之
     |        +
     |        |-  alloc_netdev_mqs(ops->priv_size, ifname, , ops->setup, ...) 
     |        |    # 此处，分配vxlan_dev，调用vxlan_setup
     |        |-  dev_set_net(dev, net)           # 设置network-namespace
     |        |-  dev->rtnl_link_ops = ops     # 设置为vxlan_link_ops	
     |
     |- dev->ifindex = ifm->if_index  # 设置设备的ifindex
     |- ops->new_link()  #此处调用vxlan_newlink，设置netdev_ops，调用register_netdevice
```

##### vxlan_setup

```C
/* Initialize the device structure. */
static void vxlan_setup(struct net_device *dev)
{
        struct vxlan_dev *vxlan = netdev_priv(dev);
        unsigned int h;

        eth_hw_addr_random(dev);       # 随机生成vxlan设备的MAC地址，VXLAN设备是独立的L2中的Host
        ether_setup(dev);       # 设置MTU，以太网广播等ethernet设备common的信息

        dev->destructor = free_netdev; 
        SET_NETDEV_DEVTYPE(dev, &vxlan_type);

        dev->features   |= NETIF_F_LLTX;
        dev->features   |= NETIF_F_SG | NETIF_F_HW_CSUM;
        dev->features   |= NETIF_F_RXCSUM;
        dev->features   |= NETIF_F_GSO_SOFTWARE;

        dev->vlan_features = dev->features;
        dev->hw_features |= NETIF_F_SG | NETIF_F_HW_CSUM | NETIF_F_RXCSUM;
        dev->hw_features |= NETIF_F_GSO_SOFTWARE;
        netif_keep_dst(dev);  
        dev->priv_flags |= IFF_NO_QUEUE;

        INIT_LIST_HEAD(&vxlan->next);  
        spin_lock_init(&vxlan->hash_lock);  # 用于保护fdb_head[]

        init_timer_deferrable(&vxlan->age_timer);       # 初始化timer对FDB进行定期清理
        vxlan->age_timer.function = vxlan_cleanup;
        vxlan->age_timer.data = (unsigned long) vxlan;

        vxlan->cfg.dst_port = htons(vxlan_port);

        vxlan->dev = dev;     # 指向vxlan虚设备的net_device

        gro_cells_init(&vxlan->gro_cells, dev);

        for (h = 0; h < FDB_HASH_SIZE; ++h)
                INIT_HLIST_HEAD(&vxlan->fdb_head[h]);  # 初始化vxlan设备的转发数据库（FDB）
}

```

##### vxlan_newlink

```C
static int vxlan_newlink(struct net *src_net, struct net_device *dev,
                         struct nlattr *tb[], struct nlattr *data[])
{
        struct vxlan_config conf;
        int err;

        err = vxlan_nl2conf(tb, data, dev, &conf, false);
        if (err)
                return err;

        return __vxlan_dev_create(src_net, dev, &conf);
}

```

`rntl_newlink`将从netlink msg解析出的配置参数保存在`tb[]`中，而这里的`vxlan_nl2conf`将这些参数转换为vxlan内部的`vxlan_config`结构。`__vxlan_dev_create`用于初始化vxlan虚拟设备、并插入per-net结构，并调用`register_netdevice`将其对应的`net_device`注册到系统中。

```C
static int __vxlan_dev_create(struct net *net, struct net_device *dev,
                              struct vxlan_config *conf)
{
        struct vxlan_net *vn = net_generic(net, vxlan_net_id);
        struct vxlan_dev *vxlan = netdev_priv(dev);
        ... ...
        err = vxlan_dev_configure(net, dev, conf, false);
        ... ...
        dev->ethtool_ops = &vxlan_ethtool_ops;

        /* create an fdb entry for a valid default destination */
        if (!vxlan_addr_any(&vxlan->default_dst.remote_ip)) {
                err = vxlan_fdb_create(vxlan, all_zeros_mac,
                                       &vxlan->default_dst.remote_ip,
                                       ... ...);
        ... ...
        }

        err = register_netdevice(dev);
        ... ...

        list_add(&vxlan->next, &vn->vxlan_list);
        return 0;
}
```

1. `vxlan_dev_configure`完成的工作：

	* 设置`dev->netdev_ops`为`vxlan_netdev_ether_ops`
	* 设置`vxlan.default_dst`，用于发送“Unknow  MAC  Destination” Unicast，以及多播、广播数据；
	* 从lower-device继承GSO等设置；
	* 设置vxlan设备的MTU：大小为Lower设备MTU减去可能的最大vxlan Header大小；
	* 保持配置信息到`vxlan->cfg`。

2. 设置`dev->ethtool_ops`为`vxlan_ethtool_ops`；
3. 为default destination创建FDB条目；
4. 调用`register_netdevice`将vxlan设备的`net_device`注册到系统中；
5. 将`vxlan_dev`加入到per-net结构`vxlan_net.vxlan_list`中。

其中，`vxlan_netdev_ether_ops`需要特别关注，其定义如下，

```C
static const struct net_device_ops vxlan_netdev_ether_ops = {
        .ndo_init               = vxlan_init,
        .ndo_uninit             = vxlan_uninit,
        .ndo_open               = vxlan_open,　　　　　　　　　　　　　　// 设备打开（使能）
        .ndo_stop               = vxlan_stop,
        .ndo_start_xmit         = vxlan_xmit,            // 数据传输（TX）
        .ndo_get_stats64        = ip_tunnel_get_stats64,
        .ndo_set_rx_mode        = vxlan_set_multicast_list,
        .ndo_change_mtu         = vxlan_change_mtu,
        .ndo_validate_addr      = eth_validate_addr,
        .ndo_set_mac_address    = eth_mac_addr,
        .ndo_fdb_add            = vxlan_fdb_add,
        .ndo_fdb_del            = vxlan_fdb_delete,
        .ndo_fdb_dump           = vxlan_fdb_dump,
        .ndo_fill_metadata_dst  = vxlan_fill_metadata_dst,
};

```

#### 打开vxlan设备

创建完的vxlan设备，还需要使能下才能工作，或者说需要打开设备，

```
$ ip link set vxlan0 up
```

之前在创建设备的时候，将vxlan设备的`net_device.netdev_ops`设置为了`vxlan_netdev_ether_ops`。

```
vxlan_newlink
    +
    |- __vxlan_dev_create
          +
          |- vxlan_dev_configure
                  +
                  |- dev->netdev_ops = &vxlan_netdev_ether_ops
```

打开设备（open），意味着其`net_device.netdev_ops.ndo_open`即`vxlan_open`被调用。

`vxlan_open`完成以下工作

1. 设置vxlan设备对应的vxlan_sock，UDP  Socket可以尝试和之前创建的复用（共享），如果无法复用这需要创建。
2. 加入多播组，接收其他VTEP发送的多播地址。要加入的多播组，也是default destition的remote_ip多播。
3. 启动老化定时器，定期清理FDB。

关于VXLAN的多播， VXLAN发送多播的情况有

* unknow MAC destination，即VTEP未学习到对端的`MAC，VTEP IP`前。
* 发送多波Ethernet帧，映射到outer多播
* 发送广播Ethernet帧，映射到outer多播

同时，还需要接收vxlan里面对方发送的多播。发生和监听的多播在vxlan中相同，可配置。具体参考RFC 7348。

```C
static int vxlan_open(struct net_device *dev)
{
        ... ...
        ret = vxlan_sock_add(vxlan);       // 查找或创建vxlan_sock
        ... ...
        if (vxlan_addr_multicast(&vxlan->default_dst.remote_ip)) {
                ret = vxlan_igmp_join(vxlan);  // 加入多播组，接受其他VTEP发送的多播。
                ... ...
        }

        if (vxlan->cfg.age_interval)         // 启动老化定时器，定期清理FDB。
                mod_timer(&vxlan->age_timer, jiffies + FDB_AGE_INTERVAL);

        return ret;
}
```

#### 创建vxlan_sock

创建UDP Socket时候会向UDP tunnel注册回调函数来接收vxlan数据。

```C
static int __vxlan_sock_add(struct vxlan_dev *vxlan, bool ipv6)
{
        ... ...
        if (!vxlan->cfg.no_share) {        // 先尝试复用已有的vxlan_sock，除非设置了no_share
                ... ...
                vs = vxlan_find_sock(vxlan->net, ipv6 ? AF_INET6 : AF_INET,
                                     vxlan->cfg.dst_port, vxlan->flags);
                ... ...
        }   
        if (!vs)  // 如果没有vxlan_sock可以复用，则创建之。
                vs = vxlan_socket_create(vxlan->net, ipv6,
                                         vxlan->cfg.dst_port, vxlan->flags);
        ... ...
                rcu_assign_pointer(vxlan->vn4_sock, vs); // 设置vxlan_dev的vxlan_sock
        vxlan_vs_add_dev(vs, vxlan);  // 将vxlan_dev加入vxlan_sock的VNI哈希表中
        return 0;       
}

```

函数`vxlan_socket_create`负责创建`vxlan_sock`。

```C
/* Create new listen socket if needed */
static struct vxlan_sock *vxlan_socket_create(struct net *net, bool ipv6,
                                              __be16 port, u32 flags)
{
        ... ...
        vs = kzalloc(sizeof(*vs), GFP_KERNEL);
        ... ...
        sock = vxlan_create_sock(net, ipv6, port, flags);  // 调用udp_sock_create分配UDP socket{}。
        ... ...
        vs->sock = sock;
        ... ...
        hlist_add_head_rcu(&vs->hlist, vs_head(net, port));  // 将新创建的vxlan_sock加入vxlan_net.sock_list[]哈希
        ... ...

        /* Mark socket as an encapsulation socket. */
        memset(&tunnel_cfg, 0, sizeof(tunnel_cfg));
        tunnel_cfg.sk_user_data = vs;                       // 隧道用户数据为vxlan_sock
        tunnel_cfg.encap_type = 1;
        tunnel_cfg.encap_rcv = vxlan_rcv;                // 设置UDP tunnel的接收函数vxlan_rcv
        tunnel_cfg.encap_destroy = NULL;
        tunnel_cfg.gro_receive = vxlan_gro_receive;
        tunnel_cfg.gro_complete = vxlan_gro_complete;

        setup_udp_tunnel_sock(net, sock, &tunnel_cfg);

        return vs;
}
```

数据接收
========

#### RX主要流程

打开（enable）VXLAN设备的时候，向UDP tunnel注册了接收数据的callback，即`vxlan_rcv`。其中，`sock.sk_user_data`即对应的`vxlan_sock`（见`vxlan_socket_create`）。

```C
vxlan_rcv(struct sock *sk, struct sk_buff *skb)
        +
        |- sanity check
        |- vs = rcu_dereference_sk_user_data(sk)    // 取出vxlan_sock
        |- vni = vxlan_vni(vxlan_hdr(skb)->vx_vni)  // 取出VXLAN首部的VNI字段
        |- vxlan = vxlan_vs_find_vni(vs, vni)      // 以VNI为Key查找vxlan_sock.vni_list，找到对应的vxlan_dev
        |  //如果没有对应的vxlan说明该VNI无人接收，则立即丢弃。
        |
        |- __iptunnel_pull_header(...)             // 根据inner Ethernet头设置skb->protocol
        |- vxlan_set_mac(vxlan, vs, skb, vni) // 重设skb相关dev/protocol字段，地址学习（data-plane learning）
        |        +
        |        |- skb_reset_mac_header(skb)
        |        |- skb->protocol = eth_type_trans(skb, vxlan->dev); // 设置skb.dev为vxlan_dev.dev，调整skb.protocol
        |        |- 重新计算checksum
        |        |- 防止循环：丢弃源MAC是vxlan_dev地址的包
        |        |- vxlan_snoop(skb->dev, &saddr, eth_hdr(skb)->h_source, vni)
        |            // control-plane学习，即学习源MAC和源VTEP的IP，记录到FDB，后续发往该MAC使用Unicast UDP。
        |            // 之前可能已经被学习过而存在于FDB。
        |
        |- skb_reset_network_header(skb);
        |- ENC处理及收包统计（vxlan->dev->tstats）
        |- gro_cells_receive(&vxlan->gro_cells, skb) // 收包 netif_rx或者加入GRO NAPI队列后调度napi_schedule
            // 此时skb->dev是vxlan->dev，后续进入正常的netif/napi收包流程
}
```

#### Data-plane学习

`vxlan_snoop`完成MAC地址学习。

```
vxlan_snoop(skb->dev, &saddr, eth_hdr(skb)->h_source, vni)
          +
          |-  f = vxlan_find_mac(vxlan, src_mac, vni);  // 查询FDB看源MAC是否已经存在
          |-  if (likely(f)) // 已经存在
          |           +
          |           |- // 源IP和FDB Entry的相同，则直接返回。否则，如果不是静态配置的，则更新Entry的VTEP源IP。
          |-  else // 不存在
                      +
                      |-  vxlan_fdb_create(...) //不存在，则创建FDB条目。
```

数据发送
========

#### vxlan_xmit

之前提到`vxlan_dev`的`net_device_ops.ndo_start_xmit`是`vxlan_xmit`，该函数用于封装overlay的Ethernet数据为vxlan数据进行发送。

> 忽略flow-based tunnelling （VXLAN_F_COLLECT_METADATA）和ARP Proxy（VXLAN_F_PROXY）功能。

```C
static netdev_tx_t vxlan_xmit(struct sk_buff *skb, struct net_device *dev)
```

1. 根据目的MAC查找FDB条目，可以支持“[RSC (route shortcircuit)](http://events.linuxfoundation.org/sites/events/files/slides/2013-linuxcon.pdf)”优化。
2. 如果查找失败，用全0 MAC进行查找默认的Destination对应的FDB条目。这个条目在`__vxlan_dev_create`的时候被创建，如果默认destination的IP合法的话。通常这个IP是VXLAN多播。如果还没找到，则丢弃。
3. 遍历FDB条目(`vxlan_fdb`）的每个remotes（`vxlan_rdst`），每个remote都发送一个skb（多个remote需要`skb_clone`）。


```C
static netdev_tx_t vxlan_xmit(struct sk_buff *skb, struct net_device *dev)
{
        ... ...
        eth = eth_hdr(skb);   
        f = vxlan_find_mac(vxlan, eth->h_dest, vni);
        did_rsc = false;      

        if (f && (f->flags & NTF_ROUTER) && (vxlan->flags & VXLAN_F_RSC) &&
            (ntohs(eth->h_proto) == ETH_P_IP ||
             ntohs(eth->h_proto) == ETH_P_IPV6)) {
                did_rsc = route_shortcircuit(dev, skb);
                if (did_rsc)  
                        f = vxlan_find_mac(vxlan, eth->h_dest, vni);
        }

        if (f == NULL) {
                f = vxlan_find_mac(vxlan, all_zeros_mac, vni);
                if (f == NULL) {
                        if ((vxlan->flags & VXLAN_F_L2MISS) &&
                            !is_multicast_ether_addr(eth->h_dest))
                                vxlan_fdb_miss(vxlan, eth->h_dest);

                        dev->stats.tx_dropped++;
                        kfree_skb(skb);
                        return NETDEV_TX_OK;
                }
        }

        list_for_each_entry_rcu(rdst, &f->remotes, list) {
                struct sk_buff *skb1;

                if (!fdst) {
                        fdst = rdst;
                        continue;
                }
                skb1 = skb_clone(skb, GFP_ATOMIC);
                if (skb1)
                        vxlan_xmit_one(skb1, dev, vni, rdst, did_rsc);
        }

        if (fdst)
                vxlan_xmit_one(skb, dev, vni, fdst, did_rsc);
        else
                kfree_skb(skb);
        return NETDEV_TX_OK;
}
```

参考资料
======

1. [Software Defined Networking using VXLAN](http://events.linuxfoundation.org/sites/events/files/slides/2013-linuxcon.pdf)
2. [vxlan.txt](https://www.kernel.org/doc/Documentation/networking/vxlan.txt)
3. [RFC7348](https://tools.ietf.org/html/rfc7348)
