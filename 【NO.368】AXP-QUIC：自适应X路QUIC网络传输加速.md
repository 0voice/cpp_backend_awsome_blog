# 【NO.368】AXP-QUIC：自适应X路QUIC网络传输加速

**导语:**  腾讯云即时通信IM实现了一种网络自适应的X路QUIC传输加速技术AXP-QUIC（Adaptive X-PATH QUIC），已应用于IM SDK客户端到服务端的数据传输。该技术建立了一套客户端弱网自评估模型，根据网络链路的RTT，丢包率，吞吐量，并结合主动探测，判断终端当前是否处于弱网络环境。同时将QUIC协议和多通道传输技术相结合，根据终端所处的网络环境，实时自动决定切换网络链路或使用多链路进行传输。**通过AXP-QUIC技术，即时通信IM能够在70%丢包的弱网络环境下，保证消息100%可靠传输，且不大幅度增加消息延时。**



随着互联网技术的发展，依托互联网通信的应用对网络传输质量也提出了更高的要求。特别是在即时通信、实时音视频、实时信令交互等对实时性或可靠性有高要求的场景，网络传输质量是影响产品体验的决定性因素之一。

通常，应用服务通过广泛的节点部署或者接入加速网络，结合最优调度，使服务接入点尽可能靠近终端用户，以降低终端到接入点的网络延时（第一公里）。数据到达接入点后，通过中继转发、最短路径、加速协议、多路传输等加速技术实现骨干网的传输加速（第二公里）。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAE0mCoxavmMcib5Xat0amiaaZWQT9RJt8CRcqWFEtZ4OMHXzO6rFEwKLtRRJHeYFYrV9QEtTGDthoyQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**移动弱网络**

在移动网络环境下，受终端物理位置变化、信号强度、AP接入等因素影响，用户网络质量具有较大的不可靠性。对于传输层协议，UDP是不可靠传输，数据的准确性需要应用层来保证。而TCP在高丢包网络环境传输效率表现差，1%的丢包情况下吞吐率降低90%（RTT 250ms）。虽然通过使用BBR等基于带宽预测的拥塞控制算法，可以提升传输性能，但在丢包率超过15%的弱网环境下，传输能力也大幅下降。

![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9fLRr2ImrIajvwOaibRS2mCIVGnmo5eJjQpso7Q0hdYss7eUFvibcCy2rw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



因此，研究如何实现在不可靠的移动弱网络环境下提供稳定可靠的数据传输，是提升移动互联网应用竞争力的关键问题。

**方案探索**

**1. 传输协议和策略**

本文探索如何实现在不可靠的终端网络环境下提供稳定可靠的数据传输。我们知道，TCP是可靠传输协议，但是存在重传歧义等问题，抗丢包能力较差。而UDP是不可靠的传输协议，但传输快，开销小。针对TCP和UDP的优缺点，为了实现快速、可靠、稳定的传输目标，一般的思路是基于UDP构建可靠的应用层传输协议，比如KCP和QUIC，再配合多发、FEC等策略，实现弱网环境下更好的传输表现。另外，随着移动操作系统的优化升级，能够支持应用同时使用cellular网络和wifi网络进行数据传输，以提升传输效率。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEmibsThfmfroQxGgsKbmm9f0xvVQ8jEBYdCCf2ZwbOpUZU6ib03AZiamNfiaBqHxw7EmFWF5sm343lyQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



**2. QUIC简介**

QUIC (Quick UDP Internet Connection) 是Google在UDP上实现的一种多路复用安全传输层，提供了多路复用和流控机制，安全等效于TLS，可靠性和拥塞控制类似于TCP。QUIC完全在用户空间中运行，可以理解为利用UDP封装实现的安全传输层。所以相比对TCP/UDP这些操作系统协议栈优化，QUIC迭代起来更方便。

![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9fG3YE2NlfHu8WWGVZ4Wibnrne8RmU5VJPX9P7koJ8MQiac22EF5pgpe6A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



QUIC作为基于UDP实现的一种通用可靠的“传输层”，具有如下关键特性：

- **建联延时**。对于TCP+TLS，建联完成需要经过TCP 3次握手+TLS握手，共3个 RTT。QUIC的Client和Server建联后，QUIC Client和Server在QUIC层缓存维护和socket五元组无关的逻辑连接session。如果需要传输数据，直接使用该session即可。这个session包含加解密的上下文信息，并且在QUIC连接数据传输过程中更新。因此，后续的数据传输是0 RTT。
- **灵活的拥塞控制****。**QUIC集成了TCP这么多年来积累的优秀思想，如FACK、TLP、 F-RTO、Early Retransmit、Pacing等，提供丰富可扩展的拥塞控制策略。另外，QUIC可以提供更丰富的信息供这些策略使用，比如每一个包携带新的seq，包括原始包和重传包。这样发送者能够区分ACK是属于重传包还是属于原始包，以避免TCP重传二义性，以及丢失重传包造成的RTO问题。QUIC ACK包同时携带了从收到包到回复ACK的延时，这样结合递增的包序号，能够精确的计算RTT。
- **无队头阻塞的多路复用****。**类似于基于TCP的HTTP/2多路复用协议有队头阻塞的问题。TCP丢包可能会影响该TCP连接上的所有流。QUIC多路复用的stream流有独立的收发窗口，某个stream的丢包，不会影响其他stream的数据传输。
- **TLS传输层安全****。**保证数据传输的安全。
- **连接迁移****。**TCP是基于4元组，在链路变化时，需要重新建立连接才能传输数据。而QUIC使用64位的Connection ID来维护客户端和服务端的逻辑连接，因此即使UDP链路发生变化，QUIC层的逻辑连接维持不变，两端收到的QUIC包能够被正常解析。

**AXP - QUIC**

综合在不可靠网络环境下实现稳定可靠传输的一般策略方法，并了解到QUIC传输协议的优势，我们将使用QUIC作为客户端和服务器之间的传输层协议。

但是在实践的过程中，我们发现，有时候终端处在弱网环境，wifi信号并未彻底断开，但数据传输实际上已经有损。并且可能此时终端cellular网络是可用的，但因系统没有切换网络，导致应用层无法及时更换传输链路。为解决这种场景下的问题，我们建立了一套客户端弱网自评估模型，根据网络链路的RTT，丢包率，吞吐率，并结合主动探测，判断终端的网络链路是否处于弱网络环境，在数据通道有损的情况下，尽可能早的主动进行连接迁移，减少对上层业务的影响。

另外，结合多网卡可多路径传输的优势，综合多链路的RTT，丢包，吞吐量等指标，AXP-QUIC客户端自动决定使用单条通道还是多条通道同时进行数据传输，以达到传输质量、速率和成本的最优。

**1. AXP-QUIC架构**

![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9fu7Af4pAm5IHMK7iaRyia9EhWiaNbjQzxyvOzYLPMIuhiaCnX8yjElKQbGg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



**2. 链路质量衡量**

移动网络质量能够通过RTT，丢包率，信号强度，带宽延时乘积等指标体现。这些指标可以通过被动网络采集或主动探测的方式获取。通过将这些指标量化转换成一个衡量链路质量的数值，这样链路之间可以进行网络质量对比。

```
PathQuality = f ( RTT, Lost, Signal, Throughput... )
```



**3. 链路选择**

我们定义如下几种链路选择的类型：

```
enum XPathType {XPathTypeNone,     // 不使用多链路XPathTypeHandover,  // 只有当主链路不可用，才会使用第二条链路XPathTypeInteractive, // 当主链路不够用，比如丢包或延时高，就会启用第二条链路XPathTypeAggregate  // 为了更大的带宽，多条链路可以一起使用}
```



在多条路径链路场景下，需要考虑slow path blocking问题。因此，在选择链路时，需要考虑各链路的传输能力差距。

```
func ChoosePath (Path p1, PathQuality q1, Path p2, PathQuality q2, XPathType xtype){if (q1 - q2 > MAX_DIFF_QUALITY)return p1, xtype;if (q2 - q1 > MAX_DIFF_QUALITY)return p2, xtype;return p1, p2, xtype}
```

**AXP - QUIC**

目前AXP-QUIC已经应用于腾讯云即时通信IM服务。按照40条/秒的消息发送速率，并测试不同上行丢包率下的表现。实测表明，该方案在对抗网络丢包方面，相比于TCP有显著的效果。

![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9fDMGPv8T7wEfteYgtKEDVh176o5SCtPqnIB3ia0cRicSoyaaUyiaQn6S8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9f60BqxiaFuVwU67NWmbVGt82eKhEe6A0rCicY4mDej2ruYf62Hqd2YqsQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



此外，对于wifi和cellular网络只有一条链路网络质量差的场景下，AXP-QUIC 能够实时感知终端各链路网络状况，并及时切换链路，获得预期最优传输效率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9fQdfNudAIvsrZt2Lg29bJIQYCWrYWfUndhL5ejVZ62WVurCDxDbKibbA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



![图片](https://mmbiz.qpic.cn/mmbiz_png/APDZeM2BxAEmibsThfmfroQxGgsKbmm9feAdR1zicnB6PDiaibDSwb1vHopbAhKJZ9ZvtAsF2He0my0G0bp5rNDD5Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

原文作者： 腾讯云音视频

原文链接：https://mp.weixin.qq.com/s/DFxC-c9L2DFc0Cv-cne7fA