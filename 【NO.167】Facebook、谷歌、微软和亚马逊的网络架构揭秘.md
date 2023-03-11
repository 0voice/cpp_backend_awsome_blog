# 【NO.167】Facebook、谷歌、微软和亚马逊的网络架构揭秘

## **0. 前言**

本文主要讲一下国外的互联网巨头的骨干网，每家公司的网络都有独特设计，其中 Facebook 和 Google 的网络主要是服务自身的产品和广大互联网用户，Amazon 和 Microsoft 在云服务的业务相对多些。

Facebook、Google、Microsoft 的网络在公开的论文都有比较详细的描述，而 Amazon 的底层网络相对的公开资料不多，在即刻构建 AWS 技术峰会 2019，AWS 的架构师分享了 AWS 的底层网络（第 3 章节或者参考文献），还是很值得学习。

## **1. Facebook Network**

目前 Facebook 公司系列产品有月活 30+亿用户，他们需要一个服务不间断、随时能访问的网站。为了实现这个目标， Facebook 在后端部署了很多先进的子系统和基础设施 ，可扩展、高性能网络就是其中之 一。

### 1.1 Facebook 全球网络概述

#### **1.1.1 概述**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2kMWaibw2qms6BCYz1HQohdRsqOjUVo1iaTTYEibQEqnvkrjM8bjDamKag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

Facebook 的网络本身就是一个大型分布式系统，针对不同任务划分成不同层次并采用不同的技术：

- 边缘（edge）
- 骨干（backbone）
- 数据中心（data centers）

遍布在用户密集区的数量庞大的**PoP/LB/cache**通过**骨干网**作为偏远地区、低成本、数量可控的超大型**数据中心**的延伸。

#### **1.1.2 Facebooke 网络流量模型**

从流量模型看，Facebook 分为两种类型。

1、外部流量：到互联网的流量（Machine-to-User）。

2、内部流量：数据中心内部的流量（Machine-to-Machine）。

其中，Facebook 数据中心内部的流量要比到互联网的流量大几个数量级，如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2zvic8wvxvMEfETFd16Xdp2fY0NOsrAjEhH55eBzTx1GdWoX5rxjvPHw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.1.3 解决方案**

机器对机器的流量通常会大量爆发，这可能会干扰并影响正常的用户流量，从而影网络可靠性目标。Facebook 将跨数据中心与面向 Internet 的流量分离到不同的网络中，并分别进行优化。

Facebook 设计了连接数据中心的网络**Express Backbone** (EBB)。在边缘互联网出口则推出**Edge Fabric** 架构。

### 1.2 Facebook 骨干网 EBB（Express Backbone）

#### **1.2.1 设计理念**

- 快速演进、模块化、便于部署
- 避免分布式流量工程（基于 RSVP-TE 带宽控制）的问题，例如带宽利用率低，路由收敛速度慢。
- 在网络的边缘利用 MPLS Segment Routing 保证网络的精确性。

#### **1.2.2 EBB 架构**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2aSyIwhvP7GxibEuOs6Q8oK54tcCjkHfJwtAvz1PAPickLTbUgCjUU9Ag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

- BGP 注入器：集中式的 BGP 注入路由的控制方式。
- sFlow 收集器：采集设备的状态传递给流量工程控制器。
- Open / R：运行在网络设备上，提供 IGP 和消息传递功能。
- LSP 代理（agent）：运行在网络设备上，代表中央控制器与设备转发表对接。

#### **1.2.3 流量工程控制平面（Traffic Engineering Controller）**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2xoeT6eRPr4taNdYdTwP34ePeGJdaX1Q2oe12cmj6aGZvu9eCz1pialA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**NetworkStateSnapshot module**：网络状态快照模块，负责构建活动的网络状态和流量矩阵**PathAllocation module**：路径分配模块，负责基于活动流量矩阵并满足某些最优性标准来计算抽象源路由。**Drivermodule**：驱动程序模块，负责将路径分配模块计算出的源路由以 MPLS 段路由的形式推送到网络设备。

### 1.3 Edge Fabric

2017 年 Sigcomm 大会，Facebook 推出了面向互联网出口的边缘网络架构**Edge Fabric**。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2uMGVR0y4AAbtwodibPQcy7bHkklgRv1NdlQxgqgFyFALccxKLZryfNQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

A PoP has Peering Routers, Aggregation SWitches, and servers. A private WAN connects to datacenters and other PoPs。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2ovvAficqPH3HNswLWKxvcF6Jyibc2O1FOic3fs0NYCsvw6Uiau1LEQwnlQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)Edge Fabric组件

- Edge Fabric 有一个 SDN/BGP 控制器。
- SDN 控制器采用多重方式搜集网络信息，如 BMP 采集器、流量采集器等。
- 控制器基于实时流量等相关信息，来产生针对某个 Prefix 的最优下一跳，指导不同 Prefix 在多个设备之间负载均衡。
- 控制器和每个 Peering Router 建立另一个 BGP 控制 session，每隔特定时间用来改写 Prefix，通过调整 Local Preference 来改变下一跳地址。
- BGP 不能感知网络质量，Edge Server 对特定流量做 eBPF 标记 DSCP，并动态随机选一小部分流量来测量主用和备用 BGP 路径的端到端性能。调度发生在 PR 上，出向拥塞的 PR 上做 SR Tunnel 重定向到非拥塞的 PR 上，如下图所示.

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2Z4jBbTQCian52CJ1vJNxeTXv15CnlwVc7uV6k8AhCPsmJ7IHGOAibMQQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)img

由此看见 Edge Fabric 有一些限制：

- SDN 控制器单点控制 PR 设备。如果 PR 设备已经 Overload，需要通过 PBR 和 ISIS SR Tunnel 转移到另一个没有拥塞的 PR，流量路径不够全局优化。
- 控制器只能通过 Prefix 来控制流量，但是同一个 prefix，可能承载视频和 Voice 流量，带宽和时延要求不同，Edge Fabric 没有 Espresso 那么灵活。

### 1.4 总结

Facebook 在骨干网、边缘网络都是使用 BGP 路由协议进行分布式控制，控制通道简单，避免多协议导致的复杂性，而对于流量工程采用集中的处理。

## **2. Google Network**

### 2.1 Google 全球网络概述

Google 全球有 30+个数据中心，100+多个 POP 站点，同时在不同运营商网络中有很多 Cache 站点。信息从 Google 数据中心信息大致经过 Data Center-POP-Cache 三级网络发送到最终用户。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2t5sM22eFDpu8BL21p7R2q9NtZLib2TIdiaWpFxOsx0rPRwGtibyW6E9xQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)Google Cloud Netwok

Google 的广域网实际上分为 B2 全球骨干网和 B4 数据中心互联网，边缘网络是 Espresso，数据中心内部则是 Jupiter，如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2AiaxSv4dcljN9VDDYtk1cMo8bg6JibhPtpYSH2qCxkKrw6Yr6fb86tUA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)Google网络构成

**B2**：面向用户的骨干网，用于连接 DC、CDN、POP、ISPs 等。B2 主要承载了面向用户的流量，和少部分内部流量（10%），带宽昂贵，整体可用性要求很高，利用率在 30%~40%之间。B2 采用商用路由器设备，并且运行 MPLS RSVP-TE 进行流量工程调节。

**B4**：数据中心内部数据交换的网络，网络节点数量可控，带宽庞大，承载的 Google 数据中心间的大部分流量。B4 承载的业务容错能力强，带宽廉价，整体利用率超过 90%。使用自研交换机设备，为 Google SDN 等新技术的试验田。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2bDIZoPicOZNTUb3Q4mgGUcd4TlX6z6XLf2z2MGb7E2EjSpwtM4LUZQw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)B4和B2架构

**Espresso**：边缘网络或者互联网出口网络，将 SDN 扩展到 Google 网络的对等边缘，连接到全球其他网络，使得 Google 根据网络连接实时性的测量动态智能化地为个人用户提供服务。

### 2.2 B2

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2sYkicbEibboCqrwqjX38DGaCm99lK55QaRC7UO4Z2Z0A56bebsTTaZfw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)B2物理架构

- PR：Peering Router，对等路由器，类似 PE 设备，主要是其他运营商网络进行对接。
- BR：Backbone Router，骨干网路由器，类似 P 设备。
- LSR：Label Switch Router，标签交互路由器。
- DR：Datacenter Route：数据中心路由器。

### 2.3 B4

B4 是业界第一个成功商用的数据中心互联的 SDN 网络。

- 交换机硬件是 Google 定制的，负责转发流量，不运行复杂的控制软件。
- OpenFlow Controller (OFC) 根据网络控制应用的指令和交换机事件，维护网络状态。
- Central TE Server 是整个网络逻辑上的中心控制器。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2CAqE9lQGXkvWqh3EwyTH1G96mCr8dfib83pDVez2XaytYzuvibEp4YBg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)B4的架构

Central TE (Traffic Engineering) Server：进行流量工程。

Network Control Server (NCS)：数据中心（Site）的控制器，其上运行着 OpenFlow Controller (OFC) 集群，使用 Paxos 协议选出一个 master，其他都是热备。

交换机（switch）：运行着 OpenFlow Agent (OFA)，接受 OFC 的指令并将 TE 规则写到硬件 flow-table 里。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2iaP59Qibz7wCMicqEHEVVfsiboyEPfR3DoMoYm8dmiayQHh0I7Uiaag44lJQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)流量工程

注：网上有很多介绍 B4 的文章，本文从略。

### 2.4 Espresso

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2W2Bq3uPUcOzyKbTyX2olpLps4hBVnjDdsPWtlq4jFnvGNesOvlMj3A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

为了更好进行流量调度，Espresso 引入了全球 TE 控制器和本地控制器（Location Controller）来指导主机（host）发出流量选择更好的外部 Peering 路由器/链路，进行 per flow Host 到 Peer 的控制，并且解耦了传统 Peering 路由器，演进为 Peering Fabric 和服务器集群（提供反向 Web 代理）。

控制和转发流程：

- 外部系统请求进入 Espresso Metro，在 Peering Fabric 上被封装成 GRE，送到负载均衡和反向 web 代理主机处理。如果可以返回高速缓存上的内容以供用户访问，则该数据包将直接从此处发回。如果 CDN 没有缓存，就发送报文通过 B2 去访问远端数据中心。
- 主机把实时带宽需求发送给全局控制器（Global Controller）。
- 全局控制器根据搜集到的全球 Internet Prefix 情况，Service 类型和带宽需求来计算调整不同应用采用不同的 Peering 路由器和端口进行转发，实现全局出向负载均衡。
- 全局控制器通知本地控制器来对 host 进行转发表更改。同时在 Peering Fabric 交换机上也配置相应的 MPLSoGRE 解封装。
- 数据报文从主机出发，根据全局控制器指定的策略，首先找到 GRE 的目的地址，Peering Fabric 收到报文之后，解除 GRE 报文头。根据预先分配给不同外部 Peering 的 MPLS 标签进行转发。

## **3. Amazon Global Network**

注:本章节主要是参考**即刻构建 AWS 技术峰会 2019**的介绍，详细文档和视频见参考文献。

### 3.1 AWS 全球网络概述

#### **3.1.1 概述**

AWS 是一家全球公有云提供商，它拥有一个全球基础设施网络，以运行和管理其支持全球客户的众多不断增长的云服务。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2pZCId0OWgibWKfFOQqJBKGI7fYmibicImPYlzZvnicnn34PJ8HUf81jRNQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

构成 AWS Global Infrastructure 的组件有：

1. Availability Zones (AZs)
2. Regions
3. Edge PoPs

- Regional Edge Caches

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2IcfQHKEdmBvYWQn8LmuRNEjl5eQ7fCwB9e2dekmavh8lGUhHhwJA5w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **3.1.2 设计理念**

- 安全性：安全是云网络的生命线。
- 可用性：需要保证当某条线路出现故障的时候，不会影响整个网络的可用性。
- 故障强隔离：当网络发生故障的时候，尽量把故障限制在某个区域内。
- 蜂窝架构：一个个网络模块构成的蜂窝式网络架构。
- 规模：支撑上百万客户的应用网络需求。
- 性能：对网络的吞吐量、延迟要求较高。

#### **3.1.3 网络通信案例解析**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw29XOZkehKs6Mhs4f7FxctvuiaO8GbRyxYh3peFXRobB5mjW9PIZSIcpw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，一个猫咪要去 AWS 的服务器中获取一张图片，流量首先通过 Internet 进入到 AWS Region，Region 包括 AZ，AZ 中有 VPC，在 VPC 中有 Server，Server 上面有图片，这是一种比较简单的抽象流程，但是如果把网络剥开一层再去看，其实会变得更加复杂，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw28dT6gCwxqEX7p3m5iavfmn4U1p90QZUPOcd03JCKHPuicVv0S7ZDSrkw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到包含了更多的 AWS 网络基础设施，流量首先进入 Edge PoP，这个也是 CDN 的站点，流量进来之后到骨干网络 Backbone，然后再进入 AWS Region，经过 Transit Center 进入 AZ。

### 3.2 AWS 全球网络的 Region 架构

#### **3.2.1 Availability Zone**

AWS Region 是由 Availability Zone 组成。可用区(Availability Zones)实质上是 AWS 的物理数据中心。在 VPC 中创建的计算资源、存储资源、网络资源和数据库资源都是托管在 AWS 的物理数据中心。

每个 AZ 至少会有一个位于同一区域内的其他 AZ，通常是一个城市，他们之间由高弹性和极低延迟的专用光纤连接相连。但是，每个 AZ 将使用单独的电源和网络连接，这使得 AZ 之间相互隔离，以便在单个 AZ 发生故障时最大限度地减少对其他 AZ 的影响。

每个 Region 有两个 Transit Center，每个 Transit Center 和下面的每个 Datacenter 都有网络互联，同样 Datacenter 之间也有网络互联，这样可以确保 AWS 网络的可用性，部分网络基础设施故障也不会影响整个 Region 的可用性。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2GsGq48ZP8EXOwx3PNCnldwmcyic6gAamqUzdz4smKgX0BVmjVLD93vw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **3.2.2 AWS Regon 数据中心构造**

AWS 采用蜂窝式的网络架构。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2svg49lYOoMqstY6dGCvu2KJerlibdREQEUmKFglmOFGOYwueFSQ3icBg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在图中间都是一个个小的模块，每个模块都有不同的一些功能，如 Access Cell 主要做主机的网络接入，Core Edge Cell 联通着 Transit Centers，进而把网络流量送进 AWS backbone。

每个 Cell 都肩负着不同的功能，Cell 和 Cell 之间都进行互联，在每一层，都可以通过平行扩展 Cell 来扩展整个网络的承载量，达到一个可伸展的网络。

每个 Cell 是一个单核路由器，端口比较少，可以控制故障域，转发架构更简单。

### 3.3 AWS 全球骨干网（Global Backbone）

AWS Direct Connect、互联网连接、区域到区域传播和 Amazon CloudFront 到 AWS 服务的连接都是需要 AWS 骨干网。

和 Region 相似，全球骨干网也是采用了蜂窝式的一个网络架构，中间是大量的光纤连接，外层是负责一些网络功能的 Cell。

Transit Center Cell 用来连接 Region 内部的数据中心，Edge Pop Cell 用来连接 PoP 节点，Backbone Cell 用来连接远端的 PoP 节点进而连接到远端 Region 的数据中心。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw290vfu6ZdywLgGr5ibTdfic0nD6OsWibNDmoz5V0v3TrakPWOjZ6EqtspQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 3.4 AWS Edge PoP

AWS Edge PoP 是部署在全球主要城市和人口稠密地区的 AWS 站点。它们远远超过可用区域的数量。

AWS Edge PoP 对外就是连接的一张张 ISP 的网络。运营商接入 AWS 的骨干网络两个地方，一个是 Edge PoPs，另外一个 AWS 区域的网络中转中心（Transit Centers）。

Edge PoP 很大的一个作用就是对外扩充 AWS 的网络，同一个 Edge PoP 可以和运营商进行多次互联，获得至外网网络最优的互联。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2nQmOpI3fsbLsJyRc195Y2mnG8eVexY9HHHibKqfvQbQAvkBvibBN3F0Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

边缘站点也是采用了蜂窝式的架构，Backbone Cell 连接 AWS 骨干网络，External Internet Cell 连接外部的 Internet 网络，同时还包括一些 AWS Edge 服务网的一些 Cell，如连接 CloudFront、Route 53、Direct Connect 和 AWS Shield，这些服务都存在于 AWS Edge PoPs 中。

### 3.5 总结

可以看到 AWS 在网络的各部分都采用了蜂窝式的架构，让这个网络的扩展性大大提升。并且通过采用主动式数据信道监控，从 AWS 服务日志采集互联网性能数据，以及互联网流量工程管理来达到互联网边缘的监控与自我修复。

## **4. Microsoft Network**

### 4.1 Microsoft Network 概述

Microsoft 在布局云计算取得很大的成功，网络的布局功不可没、其中 Azure 拥有超过 165,000 英里的私有光纤，跨越全球 60 多个区域和 170 多个网络 PoP。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2WeE0CorbHKJKay6PpqC27HPFf8KibK1vicft0aRjKbOpeCvMpVtfUpUA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)Azure网络

### 4.2 Microsoft SWAN 网络

微软 SWAN 广域网 DCI 控制器也是一个典型的 SDN 网络，从最早的静态单层 MPLS Label 构造的端到端隧道，到最新的基于 BGP-TE SR 的全球 DCI 互联解决方案，可以实现 95%的跨数据中心链路利用率。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw2tNq9FI3tRicwkUZK3UhCSWd5ZRaqQobsBbmdiaTCWM55EwwIHr1KBLHg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)Microsoft SWAN架构

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvauHZ22Jq0USicIzHuX3bflw27ZZnaIplQpkFx1R4hlpyjC8LAddLzZOdboyIcAx9Fsg26FSQmOpRVg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)Microsoft SWAN控制面

- RSVP-TE/SR-TE。
- 集中式 TE 资源分配算法。
- 服务间通过资源分配模块协作。
- 每个 Host 上都有代理，负责带宽请求和限流。

**注：具体细节可以见参考文献的论文**

## **5. 数据中心网络**

再补充一下 Google 和 Facebook 的数据中心网络设计。

### 5.1 Facebook 的 f4 和 f6 中文翻译

http://arthurchiao.art/blog/facebook-f4-data-center-fabric-zh/

http://arthurchiao.art/blog/facebook-f16-minipack-zh/

### 5.2 Google Jupiter 中文解读

http://zeepen.com/2015/12/31/20151231-dive-into-google-data-center-networks/ 原文 http://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p183.pdf

## 6. 总结

从**技术**的角度看，互联网公司的网络演进是一个 SDN 的过程。SDN 是一个网络标准化的过程，是一个通信系统互联网化的状态，贯穿着网络的控制面、转发面、管理面。

从**运营**的角度看，互联网公司的网络演进是从“所有流量 all in one” 到互联网思路构建网络，网络具有分布式、模块化、低耦合等特点。

从**布局**的角度看，互联网公司的网络布局也是技术实力全球化扩张的缩影。也希望中国的互联网公司也能不断的扩张边界，进入全球化的食物链的顶端。

**参考文献**

**Facebook 网络**

https://engineering.fb.com/2017/05/01/data-center-engineering/building-express-backbone-facebook-s-new-long-haul-network/

**Google 网络**

https://cseweb.ucsd.edu/~vahdat/papers/b4-sigcomm13.pdf

**AWS 网络**

揭秘 AWS 底层网络是如何构成的

https://blog.51cto.com/14929722/2533206

AWS 底层网络揭秘

https://www.bilibili.com/video/av99088073/

**Microsoft 网络**

https://www2.cs.duke.edu/courses/cps296.4/compsci590.4/fall14/Papers/SWAN.pdf

**Facebook 的 F4，F16 网络架构**

http://arthurchiao.art/blog/facebook-f4-data-center-fabric-zh/

http://arthurchiao.art/blog/facebook-f16-minipack-zh/

**Google Jupiter 实现**

中文解读

http://zeepen.com/2015/12/31/20151231-dive-into-google-data-center-networks/

原文

http://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p183.pdf

原文作者：云网漫步

原文链接：https://mp.weixin.qq.com/s/MPBk9wdYsE48H7OXWAd5bA