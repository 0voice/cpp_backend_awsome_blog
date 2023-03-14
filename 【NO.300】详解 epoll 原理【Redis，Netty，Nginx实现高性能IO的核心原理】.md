# 【NO.300】详解 epoll 原理【Redis，Netty，Nginx实现高性能IO的核心原理】

## 1.【Redis，Netty，Nginx 等实现高性能IO的核心原理】

### 1.1.I/O

![img](https://pic1.zhimg.com/80/v2-f2d47755f938a2b5a7d4d2e7c75458a8_720w.webp)
输入输出(input/output)的对象可以是文件(file)， 网络(socket)，进程之间的管道(pipe)。在linux系统中，都用文件描述符(fd)来表示。

### 1.2.I/O 多路复用（multiplexing）

![img](https://pic4.zhimg.com/80/v2-c2417f7394fdc1bef0b34623dee68923_720w.webp)

I/O 多路复用的本质，是通过一种机制（系统内核缓冲I/O数据），让单个进程可以监视多个文件描述符，一旦某个描述符就绪（一般是读就绪或写就绪），能够通知程序进行相应的读写操作。

select、poll 和 epoll 都是 Linux API 提供的 IO 复用方式。
Linux中提供的epoll相关函数如下：

```
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

## 2.Unix五种IO模型

我们在进行编程开发的时候，经常会涉及到同步，异步，阻塞，非阻塞，IO多路复用等概念，这里简单总结一下。

Unix网络编程中的五种IO模型：

```
Blocking IO - 阻塞IO
NoneBlocking IO - 非阻塞IO
IO multiplexing - IO多路复用
signal driven IO - 信号驱动IO
asynchronous IO - 异步IO
```

对于一个network IO，它会涉及到两个系统对象：

1. Application 调用这个IO的进程
2. kernel 系统内核

那他们经历的两个交互过程是：

阶段1： wait for data 等待数据准备；
阶段2： copy data from kernel to user 将数据从内核拷贝到用户进程中。

之所以会有同步、异步、阻塞和非阻塞这几种说法就是根据程序在这两个阶段的处理方式不同而产生的。

## 3.事件

可读事件，当文件描述符关联的内核读缓冲区可读，则触发可读事件。
(可读：内核缓冲区非空，有数据可以读取)

可写事件，当文件描述符关联的内核写缓冲区可写，则触发可写事件。
(可写：内核缓冲区不满，有空闲空间可以写入）

## 4.通知机制

通知机制，就是当事件发生的时候，则主动通知。通知机制的反面，就是轮询机制。

## 5.epoll 的通俗解释

结合以上三条，epoll的通俗解释是：

> 当文件描述符的内核缓冲区非空的时候，发出可读信号进行通知，当写缓冲区不满的时候，发出可写信号通知的机制。

## 6.epoll 数据结构 + 算法

epoll 的核心数据结构是：1个红黑树和1个链表。还有3个核心API。如下图所示：

![img](https://pic1.zhimg.com/80/v2-6c7a55178a7b2f5454aa36a9ba4a12e0_720w.webp)

## 7.就绪列表的数据结构

就绪列表引用着就绪的socket，所以它应能够快速的插入数据。

程序可能随时调用epoll_ctl添加监视socket，也可能随时删除。当删除时，若该socket已经存放在就绪列表中，它也应该被移除。（事实上，每个epoll_item既是红黑树节点，也是链表节点，删除红黑树节点，自然删除了链表节点）

所以就绪列表应是一种能够快速插入和删除的数据结构。双向链表就是这样一种数据结构，epoll使用双向链表来实现就绪队列（对应上图的rdllist）。

## 8.epoll 索引结构

既然epoll将“维护监视队列”和“进程阻塞”分离，也意味着需要有个数据结构来保存监视的socket。至少要方便的添加和移除，还要便于搜索，以避免重复添加。红黑树是一种自平衡二叉查找树，搜索、插入和删除时间复杂度都是O(log(N))，效率较好。epoll 使用了红黑树作为索引结构。

Epoll在linux内核中源码主要为 eventpoll.c 和 eventpoll.h 主要位于fs/eventpoll.c 和 include/linux/eventpool.h, 具体可以参考linux3.16。

下述为部分关键数据结构摘要, 主要介绍epitem 红黑树节点 和eventpoll 关键入口数据结构，维护着链表头节点ready list header和红黑树根节点RB-Tree root。

```
/*
 * Each file descriptor added to the eventpoll interface will
 * have an entry of this type linked to the "rbr" RB tree.
 * Avoid increasing the size of this struct, there can be many thousands
 * of these on a server and we do not want this to take another cache line.
 */
struct epitem {
    union {
        /* RB tree node links this structure to the eventpoll RB tree */
        struct rb_node rbn;
        /* Used to free the struct epitem */
        struct rcu_head rcu;
    };

    /* List header used to link this structure to the eventpoll ready list */
    struct list_head rdllink;

    /*
     * Works together "struct eventpoll"->ovflist in keeping the
     * single linked chain of items.
     */
    struct epitem *next;

    /* The file descriptor information this item refers to */
    struct epoll_filefd ffd;

    /* Number of active wait queue attached to poll operations */
    int nwait;

    /* List containing poll wait queues */
    struct list_head pwqlist;

    /* The "container" of this item */
    struct eventpoll *ep;

    /* List header used to link this item to the "struct file" items list */
    struct list_head fllink;

    /* wakeup_source used when EPOLLWAKEUP is set */
    struct wakeup_source __rcu *ws;

    /* The structure that describe the interested events and the source fd */
    struct epoll_event event;
};

/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
    /* Protect the access to this structure */
    spinlock_t lock;

    /*
     * This mutex is used to ensure that files are not removed
     * while epoll is using them. This is held during the event
     * collection loop, the file cleanup path, the epoll file exit
     * code and the ctl operations.
     */
    struct mutex mtx;

    /* Wait queue used by sys_epoll_wait() */
    wait_queue_head_t wq;

    /* Wait queue used by file->poll() */
    wait_queue_head_t poll_wait;

    /* List of ready file descriptors */
    struct list_head rdllist;

    /* RB tree root used to store monitored fd structs */
    struct rb_root rbr;

    /*
     * This is a single linked list that chains all the "struct epitem" that
     * happened while transferring ready events to userspace w/out
     * holding ->lock.
     */
    struct epitem *ovflist;

    /* wakeup_source used when ep_scan_ready_list is running */
    struct wakeup_source *ws;

    /* The user that created the eventpoll descriptor */
    struct user_struct *user;

    struct file *file;

    /* used to optimize loop detection check */
    int visited;
    struct list_head visited_list_link;
};
```

epoll使用RB-Tree红黑树去监听并维护所有文件描述符，RB-Tree的根节点。

调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个 红黑树 用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件.

当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。而且，通常情况下即使我们要监控百万计的句柄，大多一次也只返回很少量的准备就绪句柄而已，所以，epoll_wait仅需要从内核态copy少量的句柄到用户态而已.

## 9.epoll API

### 9.1. int epoll_create(int size)

功能：

内核会产生一个epoll 实例数据结构并返回一个文件描述符，这个特殊的描述符就是epoll实例的句柄，后面的两个接口都以它为中心（即epfd形参）。size参数表示所要监视文件描述符的最大值，不过在后来的Linux版本中已经被弃用（同时，size不要传0，会报invalid argument错误）

### 9.2. int epoll_ctl(int epfd， int op， int fd， struct epoll_event *event)

功能：

将被监听的描述符添加到红黑树或从红黑树中删除或者对监听事件进行修改

```
typedef union epoll_data {
void *ptr; /* 指向用户自定义数据 */
int fd; /* 注册的文件描述符 */
uint32_t u32; /* 32-bit integer */
uint64_t u64; /* 64-bit integer */
} epoll_data_t;
struct epoll_event {
uint32_t events; /* 描述epoll事件 */
epoll_data_t data; /* 见上面的结构体 */
};
```

对于需要监视的文件描述符集合，epoll_ctl对红黑树进行管理，红黑树中每个成员由描述符值和所要监控的文件描述符指向的文件表项的引用等组成。

op参数说明操作类型：

EPOLL_CTL_ADD：向interest list添加一个需要监视的描述符EPOLL_CTL_DEL：从interest list中删除一个描述符EPOLL_CTL_MOD：修改interest list中一个描述符struct epoll_event结构描述一个文件描述符的epoll行为。在使用epoll_wait函数返回处于ready状态的描述符列表时，

data域是唯一能给出描述符信息的字段，所以在调用epoll_ctl加入一个需要监测的描述符时，一定要在此域写入描述符相关信息events域是bit mask，描述一组epoll事件，在epoll_ctl调用中解释为：描述符所期望的epoll事件，可多选。常用的epoll事件描述如下：

EPOLLIN：描述符处于可读状态EPOLLOUT：描述符处于可写状态EPOLLET：将epoll event通知模式设置成edge triggeredEPOLLONESHOT：第一次进行通知，之后不再监测EPOLLHUP：本端描述符产生一个挂断事件，默认监测事件EPOLLRDHUP：对端描述符产生一个挂断事件EPOLLPRI：由带外数据触发EPOLLERR：描述符产生错误时触发，默认检测事件

### 9.3. int epoll_wait(int epfd， struct epoll_event *events， int maxevents， int timeout);

功能：

阻塞等待注册的事件发生，返回事件的数目，并将触发的事件写入events数组中。

events: 用来记录被触发的events，其大小应该和maxevents一致

maxevents: 返回的events的最大个数处于ready状态的那些文件描述符会被复制进ready list中，epoll_wait用于向用户进程返回ready list。

events和maxevents两个参数描述一个由用户分配的struct epoll event数组，调用返回时，内核将ready list复制到这个数组中，并将实际复制的个数作为返回值。

注意，如果ready list比maxevents长，则只能复制前maxevents个成员；反之，则能够完全复制ready list。

另外，struct epoll event结构中的events域在这里的解释是：

> 在被监测的文件描述符上实际发生的事件。

参数timeout描述在函数调用中阻塞时间上限，单位是ms：

timeout = -1表示调用将一直阻塞，直到有文件描述符进入ready状态或者捕获到信号才返回；timeout = 0用于非阻塞检测是否有描述符处于ready状态，不管结果怎么样，调用都立即返回；timeout > 0表示调用将最多持续timeout时间，如果期间有检测对象变为ready状态或者捕获到信号则返回，否则直到超时。epoll的两种触发方式

epoll监控多个文件描述符的I/O事件。epoll支持边缘触发(edge trigger，ET)或水平触发（level trigger，LT)，通过epoll_wait等待I/O事件，如果当前没有可用的事件则阻塞调用线程。

select和poll只支持LT工作模式，epoll的默认的工作模式是LT模式。

## 10.epoll更高效的原因

select和poll的动作基本一致，只是poll采用链表来进行文件描述符的存储，而select采用fd标注位来存放，所以select会受到最大连接数的限制，而poll不会。

select、poll、epoll虽然都会返回就绪的文件描述符数量, 但是select和poll并不会明确指出是哪些文件描述符就绪，而epoll会。

造成的区别就是，系统调用返回后，调用select和poll的程序需要遍历监听的整个文件描述符找到是谁处于就绪，而epoll则直接处理即可（直接监听到了哪些文件描述符就绪）。

select、poll都需要将有关文件描述符的数据结构拷贝进内核，最后再拷贝出来。**而epoll创建的有关文件描述符的数据结构本身就存于内核态中，系统调用返回时利用 mmap() 文件映射内存加速与内核空间的消息传递：即 epoll 使用 mmap() 减少复制开销。**

select、poll采用 **轮询** 的方式来检查文件描述符是否处于就绪态，而epoll采用回调机制。造成的结果就是，随着fd的增加，select和poll的效率会线性降低，而epoll不会受到太大影响，除非活跃的socket很多。

epoll的边缘触发模式效率高，系统不会充斥大量不关心的就绪文件描述符。

虽然epoll的性能最好，但是在**连接数少并且连接都十分活跃**的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。

## 11.epoll与select、poll的对比

### 11.1. 用户态将文件描述符传入内核的方式

select：创建3个文件描述符集并拷贝到内核中，分别监听读、写、异常动作。这里受到单个进程可以打开的fd数量限制，默认是1024。

poll：将传入的struct pollfd结构体数组拷贝到内核中进行监听。

epoll：执行epoll_create，会在**内核**的高速cache区中，建立一颗红黑树以及就绪链表(该链表存储已经就绪的文件描述符)。接着用户执行的epoll_ctl函数，添加文件描述符会在红黑树上增加相应的结点。

### 11.2. 内核态检测文件描述符读写状态的方式

select：采用**轮询**方式，遍历所有fd，最后返回一个描述符读写操作是否就绪的mask掩码，根据这个掩码给fd_set赋值。

poll：同样采用**轮询**方式，查询每个fd的状态，如果就绪则在等待队列中加入一项并继续遍历。

epoll：采用**事件回调**机制。在执行 epoll_ctl 的add操作时，不仅将文件描述符放到红黑树上，而且也注册了回调函数;内核在检测到某文件描述符可读/可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。

### 11.3. 找到就绪的文件描述符并传递给用户态的方式

select：将之前传入的fd_set拷贝传出到**用户态**并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。

poll：将之前传入的 fd 数组拷贝传出**用户态**，并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。

epoll：epoll_wait 只用观察**就绪链表**中有无数据即可，最后将链表的数据返回给数组, 并返回就绪的数量。**内核**，将就绪的文件描述符，放在传入的数组中。然后，依次遍历，处理即可。这里返回的文件描述符，是通过 mmap() ，让内核和用户空间，共享同一块内存实现传递的，减少了不必要的拷贝。

### 11.4. 重复监听的处理方式

select：将新的监听文件描述符集合拷贝传入内核中，继续以上步骤。
poll：将新的struct pollfd结构体数组拷贝传入内核中，继续以上步骤。
epoll：无需重新构建红黑树，直接沿用已存在的即可。

## 12.epoll 水平触发与边缘触发

epoll事件有两种模型：

边沿触发：edge-triggered (ET)
水平触发：level-triggered (LT)

## 13.水平触发(level-triggered)

socket接收缓冲区不为空， 有数据可读， 读事件一直触发。
socket发送缓冲区不满， 可以继续写入数据， 写事件一直触发。

## 14.边沿触发(edge-triggered)

socket的接收缓冲区状态变化时，触发读事件，即空的接收缓冲区刚接收到数据时触发读事件。
socket的发送缓冲区状态变化时，触发写事件，即满的缓冲区刚空出空间时触发读事件。

边沿触发仅触发一次，水平触发会一直触发。

## 15.事件宏

EPOLLIN ： 表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT： 表示对应的文件描述符可以写；
EPOLLPRI： 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR： 表示对应的文件描述符发生错误；
EPOLLHUP： 表示对应的文件描述符被挂断；
EPOLLET： 将 EPOLL设为边缘触发(Edge Triggered)模式（默认为水平触发），这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT： 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
libevent 采用水平触发， nginx 采用边沿触发

JDK并没有实现边缘触发，Netty重新实现了epoll机制，采用边缘触发方式；另外像Nginx也采用边缘触发。

JDK在Linux已经默认使用epoll方式，但是JDK的epoll采用的是水平触发，而Netty重新实现了epoll机制，采用边缘触发方式，netty epoll transport 暴露了更多的nio没有的配置参数，如 TCP_CORK, SO_REUSEADDR等等；另外像Nginx也采用边缘触发。

## 16.mmap() 文件映射内存

mmap是一种内存映射文件（文件映射内存，建立映射关系）的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。

实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上（参考：Linux文件读写与缓存），即完成了对文件的操作而不必再调用read,write等系统调用函数。

> Dirty Page： 页缓存对应文件中的一块区域，如果页缓存和对应的文件区域内容不一致，则该页缓存叫做脏页（Dirty Page）。对页缓存进行修改或者新建页缓存，只要没有刷磁盘，都会产生脏页。Linux支持以下5种缓冲区类型：
> Clean 未使用、新创建的缓冲区
> Locked 被锁住、等待被回写
> Dirty 包含最新的有效数据，但还没有被回写
> Shared 共享的缓冲区
> Unshared 原来被共享但现在不共享

相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。如下图所示：

![img](https://pic4.zhimg.com/80/v2-88d705451c9b9f6d21102813786bc8ef_720w.webp)

由上图可以看出，进程的虚拟地址空间，由多个虚拟内存区域构成。虚拟内存区域是进程的虚拟地址空间中的一个同质区间，即具有同样特性的连续地址范围。上图中所示的text数据段（代码段）、初始数据段、BSS数据段、堆、栈和内存映射，都是一个独立的虚拟内存区域。而为内存映射服务的地址空间处在堆栈之间的空余部分。

linux内核使用vm_area_struct结构来表示一个独立的虚拟内存区域，由于每个不同质的虚拟内存区域功能和内部机制都不同，因此一个进程使用多个vm_area_struct结构来分别表示不同类型的虚拟内存区域。各个vm_area_struct结构使用链表或者树形结构链接，方便进程快速访问，如下图所示：

![img](https://pic1.zhimg.com/80/v2-2765fa33d5a4553c76041f0925b8ec9c_720w.webp)

vm_area_struct结构中包含区域起始和终止地址以及其他相关信息，同时也包含一个vm_ops指针，其内部可引出所有针对这个区域可以使用的系统调用函数。这样，进程对某一虚拟内存区域的任何操作需要用要的信息，都可以从vm_area_struct中获得。mmap函数就是要创建一个新的vm_area_struct结构，并将其与文件的物理磁盘地址相连。

## 17.mmap在write和read时会发生什么

### **17.1.write**

- 1.进程(用户态)将需要写入的数据直接copy到对应的mmap地址(内存copy)
- 2.若mmap地址未对应物理内存，则产生缺页异常，由内核处理
- 3.若已对应，则直接copy到对应的物理内存
- 4.由操作系统调用，将脏页回写到磁盘（通常是异步的）

因为物理内存是有限的，mmap在写入数据超过物理内存时，操作系统会进行页置换，根据淘汰算法，将需要淘汰的页置换成所需的新页，所以mmap对应的内存是可以被淘汰的（若内存页是”脏”的，则操作系统会先将数据回写磁盘再淘汰）。这样，就算mmap的数据远大于物理内存，操作系统也能很好地处理，不会产生功能上的问题。

### **17.2.read**

![img](https://pic3.zhimg.com/80/v2-ee003c210c5d77d97df12bf4c2b3b666_720w.webp)

从图中可以看出，mmap要比普通的read系统调用少了一次copy的过程。因为read调用，进程是无法直接访问kernel space的，所以在read系统调用返回前，内核需要将数据从内核复制到进程指定的buffer。但mmap之后，进程可以直接访问mmap的数据(page cache)。

## 18.总结

一张图总结一下select,poll,epoll的区别：

![img](https://pic1.zhimg.com/80/v2-5a2a074550dda2645bc3f0a65d66f108_720w.webp)

epoll是Linux目前大规模网络并发程序开发的首选模型。在绝大多数情况下性能远超select和poll。目前流行的高性能web服务器Nginx正式依赖于epoll提供的高效网络套接字轮询服务。但是，在并发连接不高的情况下，多线程+阻塞I/O方式可能性能更好。

既然select，poll，epoll都是I/O多路复用的具体的实现，之所以现在同时存在，其实他们也是不同历史时期的产物：

- select出现是1984年在BSD里面实现的

- 14年之后也就是1997年才实现了poll，其实拖那么久也不是效率问题， 而是那个时代的硬件实在太弱，一台服务器处理1千多个链接简直就是神一样的存在了，select很长段时间已经满足需求

- 2002, 大神 Davide Libenzi 实现了epoll。

  

  原文链接：https://zhuanlan.zhihu.com/p/364832778

  作者：[Linux服务器研究](https://www.zhihu.com/people/shao-nian-bu-nian-shao-zhu-80)