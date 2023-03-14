# 【NO.291】Redis多线程原理详解

本篇文章为你解答以下问题：

- **0：redis单线程的实现流程是怎样的？**
- **1：redis哪些地方用到了多线程，哪些地方是单线程？**
- **2：redis多线程是怎么实现的？**
- **3：redis多线程是怎么做到无锁的？**

## **1.redis单线程的实现流程是怎样的？**

Redis一开始是单线程模型，在一个线程中要同时处理两种事件：文件事件和时间事件

文件事件主要是网络I/O的读写，请求的接收和回复

时间事件就是单次/多次执行的定时器，如主从复制、定时删除过期数据、字典rehash等

redis所有核心功能都是跑在主线程中的，像aof文件落盘操作是在子线程中执行的，那么在高并发情况下它是怎么做到高性能的呢？

由于这两种事件在同一个线程中执行，就会出现互相影响的问题，如时间事件到了还在等待/执行文件事件，或者文件事件已经就绪却在执行时间事件，这就是单线程的缺点，所以在实现上要将这些影响降到最低。那么redis是怎么实现的呢？

定时执行的时间事件保存在一个链表中，由于链表中任务没有按照执行时间排序，所以每次需要扫描单链表，找到最近需要执行的任务，时间复杂度是O(N)，redis敢这么实现就是因为这个链表很短，大部分定时任务都是在serverCron方法中被调用。从现在开始到最近需要执行的任务的开始时间，时长定位T，这段时间就是属于文件事件的处理时间，以epoll为例，执行epoll_wait最多等待的时长为T，如果有就绪任务epoll会返回所有就绪的网络任务，存在一个数组中，这时我们知道了所有就绪的socket和对应的事件（读、写、错误、挂断），然后就可以接收数据，解析，执行对应的命令函数。

如果最近要执行的定时任务时间已经过了，那么epoll就不会阻塞，直接返回已经就绪的网络事件，即不等待。

总之单线程，定时事件和网络事件还是会互相影响的，正在处理定时事件网络任务来了，正在处理网络事件定时任务的时间到了。所以redis必须保证每个任务的处理时间不能太长。

redis处理流程如下：

1：服务启动，开始网络端口监听，等待客户端请求

2：客户端想服务端发起连接请求，创建客户端连接对象，完成连接

3：将socket信息注册到epoll，设置超时时间为时间事件的周期时长，等待客户端发起请求

4：客户端发起操作数据库请求(如GET)

5：epoll收到客户端的请求，可能多个，按照顺序处理请求

6：接收请求参数，接收完成后解析请求协议，得到请求命令

7：执行请求命令，即操作redis数据库

8：将结果返回给客户端

## **2.redis哪些地方用到了多线程，哪些地方是单线程？**

Redis多线程和单线程模型对比如下图：

![img](https://pic2.zhimg.com/80/v2-7c6a1daa50f26845374d16f3d40747d1_720w.webp)

从上图中可以看出只有以下3个地方用的是多线程，其他地方都是单线程：

1：接收请求参数

2：解析请求参数

3：请求响应，即将结果返回给client

很明显以上3点各个请求都是互相独立互不影响的，很适合用多线程，特别是请求体/响应体很大的时候，更能体现多线程的威力。而操作数据库是请求之间共享的，如果使用多线程的话适合读写锁。而操作数据库本身是很快的（就是对map的增删改查），单线程不一定就比多线程慢，当然也有可能是作者偷懒，懒得实现罢了，但这次的多线程模型还是值得我们学习一下的。

## **3.redis多线程是怎么实现的？**

先大致说一下多线程的流程：

1：服务器启动时启动一定数量线程，服务启动的时候可以指定线程数，每个线程对应一个队列（list *io_threads_list[128]），最多128个线程。

2：服务器收到的每个请求都会放入全局读队列clients_pending_read，同时将队列中的元素分发到每个线程对应的队列io_threads_list中，这些工作都是在主线程中执行的。

3：每个线程（包括主线程和子线程）接收请求参数并做解析，完事后在client中设置一个标记CLIENT_PENDING_READ，标识参数解析完成，可以操作数据库了。（主线程和子线程都会执行这个步骤）

4：主线程遍历队列clients_pending_read，发现设有CLIENT_PENDING_READ标记的，就操作数据库

5：操作完数据库就是响应client了，响应是一组函数addReplyXXX，在client中设置标记CLIENT_PENDING_WRITE，同时将client加入全局写队列clients_pending_write

6：主线程将全局队列clients_pending_write以轮训的方式将任务分发到每个线程对应的队列io_threads_list

7：所有线程将遍历自己的队列io_threads_list，将结果发送给client

## **4.redis多线程是怎么做到无锁的？**

上面说了多线程的地方都是互相独立互不影响的。但是每个线程的队列就存在两个两个线程访问的情况：主线程向队列中写数据，子线程消费，redis的实现有点反直觉。按正常思路来说，主线程在往队列中写数据的时候加锁；子线程复制队列&并将队列清空，这个两个动作是加锁的，子线程消费复制后的队列，这个过程是不需要加锁的，按理来说主线程和子线程的加锁动作都是非常快的。但是redis并没有这么实现，那么他是怎么实现的呢？

redis多线程的模型是主线程负责搜集任务，放入全局读队列clients_pending_read和全局写队列clients_pending_write，主线程在将队列中的任务以轮训的方式分发到每个线程对应的队列（list *io_threads_list[128]）

1：一开始子线程的队列都是空，主线程将全对队列中的任务分发到每个线程的队列，并设置一个队列有数据的标记（*Atomic unsigned long io*threads_pending[128]），io_threads_pending[1]=5表示第一个线程的队列中有5个元素

2：子线程死循环轮训检查io_threads_pending[index] > 0，有数据就开始处理，处理完成之后将io_threads_pending[index] = 0，没数据继续检查

3：主线程将任务分发到子线程的队列中，自己处理自己队列中的任务，处理完成后，等待所有子线程处理完所有任务，继续收集任务到全局队列，在将任务分发给子线程，这样就避免了主线程和子线程同时访问队列的情况，主线程向队列写的时候子线程还没开始消费，子线程在消费的时候主线程在等待子线程消费完，子线程消费完后主线程才会往队列中继续写，就不用加锁了。因为任务是平均分配到每个队列的，所以每个队列的处理时间是接近的，等待的时间会很短。

## **5.源码执行流程**

为了方便你看源码，这里加上一些代码的执行流程

启动socket监听，注册连接处理函数，连接成功后创建连接对象connection，创建client对象，通过aeCreateFileEvent注册client的读事件

```
main -> initServer -> acceptTcpHandler -> anetTcpAccept -> anetGenericAccept -> accept(获取到socket连接句柄)
connCreateAcceptedSocket -> connCreateSocket -> 创建一个connection对象
acceptCommonHandler -> createClient创建client连接对象 -> connSetReadHandler -> aeCreateFileEvent -> readQueryFromClient
main -> aeMain -> aeProcessEvents -> aeApiPoll(获取可读写的socket) -> readQueryFromClient(如果可读) -> processInputBuffer -> processCommandAndResetClient(多线程下这个方法在当前流程下不会执行，而由主线程执行)
```

在多线程模式下，readQueryFromClient会将client信息加入server.clients_pending_read队列，listAddNodeHead(server.clients_pending_read,c);

主线程会将server.clients_pending_read中的数据分发到子线程的队列(io_threads_list)中，子线程会调用readQueryFromClient就行参数解析，主线程分发完任务后，会执行具体的操作数据库的命令，这块是单线程

如果参数解析完成会在client->flags中加一个标记CLIENT_PENDING_COMMAND，在主线程中先判断client->flags & CLIENT_PENDING_COMMAND > 0，说明参数解析完成，才会调用processCommandAndResetClient，之前还担心如果子线程还在做参数解析，主线程就开始执行命令难道不会有问题吗？现在一切都清楚了

```
main -> aeMain -> aeProcessEvents -> beforeSleep -> handleClientsWithPendingReadsUsingThreads -> processCommandAndResetClient -> processCommand -> call
```

读是多次读：socket读缓冲区有数据，epoll就会一直触发读事件，所以读可能是多次的

写是一次写：往socket写数据是在子线程中执行的，直接循环直到数据写完位置，就算某个线程阻塞了，也不会像单线程那样导致所有任务都阻塞

```
执行完相关命令后，就是将结果返回给client，回复client是一组函数，我们以addReply为例，说一下执行流程，执行addReply还是单线程的，将client信息插入全局队列server.clients_pending_write。
addReply -> prepareClientToWrite -> clientInstallWriteHandler -> listAddNodeHead(server.clients_pending_write,c)
在主线程中将server.clients_pending_write中的数据以轮训的方式分发到多个子线程中
beforeSleep -> handleClientsWithPendingWritesUsingThreads -> 将server.clients_pending_write中的数据以轮训的方式分发到多个线程的队列中io_threads_list
list *io_threads_list[IO_THREADS_MAX_NUM];是数组双向链表，一个线程对应其中一个队列
子线程将client中的数据发给客户端，所以是多线程
server.c -> main -> initThreadedIO(启动一定数量的线程) -> IOThreadMain(线程执行的方法) -> writeToClient -> connWrite -> connSocketWrite
```

网络操作对应的一些方法，所有connection对象的type字段都是指向CT_Socket

```
ConnectionType CT_Socket = {
    .ae_handler = connSocketEventHandler,
    .close = connSocketClose,
    .write = connSocketWrite,
    .read = connSocketRead,
    .accept = connSocketAccept,
    .connect = connSocketConnect,
    .set_write_handler = connSocketSetWriteHandler,
    .set_read_handler = connSocketSetReadHandler,
    .get_last_error = connSocketGetLastError,
    .blocking_connect = connSocketBlockingConnect,
    .sync_write = connSocketSyncWrite,
    .sync_read = connSocketSyncRead,
    .sync_readline = connSocketSyncReadLine
};
```

原文链接：https://zhuanlan.zhihu.com/p/382713883

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)