# Kernel虚拟设备 之 *VLAN*

> 最近为项目实现了VLAN的支持，期间把Kernel的VLAN模块（`net/802.1q`）读了遍，留笔记于此。 
> 对于VLAN帧各字段意义[WiKi](https://en.wikipedia.org/wiki/IEEE_802.1Q)有比较详细的描述。

和Bridge, Tunnel, Bonding等一样，VLAN在Kernel中使用虚拟设备实现，即`net_device`的一种扩展。
参考《ULNI》对网络栈的描述套路（个人觉得非常有效），本文也分“数据结构”，“初始化”，“数据接收（RX）”,“数据发送（TX）”几个部分进行。

### VLAN数据结构

VLAN作为一种虚拟设备(`vlan-device`)，需要构建在“真实(real-device)”设备之上，不过这里的“真实”设备可以是物理网卡（例如,`eth0`）,
也可以是其他虚拟设备，如`bonding`设备。同时、一个真实设备，可以”挂“多个VLAN设备。 例如下图所描述的情况，

[real-device](!real-device.png)

* `eth0`是物理网卡，`eth0.100`和`eth0.200`分别模拟了两个VLAN的`access port`。
* 带VLAN tag的数据被`eth0`接收后，如果tag是100,则交由`eth0.100`再进入协议栈。
* 数据发送的时候，从`eth0.100`出去的数据会由`eth.100`打上100 tag。

当然，tag的strip和insert过程可以通过HW来*offloading*以减少CUP的负担。

#### vlan设备： `vlan_dev_priv`


#### Real和VLAN设备关系: `vlan_info`

那么就需要维护”real-device"和"vlan-device"的关系，刚刚提到、不同类型的设备（物理网卡、bonding）都可作为real-device来挂载vlan-device。
所以，在每个设备要用到的`net_device`中内嵌了VLAN相关信息，即`vlan_info`。
