# 【NO.59】深入学习IO多路复用 select/poll/epoll 实现原理

> select/poll/epoll 是 Linux 服务器提供的三种处理高并发网络请求的 IO 多路复用技术，是个老生常谈又不容易弄清楚其底层原理的知识点，本文打算深入学习下其实现机制。

Linux 服务器处理网络请求有三种机制，select、poll、epoll，本文打算深入学习下其实现原理。

吃水不忘挖井人，最近两周花了些时间学习了张彦飞大佬的文章 [图解 | 深入揭秘 epoll 是如何实现 IO 多路复用的](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484905&idx=1&sn=a74ed5d7551c4fb80a8abe057405ea5e&scene=21#wechat_redirect) 和[其他文章](https://github.com/yanfeizhang/coder-kung-fu) ，及出版的书籍《深入理解 Linux 网络》，对阻塞 IO、多路复用、epoll 等的实现原理有了一定的了解；飞哥的文章描述底层源码逻辑比较清晰，就是有时候归纳总结事情本质的抽象程度不够，涉及内核源码细节的讲述较多，会让读者产生一定的学习成本，本文希望在这方面改进一下。

## **0. 结论**

本文其他的内容主要是得出了下面几个结论：

1. 服务器要接收客户端的数据，要建立 socket 内核结构，主要包含两个重要的数据结构，**（进程）等待队列**，和**（数据）接收队列**，socket 在进程中作为一个文件，可以用文件描述符 fd 来表示，为了方便理解，本文中， socket 内核对象 ≈ fd 文件描述符 ≈ TCP 连接；
2. 阻塞 IO 的主要逻辑是：服务端和客户端建立了连接 socket 后，服务端的用户进程通过 recv 函数接收数据时，如果数据没有到达，则当前的用户进程的进程描述符和回调函数会封装到一个进程等待项中，加入到 socket 的进程等待队列中；如果连接上有数据到达网卡，由网卡将数据通过 DMA 控制器拷贝到内核内存的 RingBuffer 中，并向 CPU 发出硬中断，然后，CPU 向内核中断进程 ksoftirqd 发出软中断信号，内核中断进程 ksoftirqd 将内核内存的 RingBuffer 中的数据根据数据报文的 IP 和端口号，将其拷贝到对应 socket 的数据接收队列中，然后通过 socket 的进程等待队列中的回调函数，唤醒要处理该数据的用户进程；
3. 阻塞 IO 的问题是：一次数据到达会进行**两次进程切换，**一次数据读取有**两处阻塞，单进程对单连接；**
4. 非阻塞 IO 模型解决了“**两次进程切换，两处阻塞，单进程对单连接**”中的“**两处阻塞**”问题，将“**两处阻塞**”变成了“**一处阻塞**”，但依然存在“**两次进程切换，一处阻塞，单进程对单连接**”的问题；
5. 用一个进程监听多个连接的 IO 多路复用技术解决了“**两次进程切换，一处阻塞，单进程对单连接**” 中的“**两次进程切换，单进程对单连接**”，剩下了“**一处阻塞**”，这是 Linux 中同步 IO 都会有的问题，因为 Linux 没有提供异步 IO 实现；
6. Linux 的 IO 多路复用用三种实现：select、poll、epoll。select 的问题是：

a）调用 select 时会陷入内核，这时需要将参数中的 fd_set 从用户空间拷贝到内核空间，高并发场景下这样的拷贝会消耗极大资源；（epoll 优化为不拷贝）

b）进程被唤醒后，不知道哪些连接已就绪即收到了数据，需要遍历传递进来的所有 fd_set 的每一位，不管它们是否就绪；（epoll 优化为异步事件通知）

c）select 只返回就绪文件的个数，具体哪个文件可读还需要遍历；（epoll 优化为只返回就绪的文件描述符，无需做无效的遍历）

d）同时能够监听的文件描述符数量太少，是 1024 或 2048；（poll 基于链表结构解决了长度限制）

1. poll 只是基于链表的结构解决了最大文件描述符限制的问题，其他 select 性能差的问题依然没有解决；终极的解决方案是 epoll，解决了 select 的前三个缺点；
2. epoll 的实现原理看起来很复杂，其实很简单，注意两个回调函数的使用：数据到达 socket 的等待队列时，通过**回调函数 ep_poll_callback** 找到 eventpoll 对象中红黑树的 epitem 节点，并将其加入就绪列队 rdllist，然后通过**回调函数 default_wake_function** 唤醒用户进程 ，并将 rdllist 传递给用户进程，让用户进程准确读取就绪的 socket 的数据。这种回调机制能够定向准确的通知程序要处理的事件，而不需要每次都循环遍历检查数据是否到达以及数据该由哪个进程处理，日常开发中可以学习借鉴下这种思想。

## **1. Linux 怎样处理网络请求**

### 1.1 阻塞 IO

要讲 IO 多路复用，最好先把传统的同步阻塞的网络 IO 的交互方式剖析清楚。

如果客户端想向 Linux 服务器发送一段数据 ，C 语言的实现方式是：

```
int main(){     int fd = socket();      // 创建一个网络通信的socket结构体     connect(fd, ...);       // 通过三次握手跟服务器建立TCP连接     send(fd, ...);          // 写入数据到TCP连接     close(fd);              // 关闭TCP连接}
```

服务端通过如下 C 代码接收客户端的连接和发送的数据：

```
int main(){     fd = socket(...);        // 创建一个网络通信的socket结构体     bind(fd, ...);           // 绑定通信端口     listen(fd, 128);         // 监听通信端口，判断TCP连接是否可以建立     while(1) {         connfd = accept(fd, ...);              // 阻塞建立连接         int n = recv(connfd, buf, ...);        // 阻塞读数据         doSomeThing(buf);                      // 利用读到的数据做些什么         close(connfd);                         // 关闭连接，循环等待下一个连接    }}
```

把服务端处理请求的细节展开，得到如下图所示的同步阻塞网络 IO 的数据接收流程：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141455044571769.png)

主要步骤是：

1）服务端通过 **socket() 函数**陷入内核态进行 socket 系统调用，该内核函数会创建 **socket 内核对象**，主要有两个重要的结构体，（**进程）等待队列**，和（**数据）接收队列，**为了方便理解，等待队列前可以加上进程二字，其实不加更准确，接收队列同样；进程等待队列，存放了服务端的用户进程 A 的进程描述符和回调函数；socket 的数据接收队列，存放网卡接收到的该 socket 要处理的数据；

2）进程 A 调用 recv() 函数接收数据，会进入到 **recvfrom() 系统调用函数**，发现 socket 的数据等待队列没有它要接收的数据到达时，进程 A 会让出 CPU，进入阻塞状态，进程 A 的进程描述符和它被唤醒用到的回调函数 callback func 会组成一个结构体叫等待队列项，放入 socket 的进程等待队列；

3）客户端的发送数据到达服务端的网卡；

4）网卡首先会将网络传输过来的数据通过 DMA 控制程序复制到内存环形缓冲区 RingBuffer 中；

5）网卡向 CPU 发出**硬中断**；

6）CPU 收到了硬中断后，为了避免过度占用 CPU 处理网络设备请求导致其他设备如鼠标和键盘的消息无法被处理，会调用网络驱动注册的中断处理函数，进行简单快速处理后向内核中断进程 ksoftirqd 发出**软中断**，就释放 CPU，由软中断进程处理复杂耗时的网络设备请求逻辑；

7）**内核中断进程 ksoftirqd** 收到软中断信号后，会将网卡复制到内存的数据，根据数据报文的 IP 和端口号，将其拷贝到对应 socket 的接收队列；

8）内核中断进程 ksoftirqd 根据 socket 的数据接收队列的数据，通过进程等待队列中的回调函数，唤醒要处理该数据的进程 A，进程 A 会进入 CPU 的运行队列，等待获取 CPU 执行数据处理逻辑；

9）进程 A 获取 CPU 后，会回到之前调用 **recvfrom() 函数**时阻塞的位置继续执行，这时发现 socket 内核空间的等待队列上有数据，会在内核态将内核空间的 socket 等待队列的数据拷贝到用户空间，然后才会回到用户态执行进程的用户程序，从而真的解除阻塞**；**

用户进程 A 在调用 recvfrom() 系统函数时，有两个阶段都是等待的：在数据没有准备好的时候，进程 A 等待内核 socket 准备好数据；内核准备好数据后，进程 A 继续等待内核将 socket 等待队列的数据拷贝到自己的用户缓冲区；在内核完成数据拷贝到用户缓冲区后，进程 A 才会从 recvfrom() 系统调用中返回，并解除阻塞状态。整体流程如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141455152262102.png)

在 IO 阻塞逻辑中，存在下面三个问题：

1. 进程在 recv 的时候大概率会被阻塞掉，导致一次进程切换；
2. 当 TCP 连接上的数据到达服务端的网卡、并从网卡复制到内核空间 socket 的数据等待队列时，进程会被唤醒，又是一次进程切换；并且，在用户进程继续执行完 recvfrom() 函数系统调用，将内核空间的数据拷贝到了用户缓冲区后，用户进程才会真正拿到所需的数据进行处理；
3. 一个进程同时只能等待一条连接，如果有很多并发，则需要很多进程；

总结：一次数据到达会进行**两次进程切换，**一次数据读取有**两处阻塞，单进程对单连接**。

### 1.2 非阻塞 IO

为了解决同步阻塞 IO 的问题，操作系统提供了非阻塞的 recv() 函数，这个函数的效果是：如果没有数据从网卡到达内核 socket 的等待队列时，系统调用会直接返回，而不是阻塞的等待。

如果我们要产生一个非阻塞的 socket，在 C 语言中如下代码所示：

```
// 创建socketint sock_fd = socket(AF_INET, SOCK_STREAM, 0);...// 更改socket为nonblockfcntl(sock_fd, F_SETFL, fdflags | O_NONBLOCK);// connect....while(1)  {    int recvlen = recv(sock_fd, recvbuf, RECV_BUF_SIZE) ;    ......}...
```

非阻塞 IO 模型如下图所示：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141455262925271.png)

从上图中，我们知道，非阻塞 IO，是将等待数据从网卡到达 socket 内核空间这一部分变成了非阻塞的，用户进程调用 recvfrom() 会重复发送请求检查数据是否到达内核空间，如果没有到，则立即返回，不会阻塞。不过，当数据已经到达内核空间的 socket 的等待队列后，用户进程依然要等待 recvfrom() 函数将数据从内核空间拷贝到用户空间，才会从 recvfrom() 系统调用函数中返回。

非阻塞 IO 模型解决了“**两次进程切换，两处阻塞，单进程对单连接**”中的“**两处阻塞**”问题，将“**两处阻塞**”变成了“**一处阻塞**”，但依然存在“**两次进程切换，一处阻塞，单进程对单连接**”的问题。

### 1.3 IO 多路复用

要解决“**两次进程切换，单进程对单连接**”的问题，服务器引入了 IO 多路复用技术，通过一个进程处理多个 TCP 连接，不仅降低了服务器处理网络请求的进程数，而且不用在每个连接的数据到达时就进行进程切换，进程可以一直运行并只处理有数据到达的连接，当然，如果要监听的所有连接都没有数据到达，进程还是会进入阻塞状态，直到某个连接有数据到达时被回调函数唤醒。

IO 多路复用模型如下图所示：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141455355885332.png)

从上图可知，系统调用 select 函数阻塞执行并返回数据就绪的连接个数，然后调用 recvfrom() 函数将到达内核空间的数据拷贝到用户空间，尽管这两个阶段都是阻塞的，但是由于只会处理有数据到达的连接，整体效率会有极大的提升。

到这里，阻塞 IO 模型的“**两次进程切换，两处阻塞，单进程对单连接**”问题，通过非阻塞 IO 和多路复用技术，就只剩下了“**一处阻塞**”这个问题，即 Linux 服务器上用户进程一定要等待数据从内核空间拷贝到用户空间，如果这个步骤也变成非阻塞的，也就是进程调用 recvfrom 后立刻返回，内核自行去准备好数据并将数据从内核空间拷贝到用户空间、再 notify 通知用户进程去读取数据，那就是 **IO 异步调用**，不过，Linux 没有提供异步 IO 的实现，真正意义上的网络异步 IO 是 Windows 下的 IOCP（IO 完成端口）模型，这里就不探讨了。

## **2. 详解 select、poll、epoll 实现原理**

### 2.1 select 实现原理

select 函数定义

Linux 提供的 select 函数的定义如下：

```
int select(    int nfds,                     // 监控的文件描述符集里最大文件描述符加1    fd_set *readfds,              // 监控有读数据到达文件描述符集合，引用类型的参数    fd_set *writefds,             // 监控写数据到达文件描述符集合，引用类型的参数    fd_set *exceptfds,            // 监控异常发生达文件描述符集合，引用类型的参数    struct timeval *timeout);     // 定时阻塞监控时间
```

readfds、writefds、errorfds 是三个文件描述符集合。select 会遍历每个集合的前 nfds 个描述符，分别找到可以读取、可以写入、发生错误的描述符，统称为“就绪”的描述符。然后用找到的子集替换这三个引用参数中的对应集合，返回所有就绪描述符的数量。

timeout 参数表示调用 select 时的阻塞时长。如果所有 fd 文件描述符都未就绪，就阻塞调用进程，直到某个描述符就绪，或者阻塞超过设置的 timeout 后，返回。如果 timeout 参数设为 NULL，会无限阻塞直到某个描述符就绪；如果 timeout 参数设为 0，会立即返回，不阻塞。

**文件描述符 fd**

文件描述符（file descriptor）是一个非负整数，从 0 开始。进程使用文件描述符来标识一个打开的文件。Linux 中一切皆文件。

系统为每一个进程维护了一个文件描述符表，表示该进程打开文件的记录表，而**文件描述符实际上就是这张表的索引**。每个进程默认都有 3 个文件描述符：0 (stdin)、1 (stdout)、2 (stderr)。

**socket**

socket 可以用于同一台主机的不同进程间的通信，也可以用于不同主机间的通信。操作系统将 socket 映射到进程的一个文件描述符上，进程就可以通过读写这个文件描述符来和远程主机通信。

socket 是进程间通信规则的高层抽象，而 fd 提供的是底层的具体实现。socket 与 fd 是一一对应的。通过 socket 通信，实际上就是通过文件描述符 fd 读写文件。

本文中，为了方便理解，可以认为 socket 内核对象 ≈ fd 文件描述符 ≈ TCP 连接。

**fd_set 文件描述符集合**

select 函数参数中的 fd_set 类型表示文件描述符的集合。

由于文件描述符 fd 是一个从 0 开始的无符号整数，所以可以使用 fd_set 的二进制每一位来表示一个文件描述符。某一位为 1，表示对应的文件描述符已就绪。比如比如设 fd_set 长度为 1 字节，则一个 fd_set 变量最大可以表示 8 个文件描述符。当 select 返回 fd_set = 00010011 时，表示文件描述符 1、2、5 已经就绪。

**fd_set 的 API**

fd_set 的使用涉及以下几个 API：

```
#include <sys/select.h>int FD_ZERO(int fd, fd_set *fdset);  // 将 fd_set 所有位置 0int FD_CLR(int fd, fd_set *fdset);   // 将 fd_set 某一位置 0int FD_SET(int fd, fd_set *fd_set);  // 将 fd_set 某一位置 1int FD_ISSET(int fd, fd_set *fdset); // 检测 fd_set 某一位是否为 1
```

**select 监听多个连接的用法**

服务端使用 select 监控多个连接的 C 代码是：

```
#define MAXCLINE 5       // 连接队列中的个数int fd[MAXCLINE];        // 连接的文件描述符队列int main(void){      sock_fd = socket(AF_INET,SOCK_STREAM,0)          // 建立主机间通信的 socket 结构体      .....      bind(sock_fd, (struct sockaddr *)&server_addr, sizeof(server_addr);         // 绑定socket到当前服务器      listen(sock_fd, 5);  // 监听 5 个TCP连接      fd_set fdsr;         // bitmap类型的文件描述符集合，01100 表示第1、2位有数据到达      int max;      for(i = 0; i < 5; i++)      {          .....          fd[i] = accept(sock_fd, (struct sockaddr *)&client_addr, &sin_size);   // 跟 5 个客户端依次建立 TCP 连接，并将连接放入 fd 文件描述符队列      }      while(1)               // 循环监听连接上的数据是否到达      {        FD_ZERO(&fdsr);      // 对 fd_set 即 bitmap 类型进行复位，即全部重置为0        for(i = 0; i < 5; i++)        {             FD_SET(fd[i], &fdsr);      // 将要监听的TCP连接对应的文件描述符所在的bitmap的位置置1，比如 0110010110 表示需要监听第 1、2、5、7、8个文件描述符对应的 TCP 连接        }        ret = select(max + 1, &fdsr, NULL, NULL, NULL);  // 调用select系统函数进入内核检查哪个连接的数据到达        for(i=0;i<5;i++)        {            if(FD_ISSET(fd[i], &fdsr))      // fd_set中为1的位置表示的连接，意味着有数据到达，可以让用户进程读取            {                ret = recv(fd[i], buf,sizeof(buf), 0);                ......            }        }  }
```

从注释中，我们可以看到，在一个进程中使用 select 监控多个连接的主要步骤是：

1）调用 socket() 函数建立主机间通信的 socket 结构体，bind() 绑定 socket 到当前服务器，listen() 监听五个 TCP 连接；

2）调用 accept() 函数建立和 5 个客户端的 TCP 连接，并把连接的文件描述符放入 fd 文件描述符队列；

3） 定义一个 fd_set 类型的变量 fdsr；

4）调用 FD_ZERO，将 fdsr 所有位置 0；

5）调用 FD_SET，将 fdsr 要监听的几个文件描述符的位置 1，表示要监听这几个文件描述符指向的连接；

6）调用 select() 函数，并将 fdsr 参数传递给 select；

7）select 会将 fdsr 中就绪的位置 1，未就绪的位置 0，返回就绪的文件描述符的数量；

8）当 select 返回后，调用 FD_ISSET 检测哪些位为 1，表示对应文件描述符对应的连接的数据已经就绪，可以调用 recv 函数读取该连接的数据了。

**select 的执行过程**

在服务器进程 A 启动的时候，要监听的连接的 socket 文件描述符是 3、4、5，如果这三个连接均没有数据到达网卡，则进程 A 会让出 CPU，进入阻塞状态，同时会将进程 A 的进程描述符和被唤醒时用到的回调函数组成等待队列项加入到 socket 对象 3、4、5 的进程等待队列中，注意，这时 select 调用时，fdsr 文件描述符集会从用户空间拷贝到内核空间，如下图所示：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141455568949167.png)

当网卡接收到数据，然后网卡通过中断信号通知 CPU 有数据到达，执行中断程序，中断程序主要做了两件事：

1）将网络数据写入到对应 socket 的数据接收队列里面；

2）唤醒队列中的等待进程 A，重新将进程 A 放入 CPU 的运行队列中；

假设连接 3、5 有数据到达网卡，注意，这时 select 调用结束时，fdsr 文件描述符集会从内核空间拷贝到用户空间：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141456087271156.png)

**select 的缺点**

从上面两图描述的执行过程，可以发现 select 实现多路复用有以下缺点：

1.性能开销大

1）调用 select 时会陷入内核，这时需要将参数中的 fd_set 从用户空间拷贝到内核空间，select 执行完后，还需要将 fd_set 从内核空间拷贝回用户空间，高并发场景下这样的拷贝会消耗极大资源；（epoll 优化为不拷贝）

2）进程被唤醒后，不知道哪些连接已就绪即收到了数据，需要遍历传递进来的所有 fd_set 的每一位，不管它们是否就绪；（epoll 优化为异步事件通知）

3）select 只返回就绪文件的个数，具体哪个文件可读还需要遍历；（epoll 优化为只返回就绪的文件描述符，无需做无效的遍历）

2.同时能够监听的文件描述符数量太少。受限于 sizeof(fd_set) 的大小，在编译内核时就确定了且无法更改。一般是 32 位操作系统是 1024，64 位是 2048。（poll、epoll 优化为适应链表方式）

第 2 个缺点被 poll 解决，第 1 个性能差的缺点被 epoll 解决。

### 2.2 poll 实现原理

和 select 类似，只是描述 fd 集合的方式不同，poll 使用 pollfd 结构而非 select 的 fd_set 结构。

```
struct pollfd {    int fd;           // 要监听的文件描述符    short events;     // 要监听的事件    short revents;    // 文件描述符fd上实际发生的事件};
```

管理多个描述符也是进行轮询，根据描述符的状态进行处理，但 **poll 无最大文件描述符数量的限制**，**因其基于链表存储**。

select 和 poll 在内部机制方面并没有太大的差异。相比于 select 机制，poll 只是取消了最大监控文件描述符数限制，并没有从根本上解决 select 存在的问题。

### 2.3 epoll 实现原理

epoll 是对 select 和 poll 的改进，解决了“性能开销大”和“文件描述符数量少”这两个缺点，是性能最高的多路复用实现方式，能支持的并发量也是最大。

epoll 的特点是：

1）使用**红黑树**存储**一份**文件描述符集合，每个文件描述符只需在添加时传入一次，无需用户每次都重新传入；—— 解决了 select 中 fd_set 重复拷贝到内核的问题

2）通过异步 IO 事件找到就绪的文件描述符，而不是通过轮询的方式；

3）使用队列存储就绪的文件描述符，且会按需返回就绪的文件描述符，无须再次遍历；

epoll 的基本用法是：

```
int main(void){      struct epoll_event events[5];      int epfd = epoll_create(10);         // 创建一个 epoll 对象      ......      for(i = 0; i < 5; i++)      {          static struct epoll_event ev;          .....          ev.data.fd = accept(sock_fd, (struct sockaddr *)&client_addr, &sin_size);          ev.events = EPOLLIN;          epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev);  // 向 epoll 对象中添加要管理的连接      }      while(1)      {         nfds = epoll_wait(epfd, events, 5, 10000);   // 等待其管理的连接上的 IO 事件         for(i=0; i<nfds; i++)         {             ......             read(events[i].data.fd, buff, MAXBUF)         }  }
```

主要涉及到三个函数：

```
int epoll_create(int size);   // 创建一个 eventpoll 内核对象int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);   // 将连接到socket对象添加到 eventpoll 对象上，epoll_event是要监听的事件int epoll_wait(int epfd, struct epoll_event *events,               int maxevents, int timeout);      // 等待连接 socket 的数据是否到达
```

**epoll_create**

epoll_create 函数会创建一个 struct eventpoll 的内核对象，类似 socket，把它关联到当前进程的已打开文件列表中。

eventpoll 主要包含三个字段：

```
struct eventpoll { wait_queue_head_t wq;      // 等待队列链表，存放阻塞的进程 struct list_head rdllist;  // 数据就绪的文件描述符都会放到这里 struct rb_root rbr;        // 红黑树，管理用户进程下添加进来的所有 socket 连接        ......}
```

wq：等待队列，如果当前进程没有数据需要处理，会把当前进程描述符和回调函数 default_wake_functon 构造一个等待队列项，放入当前 wq 对待队列，软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。

rbr：一棵红黑树，管理用户进程下添加进来的所有 socket 连接。

rdllist：就绪的描述符的链表。当有的连接数据就绪的时候，内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树。

eventpoll 的结构如图 2.3 所示：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141456261978920.png)

**epoll_ctl**

epoll_ctl 函数主要负责把服务端和客户端建立的 socket 连接注册到 eventpoll 对象里，会做三件事：

1）创建一个 epitem 对象，主要包含两个字段，分别存放 socket fd 即连接的文件描述符，和所属的 eventpoll 对象的指针；

2）将一个数据到达时用到的回调函数添加到 socket 的进程等待队列中，注意，跟第 1.1 节的阻塞 IO 模式不同的是，这里添加的 socket 的进程等待队列结构中，只有回调函数，没有设置进程描述符，因为**在 epoll 中，进程是放在 eventpoll 的等待队列中**，等待被 epoll_wait 函数唤醒，而不是放在 socket 的进程等待队列中；

3）将第 1）步创建的 epitem 对象插入红黑树；

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141456385278412.png)

**epoll_wait**

epoll_wait 函数的动作比较简单，检查 eventpoll 对象的就绪的连接 rdllist 上是否有数据到达，如果没有就把当前的进程描述符添加到一个等待队列项里，加入到 eventpoll 的进程等待队列里，然后阻塞当前进程，等待数据到达时通过回调函数被唤醒。

当 eventpoll 监控的连接上有数据到达时，通过下面几个步骤唤醒对应的进程处理数据：

1）socket 的数据接收队列有数据到达，会通过进程等待队列的回调函数 ep_poll_callback 唤醒红黑树中的节点 epitem；

2）ep_poll_callback 函数将有数据到达的 epitem 添加到 eventpoll 对象的就绪队列 rdllist 中；

3）ep_poll_callback 函数检查 eventpoll 对象的进程等待队列上是否有等待项，通过回调函数 default_wake_func 唤醒这个进程，进行数据的处理；

4）当进程醒来后，继续从 epoll_wait 时暂停的代码继续执行，把 rdlist 中就绪的事件返回给用户进程，让用户进程调用 recv 把已经到达内核 socket 等待队列的数据拷贝到用户空间使用。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212141456489832038.png)

## **3. 总结**

从阻塞 IO 到 epoll 的实现中，我们可以看到 **wake up 回调函数机制**被频繁的使用，至少有三处地方：一是阻塞 IO 中数据到达 socket 的等待队列时，通过回调函数唤醒进程，二是 epoll 中数据到达 socket 的等待队列时，通过回调函数 ep_poll_callback 找到 eventpoll 中红黑树的 epitem 节点，并将其加入就绪列队 rdllist，三是通过回调函数 default_wake_func 唤醒用户进程 ，并将 rdllist 传递给用户进程，让用户进程准确读取数据 。从中可知，这种回调机制能够定向准确的通知程序要处理的事件，而不需要每次都循环遍历检查数据是否到达以及数据该由哪个进程处理，提高了程序效率，在日常的业务开发中，我们也可以借鉴下这一机制。

**References**

[图解 | 深入揭秘 epoll 是如何实现 IO 多路复用的！](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484905&idx=1&sn=a74ed5d7551c4fb80a8abe057405ea5e&scene=21#wechat_redirect)

[图解 Linux 网络包接收过程](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484058&idx=1&sn=a2621bc27c74b313528eefbc81ee8c0f&scene=21#wechat_redirect)

[图解 | 深入理解高性能网络开发路上的绊脚石 - 同步阻塞网络 IO](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484834&idx=1&sn=b8620f402b68ce878d32df2f2bcd4e2e&scene=21#wechat_redirect)

[从 linux 源码看 socket 的阻塞和非阻塞](https://developer.aliyun.com/article/626844)

[Select、Poll、Epoll 详解](https://www.jianshu.com/p/722819425dbd/)

[你管这破玩意叫 IO 多路复用？](https://mp.weixin.qq.com/s?__biz=Mzk0MjE3NDE0Ng==&mid=2247494866&idx=1&sn=0ebeb60dbc1fd7f9473943df7ce5fd95&scene=21#wechat_redirect)

[I/O 多路复用，select / poll / epoll 详解](https://imageslr.com/2020/02/27/select-poll-epoll.html)

[大话 Select、Poll、Epoll](https://cloud.tencent.com/developer/article/1005481)

[IO 多路复用底层原理全解，select，poll，epoll，socket，系统中断，进程调度，系统调用](https://www.bilibili.com/video/BV1Ka4y177gs/?p=2&spm_id_from=pageDriver&vd_source=0c7b3e851a63cba48e0c6c9ad2c57629)

原文作者：mingguangtu，腾讯 IEG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/5xj42JPKG8o5T7hjXIKywg