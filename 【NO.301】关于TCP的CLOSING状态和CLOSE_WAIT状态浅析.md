# 【NO.301】关于TCP的CLOSING状态和CLOSE_WAIT状态浅析

很多资料讲了关于TCP的CLOSING和CLOSE_WAIT状态以及所谓的优雅关闭的细节，多数侧重与Linux的内核实现(除了《UNIX网络编程》)。本文不注重代码细节，只关注逻辑。所使用的工具，tcpdump，packetdrill以及ss。

关于ss可以先多说几句，它展示的信息跟netstat差不多，只不过更加详细。netstat的信息是通过procfs获取的，本质上来讲就是遍历/proc/net/netstat文件的内容，然后将其组织成可读的形式展示出来，然而ss则可以针对特定的五元组信息提供更加详细的内容，它不再通过procfs，而是用过Netlink来提取特定socket的信息，对于TCP而言，它可以提取到甚至tcp_info这种详细的信息，它包括cwnd，ssthresh，rtt，rto等。

本文展示的逻辑使用了以下三样工具：

1).packetdrill

使用packetdrill构造出一系列的包序列，使得TCP进入CLOSING状态或者CLOSE_WAIT状态。

2).tcpdump/tshark

抓取packetdrill注入的数据包以及协议栈反馈的包，以确认数据包序列确实如TCP标准所述的那样。

3).ss/netstat

通过ss抓取packetdrill相关套接字的tcp_info，再次确认细节。

我想，我使用上述的三件套解析了CLOSING状态之后，接下来的CLOSE_WAIT状态就可以当作练习了。

我来一个一个说。

## 1.关于CLOSING状态

首先我来描述一下而不是细说概念。

什么是CLOSING状态呢？我们来看一下下面的局部状态图：

![img](https://pic4.zhimg.com/80/v2-7545d0378e1a20cf51a0ba2c6a476ed3_720w.webp)

也就是说，当两端都主动发送FIN的时候，并且在收到对方对自己发送的FIN之前收到了对方发送的FIN的时候，两边就都进入了CLOSING状态，这个在状态图上显示的很清楚。这个用俗话说就是”同时关闭“。时序图我就不给出了，请自行搜索或者自己画。

有很多人都说，这种状态的TCP连接在系统中存在了好长时间并百思不得其解。这到底是为什么呢？通过状态图和时序图，我们知道，在进入CLOSING状态后，只要收到了对方对自己的FIN的ACK，就可以双双进入TIME_WAIT状态，因此，如果RTT处在一个可接受的范围内，发出的FIN会很快被ACK从而进入到TIME_WAIT状态，CLOSING状态应该持续的时间特别短。

以下是packetdrill脚本，很简单的一个脚本：

```text
0.000 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
0.000 bind(3, ..., ...) = 0
0.000 listen(3, 1) = 0

0.100 < S 0:0(0) win 32792 <mss 1460,sackOK,nop,nop,nop,wscale 7>
0.100 > S. 0:0(0) ack 1 win 5840 <mss 1460,nop,nop,sackOK,nop,wscale 7>
0.200 < . 1:1(0) ack 1 win 257
0.200 accept(3, ..., ...) = 4

// 象征性写入一些数据，装的像一点一个正常的TCP连接：握手-传输-挥手
0.250 write(4, ..., 1000) = 1000

0.300 < . 1:1(0) ack 1001 win 257

// 主动断开，发送FIN
0.400 close(4) = 0
// 在未对上述close的FIN进行ACK前，先FIN
0.500 < F. 1:1(0) ack 1001 win 260
// 至此，成功进入同时关闭的CLOSING状态。

// 由于packetdrill不能用dup调用，也好用多线程，为了维持进程不退出，只能等待
10000.000 close(4) = 0
```

同时，我启用tcpdump抓包，确认了TCP状态图的细节，即，还没有收到对方对FIN的ACK时，收到了对方的FIN：

![img](https://pic4.zhimg.com/80/v2-8048285969108595a8592e4da74716df_720w.webp)

有个异常，没有收到FIN的ACK(packetdrill没有回复，这正常，因为脚本里本来就没有这个语句)，然而也没有看到重传，此时该连接应该是处于CLOSING状态了，用ss来确认：

CLOSING 1 1 192.168.0.1:webcache 192.0.2.1:54442

cubic wscale:7,7 rto:2000 rtt:50/25 ssthresh:2 send 467.2Kbps rcv_space:5840

果然，进入了CLOSING状态且没有消失，时不我待，当过了2秒以后，ss的结果变成了：

CLOSING 1 1 192.168.0.1:webcache 192.0.2.1:54442

cubic wscale:7,7 rto:4000 rtt:50/25 ssthresh:2 send 467.2Kbps rcv_space:5840

明显在退避！如果继续观察，你会发现rto退避到了64秒之多。在我的场景中，CLOSING状态的套接字维持了两分钟之久。

然而，为什么呢？为什么CLOSING状态会维持这么久？为什么它没有继续维持下去直到永久呢？

很明显，一端的FIN发出去后，没有收到ACK，因此会退避重发，知道4次退避，即2*2*2*2*2*2秒之久。现在的问题是，为什么重发FIN始终不成功呢？要是成功了的话，估计ACK瞬间也就回来了，那么CLOSING状态也就可以进入TIME_WAIT了，但是没有成功重传FIN！

到此为止，我们知道，进入CLOSING状态之后，两边都会等待接收自己FIN的ACK，一旦收到ACK，就会进入TIME_WAIT，如此反复，如果收不到ACK，则会不断重传FIN，直到忍无可忍，将socket销毁。现在，我们集中于解释为什么重传没有成功，但是请记住，并不是每次都这样，只是在我这个packetdrill构造的场景中会有重传不成功，不然如果大概率不成功的话。岂不是每个CLOSING状态都要维持很长时间？？！！



在我的场景下，通过hook重传函数以及抓包确认，发现所有的重传虽然退避了，但是都没有真正将数据包发送出去，究其原因，最终确认问题出在以下代码上：

```text
if (atomic_read(&sk->sk_wmem_alloc) >
    min(sk->sk_wmem_queued + (sk->sk_wmem_queued >> 2), sk->sk_sndbuf))
    return -EAGAIN;
```

在Linux协议栈的实现中，tcp_retransmit_skb由tcp_retransmit_timer调用，即便是这里出了些问题没有重传成功，也还是会退避的，退避超时到期后，继续在这里出错，直到”不可容忍“销毁socket。

我们可以得知，不管如何CLOSING状态的TCP连接即便没有收到对自己FIN的ACK，也不会永久保持下去，保持多久取决于自己发送FIN时刻的RTT，然后RTT计算出的RTO按照最大的退避次数来退避，直到最终执行了固定次数的退避后，算出来的那个比较大的超时时间到期，然后TCP socket就销毁了。

因此，CLOSING状态并不可怕，起码，不管怎样，它有一个可控的销毁时限。

...

现在我来解释重传不成功的细节。

我们知道，根据上述的代码段，sk_wmem_alloc要足够大，大到它比sk_wmem_queued+sk_wmem_queued/4更大的时候，才会返回错误造成重传不成功，然而我们的packetdrill脚本中构造的TCP连接的生命周期中仅仅传输了1000个字节的数据，并且这1000个字节的数据得到了ACK，然后就结束了连接。一个socket保有一个sk_wmem_alloc字段，在skb交给这个socket的时候，该字段会增加skb长度的大小(skb本身大小包括skb数据大小)，然而当skb不再由该socket持有的时候，也就是其被更底层的逻辑接管之后，socket的sk_wmem_alloc字段自然会减去skb长度的大小，这一切的过程由以下的函数决定，即skb_set_owner_w和skb_orphan。我们来看一下这两个函数：

```text
static inline void skb_set_owner_w(struct sk_buff *skb, struct sock *sk)
{
    skb_orphan(skb);
    skb->sk = sk;
    // sock_wfree回调中会递减sk_wmem_alloc相应的大小，其大小就是skb->truesize
    skb->destructor = sock_wfree;
    /*
     * We used to take a refcount on sk, but following operation
     * is enough to guarantee sk_free() wont free this sock until
     * all in-flight packets are completed
     */
    atomic_add(skb->truesize, &sk->sk_wmem_alloc);
}
static inline void skb_orphan(struct sk_buff *skb)
{
    // 调用回调函数，递减sk_wmem_alloc
    if (skb->destructor)
        skb->destructor(skb);
    skb->destructor = NULL;
    skb->sk        = NULL;
}
```

也就是说，只要skb_orphan在skb通向网卡的路径上被正确调用，就会保证sk_wmem_alloc的值随着skb进入socket的管辖时而增加，而被实际发出后而减少。但是根据我的场景，事实好像不是这样，sk_wmem_alloc的值只要发送一个skb就会增加，丝毫没有减少的迹象...这是为什么呢？

有的时候，当你对某个逻辑理解足够深入后，一定要相信自己的判断，内核存在BUG！内核并不完美。我使用的是2.6.32老内核，这个内核我已经使用了6年多，这是我在这个内核上发现的第4个BUG了。

请注意，我的这个场景中，我使用了packetdrill来构造数据包，而packetdrill使用了tun网卡。为什么使用真实网卡甚至使用loopback网卡就不会有问题呢？这进一步引导我去调查tun的代码，果不其然，在其hard_xmit回调中没有调用skb_orphan！也就说说，但凡使用2.6.32内核版本tun驱动的，都会遇到这个问题呢。在tun的xmit中加入skb_orphan之后，问题消失，抓包你会发现大量的FIN重传包，这些重传随着退避而间隔加大(注意，用ss命令比对一下rto字段的值和tcpdump抓取的实际值)：

![img](https://pic3.zhimg.com/80/v2-5efba967d3c83a13ff0e2fc6b0c0a82e_720w.webp)

(为了验证这个，我修改了packetdrill脚本，中间增加了很多的数据传输，以便尽快重现sk_wmem_alloc在使用tun时不递减的问题)于是，我联系了前公司的同事，让他们修改OpenVPN使用的tun驱动代码，因为当时确实出现过关于TCP使用OpenVPN隧道的重传问题，然而，得到的答复却是，xmit函数中已经有skb_orphan了...然后我看了下代码，发现，公司的代码已经不存在问题了，因为我在前年搞tun多队列的时候，已经移植了3.9.6的tun驱动，这个问题已经被修复。

自己曾经做的事情，已然不再忆起...

## 2.关于CLOSE_WAIT状态

和CLOSING状态不同，CLOSE_WAIT状态可能会持续更久更久的时间，导致无用的socket无法释放，这个时间可能与应用进程的生命周期一样久！

我们先看一下CLOSE_WAIT的局部状态图。

![img](https://pic4.zhimg.com/80/v2-9dc0dd527d6b01bec0b401d8178071e3_720w.webp)

然后我来构造一个packetdrill脚本：

```text
0.000 socket(..., SOCK_STREAM, IPPROTO_TCP) = 3
0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
0.000 bind(3, ..., ...) = 0
0.000 listen(3, 1) = 0

0.100 < S 0:0(0) win 32792 <mss 1460,sackOK,nop,nop,nop,wscale 7>
0.100 > S. 0:0(0) ack 1 win 14600 <mss 1460,nop,nop,sackOK,nop,wscale 7>
0.200 < . 1:1(0) ack 1 win 257
0.200 accept(3, ..., ...) = 4
// 什么也不发了，直接断开
0.350 < F. 1:1(0) ack 1 win 260
// 协议栈会对这个FIN进行ACK，然则应用程序不关闭连接的话...
//0.450 close(4) = 0
// 该连接就会变成CLOSE_WAIT，并且只要其socket引用计数不为0，就一直僵死在那里
2000.000 close(4) = 0
```

同样的，我来展示抓包结果：

![img](https://pic4.zhimg.com/80/v2-05ed52c83129233427a63721cefc46eb_720w.webp)

最后，和描述CLOSING状态不同的是，隔了N个小时之后，我来看ss -ip的结果：

CLOSE-WAIT 1 0 192.168.0.1:webcache 192.0.2.1:53753 users:(("ppp",2399,8))

cubic wscale:7,7 rto:300 rtt:100/50 ato:40 cwnd:10 send 1.2Mbps rcv_space:14600

这个CLOSE_WAIT还在！这是为什么呢？

很遗憾，上述的packetdrill脚本并不能直观地展示这个现象，还得靠我说一说。

CLOSE_WAIT是一端在收到FIN之后，发送自己的FIN之前所处的状态，那么很显然，如果一个进程/线程始终不发送FIN，那么在该连接所隶属的socket的生命周期内，这个socket就会一直存在，我们知道，在UNIX/Linux/WinSock中，socket作为一个描述符出现，只要进程/线程继续持有它，它就会一直存在，因此大多数情况下进程/线程的生命周期内，此TCP套接字就会始终处在CLOSE_WAIT状态。进程/线程长时间持有不需要的socket描述符，更多的并不是有意的，而是在进行诸如fork/clone之类的系统调用后，dup了父亲的文件描述符，然后在孩子那里又没有及时关闭，另外的原因就是编程者对socket描述符的close接口以及shutdown接口不是很理解了。

现在，我们用一个问题来继续我们的讨论。

**什么时候进程在超长的生命周期内不会如愿关闭TCP从而发送FIN呢？**

我的答案比较直接： 不能指望close会发送FIN！

相信很多人在想断开一个TCP连接的时候，都会调用close吧。并且这种做法几乎都是正确的，以至于很多人都把这作为一种标准的做法。但是这是不对的！Why？！在《UNIX网络编程》中，曾经提到了所谓的”优雅关闭TCP连接“，何谓优雅？？！如果你充分理解close，shutdown，应该就会知道，CLOSE_WAIT出现，你应该可以给出一些解释。

**close调用**

close的参数只是一个文件描述符号，它不理解这个文件真正的细节，它只是一个文件系统内范畴的一个调用，它只是关闭文件描述符，保证此进程不会在读取它而已。如果你关闭了文件描述符4，即close(4)，你知道4代表的文件会作何反应吗？？文件系统并不知道4号描述符代表的文件到底是什么，更不知道有多少进程共享这个底层的”实体“，所以一个进程层面上逻辑根本没有权力去彻底关闭一个socket。如果你想了解close的细节，更应该去看看UNIX文件抽象或者文件系统的细节，而不是socket。请参见位于fs/open.c中的：

```text
SYSCALL_DEFINE1(close, unsigned int, fd)
{
    ...
    fdt = files_fdtable(files);
    ...
    filp = fdt->fd[fd];
    ...
    retval = filp_close(filp, files);
    ...
    return retval;
    ...
}
EXPORT_SYMBOL(sys_close);
```

在filp_close中会有fput调用：

```text
void fput(struct file *file)
{
    if (atomic_long_dec_and_test(&file->f_count))
        __fput(file);
}
```

看到那个引用计数了吗？只有当这个文件的引用计数变成0的时候，才会调用底层的关闭逻辑，对于socket而言，如果仍然还有一个进程或者线程持有这个socket对应的文件系统的描述符，那么即便你调用了close，也不会进入了socket的close逻辑，它在文件系统层面就返回了！

**shutdown调用**

这个才是真正关闭一个TCP连接的调用！shutdown并没有文件系统的语义，它专门针对内核层的TCP socket。因此，调用shutdown的逻辑，才是真正关闭了与之共享信道的tcp socket。

所谓的优雅关闭，就是在调用close之前， 首先自己调用shutdown(RD or WD)。这样的时序才是关闭TCP的必由之路！

如果你想优雅关闭一个TCP连接，请先用shutdown，然后后面跟一个close。不过有点诡异的是，Linux的shutdown(SHUT_RD)貌似没有任何效果，不过这无所谓了，本来对于读不读的，就不属于TCP的范畴，只有SHUT_WR才会实际发送一个FIN给对方。

原文地址：https://zhuanlan.zhihu.com/p/538326325

作者：linux