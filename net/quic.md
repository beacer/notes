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

QUIC其中一个主要的动机是更好的支持SPDY。

# QUIC官方文档

Google的QUIC[官方网站](https://www.chromium.org/quic)，提供了google版QUIC的文档。

# IETF文档


References
----------

* [WiKi-QUIC](https://en.wikipedia.org/wiki/QUIC)
* Jim Roskind的[QUIC设计文档](https://docs.google.com/document/d/1RNHkx_VvKWyWg6Lr8SZ-saqsQx7rFV-ev2jRFUoVD34/edit), 解释了发明QUIC的历史，背景，中心设计哲学。
* [官方网站](https://www.chromium.org/quic)
