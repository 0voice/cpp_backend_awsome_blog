# 【NO.240】HTTP/3 原理实战

015 年 HTTP/2 标准发表后，大多数主流浏览器也于当年年底支持该标准。此后，凭借着多路复用、头部压缩、服务器推送等优势，HTTP/2 得到了越来越多开发者的青睐。不知不觉的 HTTP 已经发展到了第三代，鹅厂也紧跟技术潮流，很多项目也在逐渐使用 HTTP/3。本文基于兴趣部落接入 HTTP/3 的实践，聊一聊 HTTP/3 的原理以及业务接入的方式。

## **1. HTTP/3 原理**

### 1.1 HTTP 历史

在介绍 HTTP/3 之前，我们先简单看下 HTTP 的历史，了解下 HTTP/3 出现的背景。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWnvOermFicEiaDXia4lnGho0CVXCVAqjOakxAluLcrPVdYM44kqUQeKX2w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

随着网络技术的发展，1999 年设计的 HTTP/1.1 已经不能满足需求，所以 Google 在 2009 年设计了基于 TCP 的 SPDY，后来 SPDY 的开发组推动 SPDY 成为正式标准，不过最终没能通过。不过 SPDY 的开发组全程参与了 HTTP/2 的制定过程，参考了 SPDY 的很多设计，所以我们一般认为 SPDY 就是 HTTP/2 的前身。无论 SPDY 还是 HTTP/2，都是基于 TCP 的，TCP 与 UDP 相比效率上存在天然的劣势，所以 2013 年 Google 开发了基于 UDP 的名为 QUIC 的传输层协议，QUIC 全称 Quick UDP Internet Connections，希望它能替代 TCP，使得网页传输更加高效。后经[提议](https://mailarchive.ietf.org/arch/msg/quic/RLRs4nB1lwFCZ_7k0iuz0ZBa35s)，互联网工程任务组正式将基于 QUIC 协议的 HTTP （HTTP over QUIC）重命名为 HTTP/3。

### 1.2 QUIC 协议概览

TCP 一直是传输层中举足轻重的协议，而 UDP 则默默无闻，在面试中问到 TCP 和 UDP 的区别时，有关 UDP 的回答常常寥寥几语，长期以来 UDP 给人的印象就是一个很快但不可靠的传输层协议。但有时候从另一个角度看，缺点可能也是优点。QUIC（Quick UDP Internet Connections，快速 UDP 网络连接） 基于 UDP，正是看中了 UDP 的速度与效率。同时 QUIC 也整合了 TCP、TLS 和 HTTP/2 的优点，并加以优化。用一张图可以清晰地表示他们之间的关系。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWYBhWzvwiaL1e7urxqw61NLjQjtIY3RSBRG79iaG7AKLguu0LxCdWZD5g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

那 QUIC 和 HTTP/3 什么关系呢？QUIC 是用来替代 TCP、SSL/TLS 的传输层协议，在传输层之上还有应用层，我们熟知的应用层协议有 HTTP、FTP、IMAP 等，这些协议理论上都可以运行在 QUIC 之上，其中运行在 QUIC 之上的 HTTP 协议被称为 HTTP/3，这就是”HTTP over QUIC 即 HTTP/3“的含义。

因此想要了解 HTTP/3，QUIC 是绕不过去的，下面主要通过几个重要的特性让大家对 QUIC 有更深的理解。

### 1.3 零 RTT 建立连接

用一张图可以形象地看出 HTTP/2 和 HTTP/3 建立连接的差别。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWmEEdP9aadwVFaSdqpMZyMfqAo1gMibB9UIjDxEiatGc6HJ70GG7OdYyQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

HTTP/2 的连接需要 3 RTT，如果考虑会话复用，即把第一次握手算出来的对称密钥缓存起来，那么也需要 2 RTT，更进一步的，如果 TLS 升级到 1.3，那么 HTTP/2 连接需要 2 RTT，考虑会话复用则需要 1 RTT。有人会说 HTTP/2 不一定需要 HTTPS，握手过程还可以简化。这没毛病，HTTP/2 的标准的确不需要基于 HTTPS，但实际上所有浏览器的实现都要求 HTTP/2 必须基于 HTTPS，所以 HTTP/2 的加密连接必不可少。而 HTTP/3 首次连接只需要 1 RTT，后面的连接更是只需 0 RTT，意味着客户端发给服务端的第一个包就带有请求数据，这一点 HTTP/2 难以望其项背。那这背后是什么原理呢？我们具体看下 QUIC 的连接过程。

**Step1**：首次连接时，客户端发送 Inchoate Client Hello 给服务端，用于请求连接；

**Step2**：服务端生成 g、p、a，根据 g、p 和 a 算出 A，然后将 g、p、A 放到 Server Config 中再发送 Rejection 消息给客户端；

**Step3**：客户端接收到 g、p、A 后，自己再生成 b，根据 g、p、b 算出 B，根据 A、p、b 算出初始密钥 K。B 和 K 算好后，客户端会用 K 加密 HTTP 数据，连同 B 一起发送给服务端；

**Step4**：服务端接收到 B 后，根据 a、p、B 生成与客户端同样的密钥，再用这密钥解密收到的 HTTP 数据。为了进一步的安全（前向安全性），服务端会更新自己的随机数 a 和公钥，再生成新的密钥 S，然后把公钥通过 Server Hello 发送给客户端。连同 Server Hello 消息，还有 HTTP 返回数据；

**Step5**：客户端收到 Server Hello 后，生成与服务端一致的新密钥 S，后面的传输都使用 S 加密。

这样，QUIC 从请求连接到正式接发 HTTP 数据一共花了 1 RTT，这 1 个 RTT 主要是为了获取 Server Config，后面的连接如果客户端缓存了 Server Config，那么就可以直接发送 HTTP 数据，实现 0 RTT 建立连接。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWECanehia5OEtUVNOehe2Y8T2HOfqHjhzDETiaTwSeVZqNuFIFuPjC1sg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这里使用的是 DH 密钥交换算法，DH 算法的核心就是服务端生成 a、g、p 3 个随机数，a 自己持有，g 和 p 要传输给客户端，而客户端会生成 b 这 1 个随机数，通过 DH 算法客户端和服务端可以算出同样的密钥。在这过程中 a 和 b 并不参与网络传输，安全性大大提高。因为 p 和 g 是大数，所以即使在网络中传输的 p、g、A、B 都被劫持，那么靠现在的计算机算力也没法破解密钥。

### 1.4 连接迁移

TCP 连接基于四元组（源 IP、源端口、目的 IP、目的端口），切换网络时至少会有一个因素发生变化，导致连接发生变化。当连接发生变化时，如果还使用原来的 TCP 连接，则会导致连接失败，就得等原来的连接超时后重新建立连接，所以我们有时候发现切换到一个新网络时，即使新网络状况良好，但内容还是需要加载很久。如果实现得好，当检测到网络变化时立刻建立新的 TCP 连接，即使这样，建立新的连接还是需要几百毫秒的时间。

QUIC 的连接不受四元组的影响，当这四个元素发生变化时，原连接依然维持。那这是怎么做到的呢？道理很简单，QUIC 连接不以四元组作为标识，而是使用一个 64 位的随机数，这个随机数被称为 Connection ID，即使 IP 或者端口发生变化，只要 Connection ID 没有变化，那么连接依然可以维持。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWHQtiapr3Ou7KEXpN73yBzibfnTY7H3qNNSOyJKYHXzZ85OaRPoGAxYKA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 1.5 队头阻塞/多路复用

HTTP/1.1 和 HTTP/2 都存在队头阻塞问题（Head of line blocking），那什么是队头阻塞呢？

TCP 是个面向连接的协议，即发送请求后需要收到 ACK 消息，以确认对方已接收到数据。如果每次请求都要在收到上次请求的 ACK 消息后再请求，那么效率无疑很低。后来 HTTP/1.1 提出了 Pipelining 技术，允许一个 TCP 连接同时发送多个请求，这样就大大提升了传输效率。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWM59JxDoQiaBBpyw9R3IfhwiahY1Pn7NZMwo6Wq84F7I3pvb1hopSQwkg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在这个背景下，下面就来谈 HTTP/1.1 的队头阻塞。下图中，一个 TCP 连接同时传输 10 个请求，其中第 1、2、3 个请求已被客户端接收，但第 4 个请求丢失，那么后面第 5 - 10 个请求都被阻塞，需要等第 4 个请求处理完毕才能被处理，这样就浪费了带宽资源。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWTrHQW4h5T1cgt4rBMPuvJ5pj4CtfcgQR4Tl0sgWlrmYzFXQCdJDRSQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

因此，HTTP 一般又允许每个主机建立 6 个 TCP 连接，这样可以更加充分地利用带宽资源，但每个连接中队头阻塞的问题还是存在。

HTTP/2 的多路复用解决了上述的队头阻塞问题。不像 HTTP/1.1 中只有上一个请求的所有数据包被传输完毕下一个请求的数据包才可以被传输，HTTP/2 中每个请求都被拆分成多个 Frame 通过一条 TCP 连接同时被传输，这样即使一个请求被阻塞，也不会影响其他的请求。如下图所示，不同颜色代表不同的请求，相同颜色的色块代表请求被切分的 Frame。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWpYxO466rVmFzmic38XsyZ9Ydd0Im4L671icWnZmmhuicQu58EibwkGNkEA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

事情还没完，HTTP/2 虽然可以解决“请求”这个粒度的阻塞，但 HTTP/2 的基础 TCP 协议本身却也存在着队头阻塞的问题。HTTP/2 的每个请求都会被拆分成多个 Frame，不同请求的 Frame 组合成 Stream，Stream 是 TCP 上的逻辑传输单元，这样 HTTP/2 就达到了一条连接同时发送多条请求的目标，这就是多路复用的原理。我们看一个例子，在一条 TCP 连接上同时发送 4 个 Stream，其中 Stream1 已正确送达，Stream2 中的第 3 个 Frame 丢失，TCP 处理数据时有严格的前后顺序，先发送的 Frame 要先被处理，这样就会要求发送方重新发送第 3 个 Frame，Stream3 和 Stream4 虽然已到达但却不能被处理，那么这时整条连接都被阻塞。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWmSLxgFU3bT0Ccic1wdXDuzJab4BJzYYMD4wArCKMczwjj4WbIysjqQQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

不仅如此，由于 HTTP/2 必须使用 HTTPS，而 HTTPS 使用的 TLS 协议也存在队头阻塞问题。TLS 基于 Record 组织数据，将一堆数据放在一起（即一个 Record）加密，加密完后又拆分成多个 TCP 包传输。一般每个 Record 16K，包含 12 个 TCP 包，这样如果 12 个 TCP 包中有任何一个包丢失，那么整个 Record 都无法解密。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWdoAdNLia6SY2PnmfFqHqPwtPJzfYqJkrGLba8j7s7udVUqlm9z260FQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

队头阻塞会导致 HTTP/2 在更容易丢包的弱网络环境下比 HTTP/1.1 更慢！

那 QUIC 是如何解决队头阻塞问题的呢？主要有两点。

- QUIC 的传输单元是 Packet，加密单元也是 Packet，整个加密、传输、解密都基于 Packet，这样就能避免 TLS 的队头阻塞问题；
- QUIC 基于 UDP，UDP 的数据包在接收端没有处理顺序，即使中间丢失一个包，也不会阻塞整条连接，其他的资源会被正常处理。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWuO3G08LUiaSGKNwyWeUlicOcV6Vm4jKZbBPfGcmqn5uX93KibLLDn70pw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 1.6 拥塞控制

拥塞控制的目的是避免过多的数据一下子涌入网络，导致网络超出最大负荷。QUIC 的拥塞控制与 TCP 类似，并在此基础上做了改进。所以我们先简单介绍下 TCP 的拥塞控制。

TCP 拥塞控制由 4 个核心算法组成：慢启动、拥塞避免、快速重传和快速恢复，理解了这 4 个算法，对 TCP 的拥塞控制也就有了大概了解。

- 慢启动：发送方向接收方发送 1 个单位的数据，收到对方确认后会发送 2 个单位的数据，然后依次是 4 个、8 个……呈指数级增长，这个过程就是在不断试探网络的拥塞程度，超出阈值则会导致网络拥塞；
- 拥塞避免：指数增长不可能是无限的，到达某个限制（慢启动阈值）之后，指数增长变为线性增长；
- 快速重传：发送方每一次发送时都会设置一个超时计时器，超时后即认为丢失，需要重发；
- 快速恢复：在上面快速重传的基础上，发送方重新发送数据时，也会启动一个超时定时器，如果收到确认消息则进入拥塞避免阶段，如果仍然超时，则回到慢启动阶段。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWjGibTw5K3FJaticxkIuTv0KTL4UfZswO9743OOibCebvaeDF5UxKIvpdg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

QUIC 重新实现了 TCP 协议的 Cubic 算法进行拥塞控制，并在此基础上做了不少改进。下面介绍一些 QUIC 改进的拥塞控制的特性。

#### **1.6.1 热插拔**

TCP 中如果要修改拥塞控制策略，需要在系统层面进行操作。QUIC 修改拥塞控制策略只需要在应用层操作，并且 QUIC 会根据不同的网络环境、用户来动态选择拥塞控制算法。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWaoEc2pM2rXsgLqZbtUZgxWppaLlFmibuYEUdNSOYNnzCL0SIiajk93qA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.6.2 前向纠错 FEC**

QUIC 使用前向纠错(FEC，Forward Error Correction)技术增加协议的容错性。一段数据被切分为 10 个包后，依次对每个包进行异或运算，运算结果会作为 FEC 包与数据包一起被传输，如果不幸在传输过程中有一个数据包丢失，那么就可以根据剩余 9 个包以及 FEC 包推算出丢失的那个包的数据，这样就大大增加了协议的容错性。

这是符合现阶段网络技术的一种方案，现阶段带宽已经不是网络传输的瓶颈，往返时间才是，所以新的网络传输协议可以适当增加数据冗余，减少重传操作。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWtFEqffsZuZb21l36HrmeQcae4QgjaLMYc1epb1xQpNuWA9Peqqxt8g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.6.3 单调递增的 Packet Number**

TCP 为了保证可靠性，使用 Sequence Number 和 ACK 来确认消息是否有序到达，但这样的设计存在缺陷。

超时发生后客户端发起重传，后来接收到了 ACK 确认消息，但因为原始请求和重传请求接收到的 ACK 消息一样，所以客户端就郁闷了，不知道这个 ACK 对应的是原始请求还是重传请求。如果客户端认为是原始请求的 ACK，但实际上是左图的情形，则计算的采样 RTT 偏大；如果客户端认为是重传请求的 ACK，但实际上是右图的情形，又会导致采样 RTT 偏小。图中有几个术语，RTO 是指超时重传时间（Retransmission TimeOut），跟我们熟悉的 RTT（Round Trip Time，往返时间）很长得很像。采样 RTT 会影响 RTO 计算，超时时间的准确把握很重要，长了短了都不合适。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWoC6QKq0BIKbpKlXYGH85EickJ63iaH2CrhOFTRd5AbzSVllXrd3ib71lw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

QUIC 解决了上面的歧义问题。与 Sequence Number 不同的是，Packet Number 严格单调递增，如果 Packet N 丢失了，那么重传时 Packet 的标识不会是 N，而是比 N 大的数字，比如 N + M，这样发送方接收到确认消息时就能方便地知道 ACK 对应的是原始请求还是重传请求。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWuQ8pcjaOB2nqsnjkVTxx5yXGSE10NrX9iaN54vFU1WSCQWmpBxVLOTg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.6.4 ACK Delay**

TCP 计算 RTT 时没有考虑接收方接收到数据到发送确认消息之间的延迟，如下图所示，这段延迟即 ACK Delay。QUIC 考虑了这段延迟，使得 RTT 的计算更加准确。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWdaia1QzrXBL2CkUMW2oSGzibiceEgBM9XF4w1d5uzjRKMnUicpiayFFmPYQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.6.5 更多的 ACK 块**

一般来说，接收方收到发送方的消息后都应该发送一个 ACK 回复，表示收到了数据。但每收到一个数据就返回一个 ACK 回复太麻烦，所以一般不会立即回复，而是接收到多个数据后再回复，TCP SACK 最多提供 3 个 ACK block。但有些场景下，比如下载，只需要服务器返回数据就好，但按照 TCP 的设计，每收到 3 个数据包就要“礼貌性”地返回一个 ACK。而 QUIC 最多可以捎带 256 个 ACK block。在丢包率比较严重的网络下，更多的 ACK block 可以减少重传量，提升网络效率。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWehdleicj8WJiahBoOn06vYqPICDdMWSaaKHMvHGSDQy6pWVhLAWYYekQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 1.7 流量控制

TCP 会对每个 TCP 连接进行流量控制，流量控制的意思是让发送方不要发送太快，要让接收方来得及接收，不然会导致数据溢出而丢失，TCP 的流量控制主要通过滑动窗口来实现的。可以看出，拥塞控制主要是控制发送方的发送策略，但没有考虑到接收方的接收能力，流量控制是对这部分能力的补齐。

QUIC 只需要建立一条连接，在这条连接上同时传输多条 Stream，好比有一条道路，两头分别有一个仓库，道路中有很多车辆运送物资。QUIC 的流量控制有两个级别：连接级别（Connection Level）和 Stream 级别（Stream Level），好比既要控制这条路的总流量，不要一下子很多车辆涌进来，货物来不及处理，也不能一个车辆一下子运送很多货物，这样货物也来不及处理。

那 QUIC 是怎么实现流量控制的呢？我们先看单条 Stream 的流量控制。Stream 还没传输数据时，接收窗口（flow control receive window）就是最大接收窗口（flow control receive window），随着接收方接收到数据后，接收窗口不断缩小。在接收到的数据中，有的数据已被处理，而有的数据还没来得及被处理。如下图所示，蓝色块表示已处理数据，黄色块表示未处理数据，这部分数据的到来，使得 Stream 的接收窗口缩小。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWLwmYjffpz4mUJmCqCjp05e945NcZnkHKvH8GIk2m0y6NGnIUgAC1xQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

随着数据不断被处理，接收方就有能力处理更多数据。当满足 (flow control receive offset - consumed bytes) < (max receive window / 2) 时，接收方会发送 WINDOW_UPDATE frame 告诉发送方你可以再多发送些数据过来。这时 flow control receive offset 就会偏移，接收窗口增大，发送方可以发送更多数据到接收方。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWfIwUqJOaoPCzh4ic2mOeI6bFicO65fAHVH3wWTLSFNblFd6OPqO5mFoQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

Stream 级别对防止接收端接收过多数据作用有限，更需要借助 Connection 级别的流量控制。理解了 Stream 流量那么也很好理解 Connection 流控。Stream 中，接收窗口(flow control receive window) = 最大接收窗口(max receive window) - 已接收数据(highest received byte offset) ，而对 Connection 来说：接收窗口 = Stream1 接收窗口 + Stream2 接收窗口 + ... + StreamN 接收窗口 。

## **2. HTTP/3 实践**

### 2.1 X5 内核与 STGW

X5 内核是腾讯开发的适用于安卓系统的浏览器内核，为了解决传统安卓系统浏览器内核适配成本高、不安全、不稳定等问题而开发的统一的浏览器内核。STGW 是 Secure Tencent Gateway 的缩写，意思是腾讯安全云网关。两者早在前两年便支持了 QUIC 协议。

那作为运行在 X5 上的业务，我们该如何接入 QUIC 呢？得益于 X5 和 STGW，业务在接入 QUIC 时所需要做的改动非常小，只需要两步。

**Step 1.** 在 STGW 上开启白名单，允许业务域名接入 QUIC 协议；

**Step 2.** 业务资源的 Response Header 添加 alt-svc 属性，示例：alt-svc: quic=":443"; ma=2592000; v="44,43,39"。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWoiafEXezdpgh5rA3YNfLcibgxKjRiata4CfdbT0OJnMPgrOz8jibWGMl6g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

接入 QUIC 时，STGW 的优势非常明显，由 STGW 与支持 QUIC 的客户端(这里是 X5)进行通信，而业务后台与 STGW 仍然使用 HTTP/1.1 通信，QUIC 所需要的 Server Config 等缓存信息也都是由 STGW 维护。

### 2.2 协商升级与竞速

业务域名加入了 STGW 的白名单，业务资源的 Response Header 也添加了 alt-svc 属性，那 QUIC 是如何建立连接的呢？这里有个关键的步骤：协商升级。客户端不确定服务器是否支持 QUIC，如果贸然地请求建立 QUIC 连接可能会失败，所以需要经历协商升级过程才能决定是否使用 QUIC。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWVZjCk0mIhIfnbQhbdcBDkKgN1KJdrABMsMiaibD1EFJ8VF2s72ibsLbFw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

首次请求时，客户端会使用 HTTP/1.1 或者 HTTP/2，如果服务器支持 QUIC，则在响应的数据中返回 alt-svc 头部，告诉客户端下次请求可以走 QUIC。alt-svc 主要包含以下信息：

- quic：监听的端口；
- ma：有效时间，单位是秒，承诺在这段时间内都支持 QUIC；
- 版本号：QUIC 的迭代很快，这里列出所有支持的版本号。

确认服务器支持 QUIC 之后，客户端向服务端同时发起 QUIC 连接和 TCP 连接，比较两个连接的速度，然后选择较快的协议，这个过程叫“竞速”，一般都是 QUIC 获胜。

### 2.3 QUIC 性能表现

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWkOia4ngYsTQXtJ552XqgchZMrUuO4WqUREY2dOR7AD3LVD8t4DwFyqA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

QUIC 建立连接的成功率在 90% 以上，竞速成功率也接近 90%，0 RTT 率在 55% 左右。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWZkJs7ufuV8MJ2LQw8sKQiaWPXQEYdL2ibMUtEj4K4MaczBhEOdN8m2vg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

使用 QUIC 协议时页面首屏耗时要比非 QUIC 协议减少 10%。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvWyxXibIPKnH4lowsamFU16SFtuRXc3DCwBbj8JI3sqsibvrJiaMTgZ3qYQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

从资源获取的不同阶段看，QUIC 协议在连接阶段节省的时间比较明显。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvatOuue8A7K8MQAlFoCOtDvW95OV7N3o5uV4qFNQ7DkWKx5bLgzQUupakFt5qy140gRNgAia9MR56Rw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

从页面首屏区间占比图中可以看出，使用了 QUIC 协议后，首屏耗时在 1 秒内的占比提升明显，大约在 12% 左右。

## **3. 总结**

QUIC 丢掉了 TCP、TLS 的包袱，基于 UDP，并对 TCP、TLS、HTTP/2 的经验加以借鉴、改进，实现了一个安全高效可靠的 HTTP 通信协议。凭借着 0 RTT 建立连接、平滑的连接迁移、基本消除了队头阻塞、改进的拥塞控制和流量控制等优秀的特性，QUIC 在绝大多数场景下获得了比 HTTP/2 更好的效果。

一周前，微软宣布开源自己的内部 QUIC 库 -- MsQuic，将全面推荐 QUIC 协议替换 TCP/IP 协议。

原文作者：billpchen，腾讯看点前端开发工程师

原文链接：https://mp.weixin.qq.com/s/MHYMOYHqhrAbQ0xtTkV2ig