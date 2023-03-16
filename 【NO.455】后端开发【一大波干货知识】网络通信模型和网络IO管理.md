# 【NO.455】后端开发【一大波干货知识】网络通信模型和网络IO管理

## 1.简单的C/S通信模型（accept阻塞的话，就只能一个客户端接进来）

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172810_93612.jpg)

## 2.socket()函数

```
//函数原型。返回：若是成功则为非负数，如出错则为 -1
int socket(int domin, int type, int protocol);
//调用 参数1：使用多少位地址AF_INET 为32，参数2：数据报的类型。 参数3：TCP/UDP 协议
int clientfd = socket(AF_INET, SOCK_STREAM, 0);
```

**connect() 函数,用来向服务器建立连接**

```
//函数原型。 返回：成功为 0， 出错为 -1
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen );
```

**bind()函数，绑定地址**

```
//函数原型 返回：成功为 0，出错为 -1
int     bind(int, const struct sockaddr *, socklen_t) __DARWIN_ALIAS(bind);
//使用：
    struct sockaddr_in _addr;
    _addr.sin_family = AF_INET;
    _addr.sin_port = htons(8888);
    _addr.sin_addr.s_addr = INADDR_ANY;
    bind(sockfd, (sockaddr *)&_addr, sizeof(_addr) );
```

listen() 函数，将sock从一个主动套接字转为一个监听套接字

```
//函数原型,返回一个非负数的连接描述符
int  listen(int sockfd, int backlog);
```

accept()函数，等待来自客户端的连接请求，到达监听描述符，然后在adr中填写客户端的套接字地址，并返回一个已连接描述符

```
int accept(int listenfd, struct sockaddr *addr, int *addrlen);
```

一个类似以上模型的小故事：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172811_81817.jpg)

```
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
using namespace std;
int main()
{
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);//创建socket
    struct sockaddr_in _addr;
    _addr.sin_family = AF_INET;
    _addr.sin_port = htons(8888);
    _addr.sin_addr.s_addr = INADDR_ANY;
    socklen_t socklen = sizeof(_addr);
    bind(sockfd, (sockaddr *)&_addr, socklen );
    int listenfd = listen(sockfd, 0);
    struct sockaddr_in clientAddr;
    socklen_t clientlen = sizeof(clientAddr); 
    for (;;){
        int client_sockfd = accept(sockfd,(sockaddr *)&clientAddr, &clientlen);
        char buffer[1024] = {0};
        //等待用户发信息
        ssize_t len_recv = recv(client_sockfd,buffer,1024,0);
        //不做错误处理
        printf("Recv:%s, %d Bytes\n", buffer, len_recv);
        //给用户发信息
        send(client_sockfd,buffer,len_recv,0);
        //关闭这个连接
        close(client_sockfd);
    }
    close(sockfd);
    return 0;
}
```

socket 五元组标识 ：<客户端地址，客户端端口，服务器地址，服务器端口，使用的协议tcp/udp>，五元组中的任何一个元素发生变化都表示一个新的客户端socket连接。

## 3.网络IO

网络IO会涉及到两个系统对象，一个是用户空间调用IO的进程或线程，另一个是内核空间的内核系统，比如发生IO操作read时，它会经历两个阶段：

> 等待数据准备就绪(数据准备)
> 将数据从内核拷贝到进程或者线程中。(数据读写)

因为在以上两个阶段上各有不同的情况，所以出现了多种网络IO模型

**阻塞，非阻塞，同步，异步**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172812_52174.jpg)

## 4.五种IO网络模型

### 4.1.阻塞IO（blocking IO）linux默认情况下都是阻塞IO

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172812_72821.jpg)

> 当用户进程调用了read系统函数，内核就开始第一阶段数据的准备。对于网路IO来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的数据包），这个时候 内核 就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞（数据一直没准备好，就会一直空占CPU）。当 内核一直等到数据准备好了，它就会将数据从 内核 中拷贝到用户内存，然后 内核 返回结果，用户进程才解除 阻塞 的状态，重新运行起来。

### 4.2.非阻塞 (non-blocking IO)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172813_34094.jpg)

> 从上图可以看出，应用进程调用了read函数之后，立刻就有结果返回，这个结果给用户判断结果是个error时，它就知道数据没有准备好，于是它再次发送read函数。直到内核中的数据准备好，并且又再次收到用户进程的system call，那么它马上就将数据拷贝到用户内存，然后返回。所以非阻塞模式IO中，用户进程其实就是不断的主动询问内核数据准备好了没有。
> recv() 返回值大于0时，表示接受数据完毕，返回值既是接收到的字节数
> recv() 返回0，表示连接已正常断开；
> recv() 返回-1，且errno等于EAGAIN，表示recv操作还没执行完成；如果errno不等于EAGAIN，表示recv操作遇到系统错误errno
> 设置非阻塞： fcntl( fd, F_SETFL, O_NONBLOCK ); 不推荐使用，因为循环调用read(),会占用大量资源

### 4.3.IO多路复用

> 当用户进程调用了 select，那么整个进程会被 block，而同时，kernel 会“监视”所 有 select 负责的 socket，当任何一个 socket 中的数据准备好了，select 就会返回。这个时候用户进程再调用 read 操作，将数据从 kernel 拷贝到用户进程。这个图和 blocking IO 的图其实并没有太大的不同，事实上还更差一些。因为这里需要使用两个系统调用(select 和 read)，而 blocking IO 只调用了一个系统调用(read)。但是使用 select 以后最大的优势是用户可以在一个线程内同时处理多个 socket 的 IO 请求。用户可以注册多个 socket，然后不断地调用 select 读取被激活的 socket，即可达到在同一个线程内同时处理多个 IO 请求的目的。而在同步阻塞模型中，必须通过多线程的方式才能达到这个目的。**（所以，如果处理的连接数不是很高的话，使用select/epoll 的 web server 不一定比使用 multi-threading + blocking IO 的 web server 性能更好，可能延迟还更大。select/epoll 的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。）**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172814_16482.jpg)

### 4.4.信号驱动 （signal-driven）

内核在第一阶段是异步，在第二个阶段是同步；于非阻塞IO的区别在于它提供了消息通知机制，不需要用户进程不断的轮询检查，减少了系统API的调用次数，提高效率

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172814_56702.jpg)

### 4.5.异步（asynchronous）

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215172815_87456.jpg)

```
struct aiocb { int aio_fildes off_t aio_offset volatile void *aio_buf size_t aio_nbytes int aio_reqprio struct sigevent aio_sigevent int aio_lio_opcode }
```

> 用户进程发起 read 操作之后，立刻就可以开始去做其它的事。而另一方面，从 kernel的角度，当它受到一个 asynchronous read 之后，首先它会立刻返回，所以不会对用户进程产生任何 block。然后，kernel 会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel 会给用户进程发送一个 signal，告诉它 read 操作完成了。

**总结：经上面的介绍，会发现non-blocking IO 和asynchronous IO的区别还是很明显的。在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程主动的check，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。**
**而asynchronous IO则完全不同。它就像是用户进程将整个IO操作都交给了内核去完成，然后内核做完后发信号通知。在此期间，用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据。**

原文链接：https://zhuanlan.zhihu.com/p/498435536

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)