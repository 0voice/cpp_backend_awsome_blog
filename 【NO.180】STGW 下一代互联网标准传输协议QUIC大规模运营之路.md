# 【NO.180】STGW 下一代互联网标准传输协议QUIC大规模运营之路

## **0.前言**

QUIC 作为互联网下一代标准传输协议，能够明显提升业务访问速度，提升弱网请求成功率以及改善网络变化场景下的平滑体验。

STGW 作为公司级的 7 层接入网关以及腾讯云 CLB（负载均衡器）的底层支撑框架，每天都为公司内部业务和腾讯云外部客户提供数万亿次的请求服务，对请求处理的性能、传输效率、运营的可靠性都有非常严苛的要求。

本文主要介绍 STGW 大规模运营 QUIC 过程中的一些经验和开发工作。

## 1.**QUIC 简介**

### 1.1 QUIC 的诞生和发展

> 在 QUIC 诞生之前，HTTP 协议经历了几次重要的升级：
>
> HTTP1.0 -> HTTP1.1：增加了长连接支持，大大提升了长连接场景下的性能。
>
> HTTP -> HTTPS：增加安全特性，对请求的性能综合来看会有一定的影响。
>
> HTTP1.1 -> HTTP2：主要特性是多路复用与头部压缩，提高了单连接的并发能力。

这些重要变化都是围绕安全与性能展开，对 HTTP 协议的应用和发展起到了很重要的作用。但是，它没有绕开内核 TCP 的限制，导致其协议的发展终究存在瓶颈。

GOOGLE 在引领业界从 HTTP1.1 迈向 HTTP2（GOOGLE SPDY 协议的标准版）后，再一次走在了前头，在 2012 年提出了实验性的 QUIC 协议，首次使用 UDP 重构了 TLS 和 TCP 协议。QUIC 协议不仅仅只应用于 HTTP，QUIC 在设计时除了考虑 HTTP 外，更是设计作为一个通用的传输层协议。在安全性上，GOOGLE 设计了 QUIC 加密协议作为握手协议以解决 QUIC 协议上的安全问题。一般来说，QUIC 握手协议+QUIC 传输层+HTTP2 就是我们常说的 GQUIC（这里指 web 部分）。GQUIC 协议的版本不断演化，从 Q46 开始，GQUIC 协议也不断向 IETF QUIC 和 HTTP3 靠拢。

2015 年，QUIC 的网络草案被正式提交至互联网工程任务组，这意味着新的 QUIC 协议标准将要诞生。在标准 QUIC 协议起草过程中，QUIC 协议上的标准 HTTP 协议作为 HTTP3 也同时被起草。而作为 QUIC 的标准握手协议，IETF 将 TLS1.3 应用其中。TLS1.3+QUIC+HTTP3，这就是我们常说的 IETF QUIC（这里指 web 部分）。截止目前，QUIC 标准的草案已经更新到 34 版，仍没形成正式的 RFC。但是，QUIC 已进入 IETF 最后征求意见，预计标准 QUIC/HTTP3 协议会很快问世。

### 1.2 QUIC 的关键特性

关于 QUIC 的原理，相关介绍的文章很多，这里再列举一下 QUIC 的重要特性。这些特性是 QUIC 得以被广泛应用的关键。不同业务也可以根据业务特点利用 QUIC 的特性去做一些优化。同时，这些特性也是我们去提供 QUIC 服务的切入点。

1. 低连接延时：QUIC 由于基于 UDP，无需 TCP 连接，在最好情况下，短连接下 QUIC 可以做到 0RTT 开启数据传输。而基于 TCP 的 HTTPS，即使在最好的 TLS1.3 的 early data 下仍然需要 1RTT 开启数据传输。而对于目前线上常见的 TLS1.2 完全握手的情况，则需要 3RTT 开启数据传输。对于 RTT 敏感的业务，QUIC 可以有效的降低连接建立延迟。
2. 可自定义的拥塞控制：QUIC 的传输控制不再依赖内核的拥塞控制算法，而是实现在应用层上，这意味着我们根据不同的业务场景，实现和配置不同的拥塞控制算法以及参数。GOOGLE 提出的 BBR 拥塞控制算法与 CUBIC 是思路完全不一样的算法，在弱网和一定丢包场景，BBR 比 CUBIC 更不敏感，性能也更好。在 QUIC 下我们可以根据业务随意指定拥塞控制算法和参数，甚至同一个业务的不同连接也可以使用不同的拥塞控制算法。
3. 无队头阻塞：虽然 HTTP2 实现了多路复用，但是因为其基于面向字节流的 TCP，因此一旦丢包，将会影响多路复用下的所有请求流。QUIC 基于 UDP，在设计上就解决了队头阻塞问题。同时，IETF 设计了 QPACK 编码替换 HPACK 编码，在一定程度上也减轻了 HTTP 请求头的队头阻塞问题。无队头阻塞使得 QUIC 相比 TCP 在弱网和一定丢包环境上有更强大的性能。
4. 连接迁移：当用户的地址发生变化时，如 WIFI 切换到 4G 场景，基于 TCP 的 HTTP 协议无法保持连接的存活。QUIC 基于连接 ID 唯一识别连接。当源地址发生改变时，QUIC 仍然可以保证连接存活和数据正常收发。

## 2.**QUIC 协议栈的选择**

对于协议的实现，STGW 与 CDN 业务团队在 LEGO（STGW 与 CDN 自研的高性能转发框架）上实现过完整的 HTTP2 协议。同时，STGW 也在业界最早实现了 TLS 异步代理计算的方案。对于 HTTP1.1/2 和 TLS 协议有不少工程和优化经验。QUIC 协议栈的自研目前也在按计划展开，但尚不成熟。

本文就基于开源方案，给大家简单介绍一下 QUIC 协议栈的深度定制和优化工作。

关于 QUIC 协议栈的实现，当前功能广泛，协议支持齐全的实现并不多。NGINX 官方目前实现了一个实验版本，但是该实现很多问题没解决，同时，其仅支持 IETF 最新的 DRAFT，甚至连一个完整的拥塞控制算法也没有实现。CLOUDFLARE 的 QUIC 基于 RUST 实现，性能从公开数据来看并不强。

其它的很多实现诸如 MSQUIC, NGHTTP3 等都只支持 IETF QUIC，并不支持 GQUIC。

GOOGLE 是 QUIC 协议的开创者，其基于 CHROME 的 QUIC 协议栈实现最早，功能最齐，实现上也最为标准。

不论是哪种 QUIC 协议栈，其接入都需要我们对 QUIC 的基本特性和概念有较深的理解。比如常见的连接，流，QUIC 连接 ID，QUIC 定时器，统一的调度器等等。这些概念与 QUIC 协议的内容息息相关。

下面以 CHROMIUM QUIC 为例的将 QUIC 协议栈与高性能转发框架 NGINX 与 LEGO 融合的架构图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxGiaMqAupt74SpwpWHicrD3ZkGhtvX7gGn0TTsRkNykYBAzfibJBC4elmg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)CHROMIUM QUIC接入高性能框架NGINX/LEGO

## **3.STGW 的工作**

STGW 作为公司级的 7 层接入网关以及腾讯云 CLB（负载均衡器）的底层支撑框架，为公司内部业务和腾讯云外部客户提供数万亿次请求的服务，对请求处理的性能、传输效率、运营质量的高可靠性都有非常严苟的要求。

为此，我们对 QUIC 协议栈做了大量优化和深度定制，以满足大规模运营和业务需求。主要的工作有：

1. 单机与传输性能优化

2. 1. QUIC 协议单机性能/成本优化：QUIC 将协议栈搬到应用层来，从目前一些公开的实现看，性能比 TCP 差不少，优化 QUIC 协议的性能是大规模推广 QUIC 协议很重要的一环。
   2. 与高性能转发框架融合：当前开源的 QUIC 协议栈的实现仅仅提供单核支持，并仅提供简单的 demo。要想大规模应用，需要将 QUIC 协议栈接入到我们使用的高性能网络转发框架中来，如 NGINX,LEGO 等。
   3. 传输性能，拥塞控制定制化：可以允许不同业务根据业务特性选择不同的拥塞控制算法。
   4. 做到安全高比例的 0RTT，以降低业务的连接延迟。

3. 功能特性的定制和增强

4. 1. 如何让 QUIC 连接迁移从理论走向应用：QUIC 的连接 ID 是 QUIC 协议的特性，但是实际应用中，要做到连接迁移并不容易，需要充分考虑 QUIC 包的各个路径。即使在同一台机器上，也需要正确转发到对应的核上去。
   2. QUIC 私有协议的支持：QUIC 不仅仅用于 HTTP，作为通用的传输层协议，除了支持 GQUIC,IETF HTTP3 外，QUIC 的私有协议也需要我们提供给用户。
   3. QUIC 定制化 SDK：除了高性能 QUIC 服务端外，要想使用 QUIC 需要客户端 SDK 的支持。对此我们也开发了 QUIC 的 SDK，并针对不同的场景做了定制化。
   4. 满足业务各种定制化需求：如有些业务需要 QUIC 明文传输，一些业务需要 QUIC 回源等。

5. 高可用运营

6. 1. 日常变更与平滑升级：在配置频繁变更和模块升级时，我们需要做到对 QUIC 连接无损。
   2. 抓包分析工具：分析定位为更方便。
   3. 统计监控：QUIC 的关键统计指标，需做到可视化运营。

我们围绕着这些问题展开了 QUIC 的相关工作，力求将 QUIC 特性，QUIC 运营，QUIC 性能，QUIC 定制化需求等做到最好。

## 4.**QUIC 处理性能优化**

QUIC 协议基于 UDP 将 TCP 的特性从内核移到了应用层，从当前各种 QUIC 实现来看，性能相比 TCP 差不少。TCP 长期以来使用非常广泛，这也使得其从协议栈到网卡已经经过了非常多的优化，与之相比，UDP 的优化则少了很多。除了内核和硬件外，QUIC 协议的性能也与实现有关，不同的实现版本可能也会有很大的差别。

我们对 QUIC 的性能利用火焰图等各种工具进行了详细分析，找出了一些影响 QUIC 性能的关键点：

1. 密码相关算法的开销：对于小包来说，RSA 的计算占比很高，对于大包来说，对称加解密也会占到 15%左右的比例。

2. UDP 收发包的开销：特别是对于大文件下载来说，sendmsg 占比很高，可以达到 35%-40%以上。

3. 协议栈开销：主要受协议栈实现，如 ACK 的处理，MTU 探测和发包大小，内存管理和拷贝等。

   我们基于影响 QUIC 性能的关键点进行了优化。

### 4.1 QUIC 的 RSA 硬件 OFFLOAD

在小文件请求场景中，RSA 的计算在 QUIC 请求同 HTTPS 一样，仍然是最消耗 CPU 的开销。RSA 在 HTTPS 请求可以利用硬件 offload，在 QUIC 握手过程中，RSA 同样可以利用硬件进行 offload。

使用硬件做 RSA 卸载一个很重要原因是，CPU 计算 RSA 性能较差，而专门做加解密的加速卡性能则很强。一般来说，单块 RSA 加解密卡的成本差不多是一台服务器的 5%-7%，但是其对 RSA 签名的操作性能是服务器的 2-3 倍左右，一台机器插入 2 块卡就可以带来 5 倍的 RSA 性能提升。

将 QUIC 的 RSA 计算进行硬件卸载在不同的 QUIC 协议栈上方法并不相同，下面介绍一种 RSA HOOK + ASYNC JOB 通用的 RSA 卸载方案。其特点是代码侵入性小，不需要额外修改太多 quic 协议栈或者 openssl 的代码。

Openssl1.1.0 之后，开始支持 Async Job。Async job 机制利用类协程方式切换上下文方式实现异步任务，并且提供了一个通知机制通知异步任务的返回结果。

Async Job 里的 2 个重要函数是：

> async_fibre_makecontext
>
> async_fibre_swapcontext

它们利用 setjmp，longjmp，setcontext，makecontext 这些调用，保存和切换当前上下文，从而达到状态保留和代码跳转的能力。

使用 RSA callback 将握手过程中的 RSA 进行拦截，并在 RSA 的 HOOK 函数中本地或者远程向加速卡请求 RSA 操作。同时使用 Async job 方式将同步方式异步化，从而实现 RSA 操作的异步卸载。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwx5OJYmTec2NQibibjU6tl3MuHRPch4ibVibMylWanO38Uh0IAexFvoqRjcA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在 QUIC TLS1.3 上 RSA HOOK + Async Job 进行 RSA offload。

QUIC 在进行了 RSA 的硬件 OFFLOAD 后，对于小包短连接，性能得到了很大的提升。

以 CHROMIUM QUIC 为例，在 1RTT 场景，QUIC 在使用了 RSA OFFLOAD 后，性能为原来的 256%；0RTT 场景，QUIC 在使用了 RSA OFFLOAD 后，性能为原来的 205%。在 QUIC 协议栈开销更小的实现上，这个性能提升会更加明显。

### 4.2 QUIC 发包的 GSO 优化

在大文件下载中，QUIC 发包的逻辑占比很大，通常在 35%-40%以上。因此优化发包逻辑可以提升大文件传输的性能。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxKdF7e0EY2DLr1hYKFCbW3JVCFTtPANkJtMjvt50CW8sqcbPHdiczDuA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)QUIC大文件请求火焰图

GSO(Generic Segmentation Offload)在内核 4.8 之后开始支持 UDP，其原理是让数据跨过 IP 层，链路层，在数据离开协议栈，进入网卡驱动前进行分段，不论是 TCP 还是 UDP，都是分段(每个包都附加 TCP/UDP 头部)，这样，当一个段丢失，不需要发送整个 TCP/UDP 报文。同时，路径上的 CPU 消耗也会减少。

若网卡硬件本身支持 UDP 分段，则称为 GSO OFFLOAD，其将分段工作放在网卡硬件上做，可以进一步节省 CPU。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwxz3pue6TaVib8YPeQpfStexliaTsfERae7mpNCUlhOKCW721UR5dhxVtw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)GSO原理示意图

QUIC 协议在实现上，一般为了不进行 MTU 分片，通常会在发包前就将发送数据进行分段，从而无需再进行 MTU 分片。对于大包来说，QUIC 会将每个包控制在 1400 字节左右，然后通过 sendmsg 发送出去。大文件发送场景，这种性能是很低的。如果在 sendmsg 时发送大包不做分段，然后利用内核 GSO 延迟分段，会减少路径的 CPU 消耗。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwx5d0kRjv9nTHaXDkKxWtqUciaFictdCskdz16TNIfZicYQQuXOQkF5uEUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)使用GSO发不同大小包时的吞吐

从表中可以看出，使用 GSO 连续 20 个包进行 sendmsg 发送，相比于 1400 个包单独发送，性能提升了 2-3 倍。

在实际 QUIC 场景中，发包并不是 QUIC 的全部逻辑。同时，并不是每次发包正好都可以凑齐 20 个连续包。我们对大文件下使用 GSO 进行 QUIC 压力测试，相同 CPU 使用情况下，吞吐提高了大约在 15%-20%。

### 4.3 **QUIC 协议栈的优化**

QUIC 协议栈的性能与 QUIC 协议栈实现有关。对于一些常见的协议栈实现，其优化空间主要有：

1. 一些实现如 CHROMIUM 在 0RTT 和 1RTT 请求中分别多了一次 RSA 计算，这个多余的 RSA 计算是可以去掉的。优化后，0RTT 和 1RTT 的 RSA 计算分别为 0 次和 1 次。
2. 大文件下载中服务端会收到并处理大量的 ACK。在 ACK 处理上，并不需要接收一个处理一个。可以将一轮中所有的 ACK 解析后再同时进行处理。
3. 一次发包大小尽可能接近 MTU，QUIC 协议本身也提供了 MTU 探测的特性。
4. 尽可能减少协议栈的内存拷贝。

下图是小文件 0RTT 请求场景中，协议栈优化前和优化后的性能对比：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxQ64bRvaMOhYqTUrSlXSNcLryvJnEbtsIib6hsSjtia9ZQVQqKunbbiaMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)优化前

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxSFicTVxv7TfQIXYE10VWSibgOVsBEEibEDicfcmY0soDWd3wL9icJ4eWAaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

优化后

**4.4 QUIC 性能优化小结**

目前，我们将 QUIC 协议栈无缝接入到了高性能转发模块 NGINX 与 LEGO。在小包请求上，QUIC 的性能开销基本可以达到 HTTPS 的 90%以上。如果 QUIC 使用加速卡做 RSA OFFLOAD，性能甚至比原生的 HTTPS 强。在大包请求上，优化过后的 QUIC CPU 性能可以达到 HTTPS 70%，但在大部分机型中，大文件请求通常都是网卡先到达瓶颈。总的来说，QUIC 目前的性能问题做大规模部署已经不在是大问题。当然，这里面仍然存在优化空间，我们也为此做继续的优化之中。

## 5.**QUIC 的 0RTT 优化**

下图展示了一个 HTTPS 请求与一个 QUIC 请求的对比。可以看出，一个完全握手的 HTTPS 请求，在 HTTP 请求正式发出时经历了 3 个 RTT。而 QUIC 请求可以做到发 HTTP 请求之前的 0RTT 消耗。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwx91RaNRWQic9mCxO4PGDQ4MCwq2icwbDDXCKTX90xa9GyOuhwfjnGmB8g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

为什么 QUIC 可以做到 0RTT 呢？这里分为 QUIC 握手协议和 IETF QUIC 的 TLS1.3 协议。我们以 GQUIC 握手为例，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwx6QsGdNF2Ortdw0wicNVvskEWzwJuhQQL0CJ5G5plMobPcJLDibXcCylQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

用户第一个 QUIC 包时发送一个没有带 server config 的 client hello，这个时候 server 回一个 REJ 的包，包含 server config 等信息。用户后续带上 server config 继续发 client hello，服务端会回 server hello。此时握手建立成功。

QUIC 加密握手基于 DH 的非对称秘钥交换算法进行秘钥交换，Server config 包含服务端的算法与公钥等信息，客户端利用服务端的公钥和自己的公私钥计算协商出连接的对称秘钥。

因此，第一次请求，客户端在没有保存服务端 server config 信息时，需要 1RTT 请求来完成第一次 QUIC 请求。而在后续请求中，客户端可以直接带上之前的 server config 来完成 0RTT 请求。

所以，这里的关键是：如何提升 0RTT 的比例。一种典型的场景就是，同一个用户在第一次 1RTT 请求获取到的 server config 信息，在后续多次请求中，不论路由到哪台 7 层 STGW 服务器，都能够尽可能的处理对应的 server config 信息。对此，我们尝试过很多方案，主要有：

1）4 层通过会话保持将同一个 IP 尽可能转发到同一个 7 层 STGW 服务器。这样的缺点是：1 用户的 ip 可能发生变化，2 四层基于 IP 的会话保持和基于连接 ID 的会话保持冲突，这可能导致 0RTT 提升的同时，连接迁移特性可能无法使用。

2）类似于 HTTPS 的分布式 session cache，同一个集群通过远端模块共享 server config 信息。这需要额外引进新的模块，并且会带来一定的延时。

3）类似于 session ticket，支持分布式无状态的 server config 生成。实际过程中可以根据日期和参数生成多组 SCFG，进一步提高可用性和安全性。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxNDfFofsXqvgdaYsbRVAW93lbxmgCPUwicS7KliaMrRbnx5Nk96wiaFLrg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

关于 0RTT 的优化目前我们做了不少工作，对于一些不敏感的数据传输，我们可以做到 100% 0RTT。

## 6.**QUIC 连接迁移实现**

QUIC 连接迁移是 QUIC 协议一个很重要的特点。QUIC 使用连接 ID 唯一区分连接，以应对用户的网络突然发生变化。一种典型的场景是 4G 与 wifi 之间的切换，之后用户的地址发生变化，原始的客户端 fd 已无法使用。这时只需要在客户端使用 QUIC SDK 重新创建新的 fd，并继续之前连接的发包，即可发出相同连接 ID 的包出去。

用户的 QUIC 包可能经过中间很多路径最后到达实际的业务服务器。我们以典型的腾讯业务走网关的场景分析：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxJqXSKh6Xiap0LK787MSC3tavDvYQC1ba42CZib3qhqaAUt4ypD4aZyqQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

一次 QUIC 请求经过外网后会先到四层 TGW 集群，然后转发给七层 STGW 集群的一台服务器。到达 STGW 服务器后，包会到达一个特定的 worker 进程处理。Worker 进行 QUIC 协议卸载后使用 TCP 或 UDP 转发给具体业务的一个 RS。如果用户的 QUIC 连接在处理过程中突然源地址发生了变化，我们如何继续正确的响应和维持这个 QUIC 连接？另一个场景是：用户的源地址没有发生变化，但是 7 层 STGW 服务器需要做配置变更和升级，这时 QUIC 连接是否可以维持？

### 6.1 四层基于 QUIC 连接 ID 的会话保持

当用户网络地址发生变化时，虽然源地址变化，但是 QUIC 连接 ID 仍可保持一致。包经过中间网络后首先会到 TGW 集群。为了保证用户的地址发生变化时，QUIC 连接得以维持，TGW 集群需要做正确的转发。

TGW 集群对 QUIC 的会话保持需要考虑 GQUIC 和 IETF QUIC 不同的情况。对于 GQUIC（Q043 以下）实现起来较为简单，因为 GQUIC 协议里的连接 ID 由客户产生，并在整个连接保持不变。对于 IETF QUIC，连接 ID 由客户端和服务端协商产生，需要考虑 long header 包和 short header 包等不同的场景。下图为 IETF 连接 ID 的协商过程以及 GQUIC 和 IETF QUIC 不同类型包的抓包分析。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwxc84LTFDJJ5kEZGIr99wkuiat6y2W1fLSvpdoPPytm1DeUgIrriar6RyQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)IETF 下连接 ID 的协商过程

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxYEwrSdHqfLJnQ0Ne7s5MWEfDf5uznkYO7PfF6NcFfHyrjpNG4Fyy8g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)GQUIC包

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxnZq9OdCpLau3elV1J6MHicAu6LzKqE4LOLKwX0MGZFbNmKSdiaKG0bMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxtN5SOzicKicwMy3n2P83CnfLWibPObZByKEHW1mbIiccTD07EPqGyZicr7Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)IETF不同类型包

目前 TGW 集群已经支持 QUIC 的会话保持，基本原理是在同集群不同的 TGW 服务器之间同步 QUIC 连接信息，同时能够区分不同 QUIC 协议，将相同 QUIC 连接 ID 的包转发到相同的 7 层 STGW 服务器去。

### 6.2 七层单机多核的连接迁移

当包到达 STGW 服务器时，由于 STGW 服务器多核转发，此时还需要将 QUIC 包转发到同一个进程(或线程)去处理。当前，7 层网络框架一般使用多核+REUSEPORT 模型来提供高性能转发能力。对于 QUIC 服务，上层不同的进程在同一个 UDP 端口使用 REUSEPORT 监听。LINUX 内核默认基于 4 元组 hash，因此原生情况下，不同源地址的包是无法保证到达同一个进程的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwx9855WHzJnibCTcUeBqjevl0p1tYq6GMpVoIKlfSUFEZVIcqJhVlQj2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

EBPF 在内核 4.19 引入 BPF_PROG_TYPE_SK_REUSEPORT 这个 HOOK，可以策略性的控制一个包到达后发往到哪一个 accept 队列。这使得使用 EBPF 可以实现因为用户源地址变化引起的 QUIC 连接迁移。具体方法是在 EBPF 的 REUSEPORT 钩子处，解析 QUIC 包以及连接 ID，根据 QUIC 连接 ID 将包转发到对应的 worker 去。

### 6.3 配置加载和热升级连接保持

QUIC 连接迁移还有一种典型的场景是配置加载和热升级：当 STGW 服务器进程配置变更或者进行模块升级时，原生 NGINX 对 TCP 是可以保持连接不中断的。但是对于基于 UDP 的 QUIC，在未经过优化的情况下，我们无法在配置变更和模块升级过程中保持包的正常转发。

以 NGINX 的配置和热升级变更为例，NGINX 在配置变更和热升级时，会产生新的一组 worker，同时老的 worker 进入 shutting down 状态，而老的连接状态都在老的 worker 中。此时新老 worker 共用一组 fd。若老的 worker 关闭 fd 监听，则对于老的请求的连接都会超时。若老的 worker 继续监听 fd，则存在新老 worker 惊群读同一个 fd 的问题，这使得任意新老连接的包可能会到任意一个 worker，对新老连接都存在影响。

STGW 作为一个平台，每天的配置变更需求非常多，某些集群甚至达到了几秒一次配置变更。虽然我们实现了动态配置加载可以做到绝大部分场景不需要 reload 程序，但是仍然有少部分程序要 reload。同时，热升级这种场景也是比较常见的。如果配置 reload 或者模块升级就导致存量 QUIC 连接超时或中断，必然会对业务产生很大影响。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxuCdLzQkOhhoAyoiayViaOvujZU0J8L1LZLfuaLkMPXKuzhKScJXCf6oQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)NGINX配置变更和热升级时worker的收包

那么，如何解决这个问题呢？

基于内核的 EBPF 方案可以较好的处理 4G 与 WIFI 切换的场景，但是对于 STGW 服务器配置变更和模块升级的场景，却很难实现。

为此，STGW 使用了基于共享内存的 QUIC 连接迁移方案，使用共享内存管理不同进程的所有连接信息。同时为每个 worker 设定了一个 packet queue 用于接收来自别的进程连接迁移的包的转发。

可以说，目前 STGW 完全支持 4G 与 WIFI 互切的 QUIC 连接迁移场景。同时对于线上大规模运营来说，持续的配置变更和模块升级也不会影响 QUIC 连接的保持。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxdNoAVhtdb0EbIKcCWUZ2OXKNDKEjvDCdYXIibGQN3hZhicNolGT9AQfw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)STGW基于共享内存的连接迁移和连接保持方案

### 6.4 连接迁移的应用场景

一切重连开销很大的场景可以说都是 QUIC 连接迁移的使用场景。

例如游戏，视频以及业务信道传输就是比较典型的场景。当用户在 WIFI 网络切换到 4G 时，使用原始的 TCP 方案，在网络切换后会有一个连接重建过程。一般重连后，业务会有一些初始化操作，这会消耗几个甚至十几个 RTT，现象就是应用的卡顿或者菊花旋转。在使用 QUIC 连接迁移功能后，可以保证在 WIFI 与 4G 网络切换过程中，连接的正确迁移和存活，无需建立新的连接，从而使得业务的流畅度在网络切换时会得到很大提升。。

## 7.**灵活的拥塞算法与 TCP 重定义**

TCP 拥塞控制算法的目的可以简单概括为：充分利用网络带宽、降低网络延时、优化用户体验。然而就目前而言要实现这些目标就难免有权衡和取舍。LINUX 的拥塞控制算法经过很多次迭代，主流都是使用的 CUBIC 算法。在 Linux4.19 内核后，拥塞控制算法从 CUBIC 改为了 BBR。

BBR 算法相比之前拥塞控制算法，进行了非常重大的改变。BBR 通过实时计算带宽和最小 RTT 来决定发送速率 pacing rate 和窗口大小 cwnd。BBR 主要致力于：

1）在有一定丢包率的网络链路上充分利用带宽。

2）降低网络链路上的 buffer 占用率，从而降低延迟。

BBR 完全摒弃丢包作为拥塞控制的直接反馈因素，这也导致其对丢包并不是非常敏感。通过测试我们得出，在模拟一定概率丢包的网络情况下，对 QUIC 大文件的请求，BBR 的下载性能会比 CUBIC 更好。

QUIC 将拥塞控制做在了应用层，这也使得我们能够灵活的选择不同的拥塞控制算法。目前我们在 QUIC 上支持常见的 CUBIC,BBR 等算法，并实现了业务的自主配置，根据不同业务，针对请求的不同 VIP 使用不同的拥塞控制算法。同时，我们也支持针对同一个业务不同的用户的 RTT 动态的选择拥塞控制算法。另外，我们也同 CDN 的拥塞控制算法团队密切合作，以优化拥塞控制算法在不同场景下的业务体验。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41BwxAibXTooLzSyeqslED3SiaPvb9icXgU5MEFrxiatnKxGH4YIo4IzNSCEutw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)业务可配的拥塞控制算法

除了拥塞控制，基于 QUIC 也可以在传输层针对特定的应用场景去不同的定制化。QUIC 将 TCP 的特性带到了应用层，这也使得在传输层上，我们有更多的操作可能性。例如，参照音视频领域一些常见的用法，发送数据时发送冗余数据，在一定丢包情况下，QUIC 传输层可以自动恢复数据，而不需要等待数据包的重传，以降低音视频的卡顿率。这些基于 TCP 是很难做到的。业务如果需要重定义 TCP 的一些功能或特性，来提升业务体验，QUIC 将会有很大的发挥空间。

## 8.**支持 QUIC 私有协议**

STGW 作为 7 层网关，提供通用 WEB 协议卸载和转发。因此，支持 QUIC 的 WEB 协议如 GQUIC,HTTP/3 是我们的基本能力。

但是，如前面所说，QUIC 作为通用传输层协议，不仅仅应用于 WEB，任何私有协议都可以改造到 QUIC 下。使用 QUIC 握手协议之后，客户端就可以根据自己的业务需求，发送 GQUIC,HTTP/3 等 WEB 请求，或者可以发送任意自己的私有协议。

STGW 基于 NGINX 的 STREAM 模块，对其进行深度改造，使得任意私有协议都可以跑在 QUIC 协议之下。这也大大增加了 QUIC 的应用场景。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauGdmAUPqq2d1a1wDD41Bwxp0ibxxI1J8hnw7Goz8685YlKn0micnaQGicswKIwo30ehXdyib6CwvEcPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

所以，不要觉得不是 HTTP 协议就用不了 QUIC。只要你理解了 QUIC 的特性，并且觉得 QUIC 的特性能够优化业务体验，那么，来试试 QUIC 吧。

## 9.**QUIC 定制化 SDK**

由于 QUIC 尚未标准化，当前使用 QUIC 相对来说门槛较高。

客户端方面，由于 Google 将其浏览器以 Chromium 项目开源出来，其网络协议栈 Cronet 成为业界 QUIC 客户端的主要参考对象。但 Cronet 因为 API 支持有限，代码复杂，难以满足个性化需求等，不适合直接用在我们的移动客户端上。同时，QUIC 作为一个还在高速发展的协议，服务端和客户端都在快速迭代，需要保持紧密的跟进。

基于上述痛点以及 QUIC 渐渐流行的趋势，我们提供比 Google QUIC 更定制化的 TQUIC SDK。TQUIC SDK 相比 Cronet，有体积更加轻量，简单易用，支持私有协议，连接迁移等诸多优点。目前，TQUIC SDK 已应用于公司内部多个业务之中。

## 10.**总结**

本文综合介绍了 STGW 在大规模应用 QUIC 协议过程中做的一些优化和成果。当前：

1. 我们将 QUIC 协议栈与高性能网络框架做了深度融合，并支持 QUIC WEB 协议，QUIC 私有协议，带外拥塞控制配置等大部分 QUIC 功能和特性。满足 QUIC 大规模部署与运营。
2. 我们对 QUIC 协议栈 0RTT，1RTT，小包，高带宽等多场景做了大量的性能优化，解决了 QUIC 严重消耗 CPU 资源的几个瓶颈。在小包请求上性能基本可以达到 HTTPS 的 90%。
3. 针对 RTT 敏感的短连接业务，我们大大提升了 0RTT 的比例，某些场景可以做到 100% 0RTT。
4. 更全面的连接迁移，解决了 4 层，7 层，多集群、多机器、多进程以及进程重启、重加载，模块升级等各种场景下的连接迁移问题。
5. 我们提供了定制化的 QUIC SDK，以用于客户端满足定制化 QUIC 的各种特性。

QUIC 仍然有很多特性需要充分挖掘，如 QUIC 本身基于 UDP 没有队头阻塞特性以及 QPACK 编码在 HTTP/3 的 HTTP 头部上对队头阻塞的优化等。这些特性在弱网环境下对业务都会有较好的性能提升和卡顿率降低，特别是多路复用场景。目前我们也在结合业务积累更多的实际数据，并期望在这块能够有更多的优化。

QUIC 以及相关的 HTTP/3 等协议即将形成最终的标准，我们也在不断跟进 QUIC 协议的演进。

STGW 将持续为自研业务和腾讯云 CLB 客户提供 QUIC 的统一接入和优化，帮助业务更好的提升用户体验。

原文作者：wentaomao，腾讯 TEG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/ciR-1N4z0zvGOJSoyrvMUA