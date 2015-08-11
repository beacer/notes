Kernel 之 Bridging
==================

* [网桥虚拟设备](#br-dev)


<a id="br-dev">
网桥虚拟设备
------------

Linux的网桥是虚拟设备，需要将其他设备作为其网桥端口才能工作。将这些设备（实际或虚拟皆可）“绑定”到网桥上，或者说为网桥指定Interface的过程也称为enslave。

<div align=center><img src="images/bridge.png" width="" height="" alt="bridge device"/></div>

例如，使用“brctl”（或者ip link add type bridge）添加一个网桥设备，添加后它并没有任何网桥端口（即Interface）。

    ```bash
    # brctl addbr br0
    # brctl show
    bridge name     bridge id          STP enabled     interfaces
    br0             8000.000000000000           no          
    ```

再为它加上Interface，

    ```
    # brctl addif br0 eth0
    # brctl addif br0 eth1
    # brctl show
    bridge name     bridge id          STP enabled     interfaces
    br0             8000.08002747113f           no           eth0
                                                             eth1
    ```

如果要让br0开始工作，需要将其链路（link）状态打开（UP），可能还需要为它设置个IP地址。这些方面和普通接口是一样的，可以使用ifconfig命令，或者是ip命令。

    ```bash
    # ip link set br0 up
    # ip addr add 10.0.0.1/24 dev br0 broadcast +
    ```

我们要注意的是网桥ID的定义：“优先级”+某个端口的MAC地址。而后来的标准中“优先级”其实只用了4位，后面12位作为系统ID扩展。

    ```bash
       4 bits  12 Bits                48 Bits
      +------+----------+----------------------------------+
      | Prio | SysID Ex |           MAC Address            |
      +------+----------+----------------------------------+
    ```

Prio使用4Bits，那么优先级的增量（每个Prio对应的Extension数目）是4096（2^12）。这样使用一个Prio和MAC标识的网桥表示4096个网桥（实例），它们可以出现在不同的网桥ID空间中。这么做并非随意为之，802.1Q允许的最大VLAN个数就是4096。那么就是说，每个VLAN所在的网桥ID空间，可以使用Prio+MAC标识一个网桥。事实上Cisco私有STP实现对SysID Ex的解释正是VLAN ID。

一旦形成了上图所示的网桥，eth0, eth1所连接的Ethernet LAN被连接成一个逻辑LAN。而Eth0,Eth1不需要分配IP地址（它们不需要在L3上可见），只需要为br0分配一个IP地址。当然不是说不能为eth0, eth1设置IP。 但如果果真的为eth0设置了IP（如下图所示，有两个eth0逻辑实例），那其NIC Driver收到包后到底直接交由IP层进行路由，还是交给br0处理，就需要使用ebtables进行流量划分了。

<div align=center><img src="images/bridge-2.png" width="" height="" alt="bridge device"/></div>
