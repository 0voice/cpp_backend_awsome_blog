# 【NO.603】TCP协议之Send和Recv原理及常见问题分析

## 1.Send函数

```text
int send( SOCKET s, const char FAR *buf, int len, int flags );  
```

不论是客户还是服务器应用程序都用send函数来向TCP连接的另一端发送数据。客户程序一般用send函数向服务器发送请求，而服务器则通常用send函数来向客户程序发送应答。

参数说明：

- 第一个参数指定发送端套接字描述符；
- 第二个参数指明一个存放应用程序要发送数据的缓冲区；
- 第三个参数指明实际要发送的数据的字节数。
- 第四个参数一般置0。

这里只描述同步Socket的send函数的执行流程。当调用该函数时，

send先比较待发送数据的长度len和套接字s的发送缓冲的长度，

1. 如果len大于s的发送缓冲区的长度，该函数返回SOCKET_ERROR；
2. 如果len小于或者等于s的发送缓冲区的长度，那么send先检查协议是否正在发送s的发送缓冲中的数据，如果是就等待协议把数据发送完，如果协议还没有开始发送s的发送缓冲中的数据或者s的发送缓冲中没有数据，那么 send就比较s的发送缓冲区的剩余空间和len；

针对第二种情况：

2.1 如果len大于剩余空间大小send就一直等待协议把s的发送缓冲中的数据发送完；

2.2 如果len小于剩余空间大小send就仅仅把buf中的数据copy到剩余空间里（注意并不是send把s的发送缓冲中的数据传到连接的另一端的，而是协议传的，send仅仅是把buf中的数据copy到s的发送缓冲区的剩余空间里）。如果send函数copy数据成功，就返回实际copy的字节数，如果send在copy数据时出现错误，那么send就返回SOCKET_ERROR；如果send在等待协议传送数据时网络断开的话，那么send函数也返回SOCKET_ERROR。

要注意send函数把buf中的数据成功copy到s的发送缓冲的剩余空间里后它就返回了，但是此时这些数据并不一定马上被传到连接的另一端。如果协议在后续的传送过程中出现网络错误的话，那么下一个Socket函数就会返回SOCKET_ERROR。(每一个除send外的Socket函数在执行的最开始总要先等待套接字的发送缓冲中的数据被协议传送完毕才能继续，如果在等待时出现网络错误，那么该Socket函数就返回 SOCKET_ERROR）

注意：在Unix系统下，如果send在等待协议传送数据时网络断开的话，调用send的进程会接收到一个SIGPIPE信号，进程对该信号的默认处理是进程终止。

通过测试发现，异步socket的send函数在网络刚刚断开时还能发送返回相应的字节数，同时使用select检测也是可写的，但是过几秒钟之后，再send就会出错了，返回-1。select也不能检测出可写了。

## 2.send函数发送原理

tcp协议本身是可靠的,并不等于应用程序用tcp发送数据就一定是可靠的.不管是否阻塞,send发送的大小,并不代表对端recv到多少的数据.

在阻塞模式下,send函数的过程是将应用程序请求发送的数据拷贝到发送缓存中发送并得到确认后再返回.但由于发送缓存的存在,表现为:如果发送缓存大小比请求发送的大小要大,那么send函数立即返回,同时向网络中发送数据;否则,send向网络发送缓存中不能容纳的那部分数据,并等待对端确认后再返回(接收端只要将数据收到接收缓存中,就会确认,并不一定要等待应用程序调用recv);

在非阻塞模式下,send函数的过程仅仅是将数据拷贝到协议栈的缓存区而已,如果缓存区可用空间不够,则尽能力的拷贝,返回成功拷贝的大小;如缓存区可用空间为0,则返回-1,同时设置errno为EAGAIN.

## 3.实例分析send／recv的异常情况

在实际应用中,如果发送端是非阻塞发送,由于网络的阻塞或者接收端处理过慢,通常出现的情况是,发送应用程序看起来发送了10k的数据,但是只发送了2k到对端缓存中,还有8k在本机缓存中(未发送或者未得到接收端的确认).那么此时,接收应用程序能够收到的数据为2k.假如接收应用程序调用recv函数获取了1k的数据在处理,在这个瞬间,发生了以下情况之一:

发送应用程序认为send完了10k数据,关闭了socket:

发送主机作为tcp的主动关闭者,连接将处于FIN_WAIT1的半关闭状态(等待对方的ack),并且,发送缓存中的8k数据并不清除,依然会发送给对端.如果接收应用程序依然在recv,那么它会收到余下的8k数据(这个前题是,接收端会在发送端FIN_WAIT1状态超时前收到余下的8k数据.),然后得到一个对端socket被关闭的消息(recv返回0).这时,应该进行关闭.

发送应用程序再次调用send发送8k的数据:

假如发送缓存的空间为20k,那么发送缓存可用空间为20-8=12k,大于请求发送的8k,所以send函数将数据做拷贝后,并立即返回8192;假如发送缓存的空间为12k,那么此时发送缓存可用空间还有12-8=4k,send()会返回4096,应用程序发现返回的值小于请求发送的大小值后,可以认为缓存区已满,这时必须阻塞(或通过select等待下一次socket可写的信号),如果应用程序不理会,立即再次调用send,那么会得到-1的值,在linux下表现为errno=EAGAIN.

接收应用程序在处理完1k数据后,关闭了socket:

接收主机作为主动关闭者,连接将处于FIN_WAIT1的半关闭状态(等待对方的ack).然后,发送应用程序会收到socket可读的信号(通常是select调用返回socket可读),但在读取时会发现recv函数返回0,这时应该调用close函数来关闭socket(发送给对方ack);如果发送应用程序没有处理这个可读的信号,而是继续调用send,那么第一次会像往常一样继续填充缓存区,然后返回,但如果再次调用send,进程会收到SIGPIPE信号,该信号的默认响应动作是退出进程.

交换机或路由器的网络断开:

接收应用程序在处理完已收到的1k数据后,会继续从缓存区读取余下的1k数据,然后就表现为无数据可读的现象,这种情况需要应用程序来处理超时.一般做法是设定一个select等待的最大时间,如果超出这个时间依然没有数据可读,则认为socket已不可用.发送应用程序会不断的将余下的数据发送到网络上,但始终得不到确认,所以缓存区的可用空间持续为0,这种情况也需要应用程序来处理.

如果不由应用程序来处理这种情况超时的情况,也可以通过tcp协议本身来处理,具体可以查看sysctl项中的:

net.ipv4.tcp_keepalive_intvl

net.ipv4.tcp_keepalive_probes

net.ipv4.tcp_keepalive_time

## 4.Recv函数

```text
int recv( SOCKET s,  char FAR *buf, int len, int flags);  
```

不论是客户还是服务器应用程序都用recv函数从TCP连接的另一端接收数据。

1. 第一个参数指定接收端套接字描述符；
2. 第二个参数指明一个缓冲区，该缓冲区用来存放recv函数接收到的数据；
3. 第三个参数指明buf的长度；
4. 第四个参数一般置0。

这里只描述同步Socket的recv函数的执行流程。当应用程序调用recv函数时，recv先等待s的发送缓冲中的数据被协议传送完毕，

如果协议在传送s的发送缓冲中的数据时出现网络错误，那么recv函数返回SOCKET_ERROR；

如果s的发送缓冲中没有数据或者数据被协议成功发送完毕后，recv先检查套接字s的接收缓冲区，如果s接收缓冲区中没有数据或者协议正在接收数据，那么recv就一直等待，直到协议把数据接收完毕。当协议把数据接收完毕，recv函数就把s的接收缓冲中的数据copy到buf中；

协议接收到的数据可能大于buf的长度，所以在这种情况下要调用几次recv函数才能把s的接收缓冲中的数据copy完。recv函数仅仅是copy数据，真正的接收数据是协议来完成的,recv函数返回其实际copy的字节数。如果recv在copy时出错，那么它返回SOCKET_ERROR；如果recv函数在等待协议接收数据时网络中断了，那么它返回0。

注意：在Unix系统下，如果recv函数在等待协议接收数据时网络断开了，那么调用recv的进程会接收到一个SIGPIPE信号，进程对该信号的默认处理是进程终止。

## 5.**send返回值**

在Unix系统下，如果send 、 recv 、 write在等待协议传送数据时 ， socket 被 shutdown，调用send的进程会接收到一个SIGPIPE信号，进程对该信号的默认处理是进程终止。

SIGPIPE 信号：

对一个已经收到FIN包的socket调用read方法, 如果接收缓冲已空, 则返回0, 这就是常说的表示连接关闭. 但第一次对其调用write方法 时, 如果发送缓冲没问题, 会返回正确写入(发送). 但发送的报文会导致对端发送RST报文, 因为对端的socket已经调用了close, 完全 关闭, 既不发送, 也不接收数据. 所以, 第二次调用write方法(假设在收到RST之后), 会生成SIGPIPE信号, 导致进程退出 。如果对 SIGPIPE 进行忽略处理， 二次调用write方法时, 会返回-1, 同时errno置为SIGPIPE。

处理方法：

在初始化时调用 signal(SIGPIPE,SIG_IGN) 忽略该信号（只需一次） ，SIGPIPE交给了系统处理。 此时 send 、recv或write 函数将返回-1，errno为EPIPE，可视情况关闭socket或其他处理

SIGPIPE 被忽略的情况下，如果 服务器采用了fork的话，要收集垃圾进程，防止僵尸进程的产生，可以这样处理： signal(SIGCHLD,SIG_IGN);　交给系统init去回收。 这样 子进程就不会产生僵尸进程了。

**小结**

在Linux环境下开发经常会碰到很多错误(设置errno)，其中EAGAIN是其中比较常见的一个错误(比如用在非阻塞操作中)。

从字面上来看，是提示再试一次。这个错误经常出现在当应用程序进行一些非阻塞(non-blocking)操作(对文件或socket)的时候。例如，以 O_NONBLOCK的标志打开文件/socket/FIFO，如果你连续做read操作而没有数据可读。此时程序不会阻塞起来等待数据准备就绪返 回，read函数会返回一个错误EAGAIN，提示你的应用程序现在没有数据可读请稍后再试。

又例如，当一个系统调用(比如fork)因为没有足够的资源(比如虚拟内存)而执行失败，返回EAGAIN提示其再调用一次(也许下次就能成功)。

Linux - 非阻塞socket编程处理EAGAIN错误

在linux进行非阻塞的socket接收数据时经常出现Resource temporarily unavailable，errno代码为11(EAGAIN)，这是什么意思？

这表明你在非阻塞模式下调用了阻塞操作，在该操作没有完成就返回这个错误，这个错误不会破坏socket的同步，不用管它，下次循环接着recv就可以。 对非阻塞socket而言，EAGAIN不是一种错误。在VxWorks和Windows上，EAGAIN的名字叫做EWOULDBLOCK。

另外，如果出现EINTR即errno为4，错误描述Interrupted system call，操作也应该继续。

最后，如果recv的返回值为0，那表明连接已经断开，我们的接收操作也应该结束。

当客户通过Socket提供的send函数发送大的数据包时，就可能返回一个EGGAIN的错误。该错误产生的原因是由于send函数 中的size变量大小超过了tcp_sendspace的值。tcp_sendspace定义了应用在调用send之前能够在kernel中缓存的数据 量。当应用程序在socket中设置了O_NDELAY或者O_NONBLOCK属性后，如果发送缓存被占满，send就会返回EAGAIN的错误。

为了消除该错误，有以下几种方法可以选择：

1.调大tcp_sendspace，使之大于send中的size参数

—no -p -o tcp_sendspace=65536

2.在调用send前，在setsockopt函数中为SNDBUF设置更大的值

你自己的缓冲区满了，会返回EAGAIN，你的没满，对方的缓冲区满了,肯定不关你事，可能会发送不成功，但是协议栈提供的系统调用，只管数据成功从你的缓冲区发出去，之后人家因为缓冲区满收不到数据，tcp自己有重传机制（参考Tcp/ip详解卷1）。

send()适用于已连接的数据包或流式套接口发送数据。对于数据报类套接口，必需注意发送数据长度不应超过通讯子网的IP包最大长度。IP包最大长度在WSAStartup()调用返回的WSAData的iMaxUdpDg元素中。如果数据太长无法自动通过下层协议，则返回WSAEMSGSIZE错误，数据不会被发送。

请注意成功地完成send()调用并不意味着数据传送到达。

如果传送系统的缓冲区空间不够保存需传送的数据，除非套接口处于非阻塞I/O方式，否则send()将阻塞。对于非阻塞SOCK_STREAM类型的套接口，实际写的数据数目可能在1到所需大小之间，其值取决于本地和远端主机的缓冲区大小。可用select()调用来确定何时能够进一步发送数据。

## 6.服务端判断客户端断开的经验方法

当recv()返回值小于等于0时，socket连接断开。但是还需要判断 errno是否等于 EINTR，如果errno == EINTR

则说明recv函数是由于程序接收到信号后返回的，socket连接还是正常的，不应close掉socket连接。

主动检测

```text
struct tcp_info info; 
int len=sizeof(info); 
getsockopt(sock, IPPROTO_TCP, TCP_INFO, &info, (socklen_t *)&len); 
if((info.tcpi_state==TCP_ESTABLISHED))  
    //则说明未断开  
else
    //断开
```

若使用了select等系统函数，若远端断开，则select返回1，recv返回0则断开。其他注意事项同法一。

keepalive

```text
int keepAlive = 1; // 开启keepalive属性
int keepIdle = 60; // 如该连接在60秒内没有任何数据往来,则进行探测 
int keepInterval = 5; // 探测时发包的时间间隔为5 秒
int keepCount = 3; // 探测尝试的次数.如果第1次探测包就收到响应了,则后2次的不再发.
 
setsockopt(rs, SOL_SOCKET, SO_KEEPALIVE, (void *)&keepAlive, sizeof(keepAlive));
setsockopt(rs, SOL_TCP, TCP_KEEPIDLE, (void*)&keepIdle, sizeof(keepIdle));
setsockopt(rs, SOL_TCP, TCP_KEEPINTVL, (void *)&keepInterval, sizeof(keepInterval));
setsockopt(rs, SOL_TCP, TCP_KEEPCNT, (void *)&keepCount, sizeof(keepCount));
设置后，若断开，则在使用该socket读写时立即失败，并返回ETIMEDOUT错误
```

自己实现一个心跳检测，一定时间内未收到自定义的心跳包则标记为已断开。

原文地址：https://zhuanlan.zhihu.com/p/516485158

作者：CPP后端技术