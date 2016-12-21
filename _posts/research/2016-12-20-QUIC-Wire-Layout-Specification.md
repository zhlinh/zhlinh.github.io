---
layout: article
title:  "QUIC Wire Layout Specification"
categories: research
tags: [QUIC, Google]
toc: true
image:
    teaser: /research/2016-12-20-QUIC-Wire-Layout-Specification/teaser.jpg
date: 2016-12-20
---

QUIC（Quick UDP Internet Connection，快速UDP互联网连接）是一种新的多路复用和安全
传输UDP，重新设计和优化了HTTP/2 语义(semantics)。QUIC将HTTP / 2作为主要的应用程序
协议。QUIC建立在几十年的传输和安全的实践经验基础上，并实现了一套具有巨大吸引力的
现代通用传输的机制。QUIC提供等效于HTTP / 2的多路复用和流控制，等效于TLS的安全性，
以及等效于TCP的连接语义，可靠性和拥塞控制。

---

## 1 引言

QUIC完全在用户空间运行，目前作为Chromium浏览器的一部分分发给用户，从而实现快速部署
和使用。作为UDP上的用户空间传输，QUIC允许使用现有协议难以快速部署的创新，因为它们
受到传统客户端和中间件的阻碍，或者由于操作系统长时间的开发和部署周期。

QUIC的一个重要目标是通过快速的实验部署向人们展示更好的传输设计。因此，我们希望将
一些重要的变化迁移到TCP和TLS中，当然这往往需要很长的迭代周期。

本文描述QUIC协议在标准化之前的[概念设计和流程规范](https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit?usp=sharing)。

[Chromium QUIC](https://www.chromium.org/quic)网页上提供了更详细的文档：

- 加密和传输握手: [QUIC-CRYPTO](https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g/edit?usp=sharing)
- 前向纠错和拥塞控制: [draft-iyengar-quic-loss-recovery](https://tools.ietf.org/html/draft-tsvwg-quic-loss-recovery)
- 早期部署的QUIC标准化建议: [draft-hamilton-quic-transp](https://tools.ietf.org/html/draft-tsvwg-quic-protocol)

---

## 2 优点

QUIC在功能上等同于TCP + TLS + HTTP/2，但在UDP之上实现。QUIC的主要优点包括：

- 较小的连接建立延迟
- 灵活的拥塞控制
- 避免头部(head-of-line)阻塞的多路复用
- 认证加密的报头和有效载荷
- 数据流(Stream)和连接流(connection flow)控制
- 连接迁移

### 2.1 较小的连接建立延迟

QUIC结合了加密和传输握手，减少了建立安全连接所需的往返次数。QUIC连接通常是0-RTT，
这意味着在大多数QUIC连接上，与应用程序数据可以发送之前的TCP + TLS所需的1-3次往返
相比，可以立即发送数据，而不必等待服务器的回复。

QUIC提供用于执行握手的专用流（流ID 1），有关当前握手协议的完整描述，请参阅
[QUIC Crypto Handshake](https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g/edit#heading=h.bzxklo2i5w6k)文档。
QUIC的握手将采用TLS 1.3。

![Pic2.1-TCP-1-round-trip](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.1-TCP-1-round-trip.jpg)
<center>图2.1 TCP - 1 round trip</center>

![Pic-2.2-TCP+TLS-3-round-trips](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.2-TCP+TLS-3-round-trips.jpg)
<center>图2.2 TCP+TLS - 3 round trips</center>

![Pic-2.3-QUIC-0-round-trip](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.3-QUIC-0-round-trip.jpg)
<center>图2.3 QUIC – 0 round trip</center>

### 2.2 灵活拥塞控制

QUIC具有可插入的拥塞控制以及比TCP更丰富的信令，这使得QUIC能够比TCP的拥塞控制算法
提供更丰富的信息。目前的拥塞控制是基于TCP Cubic的重新实现。

更丰富的信息的一个例子是原始和重传的每个分组携带新的分组序列号。这样QUIC发送端
可以区分重传的ACK与原始传输的ACK，从而避免TCP的重传模糊性问题。QUIC的ACK还明确地
携带在分组在接收和发送确认之间的延迟信息，并使用单调增加的分组号，这样可以进行
精确的往返时间（RTT）计算。

QUIC的ACK帧支持多达256个ACK块，因此QUIC可以比TCP（含SACK）进行更有弹性的重新排序，
且当存在重新排序或丢失时能够保持更多的分组。客户端和服务器都具有对端已接收分组的
更准确信息。

### 2.3 流和连接流控制

基于HTTP / 2的流控制，QUIC实现了流和连接流的控制。QUIC的流控制工作如下。QUIC接收端
通告在接收器愿意接收数据范围的每个流的绝对字节偏移限制。当数据在特定流上发送，接收
和转发时，接收器发送WINDOW_UPDATE帧，增加通告的流的偏移限制，允许对端在那个流上发送
更多的数据。

除了每个流的流控制，QUIC还实现连接流控制，用来限制QUIC接收端愿意分配给连接的聚合
缓冲区。连接流控制与流控制的工作方式相同，但转发分组和接收数据的偏移限制都是所有流
中的聚合。

与TCP的接收窗口自动调整类似，QUIC实现了自动调整流控制信用的流和连接流控制器。如果
限制了发送端的速率，QUIC通过发送WINDOW_UPDATE帧来增加信用的大小；如果接收程序慢时
则应限制发送端的发送速率。

### 2.4 复用

HTTP/2 on TCP会遇到TCP中的单路头部(head-of-line)阻塞。由于HTTP/2在TCP的单路字节流
的顶部复用多个流，TCP分段的丢失导致所有后续分段的阻塞，直到重传到达，而不管封装在
后续分段中的HTTP/2流。

因为QUIC是从底层设计用于多路复用操作，携带单个流的数据的丢失分组通常仅影响该特定流。
每个流的帧可以在流到达时立即分派到该流，使得没有丢失的流可以继续被重新组装并在应用
中取得进展。

注意：QUIC当前将HTTP/2的HPACK头部压缩在专用的头部流上来压缩HTTP头部，该专用的头部
压缩只对头(head-of-line)阻塞产生影响。

![Pic-2.4-HTTP-pipelining](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.4-HTTP-pipelining.jpg)
<center>图2.4 HTTP pipelining</center>

![Pic-2.5-Multiple-TCP-connections](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.5-Multiple-TCP-connections.jpg)
<center>图2.5 Multiple TCP connections</center>

![Pic-2.6-SPDY-multiplexing-over-TCP](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.6-SPDY-multiplexing-over-TCP.jpg)
<center>图2.6 SPDY: multiplexing over TCP</center>

![Pic-2.7-Head-of-line-blocking-in-SPDY](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.7-Head-of-line-blocking-in-SPDY.jpg)
<center>图2.7 Head-of-line blocking in SPDY</center>

![Pic-2.8-QUIC-multiplexing-over-UDP](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.8-QUIC-multiplexing-over-UDP.jpg)
<center>图2.8 QUIC: multiplexing over UDP</center>

![Pic-2.9-No-head-of-line-blocking-in-QUIC](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.9-No-head-of-line-blocking-in-QUIC.jpg)
<center>图2.9 No head-of-line blocking in QUIC</center>

### 2.5 验证和加密报头和有效载荷

TCP头部以纯文本形式出现在传输上，并且未经身份验证，所以TCP会出现过多的注入和头部
操作问题。例如接收窗口操纵和序列号覆盖。虽然其中的一些是主动攻击，但有些是由网络
中的中间件产生的，试图透明地提高TCP性能。然而，即使“性能增强”的中间件仍然有效地
限制了传输协议的可演进性，如在MPTCP的设计和其随后的部署中出现的问题。

QUIC分组总是被认证，通常有效载荷会被完全加密。未加密的分组报头仍然由接收器认证，
以阻止第三方的任何分组注入或操纵。QUIC保护连接免受端对端通信的欺骗或未被认证的
中间件操纵。

注意：重置连接的PUBLIC_RESET数据包目前不需要身份验证。

### 2.6 连接迁移

TCP连接由源地址，源端口，目标地址和目标端口组成的4元组标识。TCP的一个众所周知的
问题是连接不能在IP地址改变（例如，通过从WiFi切换到蜂窝）或端口号改变（当客户端
NAT绑定到期导致在服务器上看到的端口号的更改）时保持。虽然MPTCP解决了TCP的连接迁移
问题，但它仍然被缺乏中间件支持和操作系统部署所困扰。

QUIC连接由客户端随机生成的64位连接ID标识。QUIC可以在IP地址更改和NAT重新绑定时保持
连接，因为连接ID在这些迁移中保持相同。QUIC还提供了迁移客户端的自动加密验证，因为
迁移客户端继续使用相同的会话密钥来加密和解密数据包。

在连接由4元组明确地标识的情况下，例如当服务器使用临时端口向客户端发送分组时，还
提供了不发送连接ID以节约网络带宽的选项。

![Pic-2.10-Parking-lot-problem](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.10-Parking-lot-problem.jpg)
<center>图2.10 Parking lot problem</center>

### 2.7 数据包同步（Packet Pacing)

基于Chrome的实验表明，通常可以通过调整pacing速度减少数据包丢失。pacing减少了分组流
的波动，从而减少基于拥塞的损失（即由于溢出而导致的路由器中的分组丢弃）。我们应该
尝试使用pacing进一步去除与丢包统计的相关性，增加数据包散布于外部流中的可能性。

当前设计的拥塞避免是基于带宽估计来增加pacing，因此pacing的减少可能不在当前设计的
范围内。可能需要一些更好的钩子深入操作系统，以更好地促进操作系统能更准确进行pacing
缓冲。

即使使用与标准TCP类似的拥塞避免算法，也会尝试使用pacing以进一步减少丢包。例如，
TCP算法维护拥塞窗口大小（传输过程中的的分组计数）和平滑地估计RTT。这些变量提供
等效的pacing速率。发送速率将不会超过pacing速率，特别是从静止转换到主动传输时。

![Pic-2.11-Packet-Pacing](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.11-Packet-Pacing.jpg)
<center>图2.11 Packet Pacing</center>

### 2.8 前向纠错

前向纠错（又称FEC）允许传输额外的字节为分组丢失的情况提供冗余。 QUIC实现了基于
XOR的FEC的形式，提供了简单的N+1冗余。 FEC通常用于提高链路层可靠性（例如在Wifi中），
并且长期以来一直想将FEC添加到端到端传输层中，但这在TCP中是非常复杂的。 因此，QUIC
提供了一个理想的环境对FEC进行实验，并确定在什么情况下它能够提供等待时间或可靠性等
方面的优点。

![Pic-2.12-Forward-Error-Correction](/images/research/2016-12-20-QUIC-Wire-Layout-Specification/Pic-2.12-Forward-Error-Correction.jpg)
<center>图2.12 Forward Error Correction</center>

### 参考文献

- [QUIC wire specification](https://docs.google.com/document/d/1WJvyZflAO2pq77yOLbp9NsGjC1CHetAXV8I0fQe-B_U/edit?usp=sharing)
- [QUIC tech talk - QUIC: next generation multiplexed transport over UDP](https://www.youtube.com/watch?v=hQZ-0mXFmk8)
- [QUIC FEC v1](https://docs.google.com/document/d/1Hg1SaLEl6T4rEU9j-isovCo8VEjjnuCPTcLNJewj7Nk/edit?usp=sharing)

---
The End.

zhlinh

Email: zhlinhng@gmail.com

2016-12-20
