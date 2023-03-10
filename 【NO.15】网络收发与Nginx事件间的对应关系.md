# 【NO.15】网络收发与Nginx事件间的对应关系

## 1.**概述**

Nginx是一个事件驱动的框架， 所谓事件即网络事件。 Nginx每个连接自然对应两个网络事件，即 读事件和写事件。

要想理解Nginx的原理，以及Nginx再各种极端场景下的处理时，就必须要先了解网络事件。

## 2.**网络传输**

![img](https://pic4.zhimg.com/80/v2-5babcffb528891254b157864e53c4ca7_720w.webp)

假定主机 A 就是自己的电脑，主机 B 就是一台运行Nginx的[服务器](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/product/cvm%3Ffrom%3D10680)。

从主机 A 发送一个 HTTP 的 GET 请求到主机 B，这样的一个过程中主要经历了哪些事件？

通过上[图数据](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/product/konisgraph%3Ffrom%3D10680)流部分可以看出：

- 应用层里发送了一个 GET 请求 -> 到了传输层 ，这一步主要在做一件事，就是浏览器打开了一个端口，在 windows 的任务管理器中可以看到这一点，它会把这个端口记下来以及把 Nginx 打开的端口比如 80 或者 443 也记到传输层
- 然后在网络层会记下我们主机所在的 IP 和目标主机，也就是 Nginx 所在服务器公网 IP
- 到链路层以后 -> 经过以太网 -> 到达家里的路由器（网络层），家中的路由器会记录下所在运营商的一些下一段的 IP
- 通过广域网 -> 跳转到主机 B 所在的机器中 -> 报文会经过链路层 -> 网络层 -> 到传输层，在传输层操作系统就知道是给那个打开了 80 或者 443 的进程，这个进程自然就是 Nginx
- 那么 Nginx 在它的 HTTP 状态处理机里面（应用层）就会处理这个请求。

## 3.**TCP流与报文**

在上述过程中网络报文扮演了一个怎样的角色呢？

![img](https://pic3.zhimg.com/80/v2-cf52e6cd4a3ea2dfbd24be7f78a2595e_720w.webp)

- 数据链路层会在数据的前面 Header 部分和 Footer 部分添加上**源 MAC 地址和源目的地址**
- 到了网络层则是 Nginx 的公网地址（目的 IP 地址）和浏览器的公网地址（源 IP 地址）
- 到了 TCP 层（传输层），指定了 Nginx 打开的端口（目的端口）和浏览器打开的端口（源端口）
- 然后应用层就是 HTTP 协议了。

这就是一个报文，也就是说我们发送的 HTTP 协议会被切割成很多小的报文，在网络层会切割叫 MTU，以太网的每个 MTU 是 1500 字节；

> 在 TCP 层（传输层） 会考虑中间每个环节中最大的一个 MTU 值，这个时候往往每个报文只有几百字节，这个报文大小我们称为叫 MSS ，所以每收到一个 MSS 小于这么大小的一个报文时其实就是一个网络事件。 MTU： Maximum Transmit Unit，最大传输单元，即物理接口（数据链路层）提供给其上层（通常是网络层）最大一次传输数据的大小；每个以太网帧都有最小的大小64bytes，最大不能超过1518bytes，对于小于或者大于这个限制都视为错误的数据帧。一般的以太网转发设备会丢弃这些数据帧。 由于以太网帧的帧头的14字节和帧尾 CRC 校验4字节共占了18字节，剩下的承载上层协议的地方也就是 Data 域最大就只剩1500字节 MSS：Maximum Segment Size ，传输层概念，TCP 数据包每次能够传输的最大量。为了达到最佳的传输效能，TCP 协议在建立连接的时候通常要协商双方的 MSS 值，这个值TCP协议在实现的时候往往用 MTU 值代替（需要减去 IP 数据包包头的大小20Bytes和 TCP 数据段的包头20Bytes）所以往往 MSS 为1460。通讯双方会根据双方提供的 MSS 值得最小值确定为这次连接的最大 MSS 值。 MTU，最大传输单元是指一种通信协议在某一层上面所能通过的最[大数据](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/solution/bigdata%3Ffrom%3D10680)报大小（以字节为单位），它通常与链路层协议有密切的关系。 以太网传输电气方面的限制，每个以太网帧都有最小的大小64bytes，最大不能超过1518bytes，对于小于或者大于这个限制都视为错误的数据帧。一般的以太网转发设备会丢弃这些数据帧。 以太网EthernetII最大的数据帧是1518Bytes，除去以太网帧的帧头14Bytes和帧尾CRC校验部分4Bytes，那么剩下承载上层协议的地方也就是Data域最大就只能有1500Bytes，这个值我们就把它称之为MTU。 此MTU就是网络层协议非常关心的地方，因为网络层协议比如IP协议会根据这个值来决定是否把上层传下来的数据进行分片。就好比一个盒子没法装下一大块面包，我们需要把面包切成片，装在多个盒子里面一样的道理。当两台远程PC互联的时候，它们的数据需要穿过很多的路由器和各种各样的网络媒介才能到达对端，网络中不同媒介的MTU各不相同，就好比一长段的水管，由不同粗细的水管组成（MTU不同 ）通过这段水管最大水量就要由中间最细的水管决定。

![动图封面](https://pic3.zhimg.com/v2-29839370e8074f186e8dff0cc195cb06_b.jpg)





## **4.TCP 协议与非阻塞接口**

我们来看下 TCP 协议中许多事件是怎样和我们日常调用的一些接口（比如 Accept、Read、Write、Close）是怎样关联在一起的？

![img](https://pic3.zhimg.com/80/v2-5590a8a04b29e4d46ddd1a7b6aef4cee_720w.webp)

## 5.**读事件**

- 请求建立 TCP 连接事件实际上是发送了一个 TCP 报文，一个流程到达了 Nginx，对应的是读事件。因为对于 Nginx 来说，读取到了一个报文，所以就是 Accept 建立链接事件。
- 如果是 TCP 连接可读事件，就是发送了一个消息，对于 Nginx 也是一个读事件，就是 Read 读消息。
- 如果是对端（也就是浏览器）主动地关掉了，相当于 操作系统会去发送一个要求关闭链接的一个事件，对于 Nginx 来说还是一个读事件，因为它只是去读取一个报文。

## 6.**写事件**

那什么是写事件呢？

当需要向浏览器发送响应的时候，需要把消息写到操作系统中，要求操作系统发送到网络中，这就是一个写事件。

像这样的一些网络读写事件，通常在 Nginx 中或者任何一个异步事件的处理框架中，有个东西叫**事件收集、分发器**。

它会定义每类事件处理的消费者，也就是说事件是一个生产者，是通过网络中自动的生产到我们的 Nginx 中的，我们要对每种事件建立一个消费者。

> 比如连接建立事件消费者，就是对 Accept 调用，HTTP 模块就会去建立一个新的连接。还有很多读消息或者写消息，在 HTTP 状态机中不同的时间段会调用不同的方法也就是每个消费者处理。

以上就是一个事件分发、消费器，包括 AIO 像异步读写磁盘事件，还有定时器事件，比如是否超时（worker_shutdown_timeout）。

![动图封面](https://pic3.zhimg.com/v2-89e9068d77c7f035b0e184aefec4c452_b.jpg)



## 7.**WireShark抓包分析**

### **7.1WireShark设置**

首先下载一个WireShark ，设置好要抓取的 目标IP和端口，如下 。

![img](https://pic2.zhimg.com/80/v2-11aa7ac41f7b45d75f6b9118ebe06ca9_720w.webp)

点击开始

![img](https://pic1.zhimg.com/80/v2-4821c0a72a1f19d3ac8e7b953feb1564_720w.webp)

![img](https://pic2.zhimg.com/80/v2-dd62104afbdda595e9aef0db3bd2f415_720w.webp)

此时还没有流量，接下来我们访问下 nginx

![img](https://pic1.zhimg.com/80/v2-c2b8dcc221eba479722b00fb29e56cd8_720w.webp)

查看下wireshark

![img](https://pic1.zhimg.com/80/v2-4f022d05f2402e970251b6700b8570f4_720w.webp)

然后

![img](https://pic3.zhimg.com/80/v2-75e2b0a377fb0c2a95f6d586f563c2f2_720w.webp)

进行分析

### **7.2抓包分析**

本机IP

![img](https://pic1.zhimg.com/80/v2-dd624b499ce5e8d9124c99209e462f80_720w.webp)

- 浏览器首先会打开这个页面，本地打开了一个 50283端口，而 Nginx 启动的是 8888 端口。
- TCP 层主要做的是进程与进程之间通讯这件事。

![img](https://pic1.zhimg.com/80/v2-1cb85b2cf02d85f2a58d110ae6ea5090_720w.webp)

- IP 层主要解决机器与机器之间怎样互相找到的问题。

![img](https://pic2.zhimg.com/80/v2-5543d0fcf347175d0c264a6c44746d4d_720w.webp)

三次握手也就是 windows 先向 Nginx 发送了一次 [SYN]，那么相反的 Nginx 所在的服务器也会向 windows 发送一个 [SYN].

这个时候 Nginx 是没有感知到的，因为这个连接还是处于半打开的状态。直到这台 windows 服务器再次发送 [ACK] 到 Nginx 所在的服务器之上时，Nginx 所在的操作系统才会去通知 Nginx 我们收到了一个读事件，这个读事件对应是建立一个新连接，所以此时 Nginx 应该调用 Accept 方法去建立一个新的连接。

![img](https://pic3.zhimg.com/80/v2-cf7c3e23b69e32d0610beae89a6ff21e_720w.webp)

原文链接：https://zhuanlan.zhihu.com/p/583618780