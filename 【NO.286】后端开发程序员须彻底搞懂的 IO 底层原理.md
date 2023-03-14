# 【NO.286】后端开发程序员须彻底搞懂的 IO 底层原理

## 1.混乱的 IO 概念

IO是Input和Output的缩写，即输入和输出。广义上的围绕计算机的输入输出有很多：鼠标、键盘、扫描仪等等。而我们今天要探讨的是在计算机里面，主要是作用在内存、网卡、硬盘等硬件设备上的输入输出操作。

谈起IO的模型，大多数人脑子里肯定是一坨混乱的概念，“阻塞”、“非阻塞”，“同步”、“异步”有什么区别？很多同学傻傻分不清，有尝试去搜索相关资料去探究真相，结果又被淹没在茫茫的概念之中。

这里尝试简单地去解释下为啥会出现这种现象，其中一个很重要的原因就是大家看到的资料对概念的解释都站在了不同的角度，有的站在了底层内核的视角，有的直接在java层面或者Netty框架层面给大家介绍API，所以给大家造成了一定程度的困扰。

所以在开篇之前，还是要说下本文所站的视角，我们将会从底层内核的层面给大家讲解下IO。因为万变不离其宗，只有了解了底层原理，不管语言层面如何花里胡哨，我们都能以不变应万变。

## 2.用户空间和内核空间

为了便于大家理解复杂的IO以及零拷贝相关的技术，我们还是得花点时间在回顾下操作系统相关的知识。这一节我们重点看下用户空间和内核空间，基于此后面我们才能更好地聊聊多路复用和零拷贝。

![img](https://pic4.zhimg.com/80/v2-07ddbe9b3bd904da1569c21153ff5e63_720w.webp)

**硬 件 层（Hardware）**

> 包括和我们熟知的和IO相关的CPU、内存、磁盘和网卡几个硬件；

**内核空间（Kernel Space）**

> 计算机开机后首先会运行内核程序，内核程序占用的一块私有的空间就是内核空间，并且可支持访问CPU所有的指令集（ring0 - ring3）以及所有的内存空间、IO及硬件设备；

**用户空间（User Space）**

> 每个普通的用户进程都有一个单独的用户空间，用户空间只能访问受限的资源（CPU的“保护模式”）也就是说用户空间是无法直接操作像内存、网卡和磁盘等硬件的；

如上所述，那我们可能会有疑问，用户空间的进程想要去访问或操作磁盘和网卡该怎么办呢？

为此，操作系统在内核中开辟了一块唯一且合法的调用入口“System Call Interface”，也就是我们常说的系统调用，系统调用为上层用户提供了一组能够操作底层硬件的API。这样一来，用户进程就可以通过系统调用访问到操作系统内核，进而就能够间接地完成对底层硬件的操作。这个访问的过程也即用户态到内核态的切换。常见的系统调用有很多，比如：内存映射mmap()、文件操作类的open()、IO读写read()、write()等等。

## 3.IO模型

### 3.1. BIO（Blocking IO）

我们先看一下大家都熟悉的BIO模型的 Java 伪代码：

```
ServerSocket serverSocket = new ServerSocket(8080);        // step1: 创建一个ServerSocket，并监听8080端口
while(true) {                                              // step2: 主线程进入死循环
    Socket socket = serverSocket.accept();                 // step3: 线程阻塞，开启监听
    BufferedReader reader = new BufferedReader(nwe InputStreamReader(socket.getInputStream()));
    System.out.println("read data: " + reader.readLine()); // step4: 数据读取
    PrintWriter print = new PrintWriter(socket.getOutputStream(), true);
    print.println("write data");                           // step5: socket数据写入
}
```

这段代码可以简单理解成一下几个步骤：

- 创建ServerSocket，并监听8080端口；
- 主线程进入死循环，用来阻塞监听客户端的连接，socket.accept()；
- 数据读取，socket.read()；
- 写入数据，socket.write()；

**问题**

以上三个步骤：accept(…)、read(…)、write(…)都会造成线程阻塞。上述这个代码使用了单线程，会导致主线程会直接夯死在阻塞的地方。

**优化**

我们要知道一点“**进程的阻塞是不会消耗CPU资源的**”，所以在多核的环境下，我们可以创建多线程，把接收到的请求抛给多线程去处理，这样就有效地利用了计算机的多核资源。甚至为了避免创建大量的线程处理请求，我们还可以进一步做优化，创建一个线程池，利用池化技术，对暂时处理不了的请求做一个缓冲。

### 3.2.“C10K”问题

> “C10K”即“client 10k”用来指代数量庞大的客户端；

BIO看上去非常的简单，事实上采用“BIO+线程池”来处理少量的并发请求还是比较合适的，也是最优的。但是面临数量庞大的客户端和请求，这时候使用多线程的弊端就逐渐凸显出来了：

- 严重依赖线程，线程还是比较耗系统资源的（一个线程大约占用1M的空间）；
- 频繁地创建和销毁代价很大，因为涉及到复杂的系统调用；
- 线程间上下文切换的成本很高，因为发生线程切换前，需要保留上一个任务的状态，以便切回来的时候，可以再次加载这个任务的状态。如果线程数量庞大，会造成线程做上下文切换的时间甚至大于线程执行的时间，CPU负载变高。

### 3.3.NIO非阻塞模型

下面开始真正走向Java NIO或者Netty框架所描述的“**非阻塞**”，NIO叫Non-Blocking IO或者New IO，由于BIO可能会引入的大量线程，所以可以简单地理解NIO处理问题的方式是通过单线程或者少量线程达到处理大量客户端请求的目的。为了达成这个目的，首先要做的就是把阻塞的过程非阻塞化。要想做到非阻塞，那必须得要有内核的支持，同时需要对用户空间的进程暴露系统调用函数。所以，这里的“非阻塞”可以理解成系统调用API级别的，而真正底层的IO操作都是阻塞的，我们后面会慢慢介绍。

事实上，内核已经对“非阻塞”做好了支持，举个我们刚刚说的的accept()方法阻塞的例子（Tips：java中的accept方法对应的系统调用函数也叫accept），看下官方文档对其非阻塞部分的描述。

![img](https://pic3.zhimg.com/80/v2-05a1bbbb74e3e33d1e8d4cf691a5ed4e_720w.webp)

官方文档对accetp()系统调用的描述是通过把”**flags**“参数设成”**SOCK_NONBLOCK**“就可以达到非阻塞的目的，非阻塞之后线程会一直处理轮询调用，这时候可以通过每次返回特殊的异常码“**EAGAIN**”或”**EWOULDBLOCK**“告诉主程序还没有连接到达可以继续轮询。

我们可以很容易想象程序非阻塞之后的一个大致过程。所以，非阻塞模式有个最大的特点就是：**用户进程需要不断去主动询问内核数据准备好了没有！**

下面我们通过一段伪代码，看下这个调用过程：

```
// 循环遍历
while(1) {
    // 遍历fd集合
    for (fdx in range(fd1, fdn)) {
        // 如果fdx有数据
        if (null != fdx.data) {
            // 进行读取和处理
            read(fdx)&handle(fdx);
        }
    }
}
```

这种调用方式也暴露出非阻塞模式的最大的弊端，就是需要让用户进程不断切换到内核态，对连接状态或读写数据做轮询。有没有一种方式来简化用户空间for循环轮询的过程呢？那就是我们下面要重点介绍的**IO多路复用模型**。

### 3.4.IO多路复用模型

非阻塞模型会让用户进程一直轮询调用系统函数，频繁地做内核态切换。想要做优化其实也比较简单，我们假想个业务场景，A业务系统会调用B的基础服务查询单个用户的信息。随着业务的发展，A的逻辑变复杂了，需要查100个用户的信息。很明显，A希望B提供一个批量查询的接口，用集合作为入参，一次性把数据传递过去就省去了频繁的系统间调用。

多路复用实际也差不多就是这个实现思路，只不过入参这个“集合”需要你注册/填写感兴趣的事件，读fd、写fd或者连接状态的fd等，然后交给内核帮你进行处理。

那我们就具体来看看多路复用里面大家都可能听过的几个系统调用 **- select()、poll()、epoll()。**

#### 3.4.1.select()

**select()** 构造函数信息如下所示：

```
/**
 * select()系统调用
 *
 * 参数列表：
 *     nfds       - 值为最大的文件描述符+1
 *    *readfds    - 用户检查可读性
 *    *writefds   - 用户检查可写性
 *    *exceptfds  - 用于检查外带数据
 *    *timeout    - 超时时间的结构体指针
 */
int select（int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout）;
```

官方文档对**select()**的描述：

> DESCRIPTION
> select() and pselect() allow a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become “ready” for some class of I/O operation (e.g.,input possible). A file descriptor is considered ready if it is possible to perform the corresponding I/O operation (e.g., read(2)) without blocking.

select()允许程序监控多个fd，阻塞等待直到一个或多个fd到达”就绪”状态。

内核使用**select()**为用户进程提供了类似批量的接口，函数本身也会一直阻塞直到有fd为就绪状态返回。下面我们来具体看下**select()**函数实现，以便我们更好地分析它有哪些优缺点。在**select()**函数的构造器里，我们很容易看到”**fd_set**“这个入参类型。它是用位图算法bitmap实现的，使用了一个大小固定的数组（fd_set设置了FD_SETSIZE固定长度为**1024**），数组中的每个元素都是0和1这样的二进制byte，0,1映射fd对应位置上是否有读写事件，举例：如果fd == 5，那么fd_set = 000001000。

同时 **fd_set** 定义了四个宏来处理bitmap：

> FD_ZERO(&set); // 初始化，清空的作用，使集合中不含任何fd
> FD_SET(fd, &set); // 将fd加入set集合，给某个位置赋值的操作
> FD_CLR(fd, &set); // 将fd从set集合中清除，去掉某个位置的值
> FD_ISSET(fd, &set); // 校验某位置的fd是否在集合中

使用bitmap算法的好处非常明显，运算效率高，占用内存少（使用了一个byte，8bit）。我们用伪代码和图片来描述下用户进程调用select()的过程：

![img](https://pic3.zhimg.com/80/v2-a207409d2d902768399028322ae95cfe_720w.webp)

假设fds为{1, 2, 3, 5, 7}对应的bitmap为”01110101”，抛给内核空间轮询，当有读写事件时重新标记同时停止阻塞，然后整体返回用户空间。由此我们可以看到select()系统调用的弊端也是比较明显的：

- 复杂度O(n)，轮询的任务交给了内核来做，复杂度并没有变化，数据取出后也需要轮询哪个fd上发生了变动；
- 用户态还是需要不断切换到内核态，直到所有的fds数据读取结束，整体开销依然很大；
- fd_set有大小的限制，目前被硬编码成了**1024**；
- fd_set不可重用，每次操作完都必须重置；

#### 3.4.2 poll()

**poll()** 构造函数信息如下所示：

```
/**
 * poll()系统调用
 *
 * 参数列表：
 *    *fds         - pollfd结构体
 *     nfds        - 要监视的描述符的数量
 *     timeout     - 等待时间
 */
int poll（struct pollfd *fds, nfds_t nfds, int *timeout）;
### pollfd的结构体
struct pollfd{
　int fd；// 文件描述符
　short event；// 请求的事件
　short revent；// 返回的事件
}
```

官方文档对**poll()**的描述：

> DESCRIPTION
> poll() performs a similar task to select(2): it waits for one of a set of file descriptors to become ready to perform I/O.

poll() 非常像select()，它也是阻塞等待直到一个或多个fd到达”就绪”状态。

看官方文档描述可以知道，**poll()**和**select()**是非常相似的，唯一的区别在于**poll()**摒弃掉了位图算法，使用自定义的结构体**pollfd**，在**pollfd**内部封装了fd，并通过event变量注册感兴趣的可读可写事件（**POLLIN、POLLOUT**），最后把 **pollfd** 交给内核。当有读写事件触发的时候，我们可以通过轮询 **pollfd**，判断revent确定该fd是否发生了可读可写事件。

老样子我们用伪代码来描述下用户进程调用 **poll()** 的过程：

![img](https://pic1.zhimg.com/80/v2-e737ae55f45e932681bd7b969d975964_720w.webp)

**poll()** 相对于**select()**，主要的优势是使用了pollfd的结构体：

- 没有了bitmap大小1024的限制；
- 通过结构体中的revents置位；

但是用户态到内核态切换及O(n)复杂度的问题依旧存在。

#### 3.4.3 epoll()

epoll()应该是目前最主流，使用范围最广的一组多路复用的函数调用，像我们熟知的Nginx、Redis都广泛地使用了此种模式。接下来我们重点分析下，epoll()的实现采用了“三步走”策略，它们分别是**epoll_create()、epoll_ctl()、epoll_wait()。**

4.3.1 epoll_create()

```
/**
 * 返回专用的文件描述符
 */
int epoll_create（int size）;
```

用户进程通过 **epoll_create()** 函数在内核空间里面创建了一块空间（为了便于理解，可以想象成创建了一块白板），并返回了描述此空间的fd。

4.3.2 epoll_ctl()

```
/**
 * epoll_ctl()系统调用
 *
 * 参数列表：
 *     epfd       - 由epoll_create()返回的epoll专用的文件描述符
 *     op         - 要进行的操作例如注册事件,可能的取值:注册-EPOLL_CTL_ADD、修改-EPOLL_CTL_MOD、删除-EPOLL_CTL_DEL
 *     fd         - 关联的文件描述符
 *     event      - 指向epoll_event的指针
 */
int epoll_ctl（int epfd, int op, int fd , struce epoll_event *event ）;
```

刚刚我们说通过**epoll_create()**可以创建一块具体的空间“白板”，那么通过**epoll_ctl()** 我们可以通过自定义的epoll_event结构体在这块”白板上”注册感兴趣的事件了。

- 注册 - EPOLL_CTL_ADD
- 修改 - EPOLL_CTL_MOD
- 删除 - EPOLL_CTL_DEL

4.3.3 epoll_wait()

```
 /**
 * epoll_wait()返回n个可读可写的fds
 *
 * 参数列表：
 *     epfd           - 由epoll_create()返回的epoll专用的文件描述符
 *     epoll_event    - 要进行的操作例如注册事件,可能的取值:注册-EPOLL_CTL_ADD、修改-EPOLL_CTL_MOD、删除-EPOLL_CTL_DEL
 *     maxevents      - 每次能处理的事件数
 *     timeout        - 等待I/O事件发生的超时值；-1相当于阻塞，0相当于非阻塞。一般用-1即可
 */
int epoll_wait（int epfd, struce epoll_event *event , int maxevents, int timeout）; 
```

**epoll_wait()** 会一直阻塞等待，直到硬盘、网卡等硬件设备数据准备完成后发起**硬中断**，中断CPU，CPU会立即执行数据拷贝工作，数据从磁盘缓冲传输到内核缓冲，同时将准备完成的fd放到就绪队列中供用户态进行读取。用户态阻塞停止，接收到**具体数量**的可读写的fds，返回用户态进行数据处理。

整体过程可以通过下面的伪代码和图示进一步了解：

![img](https://pic4.zhimg.com/80/v2-5020b6d682a0088464dae8a94f601cc7_720w.webp)

**epoll()** 基本上完美地解决了 **poll()** 函数遗留的两个问题：

- 没有了频繁的用户态到内核态的切换；
- O(1)复杂度，返回的”nfds”是一个确定的可读写的数量，相比于之前循环n次来确认，复杂度降低了不少；

## 4.同步、异步

细心的朋友可能会发现，本篇文章一直在解释“**阻塞**”和“**非阻塞**”，“**同步**”、“**异步**”的概念没有涉及，其实在很多场景下同步&异步和阻塞&非阻塞基本上是一个同义词。阻塞和非阻塞适合从系统调用API层面来看，就像我们本文介绍的select()、poll()这样的系统调用，同步和异步更适合站在应用程序的角度来看。应用程序在同步执行代码片段的时候结果不会立即返回，这时候底层IO操作不一定是阻塞的，也完全有可能是非阻塞。所以说：

- **阻塞和非阻塞：**读写没有就绪或者读写没有完成，函数是否要一直等待还是采用轮询；
- **同步和异步：**同步是读写由应用程序完成。异步是读写由操作系统来完成，并通过回调的机制通知应用程序。

这边顺便提两种大家可能会经常听到的模式：**Reactor和Preactor。**

- **Reactor 模式：主动模式。**
- **Preactor 模式：被动模式。**

## 5.总结

本篇文章从底层讲解了下从BIO到NIO的一个过程，着重介绍了IO多路复用的几个系统调用select()、poll()、epoll()，分析了下各自的优劣，技术都是持续发展演进的，目前也有很多的痛点。

原文链接：[彻底搞懂 IO 底层原理 - vivo互联网技术 - 博客园](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/vivotech/p/14059895.html)

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)