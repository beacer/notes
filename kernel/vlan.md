# Kernel虚拟设备 之 *VLAN*

> 最近为项目实现了VLAN的支持，期间把Kernel的VLAN模块（`net/802.1q`）读了遍，留笔记于此。 
> 对于VLAN帧各字段意义[WiKi](https://en.wikipedia.org/wiki/IEEE_802.1Q)有比较详细的描述。

和Bridge, Tunnel, Bonding等一样，VLAN在Kernel中使用虚拟设备实现，即`net_device`的一种扩展。
依据《ULNI》对网络栈的描述套路（个人觉得非常有效），本文也分“数据结构”，“初始化、创建、删除”，“数据接收（RX）”,“数据发送（TX）”几个部分进行。

### VLAN数据结构

VLAN作为一种虚拟设备(`vlan-device`)，需要构建在“真实(real-device)”设备之上，这里的“真实”设备即可以是物理网卡（例如,`eth0`）,
也可以是其他虚拟设备，如`bonding`设备。同时、一个real-device，可以”挂“多个VLAN设备。 例如下图所描述的情况，

[real-device](images/vlan-real-devices.png)

* `eth0`是物理网卡，`eth0.100`和`eth0.200`分别模拟了两个VLAN的`access port`。
* 带VLAN tag的数据被`eth0`接收后，如果tag是100,则交由`eth0.100`再进入协议栈。
* 数据发送的时候，从`eth0.100`出去的数据会由`eth.100`打上100 tag。

当然，tag的strip和insert过程可以通过HW来*offloading*以减少CUP的负担。

#### VLAN设备： `vlan_dev_priv`

vlan-device也是一种`net_device`,所以只需要用通常的方式对`net_device`扩展，并实现`net_dev_ops`，`ethtool_ops`等。VLAN相关的信息保存在`vlan_dev_priv中`，分配VLAN设备`net_device`的时候一起分配。各字段的意义见注释。

``` C
struct vlan_dev_priv {
        /*
         * 下面几个字段用于将VLAN Header中的优先级映射为Kernel所使用的优先级，即skb->priority。
         * 因为不同的协议如IPv4/IPv6和VLAN都定义了自己的“优先级”，同时没有统一的处理方式。
         * Kernel的做法是全部映射。只不过这种映射可以配置。
         */
        unsigned int                            nr_ingress_mappings;
        u32                                     ingress_priority_map[8];                                                                                                                                                                
        unsigned int                            nr_egress_mappings;
        struct vlan_priority_tci_mapping        *egress_priority_map[16];

        __be16                                  vlan_proto;  // VLAN协议ETH_P_802.1Q等
        u16                                     vlan_id;     // VLAN ID
        u16                                     flags;

        struct net_device                       *real_dev;   // VLAN设备依附的“真实设备”
        unsigned char                           real_dev_addr[ETH_ALEN]; // 真实设备Link地址

        struct proc_dir_entry                   *dent;       // /proc文件目录节点
        struct vlan_pcpu_stats __percpu         *vlan_pcpu_stats; //采用per-CPU的统计（避免并发竞争）

        unsigned int                            nest_level; // 防止虚拟设备嵌套过多
};
```

使用netlink创建VLAN设备，在alloc_netdev_mqs的时候`priv_size`即`vlan_dev_priv`的大小。

> `alloc_netdev_mqs`(及其wrapper函数)和`register_netdevice`是Kernel创建、注册网络设备、包括虚拟设备所使用的两个API。

#### Real和VLAN设备关系

###### Real设备维护的VLAN信息：`vlan_info`

那么就需要维护”real-device"和"vlan-device"的关系，刚刚提到、不同类型的设备（物理网卡、bonding）都可作为real-device来挂载vlan-device。
所以，在每个设备要用到的`net_device`中内嵌了VLAN相关信息，即`vlan_info`。

``` C
struct vlan_info {                                                                                                                                                                                                                      
        struct net_device       *real_dev; /* The ethernet(like) device
                                            * the vlan is attached to.
                                            */
        struct vlan_group       grp;       // 可用<proto, vlan-id>查找对应VLAN设备的net_device结构。
        struct list_head        vid_list;  // 存放vlan_vid_info列表，该列表用于其他虚拟设备如bridge/bonding实现VLAN。
        unsigned int            nr_vids;
        struct rcu_head         rcu;
};      
```

> vid_list/nr_vids所管理的vlan_vid_info主要用于其他虚拟设备如bridge/bonding等实现VLAN支持。

“真实”设备的`net_device`中的`vlan_info`字段，用来管理attach到它的所有VLAN（及VLAN虚拟设备）。

``` C
struct net_device {
    ...
    struct vlan_info __rcu  *vlan_info;
    ...
}
```
###### VLAN设备Hash表：`vlan_group`

再看看`vlan_group`的定义，在不考虑GVRP/MVRP协议的情况下，可以认为，它的作用就是从真实设备上，快速查找VLAN设备对应的`net_device`。

``` C
struct vlan_group {     
        unsigned int            nr_vlan_devs;
        struct hlist_node       hlist;  /* linked list */
        struct net_device **vlan_devices_arrays[VLAN_PROTO_NUM]
                                               [VLAN_GROUP_ARRAY_SPLIT_PARTS];
};

```

如何理解`vlan_devices_arrays`这个多维数组呢？ 其实并不复杂，可以简单认为它是一个多级Hash表。 我们知道802.1q VLAN的VLAN ID字段只有12个bit，也就是说最多支持4096个VLAN。如果以VLAN ID作为Hash Key，那么每种VLAN协议（目前其实只支持802.1q和802.1ad QinQ两种）各需要一个4096大小的`net_device`数组作为*buckets*。只不过Kernel先把4096分成8组，做了个二级Hash，然后为每种VLAN协议分配一个Hash表。 这样一来`vlan_devices_arrays`的维度就上去了。

多维Hash表结构如图所示，

[vlan_devices_arrays](images/vlan_devices_array.png)

###### XXX：`vlan_vid_info`

#### VLAN及Real设备关系Overview

综上所述，一个Real设备和挂载（attach）的VLAN虚拟设备的数据结构关系如下，

[real-vlan-devices](images/vlan-real-devices.png)
