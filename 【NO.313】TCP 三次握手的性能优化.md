# 【NO.313】TCP 三次握手的性能优化

今天分析下 TCP 三次握手中有哪些可以优化的地方，进而提升握手的性能。

![img](https://pic3.zhimg.com/80/v2-2ce43a6a499366ee5da1f776f98c1066_720w.webp)

## 1.客户端的优化

三次握手的首要目的就是为了同步序列号。有了序列号才可以进行后续的可靠性的传输。在 TCP 中有很多功能都是依赖序列号实现的，比如流量控制、消息重传等。

在三次握手中序列号的同步是通过SYN报文同步的（SYN 全称Synchronize Sequence Number）。

三次握手的过程由协议栈自动来实现的。客户端调用 connect() 发起向服务端发送 SYN 同步报文，同时客户端的状态变为 SYN_SENT 状态，然后等待服务端回应报文。

一般情况下，客户端会在几毫秒内就会收到服务端回应的 ACK 报文。当在异常情况下，客户端迟迟收不到回应报文，则客户端会进行超时重传，而重传次数由参数 tcp_syn_retries 参数控制，默认6次。

```text
# cat /proc/sys/net/ipv4/tcp_syn_retries
6
```

当第 1 次重试发生在 1s 后，后续会以翻倍的方式在第 2、4、8、16、32 秒共做 6 次重试，最后一次重试后会等待 64 秒，若还没有收到对端发送的 ACK 报文，则中止三次握手。所以，总的耗时共 127 秒，超过 2 min。

```text
//发送重传后，需要检测当前资源使用情况
static int tcp_write_timeout(struct sock *sk)
{
struct inet_connection_sock *icsk = inet_csk(sk);
struct tcp_sock *tp = tcp_sk(sk);
int retry_until;
int mss;
/*在建立连接阶段超时，需要检测使用的路由缓存项，并获取重试次数的最大值*/
if ((1 << sk->sk_state) & (TCPF_SYN_SENT | TCPF_SYN_RECV)) {
if (icsk->icsk_retransmits)
dst_negative_advice(&sk->sk_dst_cache);
// 获取重传次数 
retry_until = icsk->icsk_syn_retries ? : sysctl_tcp_syn_retries;
} else {
...
}

/*当重传次数达到建立连接重传上限，超时重传上限或确认连接异常期间重传上限三种上限之一时，
都必须关闭套接口，并需要报告相应的错误*/
if (icsk->icsk_retransmits >= retry_until) {
/* Has it gone just too far? */
tcp_write_err(sk);
return 1;
}
return 0;
}
```

因此可以根据实际生产环境中，可以适当的调低重传次数，以便尽快的把错误暴露给应用程序。

## 2.服务端的优化

当服务端收到客户端发来的 SYN 报文后，服务端回复 SYN+ACK 报文，不仅确认了客户端的序列号，同时把本端的序列号发送给客户端。

此时服务端的状态为 SYN_RCV。在该状态下，服务器创建一个请求对象并加入到 SYN 半连接队列中，该队列中维护未完成的握手信息。当该半连接队列溢出后，服务器将无法建立起新的连接。

![img](https://pic2.zhimg.com/80/v2-f8f3f2b1b3b01eb7526ac40c9d206ea5_720w.webp)



当 SYN 半连接队列满时，服务器端直接丢弃数据包，此时客户端感知不到报文被 server 丢弃，依靠重传定时器重传。

```text
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb,
struct tcphdr *th, unsigned len)
{
...

switch (sk->sk_state) {
case TCP_CLOSE:
goto discard;

case TCP_LISTEN:
if(th->ack)
return 1;

if(th->rst)
goto discard;

//判断是否为 SYN 包
if(th->syn) {
//调用 tcp_v4_conn_request
if (icsk->icsk_af_ops->conn_request(sk, skb) < 0)
return 1;


kfree_skb(skb);
return 0;
}
goto discard;

case TCP_SYN_SENT:

...

return 0;
}


int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{
...

__u32 isn = TCP_SKB_CB(skb)->when; //重传时间戳
struct dst_entry *dst = NULL;
#ifdef CONFIG_SYN_COOKIES
int want_cookie = 0;
#else
#define want_cookie 0 /* Argh, why doesn't gcc optimize this :( */
#endif

/* Never answer to SYNs send to broadcast or multicast */
if (((struct rtable *)skb->dst)->rt_flags &
(RTCF_BROADCAST | RTCF_MULTICAST))
goto drop;

//查看半连接队列是否已满，满了报文丢弃
if (inet_csk_reqsk_queue_is_full(sk) && !isn) {
if (sysctl_tcp_syncookies) {
want_cookie = 1;
} else

goto drop;
}


//在全连接队列满的情况下，如果有 young_ack，那么直接丢弃
if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1)
goto drop;

//分配 request_sock 内核对象
req = reqsk_alloc(&tcp_request_sock_ops);
if (!req)
goto drop;

...

drop:
return 0;
}
```

从代码中可知，tcp 两个队列满了（半连接队列和全连接队列），造成 SYN 报文被丢弃。

可以通过如下命令获取由于队列已满而导致连接失败的次数。

```text
# netstat -s | grep "SYNstoLISTEN"
1192450 SYNs to LISTEN sockets dropped
```

以上命令显示的是由于队列溢出导致 SYN 被丢弃的个数。该值是个累计值，若该数值在持续增加，可以调大 SYN 半连接队列。

半连接队列大小有 tcp_max_syn_backlog 参数控制。

```text
//默认队列最大值为128
# cat /proc/sys/net/ipv4/tcp_max_syn_backlog
128

//设置队列为1024
# echo 1024 > /proc/sys/net/ipv4/tcp_max_syn_backlog

# cat /proc/sys/net/ipv4/tcp_max_syn_backlog
1024
```

从代码中可以看到，当 SYN 半连接队列已满时，有可能会丢弃。但是我们发现，当系统开启 syncookies 时，可以在不使用 SYN 半连接队列的情况下成功建立连接。

syncookies 的工作原理是这样的：服务器根据源地址和目的地址以及客户端序列号信息等生成一个hash值作为服务端的初始序列号，放在本端的SYN+ACK 的回应报文中。当客户端回应 ACK 报文时携带该值，服务端收到 ACK 报文后，取出该值进行校验，若合法，则成功建立连接。

![img](https://pic4.zhimg.com/80/v2-b2ccf5512fe90cea01f79d1adbe86073_720w.webp)

Linux下通过修改 tcp_syncookies 参数来进行开启或关闭。

其中值为 0 表示关闭该功能， 1 表示仅当 SYN 半连接队列已满时，在启用它，2 表示无条件开启。

SYN Flood 攻击就是通过消耗光服务器上的半连接队列来使得正常的用户连接请求无法被响应。因此，应当把 tcp_syncookies 设置为 1，仅在队列满时再启用。

```text
# cat /proc/sys/net/ipv4/tcp_syncookies
1
```

当客户端收到服务端发来的 SYN+ACK 报文后，就会恢复 ACK , 同时本端状态转换为 ESTABLISHED，表示连接成功。而服务端直到接收到客户端发来的ACK 后状态才会变成 ESTABLISHED。

若服务端迟迟收不到 ACK 时，就会超时超时重传 SYN+ACK 报文。当网络繁忙不稳定时，报文丢失就会很严重，因此应该调大重发次数，反之则可以调小。有关重传次数是有参数 tcp_synack_retries 参数控制。

```text
# cat /proc/sys/net/ipv4/tcp_synack_retries
5
```

该参数默认是 5 次，与客户端重发 SYN 类似，它的重试会经历 1、2、4、8、16 秒，最后一次重试后要等待 32 秒，若仍收不到 ACK 报文，才会关闭连接，故共需等待 63 秒。

当服务端收到 ACK 后，此时内核会把连接请求从半连接队列中移出，然后移动到全连接 accept 队列中，等待进程调用 accept 函数把连接取出来。若进程不能及时调用 accept 函数，就会造成 accept 队列溢出，最终会导致建立好的 TCP 连接被丢弃。

```text
struct sock *tcp_check_req(struct sock *sk,struct sk_buff *skb,
struct request_sock *req,
struct request_sock **prev)
{
...

/* In sequence, PAWS is OK. */

if (tmp_opt.saw_tstamp && !after(TCP_SKB_CB(skb)->seq, tcp_rsk(req)->rcv_isn + 1))
req->ts_recent = tmp_opt.rcv_tsval;

...

//创建子 socket 
child = inet_csk(sk)->icsk_af_ops->syn_recv_sock(sk, skb,
req, NULL); //tcp_v4_syn_recv_sock
//当全连接队列满了，会返回空
if (child == NULL)
goto listen_overflow;

//清理半连接队列
inet_csk_reqsk_queue_unlink(sk, req, prev);
inet_csk_reqsk_queue_removed(sk, req);
//把request_sock和生成的sock进行关联，并把request_sock添加到全连接队列
inet_csk_reqsk_queue_add(sk, req, child);
return child;

listen_overflow:
/*sysctl_tcp_abort_on_overflow为0，则直接丢弃客户端发来的ack，返回空，若为1，会走到
下面embryonic_reset标签处，会发送rst报文给对对端。
因此可得:
当收到三次握手中client发送的ack报文后，若全连接队列满了，会有如下处理:
1、sysctl_tcp_abort_on_overflow 为0 server会扔掉client发来的ack，不给对端回应。
2、若为1，则会发送rst报文给对端，废掉本次握手和连接。
*/
if (!sysctl_tcp_abort_on_overflow) {
inet_rsk(req)->acked = 1;
return NULL;
}

embryonic_reset:
NET_INC_STATS_BH(LINUX_MIB_EMBRYONICRSTS);
if (!(flg & TCP_FLAG_RST))
req->rsk_ops->send_reset(sk, skb);

inet_csk_reqsk_queue_drop(sk, req, prev);
return NULL;
}
```

代码中可知，当全连接队列满时，服务器端默认会把 ACK 报文丢弃，不给对端进行回应。

我们也可以设置给对端发送 RST 复位报文进行回应，告诉客户端连接已经建立失败。打开这一功能需要将 tcp_abort_on_overflow 参数设置为 1。

```text
//默认值为0 关闭
# cat /proc/sys/net/ipv4/tcp_abort_on_overflow
0

//打开后，当accept队列满了会回应 RST
# echo 1 > /proc/sys/net/ipv4/tcp_abort_on_overflow
```

通常情况下，应该把 tcp_abort_on_overflow 设置为0，因为这样可以更有利于应对突发流量。

举个例子，当 accept 队列满导致服务器丢掉了 ACK，与此同时，客户端的连接状态却是 ESTABLISHED，客户端进程就在建立好的连接上发送请求。只要服务器没有为请求回复 ACK，客户端的请求就会被多次「重发」。如果服务器上的进程只是短暂的繁忙造成 accept 队列满，那么当 accept 队列有空位时，再次接收到的请求报文由于含有 ACK，仍然会触发服务器端成功建立连接。

所以，tcp_abort_on_overflow 设为 0 可以提高连接建立的成功率，只有你非常肯定 TCP 全连接队列会长期溢出时，才能设置为 1 以尽快通知客户端。

那么怎么调整 accept 队列的长度呢？

accept 队列的长度取决于 somaxconn 和 backlog 之间的最小值，也就是 min(somaxconn, backlog)，其中：

- somaxconn 是 Linux 内核的参数，默认值是 128，可以通过 net.core.somaxconn 来设置其值；

```text
//获取参数值
# sysctl -a | grep net.core.somaxconn
128

//设置长度
# sysctl -w net.core.somaxconn=1024

# sysctl -a | grep net.core.somaxconn
1024
```

- backlog 是 listen (int sockfd, int backlog) 函数中的 backlog 大小；Tomcat、Nginx、Apache 常见的 Web 服务的 backlog 默认值都是 511。

获取accept 队列的长度如下：

```text
/*
-l 显示正在进程的socket
-n 不解析服务名称
-t 只显示 tcp socket
*/
# ss -lnt
State Recv_Q Send_Q Local Address:Port Peer Address:Port
LISTEN 0 1024 *:8090 *:*
...
```

- Recv-Q：当前 accept 队列的大小，也就是当前已完成三次握手并等待服务端 accept() 的 TCP 连接；
- Send-Q：accept 队列最大长度，上面的输出结果说明监听 8088 端口的 TCP 服务，accept 队列的最大长度为 128；

如何查看由于 accep 队列满而导致的连接丢弃？

当 accept 队列满时，服务端则会丢掉后续的 TCP 连接，丢掉的 TCP 连接的个数会统计起来，可以通过如下命令获取：

```text
// 查看tcp accept 队列溢出情况
# netstat -s | grep overflowed
1202 times the listen queue of a socket overflowed
```

其中 1202 times 表示 accept 队列溢出的次数，是个累计值。可以每隔几秒查看一次，若该值一直在增加，说明 accept 队列满了。

如果持续不断地有连接因为 accept 队列溢出被丢弃，就应该调大 backlog 以及 somaxconn 参数。

通过上面可知，可以通过如下设置进行防御 SYN 攻击

- 增大半连接队列。
- 开启 tcp_syncookies 功能。
- 减少SYN+ACK 重传次数。

## 3.通过 TFO 技术绕过三次握手过程

以上只是对三次握手的过程进程优化，但三次握手建立的连接造成的后果就是，当HTTP请求时，必须在一次RTT (Round Trip Time ， 从客户端到服务端一个往返时间) 后才能发送数据。

在 Linux 3.7 内核版本之后，提供了 TCP Fast Open 功能，这个功能可以减少 TCP 连接建立的时延。该功能就是客户端可以在首个 SYN 报文中就携带请求，这就节省了一个 RTT 的时间。

为了能够让客户端在 SYN 报文中携带请求数据，首先要解决服务端的信任问题 。因为此时服务端的 SYN 报文还没有发给客户端，所以客户端是否能够建立起连接还未可知，但此时服务器需要假定连接已经建立成功，并把请求交付给进程去处理，所以服务器必须能够信任这个客户端。

那么 TFO 怎么达成这一目的呢？它把通讯分为 2 个阶段，第一阶段为首次建立连接，这时走三次握手过程，但是在客户端的 SYN 报文中会明确告诉服务端它想使用 TFO 功能，这样服务器就会把客户端 IP 地址用只有自己知道的密钥进行加密，作为 Cookies 携带在返回的 SYN+ACK 报文中，客户端收到后会将 Cookie 缓存在本地。

之后，若客户端再次向服务端建立连接，就可以在第一个 SYN 报文中携带请求数据，同时还要附上缓存的 Cookie。很显然，这种通信方式不能再采用“先connect 再 write 请求”这种编程方式，而要改用 sendto 或者 sendmsg 函数才能实现。

服务器收到后，会用自己的秘钥验证 Cookie 是否合法，验证通过后连接才算建立成功，再把请求交给进程处理，同时给客户端返回 SYN+ACK。虽然客户端收到后还会返回 ACK, 但服务器不等收到 ACK 就可以发送 HTTP 响应了，这就减少了握手带来的1个 RTT 的时间消耗。

![img](https://pic3.zhimg.com/80/v2-4e1412ba7485f9299c167206fc12a75a_720w.webp)

为了防止 SYN 泛洪攻击，服务器的 TFO 实现必须能够自动化的定时更新密钥。

Linux 可以通过修改 tcp_fastopen 参数进行打开 TFO 功能.

```text
# cat /proc/sys/net/ipv4/tcp_fastopen
1
```

由于只有客户端和服务端同时支持时，TFO功能才能使用，所以 tcp_fastopen 参数是按比特位控制的。其中

- 第 1 个比特位为1 时，表示作为客户端时支持 TFO;
- 第 2 个比特位为1时，表示作为服务器时支持 TFO;

所以当 tcp_fastopen 的值为3时（比特为0x11）就表示完全支持 TFO 功能。

原文地址：https://zhuanlan.zhihu.com/p/527441102

作者：linux