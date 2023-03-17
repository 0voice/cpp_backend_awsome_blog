# 【NO.507】网络编程:手绘TCP状态机

**知识卡**

![img](https://pic1.zhimg.com/80/v2-6323a56fff5913e7191bbd5b1f982430_720w.webp)

**情景对话**

老王：小王，最近工作注意力不集中呀！

小王：我在等面试结果呢！

老王：你感觉如何呢？

小王：

当时情况是这样的！

------

大王：你擅长window，还是liunx?

小王：Linux（这年头谁还在写window程序）

大王；那你对网络编程一定很熟悉 吧？

小王：那是当然（都是小菜一碟）。

大王：请绘制TCP状态转换过程？

小王：。。。。（这个谁能记住他，绞尽脑汁想，5分钟过去了）

大王：还有什么要补充的吗？（耐心等待）

小王：不会写，有几个记不清楚（5分钟过去了）

大王：好，回去等通知。

老王：我来讲一讲，需要解决下面几个问题

![img](https://pic1.zhimg.com/80/v2-0016e314a66c36d46c4a4fb85b37e0dc_720w.webp)

自我提问

## 1.**问题1 socket通讯过程，和抓包格式 时间限时在2分钟**

小王：

> socket常用接口 accept,read,write close，我经常用很熟悉呀，没什么可学的了， 还有tcp协议那个图 我看多少遍？

（老王）我这里提示一下，不做深入讨论，时间限时在2分钟。

![img](https://pic3.zhimg.com/80/v2-e92b8becaba806a66065ed514770a4ee_720w.webp)

完整通讯过程

![img](https://pic1.zhimg.com/80/v2-a8106991780add30c0a10528341a522c_720w.webp)

![img](https://pic1.zhimg.com/80/v2-5c57d674a72349e63ce31fbf32862d24_720w.webp)

用户态

## 2.**问题2 ：三次握手和四次挥手过程 ,时间限时在5 分钟**

![img](https://pic1.zhimg.com/80/v2-40eaa4976148abb927c14ac5e802da8c_720w.webp)

客户端和[服务器](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/product/cvm%3Ffrom%3D10680)同时发现异常，都进行关闭这个连接

![img](https://pic2.zhimg.com/80/v2-1c9271a8f9661e129fa0cae4cee72c11_720w.webp)

我没遇到过

## 3.**问题3 在机器重启，服务重启，网络断开等情况下呢？**

小王：

> 遇到这个情况，不是epoll, SO_KEEPALIVE read返回 0代表接受。一般都是这么处理的

闲话少说，从四次挥手最有一步异常说起。

老王：RichardStevens说过这样2句话

There are two reasons for the TIME_WAIT state:

一、保证TCP协议的全双工连接能够可靠关闭

To implement TCP's full-duplex connection termination reliably

二、保证这次连接的重复数据段从网络中消失

To allow old duplicate segments to expire in the network

小王：

表示不理解，上面不是同一个意思吗，如果达到不了就消失？

（老王）错误，继续看

根据第三版《UNIX网络编程 卷1》2.7节，TIME_WAIT状态的主要目的有两个：

- 优雅的关闭TCP连接，也就是尽量保证被动关闭的一端收到它自己发出去的FIN报文的ACK确认报文；
- 处理延迟的重复报文，这主要是为了避免前后两个使用相同四元组的连接中的前一个连接的报文干扰后一个连接。 **保证TCP协议的全双工连接能够可靠关闭：（解释）**

> ACK is lost. The server will resend its final FIN, so the client must maintain state information, allowing it to resend the final ACK. If it did not maintain this information, it would respond with an RST (a different type of TCP segment), which would be interpreted by the server as an error

发生条件：

服务正常，网络正常。

- 服务正常，网络正常：

B发送FIN，进入LAST_ACK状态，A收到这个FIN包后发送ACK包，**B收到这个ACK包**，然后进入CLOSED状态

- 服务正常，网络拥塞，网络连接良好

B发送FIN，进入LAST_ACK状态，A收到这个FIN包后发送ACK包，由于某种原因，这个ACK包丢失了，**B没有收到ACK包**，然后B等待ACK包超时，又向A发送了一个FIN包 a) **假如这个时候，A还是处于TIME_WAIT状态(也就是TIME_WAIT持续的时间在2MSL内)**A收到这个FIN包后向B发送了一个ACK包，B收到这个ACK包进入CLOSED状态

b) **假如这个时候，A已经从TIME_WAIT状态变成了CLOSED状态** A收到这个FIN包后，认为这是一个错误的连接，向B发送一个**RST**包，当B收到这个RST包，进入CLOSED状态

- 服务不正常 或者网络断开 c) **假如这个时候，A挂了（假如这台机器炸掉了）【第四种情况，不在参考链接里】** B没有收到A的回应，那么会继续发送FIN包，也就是触发了TCP的重传机制，如果A还是没有回应，B还会继续 发送FIN包，直到重传超时(至于这个时间是多长需要仔细研究)，B重置这个连接，进入CLOSED状态，

小王：原来是这样

画外音

网络断了，节点重启了，是无法处理的。只能依靠Rst解决。

下面情况如果ack，不能按时到达，阻止建立新的连接。

小王：原来是这样

画外音：

> TCP连接中的一端发送了FIN报文之后如果收不到对端针对该FIN的ACK，则会反复多次重传FIN报文. 处于TIME_WAIT状态的一端在收到重传的FIN时会重新计时(rfc793 以及 linux kernel源代码tcp_timewait_state_process函数

**保证这次连接的重复数据段从网络中消失（解释）**

发生条件：

Note that it is *very* unlikely that delayed segments will cause problems like this.

Firstly the address and port of each end point needs to be the same; which is normally unlikely as the client's port is usually selected for you by the operating system from the ephemeral port range and thus changes between connections.

Secondly, the sequence numbers for the delayed segments need to be valid in the new connection which is also unlikely. However, should both of these things occur then `TIME_WAIT` will prevent the new connection's data from being corrupted.

画外音：

必须原来的ip，原来的端口发起的连接，想想一个服务器连接多个客户端，四元组 是唯一的。

![img](https://pic4.zhimg.com/80/v2-e621f23c9f23dbac8ccf1419fba682f3_720w.webp)

*Due to a shortened TIME-WAIT state, a delayed TCP segment has been accepted in an unrelated connection.*

![img](https://pic4.zhimg.com/80/v2-7503e41fdcab0719781b228847c5c7c7_720w.webp)

image.png

小王：原来是这样！

**画外音：**

**四次挥手已经完成，最有一个ack顺利达到对方，一方进入closed状态（假如3秒内完成）**

**对方依然要等待2MSL（剩余28秒），这个等待不是多余等待，而是防止**

**这个时候双方如果马上同时closed（是允许建立新的连接。这是正常通讯过程）、**

**还有延迟重发的数据包。对同一个pair连接，新老数据造成混乱**。

> tcp协议提到内核接受数据是根据port区分是那个，而不是fd。

## 4.**小王偷偷写这么几句话**

time_wait 存在的意义有2点

（1） TCP 可靠传输，保证四次挥手最后一个ack 顺利到达对方。

采用方式是：如果获取到对方重新发送fin请求，需要重新计时间，维持TIme_wait状态。

保障每次发送出去ack都最终结果（收到或者消失）

如果在网络出断网，或者服务节点重启，或者对方不启tcp重传机制上面方法是无法处理的

应该超时或者返回Rst包出路 结束last_ack状态。

（2 ） TCP基于四元组建立连接， 假如客户端端口 不随机产生，而是相同ip，相同的

端口，再次连接的话。可能出现，虽然old 连接已经消失，但是在网络中数据可能存在。

以tcp 内核中断处理 网络消息是根据 端口划分的。会造成新旧数据混乱。

> TCP不能给处于TIME_WAIT状态的连接启动新的连接。 TIME_WAIT的持续时间是2MSL，保证在建立新的连接之前老的重复分组在网络中消逝。 这个规则有一个例外：如果到达的SYN的序列号大于前一个连接的结束序列号， 源自Berkeley的实现将给当前处于TIME_WAIT状态的连接启动新的化身。 do_time_wait

![img](https://pic2.zhimg.com/80/v2-783f051280100e17186da109edb4595d_720w.webp)

tcp状态机

原文地址：https://zhuanlan.zhihu.com/p/602356909

作者：linux