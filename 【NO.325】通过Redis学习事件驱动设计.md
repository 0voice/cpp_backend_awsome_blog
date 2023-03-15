# 【NO.325】通过Redis学习事件驱动设计

## 1.**为什么C程序员都要阅读Redis源码**

主要原因就是『简洁』。如果你用源码编译过Redis，你会发现十分轻快，一步到位。其他语言的开发者可能不会了解这种痛，作为C/C++程序员，如果你源码编译安装过Nginx/Grpc/Thrift/Boost等开源产品，你会发现有很多依赖，而依赖也有自己的依赖，十分苦恼。通常半天一天就耗进去了。由衷地羡慕 npm/maven/pip/composer/...这些包管理器。而Redis则给人惊喜，一行make了此残生。

除了安装过程简洁，代码也十分简洁。使用纯C语言编写，每个模块功能都划分的很清晰。

废话不多说，本文要介绍的是Redis里的事件处理功能，与Memcache引入libevent这一臃肿的事件库不同，Redis自己实现了一个小型轻量的事件驱动库——AE。阅读它的源码是一次非常好的学习体验。

## 2.**干净整齐的跨平台兼容**

文件名说明ae.h/ae.c主要文件，根据OS平台的不同依赖以下不同文件：

| 文件        | 平台         |
| ----------- | ------------ |
| ae_epoll.c  | Linux平台    |
| ae_kqueue.c | BSD平台      |
| ae_evport.c | Solaris平台  |
| ae_select.c | 其他Unix平台 |

虽然源码文件看起来不少，但是实际上ae_epoll.c、 ae_kqueue.c、 ae_evport.c、 ae_select.c 这4个文件的功能是完全一样的，提供一致的API接口，给ae.c文件调用。这是由于实现高性能的事件驱动的API（称之为**polling API**）不存在ANSI或POSIX的标准，不同的OS内核有着自己的实现。比如Linux内核的epoll，BSD内核中的kqueue。

ae.c中有：

```text
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

请注意这里include的都是源文件，而非头文件。为什么这样？大家可以自己思考一下。

这些HAVE的宏，都是由在config.h中定义的。依据不同的操作系统，引入这4个文件中的某一个。从功能上来说，这样设计的目的与GOF设计模式中“适配器模式”（修改成一致接口）或“外观模式”（抽象复杂接口为简单接口）的思想类似，但实际上我个人感觉更类似于《POSA》（卷二）中提到的“包装门面模式”（Wrapper Facade）。Anyway，这个编程思想值得学习。

## 3.**aeEventLoop:用C++去设计，用C编码**

十几年前，以Linux之父炮轰C++为开端，社区内展开了一场C与C++孰是孰非的论战。而在国内，以原CSDN总编刘江援引此文为始，把战火烧到了国内。孟岩、云风、pongba几位大佬都身陷其中。后来以孟岩的一句『用C设计，用C++编码』在国内为这场论战定下基调。

反观Redis，他是纯C编码，但是融入了面向对象的思想。和上述观点截然相反，可谓是『用C++去设计，用C编码』。当然本文目的并非挑起语言之争，各种语言自有其利弊，开源项目的语言选择也主要是由于项目作者的个人经历和主观意愿。

定义在ae.h中的结构体 aeEventLoop 是AE库中最核心的数据结构，并且它采用了面向对象的设计思想：ae.h 中声明了多个函数，其第一个参数都是一个aeEventLoop指针，用于操纵aeEventLoop结构体。
从这个角度来说，可以将该结构体理解为面向对象语言中的类，而操纵它的函数则可以视为其成员函数。（其实C++的class编译之后大概也是类似的模式）

| 函数                 | 说明                                   |
| -------------------- | -------------------------------------- |
| aeCreateEventLoop    | 初始化一个事件循环结构体（eventLoop)   |
| aeGetSetSize         | 返回当前setsize的值                    |
| aeResizeSetSize      | 改变setsize的值（重新分配空间）        |
| aeDeleteEventLoop    | 删除事件循环，释放内存空间             |
| aeStop               | 停止事件循环，即stop值设为1            |
| aeProcessEvents      | ae核心：事件处理逻辑                   |
| aeMain               | 启动事件循环，事件循环的入口           |
| aeSetBeforeSleepProc | 注册回调函数，每次主循环在休眠前被调用 |

**aeCreateEventLoop** 和 **aeDeleteEventLoop** 可以视为“类”**aeEventLoop**的构造和析构函数，其他为成员函数。

在程序中调用AE库的时候，一般是依次调用：

1. aeCreateEventLoop
2. 给EventLoop注册文件事件or时间事件
3. aeSetBeforeSleepProc
4. aeMain
5. aeDeleteEventLoop

**AE的两种事件**

事件处理，是有别于多线程/多进程的并发模型。我也都知道Redis是单线程的。它的性能主要依靠异步事件处理功能来实现。虽然事件处理通常和网络编程混作一谈，但其实事件处理本身不一定是为网络编程服务的，它主要是服务于IO，网络通信是IO，文件读写同样是。当然Unix中万物皆文件了，socket也是一种fd。

AE支持两种事件：

1. 文件事件（IO）
2. 时间事件（毫秒级）

这两种事件都作为aeEventLoop的结构体成员存在。

aeEventLoop各成员说明:

```text
typedef struct aeEventLoop {
    int maxfd;                      /* 当前注册的最大fd */
    int setsize;                    /* 监视的fd的最大数量 */
    long long timeEventNextId;      /* 下一个时间事件的ID */
    time_t lastTime;                /* 上次时间事件处理时间 */
    aeFileEvent *events;            /* 已注册文件事件数组 */
    aeFiredEvent *fired;            /* 就绪的文件事件数组 */
    aeTimeEvent *timeEventHead;     /* 时间事件链表的头 */
    int stop;                       /* 是否停止（0：否；1：是）*/
    void *apidata;                  /* 各平台polling API所需的特定数据 */
    aeBeforeSleepProc *beforesleep; /* 事件循环休眠开始的处理函数 */
    aeBeforeSleepProc *aftersleep;  /* 事件循环休眠结束的处理函数 */
} aeEventLoop;
```

文件事件，主要依靠两个数组。一个是注册的文件事件数组，一个是已就绪的文件事件数组。

```text
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;
```



```text
typedef struct aeFiredEvent {
    int fd;
    int mask;
} aeFiredEvent;
```

## 4.**单词Fired有通知的意思，在这里你可以理解为“就绪”**

每个文件事件，其读写设置了不同的处理函数。另外mask表示事件的触发类型。当每次polling API返回就绪之后（比如epoll_wait返回），就绪会被设置到**aeFireEvent**，然后反查**aeFileEvent**获得处理函数并处理。你会发现aeFileEvent结构体里并没有记录fd。其实这是使用了HASH策略，aeEventLoop的成员 aeFileEvent数组的下标即是fd，便于快速查找。

时间事件，本质就是定时器任务，其数据结构采用一个双向链表。链表每个结点为**aeTimeEvent**结构体，主要包含事件的ID（递增）、就绪的时间，处理函数、清理函数、客户数据。

```text
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;
```

每个事件循环中，每个时间事件的ID唯一且递增，主要依赖aeEventLoop里的**timeEventNextId**来维护这个ID的递增关系。创建新的时间事件时（**aeCreateTimeEvent**）会赋值，由于只考虑了单线程，所以没有加锁逻辑，大家也不要贸然把AE用在多线程环境中。

when_sec和 when_ms 记录了时间事件的就绪时间（秒+毫秒），即当当前时间大于等于这个时间的时候，该时间事件应被处理。

时间事件的处理过程（**processTimeEvents**）主要就是：继续遍历链表，如果发现节点状态为AE_DELETED_EVENT_ID则删除该节点。如果判断当前时间已经超过节点的就绪时间就开始处理。处理函数的返回值可以指定，后续不再处理该事件（NOMORE），则该节点会被置为AE_DELETED_EVENT_ID。如果下次还需要处理，则更新该节点的时间为下次就绪时间。

## 5.**事件循环的处理逻辑**

再用一张图，回顾一下EventLoop中的两种事件，基本可以做如下理解。一个链表，一个数组。文件事件中的数组不是线性填满的，因为是采用的HASH策略，将fd作为数组下标了。

![img](https://pic4.zhimg.com/80/v2-a2e2ae526b9439fe288b8283ef6810c7_720w.webp)

aeProcessEvents 是aeEventLoop在循环过程中的的实际处理逻辑。函数原型如下：

```text
int aeProcessEvents(aeEventLoop *eventLoop, int flags);
```

flags标记，表示本次需要处理的事件类型标记和是否阻塞标记。

| 标记位         | 含义                     |
| -------------- | ------------------------ |
| AE_TIME_EVENTS | 时间事件标记             |
| AE_FILE_EVENTS | 文件事件标记             |
| AE_DONT_WAIT   | 立即返回不阻塞等待的标记 |

aeProcessEvents 具体实现代码我就不贴了。它巧妙的地方是一次柔和了文件和时间事件的两种处理过程。在函数之初，会查找时间事件的链表，找到最近就绪时间事件，然后用它的就绪时间减去当前时间的时间差作为 polling API 的休眠时间（epoll_wait的timeout参数）。然后休眠等待polling api返回。在返回之后先执行aftersleep的的处理逻辑，然后执行这段休眠时间内就绪的文件事件，最后再处理就绪的时间事件。返回值是处理过的事件总数。

也就说AE会尽量在一次处理过程中，将时间事件和文件事件一次性处理。你也许会问如果没有时间事件怎么办。当然没关系，在aeProcessEvents开始部分就根据标记位进行了判断。上面的逻辑是在文件事件和时间事件都存在的情况下，如果仅存在文件事件，则看是否设置了不阻塞的标记（AE_DONT_WAIT），若有，则polling 的超时时间设置为0。如无，即可以阻塞，则设置为-1，则polling API会阻塞直到有文件事件发生。

## 6.**ae_epoll:Linux上epoll的封装**

前文说道Redis适配各种Unix-like的操作系统。它将内核强相关的事件API（polling API）部分单独抽出来，包装出了相同的接口给AE的对外API调用。在Linux系统上的API实现为：ae_epoll.c，建议在阅读这个文件源码之前先好好回顾一下epoll的API，这样更助于快速理解。相信工作后大家写业务逻辑，应该很少接触epoll了。可以阅读这个wik，快速回顾epoll的api：LinuxAPI：epoll

ae_epoll.c 完全被 ae.c调用。各函数调用关系如下(aeApi开头的都是ae_epoll.c中的函数)：
ae.c

- aeCreateEventLoop

- - aeApiCreate

- aeResizeSetSize

- - aeApiResize

- aeDeleteEventLoop

- - aeApiFree

- aeCreateFileEvent

- - aeApiAddEvent

- aeDeleteFileEvent

- - aeApiDelEvent

- aeProcessEvent

- - aeApiPoll

- aeGetApiName

- - aeApiName

除了**aeApiName**()以外，其他函数第一个参数也都是aeEventLoop * 。用面向对象的思想来看这也是aeEventLoop的成员函数。试想若是C++，则可能会被处理成父子两个类，而aeApi系列的函数是纯虚的。
aeApi的函数也是可以做到顾名即可思义。比如：

- aeApiCreate在堆上进行内存的分配，封装epoll_create创建epfd，并写入aeEventLoop。
- aeApiAddEvent、aeApiDelEvent是封装的epoll_ctl来对aeEventLoop的监控的epoll事件进行添加和删除。
- aeApiPoll是封装的epoll_wait开启事件循环，并且每次取出就绪的fd存入aeEventLoop的fired数组中，并置位相应的mask（读or写）
- aeApiResize、aeApiFree分别进行的是内存的重分配、资源的清理（关闭epfd，free内存）和epoll本身关联不大。

## 7.**Jim：AE的缘起**

阅读完AE代码，可能只需要一下午的时间，你会惊叹于作者的设计功力。其实里面也没有太多花哨的东西，但就是如此简洁清晰的给你呈现了一个完成度如此之高的事件驱动处理库。但我想即使大家都熟悉epoll、熟悉kqueue、熟悉数据结构也不一定能设计出来AE，所以把程序员比作代码的设计师、建筑师是丝毫不为过的。

AE的设计灵感其实也是受另外一个开源项目影响，它就是 Jim。

Redis的ae.c的开篇注释中就已注明：

```text
A simple event-driven programming library. 
Originally I wrote this code for the Jim's event-loop (Jim is a Tcl interpreter)
but later translated it in form of a library for easy reuse.
 
```

Redis作者说他最初是为了Jim项目的事件驱动功能编写了一套代码，后来将其改造成了一个易于复用的库。

所以AE其实可以脱离Redis而存在。这种松耦合，是一种设计的魅力。

Jim的源码在Github上有它的镜像，其中事件循环的代码在此：

[https://github.com/msteveb/jimtcl/blob/master/jim-eventloop.c](https://link.zhihu.com/?target=https%3A//github.com/msteveb/jimtcl/blob/master/jim-eventloop.c)

快速阅读一下Jim的代码，AE整体处理逻辑确实与Jim相似。但AE也不乏创新，比如抽象出polling API这层，达到了多平台的兼容和解耦，而Jim则强耦合了select。

我们常说『站在巨人的肩膀』，虽然Jim不是巨人，但作者通过为他编写代码，从而启发了AE，即使Jim最终被世人遗忘，而它的血肉也化作了土壤，滋养后来人，这就是开源运动的意义所在，也是魅力所在。学习的本质，其实就是模仿，然后改进，这不是抄袭，这是传承。

原文地址：https://zhuanlan.zhihu.com/p/517974884

作者：linux