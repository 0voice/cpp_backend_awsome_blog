# 【NO.220】【epoll】epoll多路复用和Reactor设计思想

## 1、Reactor设计思想

### 1.1.小前言：

reactor是对epoll的一层封装 ，epoll是对io进行管理，reactor将对io的管理转化为对事件的管理

### 1.2.Reactor必要

#### 1.2.1.传统OIO模式

如图2.1所示为传统IO模式处理示意图：

![img](https://pic3.zhimg.com/80/v2-e642fdd136b6ab64c7bb3a387db57ac6_720w.webp)

图中所示一般是一个请求一个单独的处理线程。

缺点：server的accpet操作是阻塞的，业务处理中的handler中的读写请求也是阻塞的。那么这样的一种IO模式将会导致一个线程的请求没有处理完成无法处理下一个请求，这样就大大降低了吞吐量，这将是一个严重的问题。

为了解决这种问题就出现了一个经典的模式——Connection Per Thread即一个线程处理一个请求。

![img](https://pic1.zhimg.com/80/v2-2b449cc8768b2530cc6d1984d3bd1990_720w.webp)

对于每一个新的请求都会分配一个新的线程来处理，这样的好处就是每个socket的请求相互之间不受影响，每个请求的业务逻辑相互之间也不影响。任何socket的读写操作都不会影响到后面的请求。

缺点：不是每个链接都有请求发生，这样就浪费了很多的线程资源。

这个时候可以采用多路复用IO模型的方式来处理IO事件，使用Reactor将响应IO事件和业务处理分开，一个或多个线程来处理IO事件，然后将就绪得到事件分发到业务处理handlers线程去异步非阻塞处理。

## 2. Reactor模式

### 2.1.单线程Reactor模式

什么是单线程Reactor模式，单线程模式采用一个Reactor线程来【处理套接字、新连接的创建】，并且【将接收到的请求分发到处理器Handler中】。

如图2.2为简单的单线程Reactor模式示意图，Reactor和数据处理(handler)都在一个线程里，图2.2参考doug lea论文《Scalable IO in Java》论文。

![img](https://pic2.zhimg.com/80/v2-0a5e783f2636f511ec715e1e6a289041_720w.webp)

图2.2单线程Reactor模式示意图

### 2.2.单Reactor多线程模式：

![img](https://pic4.zhimg.com/80/v2-45b9e4adbd4aeb8205027d15d1ef445b_720w.webp)

### 2.3.多线程Reactor模式

多线程reactor模式的设计思想就是将handler线程放入到线程次中，在多核的情况下也可以考虑多个Selector选择器来处理事件，如图2.3为简单的多线程Reactor示意图;

![img](https://pic4.zhimg.com/80/v2-66f6e3046461028a473d5eeaae7dc05b_720w.webp)

图2.3多线程Reactor模式示意图

## 3.封装Epoll实现并发

第一次学epoll时，容易错误的认为epoll可以实现并发，其实正确的说法是借助epoll可以实现高性能并发服务器，epoll只是提供了IO复用，在IO复用，真正的并发只能通过线程进程实现。

## 4.Reactor模式：

Reactor模式实现非常简单，使用同步IO模型，即业务线程处理数据需要**主动等待或询问**，主要特点是利用epoll监听listen描述符是否有响应，及时将**客户连接**信息**放于一个队列**，epoll和队列都是在主进程/线程中，由子进程/线程来接管各个描述符，对描述符进行下一步操作，包括connect和数据读写。主程读写就绪事件。

大致流程图如下：

![img](https://pic2.zhimg.com/80/v2-ff471b48d8e48554f93eae27e08f9e49_720w.webp)

Preactor模式：

Preactor模式完全将IO处理和业务分离，使用异步IO模型，即**内核完成数据处理后主动通知给应用处理**，主进程/线程不仅要完成listen任务，还需要完成内核数据缓冲区的映射，直接将数据buff传递给业务线程，业务线程只需要处理业务逻辑即可。

大致流程如下：

![img](https://pic4.zhimg.com/80/v2-22447cac5c8bb8e44c801cc38335140b_720w.webp)

## 5.封装Epoll实现reactor模式的高性能并发服务器

### 5.1.epoll的api

首先介绍epoll的api

```
int epoll_create(int size);  // 创建epfdint epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); //向epfd注册(fd,event)// epoll_event结构体定义struct epoll_event { __uint32_t events; /* epoll 事件 */ epoll_data_t data; /* 传递的数据,用于处理ready的fd,获得上下文关系 */}// data联合体,一般用其指针域ptr,因为要从这个data读取到上下文信息typedef union epoll_data {    void *ptr;    int fd;    uint32_t u32;    uint64_t u64;} epoll_data_t;// op宏 /* EPOLL_CTL_ADD(注册新的fd到epfd) * EPOLL_CTL_MOD(修改已经注册的fd的监听事件) * EPOLL_CTL_DEL(epfd删除一个fd) */* * events : {EPOLLIN, EPOLLOUT, EPOLLPRI,             EPOLLHUP, EPOLLET, EPOLLONESHOT} */int epoll_wait(int epfd, struct epoll_event *event,             int maxevents, int timeout);  //用于轮询注册的fd,若满足相应的注册事件,                                           //  则结束epoll_wait阻塞/* @param timeout 超时时间 *     -1: 永久阻塞 *     0: 立即返回，非阻塞 *     >0: 指定微秒*/
```

### 5.2.Reactor模式:

io模式的历程:

单线程,一般阻塞->多线程,一般阻塞（一条连接一线程）->线程池(减少线程创建销毁开销)->reactor(更小粒度的线程)

所谓更小的粒度的线程是指,传统的多线程是一个连接一个线程,粒度太大,比如可以把一个连接继续细分成三个步骤:read,process,send三个步骤,每个步骤占一个线程,处理完后交给主线程调度,进入下一个处理模块

### 5.3.EPOLL实现的要点

\0. 创建epoll_fd = epoll_create(MAX_EVENT+1)

\1. 维护event数组

1.1 一个交给epoll_wait维护(空的放进去,ready的出来)

1.1.1 特定的结构体 struct epoll_event

1.2 一个自己维护(维持对所有注册事件的监控)(全局)

1.2.1 由于自己维护,想怎么写怎么写,一般的会让epoll_event.data.ptr = myevent,以作为回调函数的参数,保持一个上下文关系

\2. 创建并初始化事件(epoll_event)

2.1 结构体有两个字段 event和data , event是EPOLLIN这样的宏,data的作用是记录一些消息,这样ready的时候可以访问这个消息,比如fd, 回调函数指针,status等等

\3. 向epfd注册事件

3.1 epoll_ctl(epfd , op_macro , target_fd , &epoll_event )

3.2 op_macro是代表epoll_ctl的类型,诸如EPOLL_CTL_ADD

3.3 target_fd是要监听的fd,比如tcp监听套接字ls_socket,epoll_event就是当事件发生时,epoll_wait()里面 event[]将会出现的结构体

\4. 写回调函数

4.1 这是reactor模式的重点.

4.2 典型的,检测到ls_socket可读后(epoll_wait不再阻塞),进入回调函数(通过event[i].data.ptr->callback),假如命名为accept_fn(),在这个函数简单来看要做的事就是

1.conn_socket = accept(ls_socket)

2.创建,初始化epoll_event并注册到epfd(当然还要加进自己维护的fd列表),**由于epoll是轮询的模式,需要将conn_socket用fcntl设为O_NONBLOCK,非阻塞.**

4.3 这个不像select,需要每次重新加入fd到列表里面,注册一次即可

4.4 conn_socket被通过事件EPOLLIN注册到epfd,下一步马上的,epoll_wait检测出conn_socket可读(假设确实可读),然后回调进入recvdata函数(需要在创建并初始化epoll_*event指定, 实际上是自定义一个结构体,让data.ptr指向这个结构体就行),一般的,就以这个结构体组成自己维护的fd list,*

*4.5 recvdata函数首先*调用recv(fd,my_event->buf,…),把待发送的数据存在my_event->buf , 然后修改(fd,my*event)到epfd为发送事件EPOLLOUT,以及改变回调函数为callback_send*

4.6 来到epoll_wait,检测到该fd可写,则回调进入senddata函数,该函数调用send(fd,my*event->buf)发送数据,然后修改fd的注册事件为EPOLLIN(即EPOLL_CTLMOD),清空my_*event->buf….如此反复

注意点:

\1. callback函数如何获得my_event? 我们在epoll_event中能获得event和data,在data.ptr中找到my_event,典型的my_event可能包括回调函数指针,event_type,fd,buf[BUFLEN],last_active_time,…

```
nfd = epoll_wait(epfd , epoll_events , MAX_EVENT+1 , 1000)assert(nfd>0)for i in range(nfd):  my_event* ev = (my_event*) epoll_events[i].data.ptr  // if (epoll_events[i].events& EPOLLIN){}  用按位与的方式检测相等,因为这种flag宏一般都是位不相同的                                    // 用回调函数的方法时,不需要比较event_type,直接调用函数即可  // ev->callback_fn(ev)  不可以    callback_fn的原型应该是void(*callvback_fn)(void*),                                  // 因为定义callback原型的时候,ev还是不完整的类型  ev->callback_fn((void*)ev)
```

原文链接：https://zhuanlan.zhihu.com/p/373963070

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)