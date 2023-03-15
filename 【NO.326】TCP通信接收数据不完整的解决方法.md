# 【NO.326】TCP通信接收数据不完整的解决方法

## 1.TCP协议、Socket编程流程

**TCP/IP协议及socket封装**

![img](https://pic1.zhimg.com/80/v2-afaf8f598167f12d10b204275b21c848_720w.webp)

**套接字的编程流程：**

![img](https://pic4.zhimg.com/80/v2-5c2b1525ffcdab8903ea1c3913efe873_720w.webp)

![img](https://pic4.zhimg.com/80/v2-ae29c79d65b65bc7c62b83efd22d3b1b_720w.webp)

![img](https://pic1.zhimg.com/80/v2-b6cc5fc9eb85b03c2dd259440db5a100_720w.webp)

## 2.Send 和 Recv的基本介绍

**2.1 Send函数**

```text
int send( SOCKET s, const char FAR *buf, int len, int flags );  
```

不论是客户还是服务器应用程序都用send函数来向TCP连接的另一端发送数据。客户程序一般用send函数向服务器发送请求，而服务器则通常用send函数来向客户程序发送应答。

参数说明：

```text
第一个参数指定发送端套接字描述符；
第二个参数指明一个存放应用程序要发送数据的缓冲区；
第三个参数指明实际要发送的数据的字节数；
第四个参数一般置0。
```

这里只描述同步Socket的send函数的执行流程。当调用该函数时，

send先比较待发送数据的长度len和套接字s的发送缓冲的长度，

（1）如果len大于s的发送缓冲区的长度，该函数返回SOCKET_ERROR；

（2）如果len小于或者等于s的发送缓冲区的长度，那么send先检查协议

是否正在发送s的发送缓冲中的数据，如果是就等待协议把数据发送完，如果协议还没有开始发送s的发送缓冲中的数据或者s的发送缓冲中没有数据，那么 send就比较s的发送缓冲区的剩余空间和len，（2.1）如果len大于剩余空间大小send就一直等待协议把s的发送缓冲中的数据发送完，（2.2）如果len小于剩余空间大小send就仅仅把buf中的数据copy到剩余空间里（注意并不是send把s的发送缓冲中的数据传到连接的另一端的，而是协议传的，send仅仅是把buf中的数据copy到s的发送缓冲区的剩余空间里）。

如果send函数copy数据成功，就返回实际copy的字节数，

如果send在copy数据时出现错误，那么send就返回SOCKET_ERROR；如果send在等待协议传送数据时网络断开的话，那么send函数也返回SOCKET_ERROR。

要注意send函数把buf中的数据成功copy到s的发送缓冲的剩余空间里后它就返回了，但是此时这些数据并不一定马上被传到连接的另一端。如果协议在后续的传送过程中出现网络错误的话，那么下一个Socket函数就会返回SOCKET_ERROR。(每一个除send外的Socket函数在执行的最开始总要先等待套接字的发送缓冲中的数据被协议传送完毕才能继续，如果在等待时出现网络错误，那么该Socket函数就返回 SOCKET_ERROR）

注意：在Unix系统下，如果send在等待协议传送数据时网络断开的话，调用send的进程会接收到一个SIGPIPE信号，进程对该信号的默认处理是进程终止。

通过测试发现，异步socket的send函数在网络刚刚断开时还能发送返回相应的字节数，同时使用select检测也是可写的，但是过几秒钟之后，再send就会出错了，返回-1。select也不能检测出可写了。

**2.2 Recv函数**

```text
int recv( SOCKET s,  char FAR *buf, int len, int flags);  
```

不论是客户还是服务器应用程序都用recv函数从TCP连接的另一端接收数据。

参数说明：

```text
第一个参数指定接收端套接字描述符；
第二个参数指明一个缓冲区，该缓冲区用来存放recv函数接收到的数据；
第三个参数指明buf的长度；
第四个参数一般置0。
```

这里只描述同步Socket的recv函数的执行流程。当应用程序调用recv函数时，recv先等待s的发送缓冲中的数据被协议传送完毕，

(1)如果协议在传送s的发送缓冲中的数据时出现网络错误，那么recv函数返回SOCKET_ERROR，

(2)如果s的发送缓冲中没有数据或者数据被协议成功发送完毕后，recv先检查套接字s的接收缓冲区，如果s接收缓冲区中没有数据或者协议正在接收数据，那么recv就一直等待，直到协议把数据接收完毕。当协议把数据接收完毕，recv函数就把s的接收缓冲中的数据copy到buf中

特别提醒：

协议接收到的数据可能大于buf的长度，所以在这种情况下要调用几次recv函数才能把s的接收缓冲中的数据copy完。recv函数仅仅是copy数据，真正的接收数据是协议来完成的,recv函数返回其实际copy的字节数。

如果recv在copy时出错，那么它返回SOCKET_ERROR；

如果recv函数在等待协议接收数据时网络中断了，那么它返回0。

注意：在Unix系统下，如果recv函数在等待协议接收数据时网络断开了，那么调用recv的进程会接收到一个SIGPIPE信号，进程对该信号的默认处理是进程终止。

```text
int send (
   SOCKET s,             
   const char FAR * buf, 
   int len,              
   int flags             
);
```

## 3.常见问题

问题1：send函数每次最多可以发送多少数据？是int的最大值吗？

答：不是int的最大值

问题二：如果buffer中的数据过大，我也只需要调用一次send函数，而底层到底是一次传输成功还是陆续传输我不用管了吗？

答：recv到的数据流可能是断断续续的，你要把他们放在一起然后解码。

问题三：阻塞和非阻塞的区别？

答：Send分为阻塞和非阻塞，

阻塞模式下，如果正常的话，会直到把你所需要发送的数据发完再返回；

非阻塞模式，会根据你的socket在底层的可用缓冲区的大小，来将你的缓冲区当中的数据拷贝过去，有多大缓冲区就拷贝多少，缓冲区满了就立即返回，这个时候的返回值，只表示拷贝到缓冲区多少数据，但是并不代表发送多少数据，同时剩下的部分需要你再次调用send才会再一次拷贝到底层缓冲区。

特别注意：You can use setsockopt to enlarge the buffer.

作为一个套接字，它拥有两个缓冲，接收数据缓冲和发送数据缓冲（此缓冲不同与你自己定义的缓冲），当有数据到达时，首先进入的

就是接收数据缓冲，然后用户从这个缓冲中将数据读出来，这就是套接字接受的过程，这个缓冲的大小可以自己用**SetSocketOpt()**设定，

同时操作系统对它有一个默认大小，如果对方在很短时间内发送大量数据到达这个套接子时，可能它没来得及接收完，因此接收缓冲处于

满的状态，再有数据来的时候就进不去了，因此对方的 SEND可能就返回错误，在对方发送的数据量很小时不会出现这种情况，当数据量很大时，情况就很明显了，很容易造成收不到的情况。同样，发送方的发送缓冲也有相对应的问题。

问题四：缓冲区怎么理解？

答：socket缓冲区每一个socket在被创建之后，系统都会给它分配两个缓冲区，即输入缓冲区和输出缓冲区。

![img](https://pic4.zhimg.com/80/v2-16e9f0ce9bc0aa531c74919eab48556f_720w.webp)

send函数并不是直接将数据传输到网络中，而是负责将数据写入输出缓冲区，数据从输出缓冲区发送到目标主机是由TCP协议完成的。数据写入到输出缓冲区之后，send函数就可以返回了，数据是否发送出去，是否发送成功，何时到达目标主机，都不由它负责了，而是由协议负责。

recv函数也是一样的，它并不是直接从网络中获取数据，而是从输入缓冲区中读取数据。

输入输出缓冲区，系统会为每个socket都单独分配，并且是在socket创建的时候自动生成的。一般来说，默认的输入输出缓冲区大小为8K。套接字关闭的时候，输出缓冲区的数据不会丢失，会由协议发送到另一方；而输入缓冲区的数据则会丢失。

## 4. 对于数据接收不完整的情况，可以考虑从以下几个方面解决：

1：利用SetSocketOpt()函数将接收方套接子接收缓冲设为足够大小；

具体操作：在send()的时候，返回的是实际发送出去的字节(同步)或发送到socket缓冲区的字节(异步);系统默认的状态发送和接收一次为8688字节(约为8.5K)；在实际的过程中发送数据和接收数据量比较大，可以设置socket缓冲区，而避免了send(),recv()不断的循环收发：

```text
// 接收缓冲区
int nRecvBuf=32*1024;//设置为32K
setsockopt(s,SOL_SOCKET,SO_RCVBUF,(const char*)&nRecvBuf,sizeof(int));
//发送缓冲区
int nSendBuf=32*1024;//设置为32K
setsockopt(s,SOL_SOCKET,SO_SNDBUF,(const char*)&nSendBuf,sizeof(int));
```

2.基于winsock API，比较实用，自己写的，简单又粗暴同时还有技巧~

这样包装的目的显而易见，防止send或者 recv不完整，这样你想发一个

几MB直接调用下面方法就okay，不会少发~

```text
bool SendAll(SOCKET &sock, char*buffer, int size)
{
    while (size>0)
    {
        int SendSize= send(sock, buffer, size, 0);
        if(SOCKET_ERROR==SendSize)
            return false;
        size = size - SendSize;//用于循环发送且退出功能
        buffer+=SendSize;//用于计算已发buffer的偏移量
    }
    return true;
}

bool RecvAll(SOCKET &sock, char*buffer, int size)
{
    while (size>0)//剩余部分大于0
    {
        int RecvSize= recv(sock, buffer, size, 0);
        if(SOCKET_ERROR==RecvSize)
            return false;
        size = size - RecvSize;
        buffer+=RecvSize;
    }
    return true;
}
```

3.设置为阻塞方式：

阻塞就是干不完不准回来！

非阻塞就是你先干,我现看看有其他事没有,完了告诉我一声！

sock默认为阻塞模式，下面的代码可对sock设置为非阻塞模式

```text
int flags = fcntl(sock, F_GETFL, 0);     
fcntl(sock, F_SETFL, flags | O_NONBLOCK); 
```

假设当前代码为服务器，并且已经执行过如下代码，

当sock为阻塞模式，调用accept会阻塞直到一个请求到来

当sock为非阻塞模式，accept会返回-1，errno设置为EAGAIN或者EWOULDBLOCK

3.在recv函数之前加sleep（0.01）函数,而不是recv之后，但是感觉这样没什么效果。

4：在发送方进行数据发送时判断发送是否成功，如果不成功重发；

5：要求接收方收到数据后给发送方回应，发送方只在收到回应后才发送下一条数据。

原文地址：https://zhuanlan.zhihu.com/p/516284323

作者：linux