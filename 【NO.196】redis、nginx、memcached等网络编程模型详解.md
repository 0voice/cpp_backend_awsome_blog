# 【NO.196】redis、nginx、memcached等网络编程模型详解

说到网络编程，就要把下面四个方面处理好。

## 1.网络连接

分为两种：服务端处理接收客户端的连接，服务端作为客户端连接第三方服务

来自客户端的连接，监听accept有收到EPOLLIN事件，或者当前服务器连接上游服务器，进行connect时返回-1，errno为EINPROGRESS，此时再收到EPOLLOUT事件就代表连接上了，因为三次握手最后是需要回复ack给上游服务器。（Connect非阻塞 ，在图中箭头出表示建立成功（需要注册写事件：最终客户端还要给服务器发送个ack确认请求才能建立成功））

![img](https://pic1.zhimg.com/80/v2-2cd7005df924cfb5fdeaf291e993be5c_720w.webp)



```text
int clientfd=accept(listenfd, addr, sz);

//举例为非阻塞io， 阻塞io成功直接返回0
int connectfd=socket(AF_INET,SOCK_STREAM,0);
int ret=connect(connectfd,(struct sockaddr *)&addr,sizeof(addr));
// ret==-1 && errno==EINPROGERESS 正在建立连接
//  ret==-1 && errno==EISCONN 连接建立成功
```

## 2. 网络断开

当客户端断开时，服务端read返回0，或者收到EPOLLRDHUP事件，如果服务端要支持半关闭状态，就关闭读端shutdown(SHUT_RD)，如果不需要支持，直接close即可，大部分都是直接close，一般close前也会进行类似释放资源的操作，如果这步操作比较耗时，可以异步处理，否则导致服务端出现大量的close_wait状态。

如果是服务端主动断开连接时，通过shutdown(SHUT_WR)发送FIN包给客户端，**此时再调用write会返回-1，errno为EPIPE，代表写通道已经关闭。**这里要注意close和shutdown的区别，close时如果fd的引用不为0，是不会真正的释放资源的，比如fd1=dup(fd2)，close(fd2)不会对fd1造成影响，而shutdown则跳过了前面的引用计数检查，直接对网络进行操作，多线程多进程下用close比较好。断开连接时，如果发现接收缓冲区还有数据，直接丢弃，并回复RST包，如果是发送缓冲区还有数据，则会取消nagle进行发送，末尾加上FIN包，如果开启了SO_LINGER，则会在linger_time内等待FIN包的ack，这样保证发送缓冲区的数据被对端接收到。

```text
//主动关闭
close(fd);
shutdown(fd,SHUT_RDWR);
//主动关闭本地读端，对端写段关闭
shutdown(fd,SHUT_RD);
//主动关闭本地写端，对端读端关闭
shutdown(fd,SHUT_WR);

//被动：读端关闭
//有的网络编程需要支持半关闭状态
int n=read(fd,buf,sz);
if(n==0)
{
	close_read(fd);
	//write();
	//close(fd);
}
//被动： 写端关闭
int n=write(fd,buf,sz);
if(n==-1 && errno==EPIPE)
{
	close_write(fd);
	//close(fd);
}
```

## 3. 消息到达

如果read大于0，接收数据正常，处理对应的业务逻辑即可，如果read等于0，说明对端发送了FIN包，如果read小于0，此时要根据errno进行下一步的判断处理，如果是EWOULDBLOCK或者EAGAIN，说明接收缓冲区还没有数据，直接重试即可，如果是EINTR，说明被信号中断了，因为信号中断的优先级比系统调用高，此时也是重试read即可（read从内核态到用户态：正向错误（被信号打断），如EWOULDBLOCK EINRT 还可以正常的读下一次，其他错误直接close），如果是ETIMEDOUT，说明探活超时了，每个socket都有一个tcp_keepalive_timer，当超过tcp_keepalive_time没有进行数据交换时，开始发送探活包，如果探测失败，间隔tcp_keepalive_intvl时间发送下一次探活包，最多连续发送tcp_keepalive_probes次，如果都失败了，则关闭连接，返回ETIMEDOUT错误。

这些探活都是在传输层进行的，如果应用层的进程有死锁或者阻塞，它是检测不到的，这种情况需要在应用层自行加入心跳包机制来进行检测。一般客户端与数据库之间，反向代理与服务器直接直接用探活机制就行，但数据库之间主从复制以及客户端与服务器之间需要加入心跳机制，以防进程有阻塞。

```text
int n=read(fd,buf,sz);
if(n<0)
{
	//n==-1
	if(errno==EINTR || errno==EWOULDBLOCK)
		break;
	close(fd);
}
else if(n==0)
{
	close(fd);
}
else
{
	//处理buf
}
```

## 4. 消息发送

第四个是消息发送，write大于0，消息放入了发送缓冲区，write小于0，同样要分errno的情况处理，如果错误码为EWOULDBLOCK，说明发送缓冲区还装不下你要发送的数据，需要重试，如果是EINTR，说明write系统调用被信号中断了，同样进行重试处理，如果是EPIPE，说明写通道已经关闭了。

```text
int n=write(fd,buf,sz);
if(n==-1)
{
	if(errno==EINTR || errno==EWOULDBLOCK)
		return;
	close(fd);
}
```

**常见网络io模型**

阻塞io和非阻塞io指的是内核**数据准备阶段要不要阻塞等待**，如果内核数据准备好了，将数据从内核拷贝至用户空间还是阻塞的，所以它们都为同步io

![img](https://pic4.zhimg.com/80/v2-a15a416b83fad517ef2d8970fe7ab04b_720w.webp)

阻塞io和非阻塞io：

（1）阻塞在网络线程
（2）连接的fd阻塞属性决定了io函数是否阻塞
（3）具体差异在：io函数在数据未到达时是否立刻返回；

```text
//默认情况下，fd是阻塞的，设置非阻塞的方法如下：
int flag=fcntl(fd,F_GETFL,0);
fcntl(fd,F_SETFL,flag | O_NONBLOCK);
```



## 5.reactor的应用

目前大部分高性能网络中间件都是采用io多路复用加事件处理的机制，也就是reactor模型

### 5.1redis（单reactor）

redis是单reactor模型，只有一个epoll对象，主线程就是一个循环，不断的处理epoll事件， 首先处理accept事件，将接入的连接绑定读事件处理函数后加入epoll中，等epoll检测到连接有读事件到来时，触发读事件处理函数，但这个函数并没有真正的去读数据，而是将该有读事件到来的连接放入clients_pending_read任务队列中，主线程循环到下一次epoll_wait前，再将这些任务队列中的连接分配给各个io线程本地的任务队列io_threads_list处理，在io线程里，不断对io_threads_pending[i]原子变量进行判断，看有没有值，有就代表主线程给派了任务，然后根据io_threads_op任务类型对任务进行读或写处理，先说读处理部分，主要读取客户端数据并进行解析命令处理，将命令读到该连接对应的querybuf中，主线程忙轮询，等待所有io线程解析命令完成，再主线程开始执行命令，因为这些都是内存操作，所以单线程就可以，如果用多线程的话还有锁的问题，执行完命令后将相应结果写入连接对应的buf数组中，如果放不下就放入reply链表中，然后再将各个连接的响应客户端的任务放入clients_pending_write任务队列中，主线程再分配给各个io线程进行写处理，将数据响应给客户端，主线程此时也是忙轮询，等待所有io线程完成，如果最后发现还有数据没发送完，就注册epollout写事件sendReplyToClient，等客户端可写时再把数据发送完。可以看出网络io相关的操作使用了多线程处理，但是命令执行等纯内存操作都是单线程完成的，全程只有一个eventLoop即相当于epollfd，这种就是单reactor模型，每个io线程都有自己的任务队列io_threads_list，所以也没有多线程竞争的问题。

![img](https://pic2.zhimg.com/80/v2-3f1d06072e1621c65a1460def82a87e9_720w.webp)

### 5.2 nginx网络模型（多进程）

nginx采用的是单reactor多进程模型，因为每个连接都是处理的无状态数据，故可以通过多进程实现，多进程之间共享epollfd，在内核2.6以前，accept还存在惊群问题，即如果有连接到来，多个进程的epoll_wait都能监测到，这样多个进程都处理了该相同的连接，这是有问题的，所以nginx采用了文件锁的方式，在
ngx_process_events_and_timers函数中可以看出，哪个进程先获得了这把锁ngx-accept_mutex，就开始监听EPOLLIN事件，并进行epoll_wait接收对应的事件，接收到的事件先不处理，先放入一个队列中，如果是accept事件，就放入accept对应的ngx_posted_accept_events队列中，其它事件放入另外一个ngx_posted_events队列，然后再处理accept队列的事件回调函数，到这里才能释放文件锁，再去处理非accept队列里面的事件回调函数handler。可以看出nginx其实也是只有一个epollfd，只是被多个进程共享了，这样多个进程可以并行处理事件。

![img](https://pic3.zhimg.com/80/v2-093df57fcda7f1e853931fdae78ebf56_720w.webp)

### 5.3 memcached网络模型（多线程）

memcached是基于libevent来实现网络模型的，它是多线程多reactor模型，相比前面两种，它是有多个reactot模型的，也就是说有多个epoll进行事件循环，它也是将网络接入和网络读写io单独分离的方式，主线程主要处理网络的接入，主要看server_sockets函数，有accept事件后会调用event_handler回调函数，该事件是通过调用conn_new函数注册在主线程的event_base类型变量main_base上，所以主线程陷入事件循环后会监听accept事件。该event_handler回调函数里面做的事情就是调用drive_machine，看这个名字就知道是个状态机处理，所以accept事件来时就处理conn_listening下的事情，主要是进行accept得到客户端的sfd，然后通过round_robin算法选择一个工作线程，将该fd信息打包成CQ_ITEM放在工作线程的连接事件队列ev_queue中，最后通过pipe通知那个工作队列有连接事件过来了，其实就是往pipe中发一个”c”字符，工作线程此时处理事件回调thread_libevent_process，该回调主要是从连接事件队列ev_queue中取出主线程传过来的那个item，再调用conn_new处理该item，conn_new之前主线程也是调用的这个，主要是用来绑定fd的读写事件到event_base上去，工作线程则绑定到该线程对应的那个event_base上，这个event_base对应一个epoll，所以memcached是有多个epoll，因为每个工作线程都有自己的event_base，绑定完后，后续的读写事件回调也是event_handler函数，里面再调用drive_machine函数，只是连接的状态变成了conn_read和conn_write，这就是状态机的好处，代码逻辑很清晰。总的来说，主线程处理accept，工作线程处理后续的通信read和write，思路很清晰。

![img](https://pic3.zhimg.com/80/v2-1a0c5033e1aaeada122ba72579a76e42_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/542361454

作者：Linux