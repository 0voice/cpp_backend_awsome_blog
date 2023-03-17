# 【NO.607】UDP的可靠性传输

## 1.UDP和TCP的区别

Tcp和udp都是属于TCP/IP协议(传输层协议)。

### 1.1 TCP

TCP（Transmission Control Protocol，传输控制协议）是面向连接的协议，也就是说，在收发数据前，必须和对方建立可靠的连接。 一个TCP连接必须要经过三次握手，断开连接时需要四次挥手。

**TCP的可靠性主要体现在什么方面呢？**

\1. 应用数据被分割成TCP认为最合适发送的数据块。

这个和UDP完全不同，应用程序将产生的数据报长度将保持不变。由TCP传递给IP的信息单位称为报文段或段。最大报文段(MSS)表示TCP传往另一端的最大块数据的长度。连接建立时，双方都需要通告自己的MSS。默认情况下MSS的值为536字节(可以加上20字节的IP首部和20字节的TCP首部)。对于一个以太网，最大的MSS可达到1460字节(1500(MTU) - 20(IP) - 20(TCP))。

\2. 当TCP发出一个段之后，它启动一个定时器，等待目的端确认收到这个报文段，如果不能及时收到一个确认，将重发这个报文段。

\3. 当TCP收到另一端的数据，他将发送一个确认。这个确认不是立即发送，而是延迟几分之一秒。

\4. TCP将保持它首部和数据的校验和。这个是一个端到端的检测，目的是检测数据在传输时的任何变化，如果有收到段的检验和有差错，tcp将丢弃这个报文段和不确认收到此报文段。

\5. 既然tcp报文段作为ip数据报来传输，而ip数据报的到达可能会失序，因此tcp报文段的到达可能会失序。如果必要，tcp将对收到的数据进行重新排序。

\6. ip数据报会发生重复，tcp的接收端必须丢弃重复的数据。

\7. TCP提供流量控制。

![img](https://pic4.zhimg.com/80/v2-3f135de50afeea0ec89d214718124d8b_720w.webp)

### 1.2 UDP

UDP（UserDatagramProtocol）是一个简单的面向消息的传输层协议，尽管UDP提供标头和有效负载的完整性验证（通过校验和），但它不保证向上层协议提供消息传递，并且UDP层在发送后不会保留UDP 消息的状态。因此，UDP有时被称为不可靠的数据报协议。如果需要传输可靠性，则必须在用户应用程序中实现。

![img](https://pic2.zhimg.com/80/v2-a4ea52a543e0662958e86455434d69e9_720w.webp)

**为什么要使用UDP传输可靠性数据**

UDP(User Datagram Protocol)传输与IP传输非常类似，它的传输方式也是"Best Effort"的，所以UDP协议也是不可靠的。我们知道TCP就是为了解决IP层不可靠的传输层协议，既然UDP是不可靠的，为什么不直接使用IP协议而要额外增加一个UDP协议呢？

1. 一个重要的原因是IP协议中并没有端口(port)的概念。IP协议进行的是IP地址到IP地址的传输，这意味者两台计算机之间的对话。但每台计算机中需要有多个通信通道，并将多个通信通道分配给不同的进程使用。一个端口就代表了这样的一个通信通道。UDP协议实现了端口，从而让数据包可以在送到IP地址的基础上，进一步可以送到某个端口。
2. 对于一些简单的通信，我们只需要“Best Effort”式的IP传输就可以了，而不需要TCP协议复杂的建立连接的方式(特别是在早期网络环境中，如果过多的建立TCP连接，会造成很大的网络负担，而UDP协议可以相对快速的处理这些简单通信）
3. 在使用TCP协议传输数据时，如果一个数据段丢失或者接收端对某个数据段没有确认，发送端会重新发送该数据段。TCP重新发送数据会带来传输延迟和重复数据，降低了用户的体验。对于迟延敏感的应用，少量的数据丢失一般可以被忽略，这时使用UDP传输将能够提升用户的体验。

UDP将数据从源端发送到目的端时，无需事先建立连接，没有使用TCP中的确认技术或滑动窗口机制，因此UDP不能保证数据传输的可靠性，也无法避免接收到重复数据的情况。

UDP传输的可靠性由应用层负责，由应用程序根据需要提供报文ACK机制、重传机制、序号机制、重排机制和窗口机制。这些TCP已经都具备了。

### 1.3 如何使用UDP传输可靠性数据

这里我们主要介绍一种开源UDP的可靠性方案KCP：[https://github.com/skywind3000/kcp](https://link.zhihu.com/?target=https%3A//github.com/skywind3000/kcp)

KCP主要的优势体现在以下方面:

1. 以10%-20%带宽浪费的代价换取了比 TCP快30%-40%的传输速度。
2. RTO翻倍vs不翻倍：TCP超时计算是RTOx2，这样连续丢三次包就变成RTOx8了，十分恐怖，而KCP启动快速模式后不x2，只是x1.5（实验证明1.5这个值相对比较好），提高了传输速度。
3. 选择性重传 vs 全部重传: TCP丢包时会全部重传从丢的那个包开始以后的数据， KCP是选择性重传，只重传真正丢失的数据包。
4. 快速重传（跳过多少个包马上重传）（如果使用了快速重传，可以不考虑RTO））
5. 发送端发送了1,2,3,4,5几个包，然后收到远端的ACK: 1, 3, 4, 5，当收到ACK3时， KCP知道2被跳过1次，收到ACK4时，知道2被跳过了2次，此时可以认为2号丢失，不用等超时，直接重传2号包，大大改善了丢包时的传输速度。
6. UNA vs ACK+UNA：ARQ模型响应有两种， UNA（此编号前所有包已收到，如TCP）和ACK（该编号包已收到），光用UNA将导致全部重传，光用ACK则丢失成本太高，以往协议都是二选其一，而 KCP协议中， 除去单独的 ACK包外，所有包都有UNA信息。
7. 非退让流控：KCP正常模式同TCP一样使用公平退让法则，即发送窗口大小由：发送缓存大小、接收端剩余接收缓存大小、丢包退让及慢启动这四要素决定。但传送及时性要求很高的小数据时，可选择通过配置跳过后两步，仅用前两项来控制发送频率。以牺牲部分公平性及带宽利用率之代价，换取了开着BT都能流畅传输的效果。

名词说明：

用户数据：应用层发送的数据，如一张图片2Kb的数据

MTU：最大传输单元。即每次发送的最大数据

RTO： Retransmission TimeOut，重传超时时间。

cwnd:congestion window，拥塞窗口，表示发送方可发送多少个KCP数据包。

与接收方窗口有关，与网络状况（拥塞控制）有关，与发送窗口大小有关。

rwnd:receiver window,接收方窗口大小，表示接收方还可接收多少个KCP数据包

snd_queue:待发送KCP数据包队列

snd_nxt:下一个即将发送的kcp数据包序列号

snd_una:下一个待确认的序列号

## 2.KCP的使用方式

1. 创建 KCP对象： ikcpcb *kcp = ikcp_create(conv, user);
2. 设置传输回调函数（如UDP的send函数）： kcp->output = udp_output;
3. 真正发送数据需要调用sendto
4. 循环调用 update： ikcp_update(kcp, millisec);
5. 输入一个应用层数据包（如UDP收到的数据包） ：ikcp_input(kcp,received_udp_packet,received_udp_size);
6. 我们要使用recvfrom接收，然后扔到kcp里面做解析
7. 发送数据： ikcp_send(kcp1, buffer, 8); 用户层接口
8. 接收数据： hr = ikcp_recv(kcp2, buffer, 10);

![img](https://pic2.zhimg.com/80/v2-a93be85b4c1673decde048f2f7872e7d_720w.webp)

### 2.1 kcp配置模式

**1、工作模式：** int ikcp_nodelay(ikcpcb *kcp, int nodelay, int interval, int resend, int nc)

nodelay ：是否启用 nodelay模式， 0不启用； 1启用。

interval ：协议内部工作的 interval，单位毫秒，比如 10ms或者 20ms

resend ：快速重传模式，默认0关闭，可以设置2（2次ACK跨越将会直接重传）

nc ：是否关闭流控，默认是0代表不关闭， 1代表关闭。

普通模式： ikcp_nodelay(kcp, 0, 40, 0, 0);

极速模式： ikcp_nodelay(kcp, 1, 10, 2, 1)

**2、最大窗口：** int ikcp_wndsize(ikcpcb *kcp, int sndwnd, int rcvwnd);

该调用将会设置协议的最大发送窗口和最大接收窗口大小，默认为32，单位为包。

**3、最大传输单元：** int ikcp_setmtu(ikcpcb *kcp, int mtu);

kcp协议并不负责探测 MTU，默认 mtu是1400字节

**4、最小RTO：**不管是 TCP还是 KCP计算 RTO时都有最小 RTO的限制，即便计算出来RTO为

40ms，由于默认的 RTO是100ms，协议只有在100ms后才能检测到丢包，快速模式下为

30ms，可以手动更改该值： kcp->rx_minrto = 10;

### 2.2 kcp的协议头

![img](https://pic2.zhimg.com/80/v2-922e4cea21598d1dfd6ee4e0c4f7422d_720w.webp)

conv:连接号。 UDP是无连接的， conv用于表示来自于哪个

客户端。对连接的一种替代

cmd:命令字。如， IKCP_CMD_ACK确认命令，

IKCP_CMD_WASK接收窗口大小询问命令，

IKCP_CMD_WINS接收窗口大小告知命令，

frg:分片，用户数据可能会被分成多个KCP包，发送出去

wnd:接收窗口大小，发送方的发送窗口不能超过接收方

给出的数值

ts:时间序列

sn:序列号

una:下一个可接收的序列号。其实就是确认号，收到

sn=10的包， una为11

len：数据长度

data:用户数据

test kcp代码:[https://github.com/birate-wz/wz](https://link.zhihu.com/?target=https%3A//github.com/birate-wz/wz_utils/tree/main/kcp)

原文地址：https://zhuanlan.zhihu.com/p/612359612

作者：cpp后端技术