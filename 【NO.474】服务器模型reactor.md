# 【NO.474】服务器模型reactor

 对高并发编程，网络连接上的消息处理，可以分为两个阶段：等待消息准备好、消息处理。当使用默认的阻塞套接字时，往往是把这两个阶段合而为一，这样操作套接字的代码所在的线程就得睡眠来等待消息准备好，这导致了高并发下线程会频繁的睡眠、唤醒，从而影响了 CPU 的使用效率。

    **高并发编程方法当然就是把两个阶段分开处理。即：等待消息准备好的代码段，与处理消息的代码段是分离的**。当然，这也要求套接字必须是非阻塞的，否则，处理消息的代码段很容易导致条件不满足时，所在线程又进入了睡眠等待阶段。那么问题来了，等待消息准备好这个阶段怎么实现？它毕竟还是等待，这意味着线程还是要睡眠的！**解决办法就是，线程主动查询，或者让 1 个线程为所有连接而等待！**这就是 IO 多路复用了。多路复用就是处理等待消息准备好这件事的，但它可以同时处理多个连接！它也可能“等待”，所以它也会导致线程睡眠，然而这不要紧，因为它一对多、它可以监控所有连接。这样，当我们的线程被唤醒执行时，就一定是有一些连接准备好被我们的代码执行了。
    
    作为一个高性能服务器程序通常需要考虑处理三类事件： I/O 事件、定时事件及信号。

**reactor是什么？怎么理解？**

    reactor是一种设计模式, 是服务器的重要模型, 是一种事件驱动的反应堆模式, 高效的事件处理模型。
    
    reactor 反应堆:  事件来了才执行，事件类型可能不尽相同，所以我们需要提前注册好不同的事件处理函数。事件到来就由 epoll_wait  获取同时到来的多个事件，并且根据数据的不同类型将事件分发给事件处理机制 (事件处理器)， 也就是提前注册的哪些接口函数。
    reactor模型的设计思想和思维方式：它需要的是事件驱动，相应的事件发生，根据事件自动的调用相应的函数，所以需要提前注册好处理函数的接口到reactor中, 函数是由reactor去调用的，而不是再主函数中直接进行调用的, 需要使用回调函数。
    reactor中的 IO 使用的是select poll  epoll 多路复用IO, 以便提高 IO 事件的处理能力，提高IO事件处理效率，支持更高的并发 。
    
    和普通函数调用的不同之处在于：***\*应用程序不是主动的调用某个 API 完成处理，而是恰恰相反，Reactor 逆置了事件处理流程，应用程序需要提供相应的接口并注册到 Reactor 上，如果相应的时间发生，Reactor 将主动调用应用程序注册的接口，这些接口又称为“回调函数”\****。
    
    Reactor 模式是处理并发 I/O 比较常见的一种模式，***\*用于同步 I/O\****，***\*中心思想\****是：***\*将所有要处理的 I/O 事件注册到一个中心 I/O 多路复用器上，同时主线程/进程阻塞在多路复用器上；一旦有 I/O 事件到来或是准备就绪(文件描述符或 socket 可读、写)，多路复用器返回并将事先注册的相应 I/O 事件分发到对应的处理器中\****。

Reactor 模型有三个重要的组件：

    多路复用器：由操作系统提供，在 linux 上一般是 select, poll, epoll 等系统调用。
    
    事件分发器：将多路复用器中返回的就绪事件分到对应的处理函数中。
    
    事件处理器：负责处理特定事件的处理函数。

代码层面主要涉及以下几个方面：

reactor结构体、以及事件结构体的封装：

```
struct ntyevent



{



    int fd;



    int events;



    void *arg;



    int (*callback)(int fd, int events, void *arg);



 



    int status;



    char buffer[BUFFER_LENGTH];



    int length;



    long last_active;



};



 



 



struct ntyreactor



{



    int epfd;



    struct ntyevent *events;



};
```

reactor对应事件fd的注册、新增和删除：

```
void nty_event_set(struct ntyevent *ev, int fd, NCALLBACK callback, void *arg)



{



    ev->fd = fd;



    ev->callback = callback;



    ev->events = 0;



    ev->arg = arg;



    ev->last_active = time(NULL);



 



    return ;



}



 



int nty_event_add(int epfd, int events, struct ntyevent *ev)



{



    struct epoll_event ep_ev = {0, {0}};



    ep_ev.data.ptr = ev;



    ep_ev.events = ev->events = events;



 



    int op;



    if (ev->status == 1)



    {



        op = EPOLL_CTL_MOD;



    } else



    {



        op = EPOLL_CTL_ADD;



        ev->status = 1;



    }



    



    if (epoll_ctl(epfd, op, ev->fd, &ep_ev) < 0) 



    {



        printf("event add failed [fd=%d], events[%d]\n", ev->fd, events);



		return -1;  



    }



 



    return 0;



}



 



int nty_event_del(int epfd, struct ntyevent *ev)



{



    struct epoll_event ep_ev = {0, {0}};



 



    if (ev->status != 1)



    {



        return -1;



    }



 



    ep_ev.data.ptr = ev;



    ev->status = 0;



    epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, &ep_ev);



    



    return 0;



}
```

回调函数：

```
int recv_cb (int fd, int events, void *arg)



{



    struct ntyreactor *reactor = (struct ntyreactor*)arg;



	struct ntyevent *ev = reactor->events+fd;



 



    int len = recv(fd, ev->buffer, BUFFER_LENGTH, 0);



    nty_event_del(reactor->epfd, ev);



 



    if (len > 0)



    {



	    ev->length = len;



		ev->buffer[len] = '\0';



 



		printf("C[%d]:%s\n", fd, ev->buffer);



 



		nty_event_set(ev, fd, send_cb, reactor);



		nty_event_add(reactor->epfd, EPOLLOUT, ev);



    } else if (len == 0)



    {



        close(ev->fd);



        printf("[fd=%d] pos[%ld], closed\n", fd, ev-reactor->events);



    } else



    {



        close(ev->fd);



		printf("recv[fd=%d] error[%d]:%s\n", fd, errno, strerror(errno));



    }  



    return len;



}



 



int send_cb(int fd, int events, void *arg) {



 



	struct ntyreactor *reactor = (struct ntyreactor*)arg;



	struct ntyevent *ev = reactor->events+fd;



 



	int len = send(fd, ev->buffer, ev->length, 0);



	if (len > 0) {



		printf("send[fd=%d], [%d]%s\n", fd, len, ev->buffer);



 



		nty_event_del(reactor->epfd, ev);



		nty_event_set(ev, fd, recv_cb, reactor);



		nty_event_add(reactor->epfd, EPOLLIN, ev);



		



	} else {



 



		close(ev->fd);



 



		nty_event_del(reactor->epfd, ev);



		printf("send[fd=%d] error %s\n", fd, strerror(errno));



 



	}



 



	return len;



}



 



int accept_cb(int fd, int events, void *arg)



{



    struct ntyreactor *reactor = (struct ntyreactor*)arg;



	if (reactor == NULL) return -1;



 



    struct sockaddr_in client_addr;



	socklen_t len = sizeof(client_addr);



 



	int clientfd;



 



    if ((clientfd = accept(fd, (struct sockaddr*)&client_addr, &len)) == -1) {



		if (errno != EAGAIN && errno != EINTR) {



			



		}



		printf("accept: %s\n", strerror(errno));



		return -1;



	}



 



    int i = 0;



    do



    {



		for (i = 3;i < MAX_EPOLL_EVENTS;i ++) {



			if (reactor->events[i].status == 0) {



				break;



			}



		}



		if (i == MAX_EPOLL_EVENTS) {



			printf("%s: max connect limit[%d]\n", __func__, MAX_EPOLL_EVENTS);



			break;



		}        



 



        int flag = 0;



		if ((flag = fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0) {



			printf("%s: fcntl nonblocking failed, %d\n", __func__, MAX_EPOLL_EVENTS);



			break;



		}



        



        nty_event_set(&reactor->events[clientfd], clientfd, recv_cb, reactor);



		nty_event_add(reactor->epfd, EPOLLIN, &reactor->events[clientfd]);



    } while (0);



    



    printf(new connect [%s:%d][time:%ld], pos[%d]\n", 



		inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), reactor->events[i].last_active, i);







    return 0;



}
```

 原文作者：[当当响](https://blog.csdn.net/zhpcsdn921011)

原文链接：https://bbs.csdn.net/topics/606367428