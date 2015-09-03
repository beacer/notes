Linux Kernel之*帧的接收*
=======================

> 本文的目的是通过重读《ULNI》和目前的Kernel实现（4.x）重温网络部分帧的接收流程。虽然大部分内容都在《ULNI》中都有，毕竟2.6.12版本的Kernel已经有些年头了，书中有些地方和现在的实现会有些出入（例如`napi_struct{}`的引入，`softnet_data{}`的改变等）。借次机会再次复习复习总不会错 :-)

* [Kernel和NIC的交互](#irq-poll)
* [中断、下半部和软中断](#IRQ-BH)
  o [硬件中断](#hardware-IRQ)
  o [下半部](#bottom-Half)
  o [网络代码和SoftIRQ](#net-softirq)
* [数据结构](#data-struct)
  o [per-CPU结构：softnet_data](#softnet_data)
  o [New API结构：napi_struct](#napi_struct)

<a id="irq-poll"/>
### Kernel和NIC的交互

帧接收的过程中Kernel和网络设备的交互方式分为*中断（interrupting)*和*轮询(polling)*。

* **轮询方式**

采用Polling的方式，可以定期检查设备寄存器，如果指示有数据，就连续将数据读出。连续读取多个帧的目的在于提高吞吐量（一次轮询只读取一个帧也太低效了）。不过polling会浪费CPU周期，Polling的另一个问题是会造成一定的延迟（从NIC收到数据开始到下次轮询之间会有间隔，最差情况会造成一个轮询间隔（interval）的延迟）。

* **中断方式**

如果要保证低延迟（laytency）则可以考虑中断的方式，保证一有数据就立即通知CPU，然后中断处理函数读取数据帧，存放到输入队列，再通知内核。很显然如果每一帧都触发一个中断，会使得在高负载的情况下CPU忙于中断的处理，无法处理其他任务。考虑到硬件中断的优先级非常高，还会造成输入队列被排满没有机会的情况。

* **中断中处理多个帧**（旧式网络接收）

一种改进的方案是在`一个中断中处理多个帧`。也就是说，当收到一个中断后检查寄存器，如果有数据则持续收取直到收完所有数据，或者达到一个给定的数目（配额）后停止，显然不能因为有数据中断处理就一直不返回。一些旧式的驱动（例如使用`netif_rx()`）的驱动采用了这个模式。

>一个使用这种（旧式或netif_rx()式）方案的例子是3com的3c59x.c驱动，可以参考《ULNI》或者直接查看源码driver/net/ethernet/3com/3c59x.c。

这里只需要理解此类设备如何在一次中断中处理多个帧，先不深入到`vortex_rx()`和`netif_rx()`的实现细节和输入队列等其他话题。

* **中断与轮询结合（NAPI）**

不过更好的方式是中断和轮询的组合（称为New-API，napi方式）。NIC检测到有数据接收通过中断通知Kernel，中断处理函数并不分配skb和读取数据帧，而是将设备（早期是`net_devivce{}`本身现在是相关的napi_struct{}）排入队列，然后触发`下半部（bottom-half，BH)`（例如软中断），由下半部异步的方式负责轮询设备一次读取完所有数据或到达配额上限。

<a id="IRQ-BH"/>
### 中断、下半部和软中断

<a id="hardware-IRQ"/>
#### 硬件中断

我们知道中断是为了及时响应外部事件，而一旦处于*中断上下文（interrupt context）*中，

* 该CPU的其他中断会被关闭（硬中断hardware IRQ不能嵌套）
* CPU也不能执行其他的进程

总之，（硬）中断是不能被抢占的（nonpreemptible），也是不可重入的（non-reentrant），这样就能降低竞争条件的发生；这同时也意味着，中断处理函数的工作应该尽快完成。

<a id="bottom-Half"/>
#### 下半部（Bottom-Half，BH）

于是中断处理过程被分为“上半部top-half”和“下半部bottom-half”，上半部在中断上下文中执行，下半部则“以异步的方式完成特定的请求”。这样就能把硬中断关闭的时间大幅减少，避免因为上半部执行时间过长而丢失任何数据和信息。

下半部的解决方案有，

* 软中断（SoftIRQ）
* 微任务（tasklet）
* 内核线程（kthread）
* 工作队列（work-queue)

前两者不依赖进程环境，可用于执行“不可休眠的任务”；后两个依赖于进程环境，执行期间可以休眠。

软中断和微任务都是通过softirq框架实现，它们在并发和上锁上有一些差异，而且软中断的数目（类型）是设计编码时决定的（如`HI_SOFTIRQ`，`NET_RX_SOFTIRQ`等）；微任务基于软中断实现，但可以动态添加不同的task。软中断（包括tasklet）所执行的机会有很多，例如，

1. 硬中断处理函数(do_IRQ)的末尾，重新打开CPU的中断后，会执行Pending的软中断，这也意味着软中断是可以被硬中断所抢占的。
2. 从中断和异常事件，系统调用返回
3. 在CPU上重新打开软中断
4. 内核线程`run_ksoftirqd()`。如果其他地方都没机会，至少会在这执行一把（也算是兜底？）。命令`ps -e | grep softirq`可以找到per-CPU的softirq线程。

>考虑到主题是帧的接收以及篇幅限制，软中断和其他下半部的话题可参考《ULNI》《LDD3》等。

<a id="net-softirq"/>
### 网络代码和SoftIRQ

网络子系统的初始化函数`net_dev_init()`会“注册”两个网络相关的软中断，一个用于发送处理，一个用于接收。

```c++
static int __init net_dev_init(void)
{
    ... ...
    open_softirq(NET_TX_SOFTIRQ, net_tx_action);
    open_softirq(NET_RX_SOFTIRQ, net_rx_action);
    ... ...
}
```

> `net_dev_init()`在系统初始化（boot）期间被调用，系统初始化和宏`subsys_initcall`相关的议题见《ULNI》。

<a id="data-struct"/>
### 数据结构

<a id="softnet_data"/>
#### per-CPU结构： softnet_data{}

网络数据接收中一个非常重要的数据结构是per-CPU变量`softnet_data{}`，简称`sd`。使用Per-CPU变量可以避免上锁提高并发率和吞吐量。新版的`sd`比ULNI中的定义（2.6.12版本内核）复杂了不少，

```c++
// net/core/dev.c
DEFINE_PER_CPU_ALIGNED(struct softnet_data, softnet_data);

// include/linux/netdevice.h
struct softnet_data {
    struct list_head    poll_list;     // 需要ingress轮询的dev列表，“设备”用相关的napi_struct表示
    struct sk_buff_head process_queue; // 用于Non-NAPI设备，旧式积压队列，函数process_backlog()把
                                       // skb从sd->input_pkt_queue装异到此处，再进行收取

    /* stats */
    unsigned int        processed;
    unsigned int        time_squeeze;
    unsigned int        cpu_collision;
    unsigned int        received_rps;
    // ... RPS 和 Net FLOW ...
    struct Qdisc        *output_queue;     // TX队列规则
    struct Qdisc        **output_queue_tailp;
    struct sk_buff      *completion_queue; // 完成TX可以释放的skb
    
#ifdef CONFIG_RPS
    // ... ...
#endif
    unsigned int        dropped;         // 丢包统计，例如队列已满
    struct sk_buff_head input_pkt_queue; // 用于Non-NAPI设备，驱动分配skb，读取数据，把skb放入该队列
    struct napi_struct  backlog;         // 用于Non-NAPI设备，2.6.12时是 struct net_device backlog_dev;
    
};  
```

> 其中的某些字段和`Receive Packet Steering(RPS)`，RPS通过将接收分组分发到不同的CPU队列进行并发除了而提高PPS，http://lwn.net/Articles/328339/。

说明一下旧式的设备驱动指调用`netif_rx()`的设备或者`non-NAPI`设备。先来看看non-napi设备相关的字段，

* `sd.input_pkt_queue`

该队列仅用于non-napi设备。non-API设备会在驱动程序中（往往是中断处理函数）分配skb，从寄存器读取数据到skb，然后调用`netif_rx()`把`skb`排入`sd->input_pkt_queue`。注意，所有non-napi设备共享输入队列，即per-CPU的`sd->input_pkt_queue`。

* `sd.backlog`

用于non-napi设备，为了和NAPI接收架构兼容，所有的non-napi设备使用一个伪`napi_struct`，即这里的`sd.backlog`。NAPI设备把自己的napi_struct{}放入`sd->poll_list`，而所有的non-napi设备在`netif_rx`的时候把`sd->backlog`放入`sd->poll_list`。稍后看到napi架构的下半部（软中断处理）函数`net_rx_action`依次取出排列到`sd->poll_list`的`napi_struct{}`结构，并使用`napi_poll`轮询配额的数据（或不足配额的所有数据）。

`sd.backlog`是一个内嵌的结构，所有non-napi设备共享它。而它的`sd.backlog.poll`(即`napi.poll`)被初始化为`process_backlog()`，通过这种“共享的伪`napi_struct{}`”方式使得non-napi设备很好的融入了NAPI框架，使得*non-napi*和*NAPI设备*对下半部（`net_rx_action()`）是透明的。不得不说，这是一个值得学习的精巧设计，让我们知道如何向前兼容旧的机制，如何让下层的变化对上层透明。

* `sd.process_queue`

刚才已经提到，non-napi设备把skb放入`sd.input_pkt_queue`，然后在下半部处理中使用`sd.backlog.poll`即`proces_backlog()`函数来处理skb。改函数模拟了NAPI驱动的poll函数行为。作为对比，NAPI设备的poll从设备私有队列读取。Non-NAPI的process_backlog()函数则从non-napi设备共享的输入队列input_pkt_queue中读取配额的数据。然后放入sd.process_queue。

然后是一些通用的字段，

* `sd.poll_list`

NAPI设备在中断处理函数中将其`napi_struct{}`放入该队列，调度软中断soft-IRQ，稍后有下半部`net_rx_action()`负责轮询该设备。

<a id="napi_struct"/>
#### New API结构: napi_struct{}

> 原本可以忽略本段注释，不过考虑到ULNI使用了旧的内嵌结构`sd.backlog_dev`为了避免阅读时的误解，需要在此说明一下，该字段在2.6.12中的定义是`struct net_device backlog_dev;`。而随着`napi_struct{}`的引入而修改为此。就是说`sd.backlog.poll`原来是`sd.backlog_dev.poll`。注意，NAPI很早（在2.6.12之前）就有了，但`napi_struct{}`是新的结构。另外，net_device中原来的`dev->poll_list`, `dev->quota`, `dev->weight`也剥离到了`napi_struct{}`中集中处理。


```C++
// include/linux/netdevice.h
/*
 * Structure for NAPI scheduling similar to tasklet but with weighting
 */    
struct napi_struct {
    /* The poll_list must only be managed by the entity which
     * changes the state of the NAPI_STATE_SCHED bit.  This means
     * whoever atomically sets that bit can add this napi_struct
     * to the per-cpu poll_list, and whoever clears that bit
     * can remove from the list right before clearing the bit.
     */
    struct list_head    poll_list; 

    unsigned long       state;
    int         weight;       
    unsigned int        gro_count; 
    int         (*poll)(struct napi_struct *, int);
#ifdef CONFIG_NETPOLL         
    spinlock_t      poll_lock;
    int         poll_owner;   
#endif 
    struct net_device   *dev; 
    struct sk_buff      *gro_list; 
    struct sk_buff      *skb; 
    struct hrtimer      timer;
    struct list_head    dev_list;  
    struct hlist_node   napi_hash_node;
    unsigned int        napi_id;   
};

enum { 
    NAPI_STATE_SCHED,   /* Poll is scheduled */
    NAPI_STATE_DISABLE, /* Disable pending */
    NAPI_STATE_NPSVC,   /* Netpoll - don't dequeue from poll_list */
    NAPI_STATE_HASHED,  /* In NAPI hash */
};

```



> Netpoll和kgdb
