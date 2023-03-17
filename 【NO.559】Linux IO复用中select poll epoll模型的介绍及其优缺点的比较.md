# 【NO.559】Linux I/O复用中select poll epoll模型的介绍及其优缺点的比较

关于I/O多路复用：

I/O多路复用(又被称为“事件驱动”)，首先要理解的是，操作系统为你提供了一个功能，当你的某个socket可读或者可写的时候，它可以给你一个通知。这样当配合非阻塞的socket使用时，只有当系统通知我哪个描述符可读了，我才去执行read操作，可以保证每次read都能读到有效数据而不做纯返回-1和EAGAIN的无用功。写操作类似。操作系统的这个功能通过select/poll/epoll之类的系统调用来实现，这些函数都可以同时监视多个描述符的读写就绪状况，这样，**多个描述符的I/O操作都能在一个线程内并发交替地顺序完成，这就叫I/O多路复用，这里的“复用”指的是复用同一个线程。

## 1.I/O复用之select

1.1 介绍：

select系统调用的目的是：在一段指定时间内，监听用户感兴趣的文件描述符上的可读、可写和异常事件。poll和select应该被归类为这样的系统调用，它们可以阻塞地同时探测一组支持非阻塞的IO设备，直至某一个设备触发了事件或者超过了指定的等待时间——也就是说它们的职责不是做IO，而是帮助调用者寻找当前就绪的设备。

下面是select的原理图：



![img](https://pic2.zhimg.com/80/v2-58978583c025aa938901fc953b34601d_720w.webp)



1.2 select系统调用API如下：

```text
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

fd_set结构体是文件描述符集，该结构体实际上是一个整型数组，数组中的每个元素的每一位标记一个文件描述符。fd_set能容纳的文件描述符数量由FD_SETSIZE指定，一般情况下，FD_SETSIZE等于1024，这就限制了select能同时处理的文件描述符的总量。

1.3 下面介绍一下各个参数的含义：

1）nfds参数指定被监听的文件描述符的总数。通常被设置为select监听的所有文件描述符中最大值加1；

2）readfds、writefds、exceptfds分别指向可读、可写和异常等事件对应的文件描述符集合。这三个参数都是传入传出型参数，指的是在调用select之前，用户把关心的可读、可写、或异常的文件描述符通过FD_SET（下面介绍）函数分别添加进readfds、writefds、exceptfds文件描述符集，select将对这些文件描述符集中的文件描述符进行监听，如果有就绪文件描述符，select会重置readfds、writefds、exceptfds文件描述符集来通知应用程序哪些文件描述符就绪。这个特性将导致select函数返回后，再次调用select之前，必须重置我们关心的文件描述符，也就是三个文件描述符集已经不是我们之前传入 的了。

3）timeout参数用来指定select函数的超时时间（下面讲select返回值时还会谈及）。

```text
struct timeval
{
    long tv_sec;        //秒数
    long tv_usec;       //微秒数
};
```

1.4 下面几个函数（宏实现）用来操纵文件描述符集：

```text
void FD_SET(int fd, fd_set *set);   //在set中设置文件描述符fd
void FD_CLR(int fd, fd_set *set);   //清除set中的fd位
int  FD_ISSET(int fd, fd_set *set); //判断set中是否设置了文件描述符fd
void FD_ZERO(fd_set *set);          //清空set中的所有位（在使用文件描述符集前，应该先清空一下）
    //（注意FD_CLR和FD_ZERO的区别，一个是清除某一位，一个是清除所有位）
```

1.5 select的返回情况：

1）如果指定timeout为NULL，select会永远等待下去，直到有一个文件描述符就绪，select返回；

2）如果timeout的指定时间为0，select根本不等待，立即返回；

3）如果指定一段固定时间，则在这一段时间内，如果有指定的文件描述符就绪，select函数返回，如果超过指定时间，select同样返回。

4）返回值情况：

a)超时时间内，如果文件描述符就绪，select返回就绪的文件描述符总数（包括可读、可写和异常），如果没有文件描述符就绪，select返回0；

b)select调用失败时，返回 -1并设置errno，如果收到信号，select返回 -1并设置errno为EINTR。

1.6 文件描述符的就绪条件：

在网络编程中，

1）下列情况下socket可读：

a) socket内核接收缓冲区的字节数大于或等于其低水位标记SO_RCVLOWAT；

b) socket通信的对方关闭连接，此时该socket可读，但是一旦读该socket，会立即返回0（可以用这个方法判断client端是否断开连接）；

c) 监听socket上有新的连接请求；

d) socket上有未处理的错误。

2）下列情况下socket可写：

a) socket内核发送缓冲区的可用字节数大于或等于其低水位标记SO_SNDLOWAT；

b) socket的读端关闭，此时该socket可写，一旦对该socket进行操作，该进程会收到SIGPIPE信号；

c) socket使用connect连接成功之后；

d) socket上有未处理的错误。

## 2.I/O复用之poll

1、poll系统调用的原理与原型和select基本类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有就绪者。

2、poll系统调用API如下：

```text
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

3、下面介绍一下各个参数的含义：

1）第一个参数是指向一个结构数组的第一个元素的指针，每个元素都是一个pollfd结构，用于指定测试某个给定描述符的条件。

```text
struct pollfd
{
    int fd;             //指定要监听的文件描述符
    short events;       //指定监听fd上的什么事件
    short revents;      //fd上事件就绪后，用于保存实际发生的时间
}；
```

待监听的事件由events成员指定，函数在相应的revents成员中返回该描述符的状态（每个文件描述符都有两个事件，一个是传入型的events，一个是传出型的revents，从而避免使用传入传出型参数，注意与select的区别），从而告知应用程序fd上实际发生了哪些事件。events和revents都可以是多个事件的按位或。

2）第二个参数是要监听的文件描述符的个数，也就是数组fds的元素个数；

3）第三个参数意义与select相同。

4、poll的事件类型：



![img](https://pic3.zhimg.com/80/v2-22d23cb65fbcb15d2b3e572186159c0a_720w.webp)



在使用POLLRDHUP时，要在代码开始处定义_GNU_SOURCE

5、poll的返回情况：

与select相同。

## 3.I/O复用之epoll

首先给大家介绍一个[epoll实战视频](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1iJ411S7mv/)，点击即可观看

1、介绍：

epoll 与select和poll在使用和实现上有很大区别。首先，epoll使用一组函数来完成，而不是单独的一个函数；其次，epoll把用户关心的文件描述符上的事件放在内核里的一个事件表中，无须向select和poll那样每次调用都要重复传入文件描述符集合事件集。

2、创建一个文件描述符，指定内核中的事件表：

```text
#include<sys/epoll.h>
int epoll_create(int size);
    //调用成功返回一个文件描述符，失败返回-1并设置errno。
```

size参数并不起作用，只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符指定要访问的内核事件表，是其他所有epoll系统调用的句柄。

3、操作内核事件表：

```text
#include<sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
    //调用成功返回0，调用失败返回-1并设置errno。
```

epfd是epoll_create返回的文件句柄，标识事件表，op指定操作类型。操作类型有以下3种：

- a）EPOLL_CTL_ADD， 往事件表中注册fd上的事件；
- b）EPOLL_CTL_MOD， 修改fd上注册的事件；
- c）EPOLL_CTL_DEL， 删除fd上注册的事件。

event参数指定事件，epoll_event的定义如下：

```text
struct epoll_event
{
    __int32_t events;       //epoll事件
    epoll_data_t data;      //用户数据
};
typedef union epoll_data
{
    void *ptr;
    int  fd;
    uint32_t u32;
    uint64_t u64;
}epoll_data;
```

在使用epoll_ctl时，是把fd添加、修改到内核事件表中，或从内核事件表中删除fd的事件。如果是添加事件到事件表中，可以往data中的fd上添加事件events，或者不用data中的fd，而把fd放到用户数据ptr所指的内存中（因为epoll_data是一个联合体，只能使用其中一个数据）,再设置events。

3、epoll_wait函数

epoll系统调用的最关键的一个函数epoll_wait，它在一段时间内等待一个组文件描述符上的事件。

```text
#include<sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
    //函数调用成功返回就绪文件描述符个数，失败返回-1并设置errno。
```

timeout参数和select与poll相同，指定一个超时时间；maxevents指定最多监听多少个事件；events是一个传出型参数，epoll_wait函数如果检测到事件就绪，就将所有就绪的事件从内核事件表（epfd所指的文件）中复制到events指定的数组中。这个数组用来输出epoll_wait检测到的就绪事件，而不像select与poll那样，这也是epoll与前者最大的区别，下文在比较三者之间的区别时还会说到。

## 4.三组I/O复用函数的比较

相同点：

1）三者都需要在fd上注册用户关心的事件；

2）三者都要一个timeout参数指定超时时间；

不同点：

1）select：

a）select指定三个文件描述符集，分别是可读、可写和异常事件，所以不能更加细致地区分所有可能发生的事件；

b）select如果检测到就绪事件，会在原来的文件描述符上改动，以告知应用程序，文件描述符上发生了什么时间，所以再次调用select时，必须先重置文件描述符；

c）select采用对所有注册的文件描述符集轮询的方式，会返回整个用户注册的事件集合，所以应用程序索引就绪文件的时间复杂度为O(n)；

d）select允许监听的最大文件描述符个数通常有限制，一般是1024，如果大于1024，select的性能会急剧下降；

e）只能工作在LT模式。

2）poll：

a）poll把文件描述符和事件绑定，事件不但可以单独指定，而且可以是多个事件的按位或，这样更加细化了事件的注册，而且poll单独采用一个元素用来保存就绪返回时的结果，这样在下次调用poll时，就不用重置之前注册的事件；

b）poll采用对所有注册的文件描述符集轮询的方式，会返回整个用户注册的事件集合，所以应用程序索引就绪文件的时间复杂度为O(n)。

c）poll用nfds参数指定最多监听多少个文件描述符和事件，这个数能达到系统允许打开的最大文件描述符数目，即65535。

d）只能工作在LT模式。

3）epoll：

a）epoll把用户注册的文件描述符和事件放到内核当中的事件表中，提供了一个独立的系统调用epoll_ctl来管理用户的事件，而且epoll采用回调的方式，一旦有注册的文件描述符就绪，讲触发回调函数，该回调函数将就绪的文件描述符和事件拷贝到用户空间events所管理的内存，这样应用程序索引就绪文件的时间复杂度达到O(1)。

b）epoll_wait使用maxevents来制定最多监听多少个文件描述符和事件，这个数能达到系统允许打开的最大文件描述符数目，即65535；

c）不仅能工作在LT模式，而且还支持ET高效模式（即EPOLLONESHOT事件，读者可以自己查一下这个事件类型，对于epoll的线程安全有很好的帮助）。

select/poll/epoll总结：



![img](https://pic2.zhimg.com/80/v2-e177f887784cbb59f1bd1ce701f2d0d1_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/141447239

作者：linux