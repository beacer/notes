# Linux虚拟设备 之 *隧道 (ipip/gre)*

Linux的隧道设备`ipip`, `gre/gretap`, `sit`, `6rd`等，都是通过内核模块的形式实现的，同时Linux将基于IPv4的各种隧道的公共代码编写了一个模块`ip_tunnel` (*ip_tunnel.c/ip_tunnels.h*)．另一个，内核模块`tunnel4`（及其`xfrm`）则为各个隧道提供了数据接收、传输的框架。

## 使用隧道设备

加载`ipip`模块后，才能使用IP-in-IP隧道，加载时会自动加载所依赖的内核模块（`tunnel4`和`ip_tunnel`）。同时生成一个默认的`tunl0`。隧道的MTU比普通以太网设备小些，需要额外空间存放隧道header。

```
$ modprobe ipip
$ lsmod | grep ipip
ipip                   16384  0
tunnel4                16384  1 ipip
ip_tunnel              28672  1 ipip

$ ip link show
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
```

在Host A `10.3.3.3`和Host B `10.4.4.4`间建立ipip隧道，只需要A、B在Layer 3可达即可。并使用overlay网络`192.168.0.0/24`进行通信。下面是Host A的配置，Host B的配置类似。如果overlay网络要配置非简单的直连（on-link）地址，注意合理配置路由。

```
$ sudo ip tunnel add ipip0 mode ipip local 10.3.3.3 remote 10.4.4.4 ttl 64 dev eth0
$ sudo ip addr add 192.168.0.1/24 dev ipip0
$ sudo ip link set ipip0 up
```

查看隧道设备是否工作。

```
$ ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=31.4 ms
```

## 数据结构

Per-net Tunnel结构，保存net中所有同类型的tunnel设备，以及fb设备等。fb即"fallback"设备，是指，`no source, no destination, no key, no options`的设备。数据包中如果有"key"查找tunnel的时候key必须匹配，否则就匹配keyless tunnel。而fb tunnel作为“keyless” tunnel查找失败后，最后的选择。

```
ip_tunnel_net {
    struct net_device *fb_tunnel_dev;
    struct hlist_head tunnels[IP_TNL_HASH_SIZE];
    struct ip_tunnel __rcu *collect_md_tun;
};
```

Tunnel设备私有数据（附属于`net_device`）

```
struct ip_tunnel {
        struct ip_tunnel __rcu  *next;
        struct hlist_node hash_node;    // per-net Hash表节点
        struct net_device       *dev;
        struct net              *net;   /* netns for packet i/o */

        ... ICMP错误统计、GRE, ERSPAN, 相关信息 ...

        struct dst_cache dst_cache;     // 路由缓存

        struct ip_tunnel_parm parms;    // 配置参数

        int             mlink;
        int             encap_hlen;     /* Encap header length (FOU,GUE) */
        int             hlen;           /* tun_hlen + encap_hlen */
        struct ip_tunnel_encap encap;

        ... 6rd信息 ...

        struct ip_tunnel_prl_entry __rcu *prl;  /* potential router list */
        unsigned int            prl_count;      /* # of entries in PRL */
        unsigned int            ip_tnl_net_id;
        struct gro_cells        gro_cells;
        __u32                   fwmark;
        bool                    collect_md;
        bool                    ignore_df;
};
```

`ipip`设备的netlink操作函数，

```
static struct rtnl_link_ops ipip_link_ops __read_mostly = {
        .kind           = "ipip",
        .maxtype        = IFLA_IPTUN_MAX,
        .policy         = ipip_policy,
        .priv_size      = sizeof(struct ip_tunnel), # net_device私有数据
        .setup          = ipip_tunnel_setup,        # alloc_netdev时调用
        .validate       = ipip_tunnel_validate,
        .newlink        = ipip_newlink,
        .changelink     = ipip_changelink,
        .dellink        = ip_tunnel_dellink,
        .get_size       = ipip_get_size,
        .fill_info      = ipip_fill_info,           # dump信息
        .get_link_net   = ip_tunnel_get_link_net,
};
```

## 初始化

`ipip` 作为一个内核模块实现，初始化流程如下，

```
module_init(ipip_init);
  |- 注册per-net设备
      |- ops->init() 即 ipip_init_net()
            |- ip_tunnel_init_net(net, ipip_net_ops, "tunl0")
                  |- 初始化tunnel设备的per-net Hash: ip_tunnel_net.tunnels  
                  |- 创建fb设备 “tunl0”（需要rtnl_link_ops）
                      __ip_tunnel_create(net, ops, param)
                          |- alloc_netdev(): 私有数据为ip_tunnel{}
                          |- dev->rtnl_link_ops = ops
                          |- register_netdevice()
  |- 注册xfrm4隧道hander用于数据接收
      |- xfrm4_tunnel_register(&ipip_handler, AF_INET);
  |- 注册netlink操作：
      |- rtnl_link_register(&ipip_link_ops);
```

## 添加、删除隧道

先考察下如何创建隧道设备。至于删除过程，它基本是创建的逆过程，因此不再深入研究。

### 创建隧道接口

使用`ip tunnel ...`命令添加隧道之后内部发生了什么？我们先从整体看看，整个隧道设备的创建过程。

`ip`命令（*iproute2*）会使用*netlink*套接字组装和发送`RTM_NEWLINK`消息，最终会走到注册的*rtnetlink*函数`rtnl_newlink`。该函数完成的工作大致如下，

1. 解析netlink套接字传递的参数（也就是ip tunnel 命令跟随的参数 ），做通用参数检查;
2. 然后根据kind找到`rtnl_link_ops`，此处是`ipip_link_ops`。
3. 进一步根据kind的`.policy`解析参数，并调用类型相关的`.validate`检查。
4. 如果设备已存在（修改）则调用`ops->changelink`。
5. 如果要创建设备，调用`rtnl_create_link`，后者调用`alloc_netdev_mqs`分配`net_device`，分配并部分初始化后会调用`.setup`即`ipip_tunnel_setup`做类型特定的初始化工作。
6. 设置ifindex
7. 调用`ops->newlink`即`ipip_newlink`。通常,`->newlink`会调用`register_netdevice`将设备注册到系统中。如果->newlink为空，主动调用`register_netdevice`。
8. 通告netdev事件、其他内核模块或userspace程序可能监听该事件。

整个过程中，牵涉到了许多`rtnl_link_ops`字段,换一个角度，从`rtnl_link_ops`需要实现的Callback（"虚函数"）本身，重新总结下，

|rtnl_link_ops字段|`ipip`模块相应值|说明|
|--|--|
|.kind|"ipip"|用来查找`rtnl_link_ops`|
|.policy|ipip_policy|解析kind特定参数，例如`ipip`的*remote*, *local*等|
|.priv_size|sizeof(struct ip_tunnel)|调用`alloc_netdev_mqs`分配`net_device`附属的私有结构空间|
|.setup|ipip_tunnel_setup|调用`alloc_netdev_mqs`分配`net_device`同时调用`.setup`做类型特定初始化。|
|.validate|ipip_tunnel_validate|解析类型特定参数后，进行验证|
|.newlink|ipip_newlink|创建完`net_device`后，进行新接口设置，通常在内部调用`register_netdevice`向系统注册设备。|
|.changelink|ipip_changelink|修改设备参数|

> 如果用`ifconfig`创建隧道接口，则使用传统的`ioctl`,此处不讨论`ioctl`方式。

了解了这些netlink callback的调用背景后，以`ipip`为例，看看部分callback具体实现，

### 函数`ipip_tunnel_setup`

我们知道，`net_device`的数据收发、ioctl，打开关闭等，是通过其设备相关的虚函数表`net_device_ops`来实现的，分配了`ipip`隧道设备的net_device后，`ipip_tunnel_setup`会设置相关的`net_device_ops`。这些函数，会在设备打开、关闭和数据传输接收时提到。

然后是设置设备类型`ARPHRD_TUNNEL`，`IFF_XXX`标记，和features等。最后，会调用`ip_tunnel_setup`进行隧道通用的设置，后者目前只设置了`tunnel->ip_tnl_net_id`。

```
static const struct net_device_ops ipip_netdev_ops = {
        .ndo_init       = ipip_tunnel_init,
        .ndo_uninit     = ip_tunnel_uninit,
        .ndo_start_xmit = ipip_tunnel_xmit,
        .ndo_do_ioctl   = ipip_tunnel_ioctl,
        .ndo_change_mtu = ip_tunnel_change_mtu,
        .ndo_get_stats64 = ip_tunnel_get_stats64,
        .ndo_get_iflink = ip_tunnel_get_iflink,
};

static void ipip_tunnel_setup(struct net_device *dev)
{
        dev->netdev_ops         = &ipip_netdev_ops;

        dev->type               = ARPHRD_TUNNEL;
        dev->flags              = IFF_NOARP;
        dev->addr_len           = 4;
        dev->features           |= NETIF_F_LLTX;
        netif_keep_dst(dev);

        dev->features           |= IPIP_FEATURES;
        dev->hw_features        |= IPIP_FEATURES;
        ip_tunnel_setup(dev, ipip_net_id);  // 只是设置了tunnel->ip_tnl_net_id。
}
```

### 函数`ipip_newlink`

函数会继续解析尚未解析的参数到内部结构，例如封装参数（type, sport, dport），collect_md和fwmark。然后调用统一库函数`ip_tunnel_newlink`。

```
ipip_newlink
   +
   |- ipip_netlink_encap_parms(data, &ipencap)
   |- ipip_netlink_parms(data, &p, &t->collect_md, &fwmark);
   |- ip_tunnel_newlink(dev, tb, &p, &fwmark)
```

`ip_tunnel_newlink`完成的工作，

* 确保设备（在per-net的tunnel Hash或colect_md上）不存在，否则返回`-EEXIST`;
* 设置参数：`ip_tunnel`的`.net`，`.parms`, `.fwmark`等;
* 调用`register_netdevice`向系统**注册**网络设备;
* （如果类型是*ARPHDR_ETHER*)未通过参数指定HW地址则生成随机地址。
  - `ipip`设备类型是*ARPHRD_TUNNEL*而非*ARPHDR_ETHER*，故没有以太网地址。
* `ip_tunnel_bind_dev`：返回mtu并设置需要的headroom（dev->needed_headroom）。
  - 根据参数进行路由查找，并保存到tunnel->dst_cache，可能的话同时确定下层设备。
  - 根据下层设备（路由查找得到的外出设备或param->link）的MTU以及需要的封装header设置mtu及需要的headroom。
* `ip_tunnel_add`插入新`ip_tunnel`设备到Per-net结构
  - 将设备插入per-net的`ip_tunnel_net->tunnels`哈希表；以及，如果需要`ip_tunnel_net->collect_md_tun`。

### 函数`ipip_tunnel_init`

`register_netdevice`时会调用`dev->ndo_init`，即`ipip_tunnel_init`函数。

```
static int ipip_tunnel_init(struct net_device *dev)
{
        struct ip_tunnel *tunnel = netdev_priv(dev);

        memcpy(dev->dev_addr, &tunnel->parms.iph.saddr, 4);
        memcpy(dev->broadcast, &tunnel->parms.iph.daddr, 4);

        tunnel->tun_hlen = 0;
        tunnel->hlen = tunnel->tun_hlen + tunnel->encap_hlen;
        return ip_tunnel_init(dev);
}
```

`ip_tunnel_init`比较重要，它为tunnel设备，

* 设置析构函数;
* 分配per-CPU的统计字段;
* 分配初始化per-CPU路由缓存字段（tunnel->dst_cache）;
* 创建`gro_cell`用于*NAPI*的数据接收。
* 初始化ip_tunnel相关字段：
  - tunnel->dev
  - tunnel->net
  - tunnel->param.iph/name

```
int ip_tunnel_init(struct net_device *dev)
{
        ... ...
        /* 注册设备的“析构函数”，当设备释放的时候，释放ip_tunnel相关资源 */
        dev->needs_free_netdev = true;
        dev->priv_destructor = ip_tunnel_dev_free;

        /* 分配 per-CPU的统计字段 */
        dev->tstats = netdev_alloc_pcpu_stats(struct pcpu_sw_netstats);

        /* 分配初始化per-CPU路由缓存字段 */
        err = dst_cache_init(&tunnel->dst_cache, GFP_KERNEL);

        /* 创建`gro_cell`用于*NAPI*的数据接收。 */
        err = gro_cells_init(&tunnel->gro_cells, dev);
        if (err) {
                dst_cache_destroy(&tunnel->dst_cache);
                free_percpu(dev->tstats);
                return err;
        }    

        /* 参数等初始化 */
        tunnel->dev = dev;
        tunnel->net = dev_net(dev);
        strcpy(tunnel->parms.name, dev->name);
        iph->version            = 4;
        iph->ihl                = 5;
        ... ...  
        return 0;
}
```

## 数据接收、传输

### 数据分用：`tunnel4`模块和`xfrm`

接收过程中数据包的分用依赖于`inet_protos[]`表`IPPROTO_IPIP`也不例外，注册`IPPROTO_IPIP`的正是`tunnel4`模块。其注册的处理函数是`tunnel4_rcv`。

```
module_init(tunnel4_init)
    +
    |- inet_add_protocol(&tunnel4_protocol, IPPROTO_IPIP))
```

```
static const struct net_protocol tunnel4_protocol = {
        .handler        =       tunnel4_rcv,
        ... ...
};
```

当IP层判断收到的分组是IP-in-IP分组时，交由`tunnel4_rcv`处理。

另一方面，`tunnel4`模块维护一个为每个基于ipv4的隧道维护了个全局的隧道handler列表 `tunnel4_handlers`，其元素是`xfrm_tunnel`。

```
static struct xfrm_tunnel __rcu *tunnel4_handlers __read_mostly;

struct xfrm_tunnel {
        int (*handler)(struct sk_buff *skb);
        int (*err_handler)(struct sk_buff *skb, u32 info);

        struct xfrm_tunnel __rcu *next;
        int priority;
};
```

`ipip`模块初始化函数`ipip_init`注册了其隧道处理handler `ipip_hander`，其中接收函数是`ipip_rcv`。

```
static struct xfrm_tunnel ipip_handler __read_mostly = {
        .handler        =       ipip_rcv,
        .err_handler    =       ipip_err,
        .priority       =       1,  
};
```

### 数据接收

当IP层处理完成（此处指outer的IP header处理），进行上层协议分用，如果上层协议类型是`IPPROTO_IPIP`，则交由`tunnel4_rcv`处理。

```
ip_rcv
   +
   |- ip_rcv_finish
       +
       |- 路由查询
       |- rt->dst.input 即 ip_local_deliver # 路由查询结果Local IN
            +
            |- ip_local_deliver_finish
                  +
                  |- inet_protos[]表查询（L4分用）
                  |- ipprot->handler 即 tunnel4_rcv
```

`tunnel4_rcv`按照注册时的优先级，依次遍历并调用`xfrm_tunnel`。

```
tunnel4_rcv(skb)
    +
    |- 按照注册时的优先级，依次遍历并调用`xfrm_tunnel.hander`。
       对于ipip而言，Hanlder是ipip_rcv。

        for_each_tunnel_rcu(tunnel4_handlers, handler)
                if (!handler->handler(skb)) # ipip_rcv
                        return 0;
```

`ipip_rcv`是`ipip_tunnel_rcv`的wrapper函数。`ipip_tunnel_rcv`函数，

* 查询per-net的ipip隧道Hash表（创建时被插入`ip_tunnel_net->tunnels`）。查询条件包括：
   - link设备，即下层设备
   - 源IP
   - 目的IP
  查询会按照不同标准进行多轮尝试，具体会在`ip_tunnel_lookup`详细介绍。
* 进行xfrm安全策略检查
* 调用ip_tunnel_rcv()

```
ipip_rcv
    +
    |- ipip_tunnel_rcv
         +
         |- ip_tunnel_lookup根据link设备、源、目的IP在per-net Hash中查找隧道设备
         |- 进行xfrm安全策略检查
         |- ip_tunnel_rcv()
              +
              |- 检查checksum (ipip无)
              |- 记录seqno (ipip无)
              |- 重新设置L3 header
              |- 收包统计
              |- 设置skb->dev为隧道设备的net_device.
              |- 从tunnel设备GRO的napi收取数据。
```

#### 利用`gro_cell`进行数据接收

> 如果还记得非napi设备（旧式netif_rx设备)是如何和napi兼容的，设备将skb放入了一个per-CPU的skb队列，napi每次从队列dequeue数据。gro_cell采用类似的方法。

`gro_cell`并不复杂，一个per-CPU的cells，每个cell包含一个skb队列、一个napi结构。API也只有3个，初始化，数据接收和销毁。

```
struct gro_cell {
        struct sk_buff_head     napi_skbs;
        struct napi_struct      napi;
};

struct gro_cells {
        struct gro_cell __percpu        *cells;
};

int gro_cells_receive(struct gro_cells *gcells, struct sk_buff *skb);
int gro_cells_init(struct gro_cells *gcells, struct net_device *dev);
void gro_cells_destroy(struct gro_cells *gcells);
```

初始化`gro_cells`好理解，

```
gro_cells_init(gcells, dev)
    +
    |- gcells->cells = alloc_percpu(...)
    |- for each cpu's cell
          +
          |- 初始化cell->napi_skbs
          |- netif_napi_add(dev, &cell->napi, gre_cell_poll, weight)
          |- napi_enable(&cell->napi)
```

数据接收通过`gro_cells_receive`，它先获取当前CPU的`gro_cell`，如果队列满（超过设备最大挤压值）则丢弃分组，否则将分组skb插入`cell->napi_skbs`队列。调度设备的napi进行数据接收，此处会触发RX软中断。

```
int gro_cells_receive(struct gro_cells *gcells, struct sk_buff *skb)
{
        /* 获取当前CPU gro_cell */
        cell = this_cpu_ptr(gcells->cells);

        /* 队列满则丢弃 */
        if (skb_queue_len(&cell->napi_skbs) > netdev_max_backlog) {
                atomic_long_inc(&dev->rx_dropped);
                kfree_skb(skb);
                return NET_RX_DROP;
        }   

        /* 插入队列 */
        __skb_queue_tail(&cell->napi_skbs, skb);
        if (skb_queue_len(&cell->napi_skbs) == 1)
                napi_schedule(&cell->napi);
        return NET_RX_SUCCESS;
}
```

RX软中断处理函数`net_rx_action`会按配额调用`napi_pool`，最终调用之前注册的`gro_cell_pool`。它把数据从队列中取出，然后调用`napi_gro_receive`，`napi_skb_finish`，`netif_receive_skb_internel`。GRO的情况比较复杂，不做讨论。

```
net_rx_action
   +
   |- napi_pool
       +
       |- gro_cell_pool
           +
           |- napi_gro_receive
               +
               |- napi_skb_finish
                   +
                   |- netif_receive_skb_internel
```

#### Tunnel设备查询

函数`ip_tunnel_lookup`负责通用ipv4 tunnel的查询，

```
struct ip_tunnel *ip_tunnel_lookup(struct ip_tunnel_net *itn,
                                   int link, __be16 flags,        
                                   __be32 remote, __be32 local,   
                                   __be32 key)
```

第一轮查询按照key进行，key提取自分组，需要和tunnel所设置的key匹配。具体匹配条件如下，

1. 源地址匹配
2. 目的地匹配
3. 设备处于打开（UP）状态
4. key匹配
5. Link设备匹配

可见第一轮查询比较严格，尽量精确的找到匹配的tunnel。如果完全匹配就返回。否则需要进行第二轮。此外，第一轮如果只是link设备不匹配，把改tunnel作为候选，再进行第二轮。

```
        hash = ip_tunnel_hash(key, remote);
        head = &itn->tunnels[hash];

        hlist_for_each_entry_rcu(t, head, hash_node) {
                if (local != t->parms.iph.saddr ||
                    remote != t->parms.iph.daddr ||
                    !(t->dev->flags & IFF_UP))
                        continue;

                if (!ip_tunnel_key_match(&t->parms, flags, key))
                        continue;

                if (t->parms.link == link)
                        return t;
                else
                        cand = t;
        }
```

如果第一轮查询失败，则进行第二轮查找。此时可能有的候选（第一轮link不匹配），也可能没有。

第二轮条件基本和第一轮相同，区别是允许未设置local IP的tunnel进行匹配，对于未设置local IP的tunnel不检查源地址。同样如果匹配则返回。如果不匹配，仅仅因为link不同，且第一轮没有候选，会在第二轮设置候选。

1. 目的地匹配
2. tunnel未设置local IP
3. 设备处于打开（UP）状态
4. key匹配
5. Link设备匹配

```
        hlist_for_each_entry_rcu(t, head, hash_node) {
                if (remote != t->parms.iph.daddr ||
                    t->parms.iph.saddr != 0 ||
                    !(t->dev->flags & IFF_UP))
                        continue;

                if (!ip_tunnel_key_match(&t->parms, flags, key))
                        continue;

                if (t->parms.link == link)
                        return t;
                else if (!cand)
                        cand = t;
        }
```

目前为止的两轮查找，key和remote都是必备的。对于，没有设置remote的tunnel，可能在第三轮中找到。对于未设置remote的tunnel，Hash计算不能考虑分组中的remote地址，需要重新计算。第三轮的匹配条件是，

1. tunnel未指定remote且源地址等于tunnel的local，或者 源地址是多播且等于tunnel的remote
2. 设备处于打开（UP）状态，
3. key匹配
4. Link设备匹配

同样，如果匹配就返回; 如果不匹配，且只是link未匹配，之前又没有合适的备选，就作为备选。

```
        hash = ip_tunnel_hash(key, 0);
        head = &itn->tunnels[hash];

        hlist_for_each_entry_rcu(t, head, hash_node) {
                if ((local != t->parms.iph.saddr || t->parms.iph.daddr != 0) &&
                    (local != t->parms.iph.daddr || !ipv4_is_multicast(local)))
                        continue;

                if (!(t->dev->flags & IFF_UP))
                        continue;

                if (!ip_tunnel_key_match(&t->parms, flags, key))
                        continue;

                if (t->parms.link == link)
                        return t;
                else if (!cand)
                        cand = t;
        }
```

第三轮查找仍未找到的情况下，继续，此时如果，设置了TUNNEL_NO_KEY，则进行第四轮。如果设置了，则直接跳过第四轮，进行最后的collect_md和fb设备选择。

```
        if (flags & TUNNEL_NO_KEY)
                goto skip_key_lookup;
```

注意`ipip`模块设置了`TUNNEL_NO_KEY`，所以不会进行第四论，而是直接进入后面的查找。

第四轮查找的条件是，

1. key匹配
2. tunnel的remote和local都设置，
3. 设备处于打开（UP）状态
4. link设备匹配，

如果匹配则返回；否则如果只是link不匹配，之前未设置后备，则作为后备。

```
        hlist_for_each_entry_rcu(t, head, hash_node) {
                if (t->parms.i_key != key ||
                    t->parms.iph.saddr != 0 ||
                    t->parms.iph.daddr != 0 ||
                    !(t->dev->flags & IFF_UP))
                        continue;

                if (t->parms.link == link)
                        return t;
                else if (!cand)
                        cand = t;
        }
```

最后的查询依次在3个设备间进行，如果有设置就直接返回。它们分别是

1. cand(后备)设备
2. collect_md设备
3. fallback设备

```
skip_key_lookup:
        if (cand)
                return cand;

        t = rcu_dereference(itn->collect_md_tun);
        if (t && t->dev->flags & IFF_UP)
                return t;

        if (itn->fb_tunnel_dev && itn->fb_tunnel_dev->flags & IFF_UP)
                return netdev_priv(itn->fb_tunnel_dev);

        return NULL;
}
```

只有，所以轮次查询失败，且没有后备(只是link不匹配)、没有collect_md，和fallback的情况下，才会查询失败。通常，是不会查询失败的。

### 数据传输

链路层的数据传输从`dev_queue_xmit`开始，抛开队列规则 (`qdisc`)部分，流程简化为如下。最终会调用设备的`ndo_start_xmit`进行传输，对于`ipip`而言就是`ipip_tunnel_xmit`。

```
dev_queue_xmit
    +
    |- 可能的QoS（如队出队等，见`traffic control）
    |- validate_xmit_skb # skb处理，vlan插入tag等处理
    |- dev_hard_start_xmit(skb, dev, txq ...)
        +
        |- netdev_start_xmit
             +
             |- ops.ndo_start_xmit # 即ipip_tunnel_xmit
```

#### 函数`ipip_tunnel_xmit`

```
ipip_tunnel_xmit
    +
    |- ip_tunnel_xmit(skb, dev, tiph, ipproto)
```

隧道分为两种，一种是*point-to-point*隧道，也就是设置了非通配remote的隧道；还有一种是*NBMA* (Non-Broadcast Multi-Access)隧道，没有设置remote。fb设备`tunl0`是*NBMA*的。

NBMA隧道，需要根据inner IP的目的地址，查找next_hop地址（路由器或者直连的host），并设置为destination地址。

后续的传输流程一样。

首先是路由查找，查找的时候隧道分为 “connected”和“non-connected”。NBMA隧道，或者tos并非来自tunnel固定参数，即tos会变化的时候，则为“non-connected”。

* 确定tos地址: 或来自tunnel的设置，或继承inner IP的tos
* 进行路由查找：查找的时候，
  - 如果"connected"，尝试tunnel中缓存的路由`tunnel->dst_cache`
  - 否则，查找路由表
  - 如果“connected”，则将路由缓存到`tunnel->dst_cache`。

路由的外出设备不能是tunnel设备本身。

更新路由的路径MTU，PMTU需要考虑：
* 路由的MTU: `dst_mtu(&rt->mtu)`
* 减去tunnel设备的硬件header, outter IP长度， 隧道本身额外header长度（如GRE头，ipip无）
如果数据超过MTU，返回`-E2BIG`。

设置tos，ttl, IP_DF标记（看tunnel设置、或继承）
最后调用`iptunnel_xmit`进行outter header的封装和数据发送。

#### 函数`iptunnel_xmit`

函数`iptunnel_xmit`完成几件事，

1. 清理skb的字段
2. 设置skb的dst缓存
3. 封装outer IP首部
4. 调用`ip_local_out`发送数据
