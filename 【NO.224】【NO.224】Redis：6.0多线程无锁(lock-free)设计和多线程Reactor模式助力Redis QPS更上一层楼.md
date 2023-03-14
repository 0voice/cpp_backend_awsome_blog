# 【NO.224】Redis：6.0多线程无锁(lock-free)设计和多线程Reactor模式助力Redis QPS更上一层楼

![img](https://pic2.zhimg.com/80/v2-60bcc53ba55dfb01e77c721699f6bd71_720w.webp)

**干货:**

1. 单线程模式-----并非CPU瓶颈
2. 多线程网络模型-----多线程Reactor模式
3. 多线程I/O-----lock-free无锁模式

因为我们的主题是多线程，所以不会过多涉及单线程。

## **1. 单线程模式-并非CPU瓶颈**

咱们都知道单线程的程序是没法利用服务器的多核CPU的，那么早期的Redis为何还要使用单线程呢？咱们不妨先看一下Redis官方给出的回答：

![img](https://pic3.zhimg.com/80/v2-3a4e16bd84875870d892fef144e9670e_720w.webp)

核心意思是：CPU并非制约Redis性能表现的瓶颈所在，更多状况下是受到内存大小和网络I/O的限制，因此Redis核心网络模型使用单线程并无什么问题，若是你想要使用服务的多核CPU，能够在一台服务器上启动多个实例或者采用分片集群的方式。

咱们知道Redis的I/O线程除了在等待事件，其它的事件都是非阻塞的，没有浪费任何的CPU时间，这也是Redis可以提供高性能服务的缘由。

## **2. 多线程网络模型-多线程Reactor模式**

Redis在 6.0 版本以后正式在核心网络模型中引入了多线程，它的工做模式是这样的：

![img](https://pic2.zhimg.com/80/v2-5441df4c0e45a9f411e76faf89a58a35_720w.webp)

区别于单 Reactor 模式，这种模式再也不是单线程的事件循环，而是有多个线程（Sub Reactors）各自维护一个独立的事件循环，由 Main Reactor 负责接收新链接并分发给 Sub Reactors 去独立处理，最后 Sub Reactors 回写响应给客户端。

`Multiple Reactors` 模式一般也能够等同于 `Master-Workers` 模式，好比`Nginx(前期文章有分享哈,可以回头去看)`等就是采用这种多线程模型，虽然不一样的项目实现细节略有区别，但整体来讲模式是一致的。

### **2.1 多线程工作流程**

![img](https://pic4.zhimg.com/80/v2-8b808b124769154d14e7a8ecd240063f_720w.webp)

1. Redis 服务器启动，开启主线程事件循环（Event Loop），注册 acceptTcpHandler 链接应答处理器到用户配置的监听端口对应的文件描述符，等待新链接到来；
2. 客户端和服务端创建网络链接；
3. acceptTcpHandler 被调用，主线程使用 AE 的 API 将 readQueryFromClient 命令读取处理器绑定到新链接对应的文件描述符上，并初始化一个 client 绑定这个客户端链接；
4. 客户端发送请求命令，触发读就绪事件，服务端主线程不会经过 socket 去读取客户端的请求命令，而是先将 client 放入一个 LIFO 队列 clients_pending_read；
5. 在事件循环（Event Loop）中，主线程执行 beforeSleep -->handleClientsWithPendingReadsUsingThreads，利用 Round-Robin 轮询负载均衡策略，把 clients_pending_read队列中的链接均匀地分配给 I/O 线程各自的本地 FIFO 任务队列 io_threads_list[id] 和主线程本身，并且用io_threads_pending[id]来记录每个线程的分配任务数量，因为线程需要读取这个io_threads_pending[id]这个数量来消费任务，消费完成会初始化为0。I/O 线程经过 socket 读取客户端的请求命令(是通过io_threads_op这个变量来判断是读(IO_THREADS_OP_READ)还是写(IO_THREADS_OP_WRITE)， 这里是io_threads_op == IO_THREADS_OP_READ)，存入 client->querybuf 并解析第一个命令，但不执行命令，主线程忙轮询，等待全部 I/O 线程完成读取任务；
6. 主线程和全部 I/O 线程都完成了读取任务(通过遍历io_threads_pending[id]，把每个线程的分配任务数量累加起来如果和等于0代表多线程已经消费完了任务)，主线程结束忙轮询，遍历 clients_pending_read 队列，执行全部客户端链接的请求命令，先调用 processCommandAndResetClient 执行第一条已经解析好的命令，而后调用 processInputBuffer 解析并执行客户端链接的全部命令，在其中使用 processInlineBuffer 或者 processMultibulkBuffer 根据 Redis 协议解析命令，最后调用 processCommand 执行命令；
7. 根据请求命令的类型（SET, GET, DEL, EXEC 等），分配相应的命令执行器去执行，最后调用 addReply 函数族的一系列函数将响应数据写入到对应 client 的写出缓冲区：client->buf 或者 client->reply ，client->buf 是首选的写出缓冲区，固定大小 16KB，通常来讲能够缓冲足够多的响应数据，可是若是客户端在时间窗口内须要响应的数据很是大，那么则会自动切换到 client->reply 链表上去，使用链表理论上可以保存无限大的数据（受限于机器的物理内存），最后把 client 添加进一个 LIFO 队列 clients_pending_write；
8. 在事件循环（Event Loop）中，主线程执行 beforeSleep --> handleClientsWithPendingWritesUsingThreads，利用 Round-Robin 轮询负载均衡策略，把 clients_pending_write 队列中的链接均匀地分配给 I/O 线程各自的本地 FIFO 任务队列 io_threads_list[id] 和主线程本身，并且用io_threads_pending[id]来记录每个线程的分配任务数量，因为线程需要读取这个io_threads_pending[id]这个数量来消费任务，消费完成会置为0。I/O 线程经过调用 writeToClient(io_threads_op == IO_THREADS_OP_WRITE)把 client 的写出缓冲区里的数据回写到客户端，主线程忙轮询，等待全部 I/O 线程完成写出任务；
9. 主线程和全部 I/O 线程都完成了写出任务(通过遍历io_threads_pending[id]，把每个线程的分配任务数量累加起来如果和等于0代表多线程已经消费完了任务)， 主线程结束忙轮询，遍历 clients_pending_write 队列，若是 client 的写出缓冲区还有数据遗留，则注册 sendReplyToClient 到该链接的写就绪事件，等待客户端可写时在事件循环中再继续回写残余的响应数据。

这里大部分逻辑和以前的单线程模型是一致的，变更的地方仅仅是把读取客户端请求命令和回写响应数据的逻辑异步化了，交给 I/O 线程去完成，这里须要特别注意的一点是：I/O 线程仅仅是读取和解析客户端命令而不会真正去执行命令，客户端命令的执行最终仍是要回到主线程上完成。

### **2.2 图解详细流程**

![img](https://pic3.zhimg.com/80/v2-397545336d30362438357f4871745b92_720w.webp)

## **3. I/O多线程lock-free（无锁设计）**

主线程和 I/O 线程之间共享的变量有三个：io_threads_pending 计数器、io_threads_op I/O 标识符和 io_threads_list 线程本地任务队列。

- io_threads_pending

- - 统计每个线程分配任务的数量，主线程在分配的时候，子线程是被换新的，它一直在执行100w次的CPU空转即自旋操作；当主线程把任务分配好之后，子线程会判断io_threads_pending[id]不为0就去消费任务，消费完成会置为0。

- io_threads_op

- - IO_THREADS_OP_WRITE socket可写
  - IO_THREADS_OP_READ socket可读

- io_threads_list

- - 线程本地任务队列(链表)

lock-free无锁设计核心：

\1. 原子变量，不需要加锁保护

- io_threads_pending变量在声明的时候加上了_Atomic限定符：_Atomic unsigned long; _Atomic是C11标准中引入的原子操作：被_Atomic修饰的变量被认为是原子变量，对原子变量的操作是不可分割的(Atomicity)，且操作结果对其他线程可见，执行的顺序也不能被重排。所以io_threads_pending是属于线程安全的变量。

\2. 交错访问来规避共享数据竞争

- io_threads_op 和 io_threads_list 这两个变量则是经过控制主线程和 I/O 线程交错访问来规避共享数据竞争问题。
- I/O 线程启动以后会经过忙轮询和锁休眠等待主线程的信号，在这以前它不会去访问本身的本地任务队列 io_threads_list[id]，而主线程会在分配完全部任务到各个 I/O 线程的本地队列以后才去唤醒 I/O 线程开始工做，而且主线程以后在 I/O 线程运行期间只会访问本身的本地任务队列 io_threads_list[0] 而不会再去访问 I/O 线程的本地队列，这也就保证了主线程永远会在 I/O 线程以前访问 io_threads_list 而且以后再也不访问，保证了交错访问。
- io_threads_op 同理，主线程会在唤醒 I/O 线程以前先设置好 io_threads_op 的值，而且在 I/O 线程运行期间不会再去访问这个变量，这也就变相保证了原子性。在源码src/server.h中 : extern int io_threads_op;

## **4. 源码**

源码真的太多了，拷贝到这里实在影响阅读，因此为了大家能迅速定位，我这里贴出地址哈。

- 子线程入口Main函数 处理read，write和解析操作

- - IOThreadMain：源码文件3665行
  - [https://github.com/redis/redis/blob/64f6159646337b4a3b56a400522ad4d028d55dac/src/networking.c](https://link.zhihu.com/?target=https%3A//github.com/redis/redis/blob/64f6159646337b4a3b56a400522ad4d028d55dac/src/networking.c)

- 主线程执行回复入口函数

- - handleClientsWithPendingWritesUsingThreads：3810行
  - 地址同上

- 主线程执行读取数据入口函数

- - handleClientsWithPendingReadsUsingThreads：3935行
  - 地址同上

- 主线程初始化多线程入口函数

- - initThreadedIO: 3712行
  - 地址同上

- 其他源码

- - readQueryFromClient：2266行
  - postponeClientRead：3908行
  - 地址同上

原文地址：https://zhuanlan.zhihu.com/p/595289451

作者：linux