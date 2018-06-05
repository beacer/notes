QUIC学习笔记
===========

# QUIC概览

> Quick UDP Internet Connections

QUIC通过*UDP*在两端支持一组*复用的连接*, 提供等同于*TLS/SSL的安全*，同时减少连接和传输*延时*，以及通过双向带宽评估*避免拥塞*。

* 基于UDP（兼容legacy设备）
* 连接复用 （Multiplexed Connections,彻底解决TCP/HTTP的头端阻塞问题，支持roaming）
* 安全(SSL/TLS)
* 减少连接和传输Latency (0-RTT)
* 流量和拥塞控制（双向带宽评估, 连接和流级别的控制，丰富的语义)
* 应用层实现便于更快的迭代。

> wikipedia: QUIC supports a set of multiplexed connections between two endpoints over User Datagram Protocol (UDP), and was designed to provide security protection equivalent to TLS/SSL, along with reduced connection and transport latency, and bandwidth estimation in each direction to avoid congestion.

时间线：

| Year | Event |
|------|-------|
| 2012 | designed by Jim Roskind at Google
| 2013 | announced publicly in 2013
| 2015 | an Internet Draft submitted to the IETF.
| 2016 | QUIC working group was established.
| 2018 | supported by approximately 0.8 percent of web servers.

### 目标与特性

* C/S建立端对端QUIC连接(connection)，连接由复用的流(stream)组成, 每个流相当于独立的TCP连接，
* 降低延迟(0-RTT)

  0-RTT和TLS的session复用，无状态恢复(ticket机制)很像。优化包括了不同的阶段：握手（handshake）阶段,加密设置(encryption setup), 初始化数据请求(initial data requests)

* 类似SPDY(HTTP/2.0)的流复用

  设计QUIC的主要动机之一是解决TCP头端阻塞.复用流还能支持移动roaming(IP地址变化).

* 更好的处理丢包

  加密块(cryptographic block)边界与分组(packet)边界对齐，减少丢包的影响。
TCP使用拥塞窗口(congestion window)避免拥塞，影响到多个复用的连接（单个init-cwnd）。QUIC使用更现代的技术，例如分组测距(packet pacing)不间断带宽预测；主动推测(proactive speculative)重传（重要的分组重复发送）。

> 用户态开发周期短，有效的优化可以方向移植到TCP和TLS（开发周期较长），例如BBR算法。

### 支持情况

* Client支持
  - Google Chrome `chrome://net-internals/#quic`
  - Opera
* Server支持
  - Google的[prototype-server](https://cs.chromium.org/chromium/src/net/tools/quic/quic_server.cc)
  - [Caddy](https://github.com/beacer/caddy)项目及使用的库[quic-go](https://github.com/lucas-clemente/quic-go)
  - LiteSpeed Technologies的[负载均衡器](https://www.litespeedtech.com/products/litespeed-web-adc)及WebServer。

### 源代码

* [Chromium](http://www.chromium.org/developers/how-tos/get-the-code),Google Chrome web browser, C++
* [quic-go](https://github.com/lucas-clemente/quic-go) , Go, Caddy项目使用。
* [LSQUIC Client Library](https://github.com/litespeedtech/lsquic-client), C, LiteSpeed Web Server的实现。
* [Quicr](https://github.com/Ralith/quicr)及[Quinn](https://github.com/djc/quinn), Rust语言.


# QUIC设计文档

Jim Roskind的[QUIC设计文档](https://docs.google.com/document/d/1RNHkx_VvKWyWg6Lr8SZ-saqsQx7rFV-ev2jRFUoVD34/edit), 解释了发明QUIC的历史，背景，中心设计哲学。

### 设计动机

##### 降低时延

减低Internet时延,提供更好的交互式服务。带宽在不断增长，而时延受制于光速等因素无法减少。我们需要更少的时延以及更少费时的重传((less time-consuming retransmits)。

> 时延包括：传播时延，传输时延，处理时延，排队时延 - 《Web性能权威指南》

##### 连接复用

“IP和socket对”的资源有限，现有的优化使用多个连接，只是传输冗余的信息。

* 复用的传输可以降低端口的使用数，
* 统一报告及响应通道的特性（如丢包）；
* 允许更高层的协议压缩冗余数据（例如SPDY/H2的headers）。
* 让3层负载均衡器将同一个client的连接流量让同一个Server处理。

> 此外，还能解决TCP的头端阻塞问题；以及支持roaming。
> 比如浏览器为实现HTTP/1.0 pipeline、避免头端阻塞，使用多个TCP连接 - 《Web性能权威指南》

##### 支持SPDY

QUIC其中一个主要的动机是更好的支持SPDY。SPDY可以工作在TCP/SSL上，通过请求pipeline降低延迟（发送多个请求，不需要等之前的请求完成），是一样压缩降低带宽（二进制协议，首部压缩）．但SPDY遇到了几个问题，

* 单个分组的延迟会引入头端阻塞（TCP有序交付模型的头端阻塞问题），需要更好的传输复用技术．
* 不适宜的TCP拥塞避免导致额外的带宽减少，也串行化延时开销．单个SPDY连接代替Ｋ个独立的连接．一个包丢失，整个TCP连接（所有SPDY连接）的拥塞窗口减半（CUBIC是30%)．复用同一个TCP连接，无法做到按stream流控．
* TLS/SSL会话恢复延迟（resumption delay）．在传数据前，TLS握手至少需要一个额外的RTT（来恢复session），比完全握手快．但是这是TLS实现的问题，而非功能或安全上的必须．
* TLS引入了解密依赖，前面的分组必须先解密，才能解密后续分组．

> TLS 1.3的完全握手只需要1个额外的RTT．

### 设计目标

QUIC设计目标，

* 广泛的部署与当今的Internet

  要能通过middle-boxes，无需client的内核升级，或提升至特权(elevated privileges)等．这个目标可以使用UDP-based用户态协议解决．middle-boxex和防火墙通常阻挡或者大幅降级除了TCP/UDP之外的传输，因此不考虑重新发明（revolution)新的传输层。

* 减少丢包引起的头端阻塞(head-of-line blocking)

  即一个连接分多个stream，一个stream的丢包不影响其他复用的stream．可以发展TCP解决服用流相互阻塞，例如MPTCP，但是依然解决不了TCP Socket接口的in-order交付模型，而更改内核会违反目标1。依然使用流复用和UDP解决，UDP的API没有in-order交付模型。

* 低延时

　最小化RTT开销，包括连接建立（setup），恢复（resumption)以及应对丢包。
  - 明显减少连接建立延迟(0-RTT，加密hello, initial-request)
  - 尝试使用FEC减少丢包的重传。

  > FEC已经从最新的QUIC版本中移除，原因是在许多网络环境中效果不如预期。

* 改善移动支持

  包括延迟和效率（radio关闭的时候TCP连接也随之关闭），关闭连接后重新建立或恢复非常耗时。QUIC支持0-RTT会话恢复。

* 相对TCP更有好的拥塞避免支持

  独立的stream流量控制，而非整个连接。防止快发生端造成慢接收端缓冲溢出。提供TCP友好、兼容的协议。

* 相当于TLS的私密性（不需要按序传输按序解密）

  防止middle-boxes修改新协议的唯一方法是加密。同时要提供等同于TLS的 *抗干扰*，*私密性*, *重放包含*, *认证* 特性。

* 可靠且安全的资源需求可伸缩性。
* 降低带宽消耗，增加通道状态响应（多个复用的stream采取统一的channel状态信号）
* 在多个复用stream中实现可靠传输
* 对于代理（proxy）高效的复用和解复用属性。
* 重用，发展现有的协议

### API元素

建立复用流的API有几个复杂性问题待消除:

* 添加新的stream到connection中的API
* 从不同的stream分别read/write的API
* 设置每个不同的stream包括：可靠性，性能配置(根据不同网络环境tradeoff)

##### Stream特性

不同的stream有不同的特性，可以被app单独修改。

* 可调整的冗余（redundancy）级别（针对不同的带宽、延时）
* 可调整的优先级级别（模仿SPDY所发展的优先级语义）
* 控制通道，被视为带外流（out-of-band stream），
  - 用于对其他stream状态改变发送信号；
  - 由不同目的的控制帧组成；
  - 用于加密协商。

> FEC不再支持。

##### 有序数据交付

需要提供类似TCP的可靠、有序交付(in-order-delivery)，需要针对stream的API最大程度模拟TCP Socket。以便服务大部分现有应用，且只需要最小改动。需要API最大程度模拟SPDY。

##### 连接状态

历史上，应用和连接(TCP连接)是分离的。应用完成发送后，可以选择关闭连接，但数据可能还在本地队列中没有发送或没被应答。这样会造成关闭链接，终止应用直接的竞争。为更好对应用进行有效而紧密的绑定（tight binding），以下统计对应用可见：

1. RTT (平滑预测)
2. 分组大小（包括所有的overhead, payload）
3. 带宽（整个连接，平滑预测）
4. 峰值持续带宽（Peak Sustained Bandwidth，整个连接）
5. 拥塞窗口大小
6. 队列大小
7. 队列中的字节数
8. Per-stream队列大小

通告（Notification）或事件(Event)访问，

1. 队列大小下降到0
2. 所有需要的ACK已经接收（连接可以无状态损失的关闭）

### 协议哲学

协议有4个阶段需要考虑性能效率：

* Startup

  在连接建立阶段降低延迟，包括恢复(Resumption)连接的情况。

* Steady State

  确保进入稳定状态后的高效率和低延时，

* Idle Entry

  有效快速（低延时）的切换到idle状态。包括应对TCP尾部丢包（tail-drop)引起的严重延迟。

* Idle Departure

##### 通过无状态UDP进行连接：克服NATs

最基本的问题：如何让UDP *数据报* 适应 *基于连接*(connection based)的协议，尤其是在有middle-boxex和firewall NAT服务的情况下。后者不但无法提供帮助，还会成为阻碍。

Middle-box要断开某个TCP连接，可以向两端发起`RST`。对于UDP而已，没有类似的通告方法。当middle-box解绑UDP流量后，endpoint无法发送数据到对方，也没有`NACK`，直接变成黑洞。TCP继续发送至少会引发对端`RST`。更复杂的是，如果client继续发，middle-boxes可能又建立新的绑定，新绑定可能使用了不同的源端口。这种情况下QUIC需要将新流量视为现有连接的后续流量。

调查显示NAT boxes通常会在UDP端口映射idle 30-120秒解帮，或者更快。过早的unbinding，也许因为NAT表LRU回收，或者其他脆弱的实现。这些unbinding会对QUIC造成问题。

> 即引入CID解决。

###### CID: 连接标识的关键

NAT可以导致源端口在连接生成周期内的改变（unbind/rebind)，源IP，源端口不足以识别连接。所以引入连接标识符(`CID`)，CID是一个伪随机的nonce，64bits。在client发送Server的第一个分组携带，后续分组可显式或隐式的存在CID。

Server使用CID，

* 标识不同的连接，
* 为连接”恢复（resurrect)” 会话加密上下文（参考TLS的session-ID/ticket-ID），包括加密密钥和认证密钥。

> 这里指的是Client LAN侧的NAPT，实际上LB这种甚至会导致4元组改变，还有一种情况是client切换链路（WiFi，LTE）的时候导致源IP的变化。总之，连接的四元组都可能变化。

###### NAT绑定的keep-alive

一种补救措施是使用keepalive，但是对于移动设备而言会有问题，因为它关闭radio电源省电。

###### UDP分组分片

是否需要禁止分片？好处与坏处。今后的趋势是分片会逐步减少。

> IPv6不提倡分片，分片信息不在基础header中，而且中间设备不允许分片。而是需要加强端进行PMTU发现。

分片后，最初的分组含有UDP和CID，但后续的分片缺乏这些信息。而重装需要用到IP首部的"identification"，共16 bit，一旦同时有2^16个分组在传输，就可能有冲突。NAT会加剧冲突。

所以接受方要么放弃重装努力，要么可能提供一个错误的重装分组。未能重装的分组，会被鉴权hash识别为垃圾数据，而导致协议错误，比丢弃分片更严重。

分片还影响带宽利用率，浪费目标server的时间。

现有的API无法让app了解到分片的存在。如果有这样的API，就可以避免易出错（error-prone）的“路径MTU发现，PMTU exploration”，而是采用观察分片发生和回应的方法来应对。

##### 连接建立(establishment)与恢复(resumption)




















# QUIC官方网站

Google的QUIC[官方网站](https://www.chromium.org/quic)，提供了google版QUIC的文档。

# IETF文档


References
----------

* [WiKi-QUIC](https://en.wikipedia.org/wiki/QUIC)
* Jim Roskind的[QUIC设计文档](https://docs.google.com/document/d/1RNHkx_VvKWyWg6Lr8SZ-saqsQx7rFV-ev2jRFUoVD34/edit), 解释了发明QUIC的历史，背景，中心设计哲学。
* [官方网站](https://www.chromium.org/quic)
