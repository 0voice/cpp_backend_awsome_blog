# 【NO.17】三次握手时，客户端发送的 SYN 报文为什么会被丢弃？

## 1.**楔子**

我们知道当客户端和服务端建立连接时，会进行三次握手。客户端先向服务端发送 SYN 报文，表示想要建立连接，这是第一次握手；然后服务端收到 SYN 之后会给客户端回复 SYN + ACK，表示同意建立连接，也就是第二次握手。

![img](https://pic2.zhimg.com/80/v2-bcf301db24115a25c719013ac325946d_720w.webp)

但如果第二次握手的时候，服务端没有回复，那么说明客户端发送的 SYN 报文被服务端忽略了。

![img](https://pic3.zhimg.com/80/v2-fb0210147cdffa269e7644d2141b111a_720w.webp)

然后客户端在规定时间内，因收不到服务端的反馈就会触发超时，于是重传 SYN 报文，直到达到最大的重传次数。以上便是 SYN 报文被丢弃的过程，但问题是它为什么会被丢弃呢？主要有以下原因：

- 开启 tcp_tw_recycle 参数，并且处于 NAT 环境下；
- Accpet 队列满了；
- SYN 队列满了；

下面来解释一下。

## 2.**tcp_tw_recycle 参数**

TCP 四次挥手过程中，主动断开连接方会有一个 TIME_WAIT 状态，这个状态会持续 2 MSL 后才转变为 CLOSED 状态。

![img](https://pic4.zhimg.com/80/v2-4f3f8f1729f599e3db2e097e91a95627_720w.webp)

在 Linux 操作系统下，TIME_WAIT 状态的持续时间是 60 秒，你可以通过修改 Linux 源代码来改变这个值，但不推荐。

![img](https://pic3.zhimg.com/80/v2-e4d5ffe3a39b315917c682b5630e2006_720w.webp)

因此在 60 秒内，端口会一直被客户端占用。而端口资源是有限的，一般可开启的端口为 32768 ~ 60999。

![img](https://pic3.zhimg.com/80/v2-64d4cb416421f0cc97060ce0be29feb2_720w.webp)

当然你也可以修改这两个值，但总之如果主动断开连接方的 TIME_WAIT 状态过多，占满了所有端口资源，则会导致无法创建新连接。问题来了，既然 TIME_WAIT 有缺陷，那为什么还要保留这个特性呢？不用想，肯定是有着其它作用，而作用有两个：

- 防止旧的连接数据包；
- 保证连接正确关闭；

我们分别解释。

**原因一：防止旧的连接数据包**

假设 TIME-WAIT 没有等待时间或时间过短，被延迟的数据包抵达后会发生什么呢？

![img](https://pic1.zhimg.com/80/v2-cbf297f554c7c4d609d76cc639536d30_720w.webp)

如上图黄色框框显示的那样，服务端在关闭连接之前发送的 Seq = 301 报文，被网络延迟了。这时有相同端口的 TCP 连接被复用后，被延迟的 Seq = 301 抵达了客户端，那么客户端有可能正常接收这个过期的报文，这就会产生数据错乱等严重的问题。

所以 TCP 就设计出了这么一个机制，因为 1MSL 表示报文的最大生存时间，那么经过 2MSL 便可以让两个方向上的数据包在网络中都自然消失，那么再出现的数据包一定都是新建立连接所产生的。

**相关视频推荐**

[tcpip，accept，11个状态，细枝末节的秘密，还有哪些你不知道](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1gf4y1k78E/)

[通过10道经典网络面试题，搞懂tcp/ip协议栈所有知识点](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1oW4y1E7r3/)

[C/C++开发哪个方向更有前景，游戏，c++后端，网络处理，音视频开发，嵌入式开发，桌面开发](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1HV4y1578G/)

学习地址：**[c/c++ linux服务器开发/后台架构师](https://link.zhihu.com/?target=https%3A//jq.qq.com/%3F_wv%3D1027%26k%3DnzAMj4Qa)**

需要C/C++ Linux服务器架构师学习资料加qun**[812855908](https://link.zhihu.com/?target=https%3A//jq.qq.com/%3F_wv%3D1027%26k%3DT2C5XZV4)**获取（资料包括**C/C++，Linux，golang技术，Nginx，ZeroMQ，MySQL，Redis，fastdfs，MongoDB，ZK，流媒体，CDN，P2P，K8S，Docker，TCP/IP，协程，DPDK，ffmpeg**等），免费分享

![img](https://pic3.zhimg.com/80/v2-0fb9145c6e395f48de7c1d3a1d500dc2_720w.webp)

**原因二：保证连接正确关闭**

其实在四次挥手示意图中应该就能发现端倪，我们知道服务端在传输完数据之后会发送 FIN 表示正式关闭连接，然后处于 LAST_ACK 状态，等待客户端的最后一次确认。

如果客户端再发送一次 ACK 给服务端，那么服务端收到之后就会进入 CLOSED 状态，但问题是这最后一次 ACK 报文如果在网络中丢失了该怎么办？

如果没有 TIME_WAIT，那么客户端把 ACK 报文发送之后就进入 CLOSED 了，但 ACK 报文并没有到达服务端，这时服务端就会一直处于 LAST_ACK 状态。如果后续客户端再发起新的建立连接的 SYN 报文后，服务端就不会再返回 SYN + ACK 了，而是直接发送 RST 报文表示终止连接的建立。

因此客户端在发送完 ACK 之后不能直接 CLOSED，而是要等一段时间。如果服务端在发「FIN 关闭连接报文」之后的规定时间内没有收到来自客户端的 ACK 报文，那么服务端就知道这个 ACK 报文在网络中丢失了，此时会重新给客户端发送 FIN 报文。

所以客户端要等待（此时处于 TIME_WAIT 状态），因为它不知道自己最后发送的 ACK 报文是否成功抵达服务端，它只知道服务端收不到 ACK 报文时，会再度给自己发送 FIN 报文，因此只能默默等待 2MSL（发送 ACK 加上当服务端收不到时返回 FIN，整个过程最多 2MSL）。

如果再次收到服务端的 FIN，那么它要再次发送 ACK；但如果等了 2MSL 后，服务端没有再次发送 FIN，那么它就知道自己上一次发送的 ACK 被服务端成功接收了，此时也会进入 CLOSED 状态。至此，四次挥手完成，客户端和服务端之间连接断开。

以上便是 TIME_WAIT 状态的作用，之所以单独花些时间介绍它，是因为 TIME_WAIT 是导火索。由于 TIME_WAIT 状态的连接过多会造成内存资源和本地端口资源的占用，所以 Linux 内核提供了两个系统参数来快速回收处于 TIME_WAIT 状态的连接，而这两个参数默认都是关闭的：

- 参数一：net.ipv4.tcp_tw_reuse，如果开启该选项的话，客户端（连接发起方） 在调用 connect() 函数时，内核会随机找一个 time_wait 状态超过 1 秒的连接给新的连接复用，所以该选项只适用于连接发起方；
- 参数二：net.ipv4.tcp_tw_recycle，如果开启该选项的话，允许处于 TIME_WAIT 状态的连接被快速回收；
- 要使得上面两个选项在开启之后能够生效，还有一个前提条件，就是要打开 TCP 时间戳，也就是将 net.ipv4.tcp_timestamps 设置为 1（默认为 1）；

![img](https://pic1.zhimg.com/80/v2-ac1fe99e407cada3ed5f72df76c7dd70_720w.webp)

但是重点来了，tcp_tw_recycle 在使用了 NAT 的网络下是不安全的。因为对于服务器来说，如果同时开启了 tcp_tw_recycle 和 tcp_timestamps 选项，则会额外开启一种称之为 per-host 的 PAWS 机制，正是这个机制导致了 SYN 报文可能出现丢失。

目前的信息量估计稍微有点大，我们先简单回顾一下什么是 NAT，然后再来介绍一下什么是 PAWS，最后再来说 per-host 的 PAWS。

## 3.**什么是 NAT**

这里需要先解释一下什么是 NAT，NAT 指的是网络地址转换，简单来说就是将内部网络的私有 IP 转成公有 IP。估计很多人分不清 NAT 和桥接（Bridged）之间的区别，我们以 VMware 为例来解释一遍。

提个问题，我们在使用 VMware 虚拟出一个 CentOS 之后，这个 CentOS 要如何连接外网呢？

***第一种模式：桥接模式***

VMware 在安装之后会创建一个虚拟网桥以及一个虚拟交换机，所有以桥接模式设置的虚拟机都会连接到虚拟交换机的一个接口上，共处于一个二层网络中。所以桥接下的网卡与网卡都是交换模式的，相互可以直接访问而不干扰。

而虚拟机虚拟的网卡和主机网卡通信则是需要借助虚拟网桥，所以在桥接模式下，虚拟机 IP 地址需要与主机在同一个网段。如果需要联网，则网关与 DNS 需要与主机网卡一致。其网络结构如下图所示：

![img](https://pic4.zhimg.com/80/v2-223d583be8c11dd669904cd79d21e7c7_720w.webp)

桥接模式下的多个虚拟机之间可以直接通信，而虚拟机和主机之间则要借助于虚拟网桥，因此虚拟机是可以直接 ping 通外网 ip 的（前提是主机可以）。

似乎桥接模式还是蛮不错的，但它有一个缺点，就是它要和主机在同一个网段，并且要为其分配一个独立的 IP。如果你当前网段的可使用 IP 不多或者对 IP 管理比较严格的话，那么桥接模式就不适用了，因为 IP 地址占用严重（每一个虚机都要有一个独立的 IP）。

所以就有了 NAT。

***第二种模式：NAT 模式***

上面说道，如果你的网络 IP 资源紧缺，但又希望你的虚拟机能够联网，这时候 NAT 模式是最好的选择。NAT 模式借助虚拟 NAT 设备和虚拟 DHCP 服务器，首先主机网卡直接与虚拟 NAT 设备相连，然后虚拟 NAT 设备和虚拟 DHCP 服务器、以及虚机网卡一起连接在虚拟交换机上。

因此，通过 NAT 设备和 DHCP，虚拟机会共用主机网卡实现对外上网，对外暴露的都是主机 IP。就类似于局域网内的 IP 会共用一个公网 IP 一样，背后用的同样是 NAT 技术，每个局域网内的 IP 只需要保证在当前局域网内不冲突即可。至于对外上网，用的是同一个公网 IP，并且局域网内的 IP 和外网 IP 可以不在同一个网段。

不是很复杂，这里不画图了。

多提一句，Docker 也是类似的，Docker 在安装之后会创建一个 docker0 虚拟网桥，默认网络模式下每个容器也都有各自的 Network NameSpace、并且会设置 IP 等。这些容器也在一个二层网络中，并且通过 docker0 网桥以及 iptables nat 表连接至主机网卡，和外网进行通信。

## 4.**什么是 PAWS 机制**

说完了 NAT 之后再来看看什么是 PAWS？

当 tcp_timestamps 选项开启之后， PAWS 机制会自动开启，它的作用是防止 TCP 包中的序列号发生绕回。正常来说每个 TCP 包都会有自己唯一的 Seq 号（序列号），出现 TCP 数据包重传的时候会复用 Seq 号，这样接收方能通过 Seq 号来判断数据包的唯一性，也能在重复收到某个数据包的时候判断数据是不是重传的，最关键的是还可以保证数据包按序接收、不会错乱。

但问题是 TCP 的这个 Seq 号是有限的，一共 32 bit，随着传输对的不断进行，Seq 会不断递增，当溢出之后从 0 开始再次依次递增。

所以当 Seq 号出现溢出后单纯通过 Seq 号无法标识数据包的唯一性，某个数据包延迟或因重发而延迟时可能导致连接传递的数据被破坏，比如：

![img](https://pic1.zhimg.com/80/v2-80eb8fe3e17eeece31adb1769dcf7cdc_720w.webp)

上图 Seq=A 数据包出现了重传，并在 Seq 号耗尽再次从 A 递增时，第一次发的 A 数据包延迟到达了 server。由于 TCP 会通过序列号来去除重复数据，那么两个 Seq=A 的包肯定会丢弃一个。这种情况下如果没有别的机制来保证，server 会认为延迟到达的 A 数据包是正确的而接收，反而是将正常的第三次发的 Seq 为 A 的数据包丢弃，造成数据传输错误。

PAWS 就是为了避免这个问题而产生的，在开启了 tcp_timestamps 选项的情况下，一台机器发的所有 TCP 包都会带上发送时的时间戳（根据 CPU tick 计算得到，放在 TCP option 中）。

PAWS 要求连接双方维护最近一次收到的数据包的时间戳（Recent TSval），每收到一个新数据包都会读取数据包中的时间戳值跟 Recent TSval 值做比较。如果发现收到的数据包中时间戳不是递增的，则表示该数据包是过期的，就会直接丢弃这个数据包。

对于上面图中的例子，如果有了 PAWS 机制，就能做到在收到 Delay 到达的 A 号数据包时，识别出它是个过期的数据包而将其丢掉，因为时间戳不递增。

## **5.什么是 per-host 的 PAWS 机制**

当同时开启了 tcp_tw_recycle 和 tcp_timestamps 时，就会开启一种叫 per-host 的 PAWS 机制。而从名字上就可以得出，per-host 是针对每一个 IP 做 PAWS 检查，不同 IP 之间的 PAWS 检查是独立的。

但如果客户端网络环境使用了 NAT，那么客户端环境的每一台机器通过 NAT 网关后，都会是相同的 IP 地址，那么在服务端看来就好像跟一个客户端打交道一样，无法区分出来。

举个例子，当客户端 A 通过 NAT 网关和服务器建立 TCP 连接，然后服务器主动关闭并且快速回收 TIME_WAIT 状态的连接后，客户端 B 也通过 NAT 网关和服务器建立 TCP 连接。注意客户端 A 和 客户端 B 因为经过相同的 NAT 网关，所以使用相同的 IP 地址与服务端建立 TCP 连接。如果客户端 B 的 timestamp 比客户端 A 的 timestamp 小，那么由于服务端的 per-host 的 PAWS 机制的作用，服务端就会丢弃客户端主机 B 发来的 SYN 包。

因此 tcp_tw_recycle 在使用了 NAT 的网络下是存在问题的，因为使用 NAT 之后 IP 地址相同，都是同一个公网 IP。当然，如果不是对每一个 IP 做 PAWS 检查，而是对 IP 加端口组合起来做 PAWS 检查，那么就不会存在这个问题了。

以上就是 SYN 丢弃的原因之一，回顾整个过程，我们先是介绍了 TIME_WAIT，然后引出了 tcp_tw_recycle，接着引出了 NAT, PAWS，最后引出了 per-host PAWS。

> tcp_tw_recycle 在 Linux 4.12 版本后，直接取消了这一参数。所以不存在开启 tcp_tw_recycle 来优化 TCP 这一说。

## **6.Accept 队列满了**

在 TCP 三次握手的时候，Linux 内核会维护两个队列，分别是：

- 半连接队列，也称 SYN 队列；
- 全连接队列，也称 Accept 队列；

服务端收到客户端发起的 SYN 请求后，内核会把该连接存储到半连接队列，并向客户端响应 SYN+ACK。接着客户端会返回 ACK，服务端收到第三次握手的 ACK 后，内核会从半连接队列里面将连接取出，然后添加到全连接队列，等待进程调用 accept 函数时把连接取出来。

![img](https://pic1.zhimg.com/80/v2-38fea32a06ed5325f1751c2ce584afc0_720w.webp)

所以整个过程如下：

- \1. 客户端发送 SYN 报文；
- \2. 服务端将连接插入到半连接队列；
- \3. 服务端向客户端返回 SYN + ACK；
- \4. 客户端收到之后再向服务端返回 ACK；
- \5. 服务端将连接从半连接队列中取出，移入全连接队列；
- \6. 进程调用 accept 函数，从全连接队列中取出已完成连接建立的 socket连接；

因此半连接队列（SYN 队列）用来存储 SYN_RECV 状态、未完成建立的连接，全连接队列（Accept 队列）用来存储 ESTABLISH 状态、已完成建立的连接。

而我们也可以很容易得出结论，客户端返回成功是在第二次握手之后，服务端 accept 成功是在三次握手之后，因为调用 accept 就相当于从全连接队列中取出一个连接和客户端进行通信。

那么如何查看 SYN 队列和 Accept 队列的大小呢？

- net.ipv4.tcp_max_syn_backlog：查看半连接队列长度；
- net.core.somaxconn：查看全连接队列的长度；

![img](https://pic4.zhimg.com/80/v2-702a46bf1cd8946a1db5c1079d782aaf_720w.webp)

Linux 一切皆文件，如果想要修改队列大小的话，直接修改相应的文件即可。当然准确来说：

- max(64, tcp_max_syn_backlog) 才是半连接队列的长度；
- min(backlog, somaxconn) 才是全连接队列的长度，这里的 backlog 就是我们编写 socket 代码时，在 listen 方法里面指定的值；

但是在服务端并发处理大量请求时，如果 TCP Accpet 队列过小，或者应用程序调用 accept 方法不及时，就会造成 Accpet 队列已满。这时后续的连接就会被丢弃，这样就会出现服务端请求数量上不去的现象。

![img](https://pic2.zhimg.com/80/v2-1198ed32013d00b53fbcd51d4943dfb9_720w.webp)

所以 Accept 队列满了，也会造成 SYN 报文的丢失。那么如何查看 Accept 队列是否已满呢？可以使用命令 ss -lnt 来查看，其中 -l 表示显示正在监听的 socket，-n 表示不解析服务名称，-t 表示只显示 tcp socket，同理 -u 表示只显示 udp socket。

![img](https://pic4.zhimg.com/80/v2-403e3a86317bfedd12ec9983ea8a0477_720w.webp)

- Recv-Q：当前 Accpet 队列的大小，也就是当前已完成三次握手并等待服务端 accept() 的 TCP 连接个数；
- Send-Q：当前 Accpet 队列的最大长度，我们以监听 8088 端口的 TCP 服务进程为例，输出结果说明了 Accpet 队列的最大长度为 50，显然在 listen 的时候指定了 50；

如果 Recv-Q 的大小超过 Send-Q，就说明发生了 Accpet 队列满的情况。要解决这个问题，我们可以：

- 调大 Accpet 队列的最大长度，调大的方式是通过增大 backlog 以及 somaxconn 的值；
- 检查系统或者代码为什么调用 accept() 不及时；

注意：使用 ss 命令获取 Recv-Q 和 Send-Q 时，连接一定要在 LISTEN 状态（通过参数 -l 指定），如果不是 LISTEN的话，那么 Recv-Q 和 Send-Q 就不再表示队列大小了。

![img](https://pic3.zhimg.com/80/v2-b1d8b226338110c7b6fedaddbfdc85a2_720w.webp)

代码位于 net/ipv4/tcp_diag.c 中，可以看一下。如果我们使用 ss 命令的时候指定 -l 参数的话：

![img](https://pic3.zhimg.com/80/v2-eea828d94066cadb3b05728d0fb46c5a_720w.webp)

此时显示状态为 ESTABLISHED 的连接，连接既然都已经建立了，那么 Recv-Q 和 Send-Q 就不再表示队列大小了。

再来补充一下，当全连接队列满了之后，除了丢弃连接还有没有其它的做法呢？实际上，丢弃连接只是 Linux 的默认行为，我们还可以选择向客户端发送 RST 复位报文，告诉客户端连接已经建立失败，而这取决于 tcp_abort_on_overflow 的值。

- 如果为 0：表示当全连接队列满了，server 会扔掉 client 发过来的 ack；
- 如果为 1：表示当全连接队列满了，那么 server 发送一个 reset 包给 client，表示废掉这个握手过程和这个连接；

如果要想知道客户端连接不上服务端，是不是服务端 TCP 全连接队列满的原因，那么可以把 tcp_abort_on_overflow 设置为 1。这时如果在客户端异常中可以看到很多 connection reset by peer 的错误，那么就可以证明是由于服务端 TCP 全连接队列溢出的问题。但不管设置为 0 还是 1，最终 SYN 报文都不会得到正常的应答。

> connection reset by peer 这个错误应该有很多人遇见吧，我在连接消息队列的时候就遇到过。

但通常情况下，应当把 tcp_abort_on_overflow 设置为 0，因为这样更有利于应对突发流量。

举个例子，当 TCP 全连接队列满导致服务器丢掉了 ACK，与此同时，客户端的连接状态却是 ESTABLISHED（因为它发生在第二次握手之后），进程就在建立好的连接上发送请求。只要服务器没有为请求回复 ACK，请求就会被多次重发。

如果服务器上的进程只是短暂的繁忙造成 Accept 队列满，那么当 TCP 全连接队列有空位时，再次收到的请求报文由于含有 ACK，仍然会触发服务器端成功建立连接。

所以 tcp_abort_on_overflow 设为 0 可以提高连接建立的成功率，只有你非常肯定 TCP 全连接队列会长期溢出时，才能设置为 1 以尽快通知客户端。

## 7.**SYN 队列满了**

因为 SYN 发送之后连接会先进入半连接队列，之后再进入全连接队列。如果全连接队列满了会导致 SYN 报文丢弃，那么半连接队列满了应该也会导致 SYN 报文被丢弃吧。答案是肯定的，我们先来看看什么情况下半连接队列会满。

![img](https://pic4.zhimg.com/80/v2-793e3cb726a18198902e7aed35ccc113_720w.webp)

当服务端收到来自客户端的 SYN 报文时，就会进入 SYN_RECV 状态，此时连接会进入半连接队列。但服务端发出去的 SYN + ACK 报文却迟迟得不到客户端的应答，久而久之就会占满服务端的半连接队列。而所谓的 SYN 攻击就是采用这种方式，短时间伪造大量不同 IP 地址的 SYN 报文，但是故意不回复 ACK，直到服务端的半连接队列已满，使得其不能为正常用户服务。

![img](https://pic2.zhimg.com/80/v2-504ea77e3c6c932cc10e1a02c6352559_720w.webp)

我们知道当收到客户端的 ACK 报文之后会将连接从半连接队列移除并放入到全连接队列，但是 SYN 攻击的特点就是在发送完 SYN 报文之后故意不发 ACK 报文，因此最终半连接队列会被塞满，全连接队列会为空。咦，那出现这种情况除了丢弃连接之外还有什么解决办法呢？答案是启动 cookie。

![img](https://pic2.zhimg.com/80/v2-7a640d4bd97d264c9e24a240108fae2d_720w.webp)

通过设置 net.ipv4.tcp_syncookies = 1 实现。

![img](https://pic2.zhimg.com/80/v2-b1f8e7b4ca42fe830fc1d251f5454201_720w.webp)

默认值就是 1，那么它的含义是什么呢？

- 当 SYN 队列满之后，后续服务器收到 SYN 包，不进入半连接队列；
- 而是根据当前状态计算出一个 cookie 值，再以 SYN + ACK 中序列号的形式返回客户端；
- 服务端接收到客户端的应答报文时，服务器会检查这个 ACK 包中 cookie 值的合法性。如果合法，直接放入到全连接队列；
- 最后应用通过调用 socket 接口 accpet()，从全连接队列取出连接；

![img](https://pic1.zhimg.com/80/v2-a96c182ace6eef8a80bff0e6b3cb4090_720w.webp)

tcp_syncookies 参数的取值有三种，值为 0 时表示关闭该功能，2 表示无条件开启功能，而 1 则表示仅当 SYN 半连接队列放不下时，再启用它。

由于 syncookie 仅用于应对 SYN 泛洪攻击（攻击者恶意构造大量的 SYN 报文发送给服务器，造成 SYN 半连接队列溢出，导致正常客户端的连接无法建立），毕竟这种方式建立的连接，许多 TCP 特性都无法使用。所以应当把 tcp_syncookies 设置为 1，仅在队列满时再启用。

因此 SYN 队列满了也会丢弃 SYN 报文，但连接是可以建立的。当然除了这种方式，我们还可以适当增加半连接队列的大小，但是不能单纯增大 tcp_max_syn_backlog 的值，还需要一同增大 somaxconn 和 backlog，也就是增大全连接队列，只增大 tcp_max_syn_backlog 没有意义。

```
# 增大 tcp_max_syn_backlog
echo 1024 > /proc/sys/net/ipv4/tcp_max_syn_backlog
# 增大 somaxconn
echo 1024 > /proc/sys/net/ipv4/somaxconn
```

至于 backlog 则是在应用程序中设置，比如 Python 是在调用 listen 方法的时候设置：

![img](https://pic1.zhimg.com/80/v2-63889b9226585201440af89a8de93400_720w.webp)

再比如 Nginx，是在配置文件中设置（配置完之后要重启 Nginx）：

```
server {
    listen 8088 default backlog=1024;
    server_name localhost;
    ......
}
```

另外当服务端受到 SYN 攻击时，就会有大量处于 SYN_RECV 状态的 TCP 连接，处于这个状态的 TCP 会重传 SYN+ACK ，当重传超过次数达到上限后，就会断开连接。而这个重传次数默认为 5：

![img](https://pic1.zhimg.com/80/v2-619b5808475fa10eaabef96a80ddee88_720w.webp)

那么针对 SYN 攻击的场景，我们可以减少 SYN+ACK 的重传次数，以加快处于 SYN_RECV 状态的 TCP 连接断开。

## 8.**小结**

以上就是关于 SYN 报文会在何时丢弃的相关内容，通过 SYN 报文丢弃我们引出了一些参数配置，以及半连接队列和全连接队列，并探讨了两个队列满了会引发什么后果、要如何处理等等。

当然私下里，建议没事也可以好好学习一下 TCP，该协议已经发展好几十年了，其性能已经做足了优化，对互联网的发展起到了举足轻重的作用，掌握它对我们的职业生涯是非常有帮助的。

原文链接：https://zhuanlan.zhihu.com/p/583279182