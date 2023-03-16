# 【NO.435】深入剖析阻塞式socket的timeout

## 1.前言

**网络编程中超时时间是一个重要但又容易被忽略的问题,对其的设置需要仔细斟酌。**

本文讨论的是socket设置为阻塞模式，如果socket处于阻塞模式运行时，就需要考虑处理socket操作超时的问题。
所谓阻塞模式，是指其完成指定的操作之前阻塞当前的进程或线程，直到操作有结果返回.
在我们直接调用socket操作函数时，如果不进行特意声明的话，它们都是工作在阻塞模式的，
如 connect, send, recv等.

**简单分类的话，可以将超时处理分成两类：**

**连接(connect)超时;**
**发送(send), 接收(recv)超时;**

## 2.连接超时

从字面上看，连接超时就是在一定时间内还是连接不上目标主机。你所建立的socket连接其实最终都要进行系统调用进入内核态，剩下的就是等待内核通知连接建立。所以自行在代码中设置了超时时间（一般是叫connectTimeout或者socketTimeout），那么这个超时时间一到如果内核还没成功建立连接，那就认为是连接超时了。如果他们没设置超时时间，那么这个connectTimeout就取决于内核什么时候抛出超时异常了。

因此，我们需要分析一下内核是怎么来判断连接超时的。

**内核层的超时分析**

我们都知道一个连接的建立需要经过3次握手，所以连接超时简单的说是是客户端往服务端发的SYN报文没有得到响应（服务端没有返回ACK报文）。

由于网络本身是不稳定的，丢包是很常见的事情（或者对方主机因为某些原因丢弃了该包），因此内核在发送SYN报文没有得到响应后，往往还是进行多次重试。同时，为了避免发送太多的包影响网络，重试的时间间隔还会不断增加。

在linux中，重试的时间间隔会呈指数型增长，为2的N次方，即：

第一次发送SYN报文后等待1s（2的0次幂）后再重试

第二次发送SYN报文后等待2s（2的1次幂）后再重试

第三次发送SYN报文后等待4s（2的2次幂）后再重试

第四次发送SYN报文后等待8s（2的3次幂）后再重试

第五次发送SYN报文后等待16s（2的4次幂）后再重试

第六次发送SYN报文后等待32s（2的5次幂）后再重试

第七次发送SYN报文后等待64s（2的6次幂）后再重试

对于重试次数，由linux的net.ipv4.tcp_syn_retries来确定，默认值一般是6（有些linux发行版可能不太一样），我们可以通过sysctl net.ipv4.tcp_syn_retries查看。比如重试次数是6次，那么我们可以得出超时时间应该是 1+2+4+8+16+32+64=127秒 （上面的第一条是第一次发送SYN报文，不算重试）。

如果我们想修改重试次数，可以输入命令sysctl -w net.ipv4.tcp_syn_retries=5来修改（需要root权限）。如果希望重启后生效，将net.ipv4.tcp_syn_retries = 5放入/etc/sysctl.conf中，之后执行sysctl -p 即可生效。

在一些linux发行版中，重试时间可能会变动。如果想确定操作系统具体的超时时间，可以通过下面这条命令来判断：

```text
gaoke@ubuntu:~$ date; telnet 10.16.15.15 5000; date
Sat Apr  2 14:27:33 CST 2022
Trying 10.16.15.15...
telnet: Unable to connect to remote host: Connection timed out
Sat Apr  2 14:29:40 CST 2022
```

**综合分析**

如果应用层面设置了自己的超时时间，同时内核也有自己的超时时间，那么应该以哪个为准呢？答案是哪个超时时间小以哪个为准。

个人认为，在我们的实际应用中，这个超时时间不宜设置的太长，通常建议2-10s。比如在分布式系统中，我们通常会在多台节点中根据一定策略选择一台进行连接。在有机器宕机的情况下，如果连接超时时间设置的比较长，而我们客户端的线程池又比较小，就很可能大多数的线程都在等待建立连接，过了较长时间才发现连接不上，影响应用的整体吞吐量。

**connect系统调用**

我们观察一下此系统调用的kernel源码，调用栈如下所示:

```text
connect[用户态]
    |->SYSCALL_DEFINE3(connect)[内核态]
            |->sock->ops->connect
```

![img](https://pic2.zhimg.com/80/v2-1f2b925d0c0cf1e582a940e12cfd9c21_720w.webp)

最终调用的tcp_connect源码如下：

```text
int tcp_connect(struct sock *sk) {
    ......
    // 发送SYN
    err = tcp_transmit_skb(sk, buff, 1, sk->sk_allocation);
    ...
    /* Timer for repeating the SYN until an answer. */
    // 由于是刚建立连接，所以其rto是TCP_TIMEOUT_INIT
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                inet_csk(sk)->icsk_rto, TCP_RTO_MAX);
    return 0;    
}
```

又上面代码可知，在tcp_connect设置了重传定时器之后return回了tcp_v4_connect再return到inet_stream_connect。

我们可以采用设置SO_SNDTIMEO来控制connect系统调用的超时,如下所示:

```text
setsockopt(sockfd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len);
```

**不设置SO_SNDTIMEO**

如果不设置SO_SNDTIMEO,那么会由tcp重传定时器在重传超过设置的时候后超时,如下图所示:

![img](https://pic4.zhimg.com/80/v2-b63b820c7acc39c804edc6169a1e8cff_720w.webp)

我们如何查看syn重传次数？:

```text
cat /proc/sys/net/ipv4/tcp_syn_retries
```

![img](https://pic1.zhimg.com/80/v2-a7ecd3ce874b2ae2c0d6a582d78cf524_720w.webp)

对于系统调用，connect的超时时间为:

| tcp_syn_retries | timeout              |
| --------------- | -------------------- |
| 1               | min(so_sndtimeo,3s)  |
| 2               | min(so_sndtimeo,7s)  |
| 3               | min(so_sndtimeo,15s) |
| 4               | min(so_sndtimeo,31s) |
| 5               | min(so_sndtimeo,63s) |

kernel代码版本细微变化

值得注意的是，linux本身官方发布的2.6.32源码对于tcp_syn_retries2的解释和RFC并不一致,不同内核小版本上的实验会有不同的connect timeout表现的原因(有的抓包到的重传SYN时间间隔为3,6,12......)。

以下为代码对比:

```text
========================>linux 内核版本2.6.32-431<========================
#define TCP_TIMEOUT_INIT ((unsigned)(1*HZ))    /* RFC2988bis initial RTO value    */

static inline bool retransmits_timed_out(struct sock *sk,
                     unsigned int boundary,
                     unsigned int timeout,
                     bool syn_set)
{
    ......
    unsigned int rto_base = syn_set ? TCP_TIMEOUT_INIT : TCP_RTO_MIN;
    ......
    timeout = ((2 << boundary) - 1) * rto_base;
    ......

}
========================>linux 内核版本2.6.32.63<========================
#define TCP_TIMEOUT_INIT ((unsigned)(3*HZ))    /* RFC 1122 initial RTO value    */

static inline bool retransmits_timed_out(struct sock *sk,
                     unsigned int boundary
{
    ......
    timeout = ((2 << boundary) - 1) * TCP_RTO_MIN;
    ......
}
```

另外，tcp_syn_retries重传次数可以在单个socket中通过setsockopt设置。

## 3.发送超时

**在tcp连接建立之后，写操作可以理解为向对端发送tcp报文的过程。在tcp的实现中，每一段报文都需要有对端的回应，即ACK报文。和连接时发送SYN报文一样，如果超过一定时间没有收到响应，内核会再次重发该报文。和SYN报文的重试不同的是，linux有另外的参数来控制这个重试次数，即net.ipv4.tcp_retries2，可以通过sysctl net.ipv4.tcp_retries2查看其值。**

另外，这个数据报文重试时间间隔的计算方式也和SYN报文不一样，由于计算方式比较复杂，这里就不详细介绍。

一般linux发行版的net.ipv4.tcp_retries2的默认值为5或者15，对应的超时时间如下表：

tcp_retries2对端无响应

525.6s-51.2s,根据动态rto定

15924.6s-1044.6s,根据动态rto定

和SYN报文的超时时间一样，如果应用层设置了超时时间，哪么具体的超时时间以内核和应用层的超时时间的最小值为准。

socket的write系统调用最后调用的是tcp_sendmsg,源码如下所示:

```text
int tcp_sendmsg(struct kiocb *iocb, struct socket *sock, struct msghdr *msg,
        size_t size){
    ......
    timeo = sock_sndtimeo(sk, flags & MSG_DONTWAIT);
    ......
    while (--iovlen >= 0) {
        ......
        // 此种情况是buffer不够了
        if (copy <= 0) {
    new_segment:
          ......
          if (!sk_stream_memory_free(sk))
              goto wait_for_sndbuf;

          skb = sk_stream_alloc_skb(sk, select_size(sk),sk->sk_allocation);
          if (!skb)
              goto wait_for_memory;
        }
        ......
    }
    ......
    // 这边等待write buffer有空间
wait_for_sndbuf:
        set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
wait_for_memory:
        if (copied)
            tcp_push(sk, flags & ~MSG_MORE, mss_now, TCP_NAGLE_PUSH);
            // 这边等待timeo长的时间
        if ((err = sk_stream_wait_memory(sk, &timeo)) != 0)
            goto do_error;
        ......
out:
    // 如果拷贝了数据，则返回
    if (copied)
        tcp_push(sk, flags, mss_now, tp->nonagle);
    TCP_CHECK_TIMER(sk);
    release_sock(sk);
    return copied;        
out_err:
    // error的处理
    err = sk_stream_error(sk, flags, err);
    TCP_CHECK_TIMER(sk);
    release_sock(sk);
    return err;        
}
```

从上面的内核代码看出，如果socket的write buffer依旧有空间的时候，会立马返回，并不会有timeout。但是write buffer不够的时候，会等待SO_SNDTIMEO的时间(nonblock时候为0)。但是如果SO_SNDTIMEO没有设置的时候,默认初始化为MAX_SCHEDULE_TIMEOUT,可以认为其超时时间为无限。那么其超时时间会有另一个条件来决定，我们看下sk_stream_wait_memory的源码:

```text
int sk_stream_wait_memory(struct sock *sk, long *timeo_p){
        // 等待socket shutdown或者socket出现err
        sk_wait_event(sk, ¤t_timeo, sk->sk_err ||
                          (sk->sk_shutdown & SEND_SHUTDOWN) ||
                          (sk_stream_memory_free(sk) &&
                          !vm_wait));
}
```

在write等待的时候，如果出现socket被shutdown或者socket出现错误的时候，则会跳出wait进而返回错误。在不考虑对端shutdown的情况下,出现sk_err的时间其实就是其write的timeout时间,那么我们看下什么时候出现sk->sk_err。

**SO_SNDTIMEO不设置,write buffer满之后ack一直不返回的情况(例如，物理机宕机)**

物理机宕机后，tcp发送msg的时候,ack不会返回，则会在重传定时器tcp_retransmit_timer到期后timeout,其重传到期时间通过tcp_retries2以及TCP_RTO_MIN计算出来。

tcp_retries2的设置位置为:

```text
cat /proc/sys/net/ipv4/tcp_retries2
```

**SO_SNDTIMEO不设置,write buffer满之后对端不消费，导致buffer一直满的情况**

和上面ack超时有些许不一样的是，一个逻辑是用TCP_RTO_MIN通过tcp_retries2计算出来的时间。另一个是真的通过重传超过tcp_retries2次数来time_out，两者的区别和rto的动态计算有关。但是可以大致认为是一致的。

**上述逻辑如下图所示:**

![img](https://pic2.zhimg.com/80/v2-7dfb1c3a893d4a66f0f8f775eb838f1d_720w.webp)

**write_timeout表格**

| tcp_retries2 | buffer未满 | buffer满                                      |
| ------------ | ---------- | --------------------------------------------- |
| 5            | 立即返回   | min(SO_SNDTIMEO,(25.6s-51.2s)根据动态rto定    |
| 15           | 立即返回   | min(SO_SNDTIMEO,(924.6s-1044.6s)根据动态rto定 |

## 4.接收超时

**在tcp协议中，读的操作和写操作的逻辑是相通的。**

tcp连接建立后，两边的通信无非就是报文的互传。写操作是将数据放到tcp报文中发送给对端，然后等待对端响应，一定时间没有得到响应就是超时。而读操作其实就是发送一个读取数据的报文给对端，然后对端返回带有数据的报文，一定时间没有收到对端的报文则认为超时。对于tcp协议而言，其实不会分辨他们发送的报文具体是要干嘛，因此readTimeout的判断逻辑和writeTimeout基本一样。它的重传次数也是由参数net.ipv4.tcp_retries2控制。在应用层面也一般是统一叫socketTimeout。

**read系统调用**

socket的read系统调用最终调用的是tcp_recvmsg, 其源码如下:

```text
int tcp_recvmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,
        size_t len, int nonblock, int flags, int *addr_len)
{
    ......
    // 这边timeo=SO_RCVTIMEO
    timeo = sock_rcvtimeo(sk, nonblock);
    ......
    do{
        ......
        // 下面这一堆判断表明，如果出现错误,或者已经被CLOSE/SHUTDOWN则跳出循环
        if(copied) {
            if (sk->sk_err ||
                sk->sk_state == TCP_CLOSE ||
                (sk->sk_shutdown & RCV_SHUTDOWN) ||
                !timeo ||
                signal_pending(current))
                break;
        } else {
            if (sock_flag(sk, SOCK_DONE))
                break;

            if (sk->sk_err) {
                copied = sock_error(sk);
                break;
            }
            // 如果socket shudown跳出
            if (sk->sk_shutdown & RCV_SHUTDOWN)
                break;
            // 如果socket close跳出
            if (sk->sk_state == TCP_CLOSE) {
                if (!sock_flag(sk, SOCK_DONE)) {
                    /* This occurs when user tries to read
                     * from never connected socket.
                     */
                    copied = -ENOTCONN;
                    break;
                }
                break;
            }
            .......
        }
        .......

        if (copied >= target) {
            /* Do not sleep, just process backlog. */
            release_sock(sk);
            lock_sock(sk);
        } else /* 如果没有读到target自己数(和水位有关,可以暂认为是1)，则等待SO_RCVTIMEO的时间 */
            sk_wait_data(sk, &timeo);    
    } while (len > 0);
    ......
}
```

上面的逻辑如下图所示:

![img](https://pic1.zhimg.com/80/v2-5670ad1fb88cb0dec2c45c31e0abd03c_720w.webp)

重传以及探测定时器timeout事件的触发时机如下图所示:

![img](https://pic3.zhimg.com/80/v2-0323bbdba2c5e5f382e541e01b84e386_720w.webp)

如果内核层面ack正常返回而且对端窗口不为0，仅仅应用层不返回任何数据,那么就会无限等待，直到对端有数据或者socket close/shutdown为止，如下图所示:

![img](https://pic4.zhimg.com/80/v2-e7edb495529db43528b4e217ccf0d20f_720w.webp)

很多应用就是基于这个无限超时来设计的,例如activemq的消费者逻辑。

**ReadTimeout超时表格**

C系统调用:

| tcp_retries2 | 对端无响应                                    | 对端内核响应正常                 |
| ------------ | --------------------------------------------- | -------------------------------- |
| 5            | min(SO_RCVTIMEO,(25.6s-51.2s)根据动态rto定    | SO_RCVTIMEO==0?无限,SO_RCVTIMEO) |
| 15           | min(SO_RCVTIMEO,(924.6s-1044.6s)根据动态rto定 | SO_RCVTIMEO==0?无限,SO_RCVTIMEO) |

Java系统调用

| tcp_retries2 | 对端无响应                                   | 对端内核响应正常               |      |
| ------------ | -------------------------------------------- | ------------------------------ | ---- |
| 5            | min(SO_TIMEOUT,(25.6s-51.2s)根据动态rto定    | SO_TIMEOUT==0?无限,SO_RCVTIMEO |      |
| 15           | min(SO_TIMEOUT,(924.6s-1044.6s)根据动态rto定 | SO_TIMEOUT==0?无限,SO_RCVTIMEO |      |

## 5.对端物理机宕机之后的超时

**对端物理机宕机后还依旧有数据发送**

对端物理机宕机时对端内核也gg了(不会发出任何包通知宕机)，那么本端发送任何数据给对端都不会有响应。其超时时间就由上面讨论的
min(设置的socket超时[例如SO_TIMEOUT],内核内部的定时器超时来决定)。

**对端物理机宕机后没有数据发送，但在read等待**

这时候如果设置了超时时间timeout，则在timeout后返回。但是，如果仅仅是在read等待，由于底层没有数据交互，那么其无法知道对端是否宕机，所以会一直等待。但是，内核会在一个socket两个小时都没有数据交互情况下(可设置)启动keepalive定时器来探测对端的socket。如下图所示:

![img](https://pic3.zhimg.com/80/v2-b4cbb2fff7bff2ff37dfb5fac067b86e_720w.webp)

大概是2小时11分钟之后会超时返回。keepalive的设置由内核参数指定：

```text
cat /proc/sys/net/ipv4/tcp_keepalive_time 7200 即两个小时后开始探测
cat /proc/sys/net/ipv4/tcp_keepalive_intvl 75 即每次探测间隔为75s
cat /proc/sys/net/ipv4/tcp_keepalve_probes 9 即一共探测9次
```

可以在setsockops中对单独的socket指定是否启用keepalive定时器。

**对端物理机宕机后没有数据发送，也没有read等待**

和上面同理，也是在keepalive定时器超时之后，将连接close。所以我们可以看到一个不活跃的socket在对端物理机突然宕机之后,依旧是ESTABLISHED状态，过很长一段时间之后才会关闭。

## 6.进程宕机后的超时

物理机突然宕机和进程宕掉的表现不一样。一个tcp连接建立后，如果一端的物理机突然宕机，另外一端是完全不知情的，它会像往常一样继续发送相关报文，直到超时时间到了才返回。另外，一般操作系统会有机制检测来释放该tcp连接。而如果只是进程宕掉，在进程退出的时候，操作会负责回收这个进程所属的所有tcp连接，在这时会向这些tcp连接的对端发送FIN报文，表示要关闭连接了，这时候对端是可以知道连接已经关闭的。（如果进程退出后还收到来自对端的报文，那么内核会立马发送reset给对端，从而不会卡住对端的线程资源）

所以如果仅仅是对端进程宕机的话(进程所在内核会close其所拥有的所有socket)，由于fin包的发送，本端内核可以立刻知道当前socket的状态。如果socket是阻塞的，那么将会在当前或者下一次write/read系统调用的时候返回给应用层相应的错误。如果是nonblock，那么会在select/epoll中触发出对应的事件通知应用层去处理。
如果fin包没发送到对端，那么在下一次write/read的时候内核会发送reset包作为回应。

**nonblock**

设置为nonblock=true后，由于read/write都是立刻返回，且通过select/epoll等处理重传超时/probe超时/keep alive超时/socket close等事件，所以根据应用层代码决定其超时特性。定时器超时事件发生的时间如上面几小节所述，和是否nonblock无关。nonblock的编程模式可以让应用层对这些事件做出响应。

## 7.总结

网络编程中超时时间是个重要但又容易被忽略的问题，这个问题只有在遇到物理机宕机，偶尔的网络抖动等平时遇不到的现象时候才会凸显。希望本篇文章可以对读者在以后遇到类似超时问题时有所帮助,只要涉及到阻塞式的网络请求，就一定要加超时，否则你将进入自己曾经种下的苦果，加班，掉的不是汗水，是头发。

原文地址：https://zhuanlan.zhihu.com/p/535405145

作者：linux