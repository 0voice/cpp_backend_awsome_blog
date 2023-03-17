# 【NO.620】c++ 协程_关于协程的实现与原理，多年程序员深度总结

**前言**

协程这个概念很久了，好多程序员是实现过这个组件的，网上关于协程的文章，博客，论坛都是汗牛充栋，在知乎，github上面也有很多大牛写了关于协程的心得体会。突发奇想，我也来实现一个这样的组件，并测试了一下性能。借鉴了很多大牛的思想，阅读了很多大牛的代码。于是把整个思考过程写下来。实现代码

[https://github.com/wangbojing/NtyCo](https://link.zhihu.com/?target=https%3A//github.com/wangbojing/NtyCo)

代码简单易读，如果在你的项目中，NtyCo能够为你解决些许工程问题，那就荣幸之至。

本文章的设计思路，是在每一个章的最前面以问题提出，每章节的学习目的。大家能够带着每章的问题来读每章节的内容，方便读者能够方便的进入每章节的思考。读者读完以后加上案例代码阅读，编译，运行，能够对神秘的协程有一个全新的理解。能够运用到工程代码，帮助你更加方便高效的完成工程工作。

本文章仅代表本人观点，若不有严谨的地方，欢迎抛转。

## 1.第一章 协程的起源

**问题：协程存在的原因？协程能够解决哪些问题？**

在我们现在CS，BS开发模式下，服务器的吞吐量是一个很重要的参数。其实吞吐量是IO处理时间加上业务处理。为了简单起见，比如，客户端与服务器之间是长连接的，客户端定期给服务器发送心跳包数据。客户端发送一次心跳包到服务器，服务器更新该新客户端状态的。心跳包发送的过程，业务处理时长等于IO读取（RECV系统调用）加上业务处理（更新客户状态）。吞吐量等于1s业务处理次数。

业务处理（更新客户端状态）时间，业务不一样的，处理时间不一样，我们就不做讨论。

那如何提升recv的性能。若只有一个客户端，recv的性能也没有必要提升，也不能提升。若在有百万计的客户端长连接的情况，我们该如何提升。以Linux为例，在这里需要介绍一个“网红”就是epoll。服务器使用epoll管理百万计的客户端长连接，代码框架如下：

```text
while (1) {

 int nready = epoll_wait(epfd, events, EVENT_SIZE, -1);

 

 for (i = 0;i < nready;i ++) {

 

 int sockfd = events[i].data.fd;

 if (sockfd == listenfd) {

 int connfd = accept(listenfd, xxx, xxxx);

 

            setnonblock(connfd);

 

            ev.events = EPOLLIN | EPOLLET;

            ev.data.fd = connfd;

            epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);

 

        } else {

            handle(sockfd);

        }

    }

}
```

对于响应式服务器，所有的客户端的操作驱动都是来源于这个大循环。来源于epoll_wait的反馈结果。

对于服务器处理百万计的IO。Handle(sockfd)实现方式有两种。

**第一种**，handle(sockfd)函数内部对sockfd进行读写动作。代码如下

```text
int handle(int sockfd) {

 

 recv(sockfd, rbuffer, length, 0);

 

    parser_proto(rbuffer, length);

 

 send(sockfd, sbuffer, length, 0);

 

}
```



handle的io操作（send,recv）与epoll_wait是在同一个处理流程里面的。这就是IO同步操作。

优点：

\1. sockfd管理方便。

\2. 操作逻辑清晰。

缺点：

\1. 服务器程序依赖epoll_wait的循环响应速度慢。

\2. 程序性能差

**第二种**，handle(sockfd)函数内部将sockfd的操作，push到线程池中，代码如下：

```text
int thread_cb(int sockfd) {

 // 此函数是在线程池创建的线程中运行。

    // 与handle不在一个线程上下文中运行

 recv(sockfd, rbuffer, length, 0);

    parser_proto(rbuffer, length);

 send(sockfd, sbuffer, length, 0);

}

 

int handle(int sockfd) {

 //此函数在主线程 main_thread 中运行

    //在此处之前，确保线程池已经启动。

    push_thread(sockfd, thread_cb); //将sockfd放到其他线程中运行。

}
```

Handle函数是将sockfd处理方式放到另一个已经其他的线程中运行，如此做法，将io操作（recv，send）与epoll_wait 不在一个处理流程里面，使得io操作（recv,send）与epoll_wait实现解耦。这就叫做IO异步操作。

优点：

\1. 子模块好规划。

\2. 程序性能高。

缺点：

正因为子模块好规划，使得模块之间的sockfd的管理异常麻烦。每一个子线程都需要管理好sockfd，避免在IO操作的时候，sockfd出现关闭或其他异常。

**上文有提到IO同步操作，程序响应慢，IO异步操作，程序响应快。**

**下面来对比一下IO同步操作与IO异步操作。**

代码如下：

[https://github.com/wangbojing/c1000k_test/blob/master/server_mulport_epoll.c](https://link.zhihu.com/?target=https%3A//github.com/wangbojing/c1000k_test/blob/master/server_mulport_epoll.c)

在这份代码的486行，#if 1, 打开的时候，为IO异步操作。关闭的时候，为IO同步操作。

接下来把我测试接入量的结果粘贴出来。

IO异步操作，每1000个连接接入的服务器响应时间（900ms左右）。

IO同步操作，每1000个连接接入的服务器响应时间（6500ms左右）。

IO异步操作与IO同步操作

**对比项**

**IO同步操作**

**IO异步操作**

Sockfd管理

管理方便

多个线程共同管理

代码逻辑

程序整体逻辑清晰

子模块逻辑清晰

程序性能

响应时间长，性能差

响应时间短，性能好

有没有一种方式，有异步性能，同步的代码逻辑。来方便编程人员对IO操作的组件呢？ 有，采用一种轻量级的协程来实现。在每次send或者recv之前进行切换，再由调度器来处理epoll_wait的流程。

就是采用了基于这样的思考，写了NtyCo，实现了一个IO异步操作与协程结合的组件。[https://github.com/wangbojing/NtyCo](https://link.zhihu.com/?target=https%3A//github.com/wangbojing/NtyCo)，

## 2.第二章 协程的案例

**问题：协程如何使用？与线程使用有何区别？**

在做网络IO编程的时候，有一个非常理想的情况，就是每次accept返回的时候，就为新来的客户端分配一个线程，这样一个客户端对应一个线程。就不会有多个线程共用一个sockfd。每请求每线程的方式，并且代码逻辑非常易读。但是这只是理想，线程创建代价，调度代价就呵呵了。

先来看一下每请求每线程的代码如下：

```text
while(1) {

 socklen_t len = sizeof(struct sockaddr_in);

 int clientfd = accept(sockfd, (struct sockaddr*)&remote, &len);

 

 pthread_t thread_id;

 pthread_create(&thread_id, NULL, client_cb, &clientfd);

 

}
```

这样的做法，写完放到生产环境下面，如果你的老板不打死你，你来找我。我来帮你老板，为民除害。

如果我们有协程，我们就可以这样实现。参考代码如下：

```text
https://github.com/wangbojing/NtyCo/blob/master/nty_server_test.c


while (1) {

    socklen_t len = sizeof(struct sockaddr_in);

    int cli_fd = nty_accept(fd, (struct sockaddr*)&remote, &len);

 

    nty_coroutine *read_co;

    nty_coroutine_create(&read_co, server_reader, &cli_fd);

 

}
```

这样的代码是完全可以放在生成环境下面的。如果你的老板要打死你，你来找我，我帮你把你老板打死，为民除害。

**线程的API思维来使用协程，函数调用的性能来测试协程。**

NtyCo封装出来了若干接口，一类是协程本身的，二类是posix的异步封装

协程API：while

\1. 协程创建

int nty_coroutine_create(nty_coroutine **new_co, proc_coroutine func, void *arg)

\2. 协程调度器的运行

void nty_schedule_run(void)

POSIX异步封装API：

```text
int nty_socket(int domain, int type, int protocol)

int nty_accept(int fd, struct sockaddr *addr, socklen_t *len)

int nty_recv(int fd, void *buf, int length)

int nty_send(int fd, const void *buf, int length)

int nty_close(int fd)
```

接口格式与POSIX标准的函数定义一致。

## 3.第三章 协程的实现之工作流程

**问题：协程内部是如何工作呢？**

先来看一下协程服务器案例的代码， 代码参考：[https://github.com/wangbojing/NtyCo/blob/master/nty_server_test.c](https://link.zhihu.com/?target=https%3A//github.com/wangbojing/NtyCo/blob/master/nty_server_test.c)

分别讨论三个协程的比较晦涩的工作流程。第一个协程的创建；第二个IO异步操作；第三个协程子过程回调

### 3.1 创建协程

当我们需要异步调用的时候，我们会创建一个协程。比如accept返回一个新的sockfd，创建一个客户端处理的子过程。再比如需要监听多个端口的时候，创建一个server的子过程，这样多个端口同时工作的，是符合微服务的架构的。

创建协程的时候，进行了如何的工作？创建API如下：

int nty_coroutine_create(nty_coroutine **new_co, proc_coroutine func, void *arg)

参数1：nty_coroutine **new_co，需要传入空的协程的对象，这个对象是由内部创建的，并且在函数返回的时候，会返回一个内部创建的协程对象。

参数2：proc_coroutine func，协程的子过程。当协程被调度的时候，就会执行该函数。

参数3：void *arg，需要传入到新协程中的参数。

协程不存在亲属关系，都是一致的调度关系，接受调度器的调度。调用create API就会创建一个新协程，新协程就会加入到调度器的就绪队列中。

创建的协程具体步骤会在《协程的实现之原语操作》来描述。

### 3.2 实现IO异步操作

大部分的朋友会关心IO异步操作如何实现，在send与recv调用的时候，如何实现异步操作的。

先来看一下一段代码：

```text
while (1) {

 int nready = epoll_wait(epfd, events, EVENT_SIZE, -1);

 

 for (i = 0;i < nready;i ++) {

 

 int sockfd = events[i].data.fd;

 if (sockfd == listenfd) {

 int connfd = accept(listenfd, xxx, xxxx);

 

            setnonblock(connfd);

 

            ev.events = EPOLLIN | EPOLLET;

            ev.data.fd = connfd;

            epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);

 

        } else {

 

            epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);

 recv(sockfd, buffer, length, 0);

 

 //parser_proto(buffer, length);

 

 send(sockfd, buffer, length, 0);

            epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, NULL);

        }

    }

}
```



在进行IO操作（recv，send）之前，先执行了 epoll_ctl的del操作，将相应的sockfd从epfd中删除掉，在执行完IO操作（recv，send）再进行epoll_ctl的add的动作。这段代码看起来似乎好像没有什么作用。

如果是在多个上下文中，这样的做法就很有意义了。能够保证sockfd只在一个上下文中能够操作IO的。不会出现在多个上下文同时对一个IO进行操作的。协程的IO异步操作正式是采用此模式进行的。

把单一协程的工作与调度器的工作的划分清楚，先引入两个原语操作 resume，yield会在《协程的实现之原语操作》来讲解协程所有原语操作的实现，yield就是让出运行，resume就是恢复运行。调度器与协程的上下文切换如下图所示

在协程的上下文IO异步操作（nty_recv，nty_send）函数，步骤如下：

\1. 将sockfd 添加到epoll管理中。

\2. 进行上下文环境切换，由协程上下文yield到调度器的上下文。

\3. 调度器获取下一个协程上下文。Resume新的协程

IO异步操作的上下文切换的时序图如下：

### 3.3 回调协程的子过程

在create协程后，何时回调子过程？何种方式回调子过程？

首先来回顾一下x86_64寄存器的相关知识。汇编与寄存器相关知识还会在《协程的实现之切换》继续深入探讨的。x86_64 的寄存器有16个64位寄存器，分别是：%rax, %rbx,

%rcx, %esi, %edi, %rbp, %rsp, %r8, %r9, %r10, %r11, %r12, %r13, %r14, %r15。

%rax 作为函数返回值使用的。

%rsp 栈指针寄存器，指向栈顶

%rdi, %rsi, %rdx, %rcx, %r8, %r9 用作函数参数，依次对应第1参数，第2参数。。。

%rbx, %rbp, %r12, %r13, %r14, %r15 用作数据存储，遵循调用者使用规则，换句话说，就是随便用。调用子函数之前要备份它，以防它被修改

%r10, %r11 用作数据存储，就是使用前要先保存原值

以NtyCo的实现为例，来分析这个过程。CPU有一个非常重要的寄存器叫做EIP，用来存储CPU运行下一条指令的地址。我们可以把回调函数的地址存储到EIP中，将相应的参数存储到相应的参数寄存器中。实现子过程调用的逻辑代码如下：

```text
void _exec(nty_coroutine *co) {

    co->func(co->arg); //子过程的回调函数

}

 

void nty_coroutine_init(nty_coroutine *co) {

 //ctx 就是协程的上下文

    co->ctx.edi = (void*)co; //设置参数

    co->ctx.eip = (void*)_exec; //设置回调函数入口

 //当实现上下文切换的时候，就会执行入口函数_exec , _exec 调用子过程func

}
```

## 4.第四章 协程的实现之原语操作

**问题：协程的内部原语操作有哪些？分别如何实现的？**

协程的核心原语操作：create, resume, yield。协程的原语操作有create怎么没有exit？以NtyCo为例，协程一旦创建就不能有用户自己销毁，必须得以子过程执行结束，就会自动销毁协程的上下文数据。以_exec执行入口函数返回而销毁协程的上下文与相关信息。co->func(co->arg) 是子过程，若用户需要长久运行协程，就必须要在func函数里面写入循环等操作。所以NtyCo里面没有实现exit的原语操作。

create：创建一个协程。

\1. 调度器是否存在，不存在也创建。调度器作为全局的单例。将调度器的实例存储在线程的私有空间pthread_setspecific。

\2. 分配一个coroutine的内存空间，分别设置coroutine的数据项，栈空间，栈大小，初始状态，创建时间，子过程回调函数，子过程的调用参数。

\3. 将新分配协程添加到就绪队列 ready_queue中

实现代码如下：



```text
int nty_coroutine_create(nty_coroutine **new_co, proc_coroutine func, void *arg) {

 

    assert(pthread_once(&sched_key_once, nty_coroutine_sched_key_creator) == 0);

    nty_schedule *sched = nty_coroutine_get_sched();

 

 if (sched == NULL) {

        nty_schedule_create(0);

 

        sched = nty_coroutine_get_sched();

 if (sched == NULL) {

            printf("Failed to create schedulern");

 return -1;

        }

    }

 

    nty_coroutine *co = calloc(1, sizeof(nty_coroutine));

 if (co == NULL) {

        printf("Failed to allocate memory for new coroutinen");

 return -2;

    }

 

 //

 int ret = posix_memalign(&co->stack, getpagesize(), sched->stack_size);

 if (ret) {

        printf("Failed to allocate stack for new coroutinen");

        free(co);

 return -3;

    }

 

    co->sched = sched;

    co->stack_size = sched->stack_size;

    co->status = BIT(NTY_COROUTINE_STATUS_NEW); //

    co->id = sched->spawned_coroutines ++;

co->func = func;

 

    co->fd = -1;

co->events = 0;

 

    co->arg = arg;

    co->birth = nty_coroutine_usec_now();

    *new_co = co;

 

    TAILQ_INSERT_TAIL(&co->sched->ready, co, ready_next);

 

 return 0;

}
```

yield： 让出CPU。

void nty_coroutine_yield(nty_coroutine *co)

参数：当前运行的协程实例

调用后该函数不会立即返回，而是切换到最近执行resume的上下文。该函数返回是在执行resume的时候，会有调度器统一选择resume的，然后再次调用yield的。resume与yield是两个可逆过程的原子操作。

resume：恢复协程的运行权

int nty_coroutine_resume(nty_coroutine *co)

参数：需要恢复运行的协程实例

调用后该函数也不会立即返回，而是切换到运行协程实例的yield的位置。返回是在等协程相应事务处理完成后，主动yield会返回到resume的地方。

## 5.第五章 协程的实现之切换

**问题：协程的上下文如何切换？切换代码如何实现？**

首先来回顾一下x86_64寄存器的相关知识。x86_64 的寄存器有16个64位寄存器，分别是：%rax, %rbx, %rcx, %esi, %edi, %rbp, %rsp, %r8, %r9, %r10, %r11, %r12,

%r13, %r14, %r15。

%rax 作为函数返回值使用的。

%rsp 栈指针寄存器，指向栈顶

%rdi, %rsi, %rdx, %rcx, %r8, %r9 用作函数参数，依次对应第1参数，第2参数。。。

%rbx, %rbp, %r12, %r13, %r14, %r15 用作数据存储，遵循调用者使用规则，换句话说，就是随便用。调用子函数之前要备份它，以防它被修改

%r10, %r11 用作数据存储，就是使用前要先保存原值。

上下文切换，就是将CPU的寄存器暂时保存，再将即将运行的协程的上下文寄存器，分别mov到相对应的寄存器上。此时上下文完成切换。如下图所示：

切换_switch函数定义：

int _switch(nty_cpu_ctx *new_ctx, nty_cpu_ctx *cur_ctx);

参数1：即将运行协程的上下文，寄存器列表

参数2：正在运行协程的上下文，寄存器列表

我们nty_cpu_ctx结构体的定义，为了兼容x86，结构体项命令采用的是x86的寄存器名字命名。

typedef struct _nty_cpu_ctx {

void *esp; //

void *ebp;

void *eip;

void *edi;

void *esi;

void *ebx;

void *r1;

void *r2;

void *r3;

void *r4;

void *r5;

} nty_cpu_ctx;

_switch返回后，执行即将运行协程的上下文。是实现上下文的切换

_switch的实现代码：

```text
0: __asm__ (

1: "    .text                                  n"

2: "       .p2align 4,,15                                   n"

3: ".globl _switch                                          n"

4: ".globl __switch                                         n"

5: "_switch:                                                n"

6: "__switch:                                               n"

7: "       movq %rsp, 0(%rsi)      # save stack_pointer     n"

8: "       movq %rbp, 8(%rsi)      # save frame_pointer     n"

9: "       movq (%rsp), %rax       # save insn_pointer      n"

10: "       movq %rax, 16(%rsi)                              n"

11: "       movq %rbx, 24(%rsi)     # save rbx,r12-r15       n"

12: "       movq %r12, 32(%rsi)                              n"

13: "       movq %r13, 40(%rsi)                              n"

14: "       movq %r14, 48(%rsi)                              n"

15: "       movq %r15, 56(%rsi)                              n"

16: "       movq 56(%rdi), %r15                              n"

17: "       movq 48(%rdi), %r14                              n"

18: "       movq 40(%rdi), %r13     # restore rbx,r12-r15    n"

19: "       movq 32(%rdi), %r12                              n"

20: "       movq 24(%rdi), %rbx                              n"

21: "       movq 8(%rdi), %rbp      # restore frame_pointer  n"

22: "       movq 0(%rdi), %rsp      # restore stack_pointer  n"

23: "       movq 16(%rdi), %rax     # restore insn_pointer   n"

24: "       movq %rax, (%rsp)                                n"

25: "       ret                                              n"

26: );
```



按照x86_64的寄存器定义，%rdi保存第一个参数的值，即new_ctx的值，%rsi保存第二个参数的值，即保存cur_ctx的值。X86_64每个寄存器是64bit，8byte。

Movq %rsp, 0(%rsi) 保存在栈指针到cur_ctx实例的rsp项

Movq %rbp, 8(%rsi)

Movq (%rsp), %rax #将栈顶地址里面的值存储到rax寄存器中。Ret后出栈，执行栈顶

Movq %rbp, 8(%rsi) #后续的指令都是用来保存CPU的寄存器到new_ctx的每一项中

Movq 8(%rdi), %rbp #将new_ctx的值

Movq 16(%rdi), %rax #将指令指针rip的值存储到rax中

Movq %rax, (%rsp) # 将存储的rip值的rax寄存器赋值给栈指针的地址的值。

Ret # 出栈，回到栈指针，执行rip指向的指令。

上下文环境的切换完成。

## 6.第六章 协程的实现之定义

**问题：协程如何定义? 调度器如何定义？**

先来一道设计题：

设计一个协程的运行体R与运行体调度器S的结构体

\1. 运行体R：包含运行状态{就绪，睡眠，等待}，运行体回调函数，回调参数，栈指针，栈大小，当前运行体

\2. 调度器S：包含执行集合{就绪，睡眠，等待}

这道设计题拆分两个个问题，一个运行体如何高效地在多种状态集合更换。调度器与运行体的功能界限。

### 6.1 运行体如何高效地在多种状态集合更换

新创建的协程，创建完成后，加入到就绪集合，等待调度器的调度；协程在运行完成后，进行IO操作，此时IO并未准备好，进入等待状态集合；IO准备就绪，协程开始运行，后续进行sleep操作，此时进入到睡眠状态集合。

就绪(ready)，睡眠(sleep)，等待(wait)集合该采用如何数据结构来存储？

就绪(ready)集合并不没有设置优先级的选型，所有在协程优先级一致，所以可以使用队列来存储就绪的协程，简称为就绪队列（ready_queue）。

睡眠(sleep)集合需要按照睡眠时长进行排序，采用红黑树来存储，简称睡眠树(sleep_tree)红黑树在工程实用为<key, value>, key为睡眠时长，value为对应的协程结点。

等待(wait)集合，其功能是在等待IO准备就绪，等待IO也是有时长的，所以等待(wait)集合采用红黑树的来存储，简称等待树(wait_tree)，此处借鉴nginx的设计。

数据结构如下图所示：

Coroutine就是协程的相应属性，status表示协程的运行状态。sleep与wait两颗红黑树，ready使用的队列，比如某协程调用sleep函数，加入睡眠树(sleep_tree)，status |= S即可。比如某协程在等待树(wait_tree)中，而IO准备就绪放入ready队列中，只需要移出等待树(wait_tree)，状态更改status &= ~W即可。有一个前提条件就是不管何种运行状态的协程，都在就绪队列中，只是同时包含有其他的运行状态。

### 6.2 调度器与协程的功能界限

**每一协程都需要使用的而且可能会不同属性的，就是协程属性。每一协程都需要的而且数据一致的，就是调度器的属性。**比如栈大小的数值，每个协程都一样的后不做更改可以作为调度器的属性，如果每个协程大小不一致，则可以作为协程的属性。

**用来管理所有协程的属性，作为调度器的属性。**比如epoll用来管理每一个协程对应的IO，是需要作为调度器属性。

按照前面几章的描述，定义一个协程结构体需要多少域，我们描述了每一个协程有自己的上下文环境，需要保存CPU的寄存器ctx；需要有子过程的回调函数func；需要有子过程回调函数的参数 arg；需要定义自己的栈空间 stack；需要有自己栈空间的大小 stack_size；需要定义协程的创建时间 birth；需要定义协程当前的运行状态 status；需要定当前运行状态的结点（ready_next, wait_node, sleep_node）；需要定义协程id；需要定义调度器的全局对象 sched。

协程的核心结构体如下：

```text
typedef struct _nty_coroutine {

 

    nty_cpu_ctx ctx;

    proc_coroutine func;

 void *arg;

 size_t stack_size;

 

    nty_coroutine_status status;

    nty_schedule *sched;

 

 uint64_t birth;

 uint64_t id;

 

 void *stack;

 

 RB_ENTRY(_nty_coroutine) sleep_node;

 RB_ENTRY(_nty_coroutine) wait_node;

 

 TAILQ_ENTRY(_nty_coroutine) ready_next;

 TAILQ_ENTRY(_nty_coroutine) defer_next;

 

} nty_coroutine;
```



调度器是管理所有协程运行的组件，协程与调度器的运行关系。

调度器的属性，需要有保存CPU的寄存器上下文 ctx，可以从协程运行状态yield到调度器运行的。从协程到调度器用yield，从调度器到协程用resume

以下为协程的定义。

```text
typedef struct _nty_coroutine_queue nty_coroutine_queue;

 

typedef struct _nty_coroutine_rbtree_sleep nty_coroutine_rbtree_sleep;

typedef struct _nty_coroutine_rbtree_wait nty_coroutine_rbtree_wait;

 

typedef struct _nty_schedule {

 uint64_t birth;

nty_cpu_ctx ctx;

 

 struct _nty_coroutine *curr_thread;

 int page_size;

 

 int poller_fd;

 int eventfd;

 struct epoll_event eventlist[NTY_CO_MAX_EVENTS];

 int nevents;

 

 int num_new_events;

 

    nty_coroutine_queue ready;

    nty_coroutine_rbtree_sleep sleeping;

    nty_coroutine_rbtree_wait waiting;

 

} nty_schedule;
```

## 7.第七章 协程的实现之调度器

**问题：协程如何被调度？**

**调度器的实现，有两种方案，一种是生产者消费者模式，另一种多状态运行。**

### 7.1 生产者消费者模式

逻辑代码如下：

```text
while (1) {

 

 //遍历睡眠集合，将满足条件的加入到ready

        nty_coroutine *expired = NULL;

 while ((expired = sleep_tree_expired(sched)) != ) {

            TAILQ_ADD(&sched->ready, expired);

        }

 

 //遍历等待集合，将满足添加的加入到ready

        nty_coroutine *wait = NULL;

 int nready = epoll_wait(sched->epfd, events, EVENT_MAX, 1);

 for (i = 0;i < nready;i ++) {

            wait = wait_tree_search(events[i].data.fd);

            TAILQ_ADD(&sched->ready, wait);

        }

 

 // 使用resume回复ready的协程运行权

 while (!TAILQ_EMPTY(&sched->ready)) {

            nty_coroutine *ready = TAILQ_POP(sched->ready);

            resume(ready);

        }

    }
```

### 7.2 多状态运行

实现逻辑代码如下：

```text
while (1) {

 

 //遍历睡眠集合，使用resume恢复expired的协程运行权

        nty_coroutine *expired = NULL;

 while ((expired = sleep_tree_expired(sched)) != ) {

            resume(expired);

        }

 

 //遍历等待集合，使用resume恢复wait的协程运行权

        nty_coroutine *wait = NULL;

 int nready = epoll_wait(sched->epfd, events, EVENT_MAX, 1);

 for (i = 0;i < nready;i ++) {

            wait = wait_tree_search(events[i].data.fd);

            resume(wait);

        }

 

 // 使用resume恢复ready的协程运行权

 while (!TAILQ_EMPTY(sched->ready)) {

            nty_coroutine *ready = TAILQ_POP(sched->ready);

            resume(ready);

        }

    }
```

## 8.第八章 协程性能测试

测试环境：4台VMWare 虚拟机

1台服务器 6G内存，4核CPU

3台客户端 2G内存，2核CPU

操作系统：ubuntu 14.04

服务器端测试代码：[https://github.com/wangbojing/NtyCo](https://link.zhihu.com/?target=https%3A//github.com/wangbojing/NtyCo)

客户端测试代码：[https://github.com/wangbojing/c1000k_test/blob/master/client_mutlport_epoll.c](https://link.zhihu.com/?target=https%3A//github.com/wangbojing/c1000k_test/blob/master/client_mutlport_epoll.c)

按照每一个连接启动一个协程来测试。每一个协程栈空间 4096byte

6G内存 –> 测试协程数量100W无异常。并且能够正常收发数据。

原文地址：https://zhuanlan.zhihu.com/p/430196805

作者：linux