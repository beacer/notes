# 通过QEMU+GDB调试Linux内核

## 环境

宿主机（调试机）环境:

* Ubuntu 16.04.3 LTS
* Intel(R) Core(TM) i5-4590 CPU @ 3.30GHz
* `4.4.0-112-generic x86_64`

qemu虚拟机（被调试）：

* kernel: 主线版本 v4.9

## 内核编译

下载并配置并编译内核,

```shell
$ cd linux
$ make defconfig
$ make menuconfig
```

建议的配置如下，注意`[*]`或`<*>`代表选择，`[ ]`代表关闭．

```shell
Kernel hacking  --->
  [*] KGDB: kernel debugger  --->
    <*> KGDB: use kgdb over the serial console (NEW)
    [*] KGDB: Allow debugging with traps in notifiers
  Compile-time checks and compiler options  --->
    [*] Compile the kernel with debug info
    [*]   Provide GDB scripts for kernel debugging

Kernel hacking  --->
  Memory Debugging  --->
    [ ] Testcase for the marking rodata read-only

Processor type and features  --->
  [ ] Randomize the address of the kernel image (KASLR)
```

编译内核，用虚拟机跑调试内核，不需要install．可以使用`-j`选项设置make的cpu数提高编译速度．

> 新版本Kernel需要关闭KASLR，否则无法设置断点调试.

```shell
make # or make -j#cpu
make modules
```

看看是否编译成功，

```
$ ls vmlinux ./arch/x86/boot/bzImage
./arch/x86/boot/bzImage  vmlinux
```

## 制作根文件系统

下面，我们使用[busybox](https://www.busybox.net/)制作一个简单的根文件系统．

### 编译busybox

首先需要编译busybox，下载个稳定版本的busybox并编译，

```shell
$ wget http://busybox.net/downloads/busybox-1.27.2.tar.bz2
$ tar vjxf busybox-1.27.2.tar.bz2
$ cd busybox-1.27.2/
$ make menuconfig
```

相关的配置如下，同样`[*]`的代表打开，`[ ]`代表关闭.可以适当增加删减一些配置.

```shell
Busybox Settings  --->
  [*] Don't use /usr (NEW)
  --- Build Options
  [*] Build BusyBox as a static binary (no shared libs)
  --- Installation Options ("make install" behavior)
  (./_install) BusyBox installation prefix (NEW)

Miscellaneous Utilities  --->
  [ ] flash_eraseall
  [ ] flash_lock
  [ ] flash_unlock
  [ ] flashcp
```

编译busybox，会安装到`./_install`目录下

```shell
$ make # or make -j#cpu
$ make install

$ ls _install/
bin  linuxrc  sbin
```

### 制作根文件系统

根文件系统镜像大小256MiB，格式化为ext3文件系统．

```shell
# in working-dir
$ dd if=/dev/zero of=rootfs.img bs=1M count=256
$ mkfs.ext3 rootfs.img
```

将文件系统mount到本地路径，复制busybox相关的文件，并生成必要的文件和目录

```
# in working-dir
$ mkdir /tmp/rootfs-busybox
$ sudo mount -o loop $PWD/rootfs.img /tmp/rootfs-busybox

$ sudo cp -a busybox-1.27.2/_install/* /tmp/rootfs-busybox/
$ pushd /tmp/rootfs-busybox/
$ sudo mkdir dev sys proc etc lib mnt
$ popd
```

还需要制作系统初始化文件,

```shell
# in working-dir
$ sudo cp -a busybox-1.27.2/examples/bootfloppy/etc/* /tmp/rootfs-busybox/etc/
```

Busybox所使用的`rcS`，内容可以写成

```shell
$ cat /tmp/rootfs-busybox/etc/init.d/rcS
#! /bin/sh

/bin/mount -a
/bin/mount -t sysfs sysfs /sys
/bin/mount -t tmpfs tmpfs /dev
/sbin/mdev -s
```

接下来就不需要挂载的虚拟磁盘了

```
$ sudo umount /tmp/rootfs-busybox
```

## 安装qemu

安装并运行编译的调试内核

```shell
$ sudo apt-get install qemu # 或 Fedora/CentOS: yum install qemu
$ sudo qemu-system-x86_64 -kernel linux/arch/x86/boot/bzImage -append 'root=/dev/sda' -boot c -hda rootfs.img -k en-us
```

> tips: 
> * 使用`ctrl+alt+2`切换qemu控制台,使用`ctrl+alt+1`切换回调试kernel
> * 使用`ctrl+alt`将被qemu VM捕获的鼠标焦点切换回host


## 通过qemu和gdb调试kernel

使用qemu运行Kernel，然后切换到qenu控制台，输入

```shell
(qemu) gdbserver tcp::1234
Waiting for gdb connection on device 'tcp::1234'
```

打开另一个终端，使用调试工具(gdb/ddd/cgdb)调试vmlinux文件，并连接gdb server．

```shell
$ cd linux
$ gdb vmlinux
... ...
Reading symbols from vmlinux...done.
(gdb) target remote 127.0.0.1:1234
Remote debugging using 127.0.0.1:1234
default_idle () at arch/x86/kernel/process.c:355
(gdb) b ip_rcv
Breakpoint 1 at 0xffffffff817dab80: file net/ipv4/ip_input.c, line 413.
(gdb) c
Continuing.

Breakpoint 1, ip_rcv (skb=0xffff880007175000, dev=0xffff88000726f000, pt=0xffffffff8233ec20 <ip_packet_type>, orig_
dev=0xffff88000726f000) at net/ipv4/ip_input.c:413
(gdb)
```

> 要触发上述断点，可以切换回qemu的调试kernel．为`lo`接口添加`127.0.0.1/8`地址并使能`lo`接口，然后`ping`环回地址．

## 调试网络

### 网络设备设置

我们使用qemu的**bridge模式**设置虚机网络，该模式需要在宿主机配置网桥，并使用该网桥配置地址和默认路由．具体见host的`/etc/qemu-ifup`文件．
然后使用`-net`参数启动qemu虚机．

> 如果通过宿主机eth0远程登录，该操作可能导致网络登录中断．

```shell
host $ sudo brctl addbr br0
host $ sudo brctl addif br0 eth0
host $ sudo ifconfig eth0 0
host $ sudo dhclient br0
host $ sudo qemu-system-x86_64 -kernel linux/arch/x86/boot/bzImage \
       -append 'root=/dev/sda' -boot c -hda rootfs.img -k en-us \
       -net nic -net tap,ifname=tap0
```

`tap0`是在宿主机中对应的接口名．我们可以在宿主机中看到网桥及其两个端口．tap设备的另一端是VM的eth0．

```shell
host $ brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.1866da0573d1       no              eth0
                                                        tap0
```

为简单测试VM和宿主机的网络联通性，我们在宿主机的`br0`和虚机的`eth0`分别配置两个私有地址来测试．

```shell
# qemu VM（调试Kernel）
/ # ip addr add 192.168.0.2/24 dev eth0 
/ # ip link set eth0 up
```

```shell
# 宿主机
host $ sudo ip addr add 192.168.0.1/24 dev br0
host $ ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.328 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.282 ms
... ...
```

这样可以基本满足调试Kernel的网络协议栈的环境了.

> 或者在VM中运行`udhcpc eth0`让VM获取和host相同网络的IP，不过这需要DHCP Server的支持．

## 使用nfs挂载rootfs

调试内核模块（或其他用户态程序的时候），挂载静态的ext3文件系统并不方便．为此我们可以采用nfs的形式挂载qemu kernel的rootfs，这样就能方便的在host中修改，编译内核模块，并在qemu kernel中配合gdb进行调试．根文件系统制作方法和之前相同，

```
# host working dir
host $ mkdir rootfs.nfs
host $ cp -a busybox-1.27.2/_install/* rootfs.nfs/
host $ pushd rootfs.nfs/
host $ mkdir dev sys proc etc lib mnt
host $ popd
host $ cp -a busybox-1.27.2/examples/bootfloppy/etc/* rootfs.nfs/etc/
host $ cat rootfs.nfs/etc/init.d/rcS
#! /bin/sh

/bin/mount -a
/bin/mount -t sysfs sysfs /sys
/bin/mount -t tmpfs tmpfs /dev
/sbin/mdev -s

host $ chmod -R 777 rootfs.nfs/
```

配置host的nfs服务并启动,

```
host $ apt-get install nfs-kernel-server
host $ cat /etc/exports
/path/to/working/dir/rootfs.nfs *(rw,insecure,sync,no_root_squash)

host $ service nfs-kernel-server restart
```

使用nfs挂载qemu Kernel的根文件系统

```
host $ sudo qemu-system-x86_64 -kernel linux/arch/x86/boot/bzImage \
        -append 'root=/dev/nfs nfsroot="192.168.1.1:/path/to/working/dir/rootfs.nfs/" rw ip=192.168.1.2' \
	-boot c -k en-us -net nic -net tap,ifname=tap0
```

其中`nfsroot`为host的IP及要挂载的根文件系统在host中的路径，`ip`参数填写qemu Kernel将使用的IP地址．

### 调试内核模块

> Note: 编译内核模块的时候，源码树和虚机Kernel编译需要是同一份．不然会出现模块版本不匹配无法运行的情况.

## 参考

* https://www.jianshu.com/p/02557f0d29dc
* http://blog.csdn.net/ganggexiongqi/article/details/5877756
* https://www.binss.me/blog/how-to-debug-linux-kernel/
* https://www.jianshu.com/p/110b60c14a8b
* https://help.ubuntu.com/community/SettingUpNFSHowTo
