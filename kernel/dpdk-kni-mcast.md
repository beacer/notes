最近尝试了`dpdk`的`kni`驱动，使得非高速转发路径的数据能利用Kernel协议栈，这确实方便了非面向性能的应用。不过在`kni`虚拟接口上运行多播应用的时候的时候发现，根本不工作？

#### 谈谈NIC的多播接收

接收多播对于一个网卡来说是必须的，然而初始情况下Ethernet网卡只接收两类数据帧：

* 目的MAC是网卡物理地址的帧；
* 以及以太网广播帧。

除非，

* 打开了混杂模式，那么所有见到的帧都会接收（例如用`tcpdump`时）；
* 或者，明确指示它接收某个以太网多播（或某组多播或其他Unicast）。

这也是为什么多播的效率好于广播的原因，本机不关心的多播在硬件层就ignore，不必麻烦协议栈。由于应用程序或协议（`IGMP/MLD`、`IPv6 SLAAC/ND`，`OSPF`等）本身的需求，要加入一个多播组的时候，例如`224.0.0.1`，将其转化为以太网多播地址（`01:00:5e:00:00:01`）然后告知网卡要接受这个多播帧。

> 网卡一般会有一个表维护要加入哪些多播，因为资源的限制，不可能做到为每个以太网多播维护一个很大的表，会采用哈希的形式多对一映射，或者限制多播表的大小。

#### 怎么`kni`设备收不到多播？

回到`kni`的情况，网卡本身显然是支持多播的，以防万一，查了下DPDK的NIC驱动也是支持的；那么只能怀疑`kni`虚拟设备的驱动了，果然，`kni`驱动“[实现了](http://dpdk.org/dev/patchwork/patch/5074/)”设置多播的函数`ndo_set_rx_mode()`，只不过是个空函数而已。

> `ndo_xxx`系列函数（包括`ndo_set_rx_mode`）是Kernel `net_device`层和具体设备驱动的一个Callback（虚函数），实现`ndo_set_rx_mode`目的是控制网卡的接收模式。换句话说就是设置NIC地址过滤表，除了自己地址和广播外，额外接收哪些硬件地址，比如，
> * `Secondary Unicast`（自己硬件地址之外的地址）
> * `Promiscuous Mode` （全都给收上来吧）
> * `Multicast Mode` （告诉我哪些多播要收，还是Multicast全收）

kni是虚拟设备，转交PMD设备的包，不关心是不是多播，本来是“没必要”实现这个逻辑的。不过既然KNI是建立在实际PMD设备之上的虚拟Linux设备，Linux只能操作到`kni`驱动，需要把DPDK PMD控制下的物理网卡的多播打开才行。

Double check下最新的Release，没有填坑的迹象。

#### 那么只能自己动手了

网卡硬件是OK的，PDM驱动也是支持多播的（虽然只是提供了个API让用户自己设置）。那么目标很明确，通过`kni`驱动把需要加入的MAC多播告知实际控制网卡的`PMD`驱动即可。分成几个步骤，

* 从Kernel取出MAC多播列表
* 通过某种方式从kni驱动通知到PMD驱动
* 通过PMD驱动提供的API将多播列表设置到实际网卡中。

###### Kernel把多播列表存在哪？

去设置多播的地方找找（除非你知道它们就在`net_device->mc`里面），

```C
ip_mc_join_group()
    |- ip_mc_find_dev
    |- ip_mc_inc_group
          |- IPv4多播被加入了dev.in_device.mc_hash/.mc_list
          |- igmp_group_added
                |- ip_mc_filter_add
                     |- dev_mc_add
                           |- __hw_addr_add_ex
                           |     分配netdev_hw_addr加入dev->mc列表
                           |- __dev_set_rx_mode
                                |- dev.ops.ndo_set_rx_mode
                                   设置具体驱动的RX Mode
```

硬件多播被作为`netdev_hw_addr`保存在了`dev->mc`列表之中，并通过之前提到的`ndo_set_rx_mode`传递给具体的驱动。“Helper”宏`netdev_for_each_mc_addr`可以用来遍历该链表。

###### `kni`怎么通知`PMD`？

DPDK的PMD是用户态驱动，`kni`驱动是内核态的，他们之间的通过内存映射实现的FIFO来通信，包括数据包和控制消息。具体可以参考[kni的文档](http://dpdk.org/doc/guides/prog_guide/kernel_nic_interface.html)和代码。

`kni`和`PMD`控制通道是`sync FIFO`，对应了`kni_dev.sync_kva/va`。具体说来是用了上面的那段内存来传输`rte_kni_request`结构的，Kernel可以使用API `kni_net_process_request`发送请求到用户态；用户态由`rte_kni_handle_request`处理请求，返回处理结果。

但是Requset结构目前只支持通知 MTU改变和link状态改变；为了传递多播列表需要自定义一个消息类型以及保存多播的结构放到Requset结构。

考虑到毕竟MTU和Link UP/Down这种消息相对一个多播列表而言数据量很小，担心一个多播列表保存Request超过`kni_dev.sync_kva/va`的大小（毕竟表面上看不出它到底多大呀！）。追查了分配这个buffer的地方：`rte_kni_init()/memzone "kni_sync_%d"`，差不多有64K大，放心了。

> 利用`sync FIFO`是不是好?不好说，只是使用别的kernel/user-space通信方式，也有点繁琐。

定义数据结构，从`net_dev->mc`提取MAC多播，用Request传递给PMD，记得处理Response。

###### PMD驱动多播列表设置到实际网卡中

这个简单，在`rte_kni_handle_request`接收到Kernel的消息直接调用API即可。

最后还有一点要注意，`ndo_set_rx_mode`是在原子（spinlock）上下文，而`kni_net_process_request`调用了`mutex`，原子上下文不能使用`mutex`这种会引发上下文切换的函数。所以在`ndo_set_rx_mode`里将多播列表暂时保存起来然后利用`kni`的`kthread`线程去检查并发送请求，后者访问`dev->mc`数据的时候记得上锁（`netdev_addr_lock_bh`）并一次性取出。

多播在`kni`虚拟接口上正常接收，可以，这很DIY。