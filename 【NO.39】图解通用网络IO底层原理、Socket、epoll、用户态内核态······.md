# 【NO.39】图解通用网络IO底层原理、Socket、epoll、用户态内核态······

## 1.LInux 操作系统中断

### 1.1 什么是系统中断

> 这个没啥可说的，大家都知道；

CPU 在执行任务途中接收到中断请求，需要保存现场后去处理中断请求！保存现场称为中断处理程序！处理中断请求也就是唤醒对应的任务进程来持有CPU进行需要的操作！

有了中断之后，提升了操作系统的性能！可以异步并行处理很多任务！

- **软中断（80中断）**

由CPU产生的；CPU检查到程序代码段发生异常会切换到内核态；

- **硬中断**

由硬件设备发起的中断称为硬中断！可以发生在任何时间；比方说网卡设备接收到一组报文；对应的报文会被[DMA](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3DDMA%26spm%3D1001.2101.3001.7020)设备进行拷贝到网卡缓冲区！然后网卡就会向CPU发起中断信号（IRQ）：

CPU收到信号后就会执行网卡对应的中断处理程序！

### 1.2 内核在系统中断时做了什么事

每种中断都有它对应的中断处理程序；

对应到内核的某一个代码段；

CPU接收到中断后；首先需要将寄存器中数据保存到进程描述符！PCB！

随后切换到内核态处理中断处理程序！执行网卡的程序；

执行完毕之后切换到用户态，根据PCB内容恢复现场！然后就可继续执行代码段了！

### 1.3 硬件中断触发的过程

![img](https://pic1.zhimg.com/80/v2-4f58845894548325a99a27258dba7dec_720w.webp)

**中断请求寄存器**： 保存需要发送中断请求的设备记录！

**优先级解析器**：中断请求是有优先级之分的，因为CPU不能同时执行多个中断请求！

**正在服务寄存器**：正在执行的请求！比方我正在打字，这里面记录的就是键盘IRQ1 ！

![img](https://pic2.zhimg.com/80/v2-30147da63326ce5299ff88fbf0258a51_720w.webp)

操作系统启动时需要将硬件向量值与处理程序地址进行映射！当硬件发送中断信息时只会发送向量值，通过匹配找到对应的处理程序！

## 2.[Socket](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3DSocket%26spm%3D1001.2101.3001.7020)基础

### 2.1Socket读写缓冲区机制

![img](https://pic1.zhimg.com/80/v2-cb1cbee2218d93c2e549ff61e275116c_720w.webp)

所谓socket,在底层也无非就是一个对象，通过对象绑定两个缓冲区，也就是数据队列，然后调用系统API对这两个缓冲区的数据进行操作罢了！

发数据；用户态转内核态，将数据拷贝到send缓存区，然后调用write系统调用将数据拷贝到网卡，再由网卡通过TCP/IP协议进行数据包的网络发送！

socket**两种工作模式**

- BIO

总结：读数据读不到就一直等，发数据发不了就一直等！

- NIO

读数据读不到就等一会再读，取数据取不到就等一会再取！

接受端缓冲区打满了，线程又抢占不到CPU去清理缓冲区，怎么办！

最后发送端的数据缓冲区也会被打满！

## 3.系统调用；用户态------内核态

### 3.1**系统调用**：

int 0X80对应的就是系统调用中断处理程序；向量值为128；system_call;

![img](https://pic2.zhimg.com/80/v2-b234bc234ccced64cf5c14fe51801881_720w.webp)

IRQ是有限的，不可能为每一个系统调用都分配一个向量值，所以统一使用80中断来进行系统调用的路由！



### 3.2 为什么要有这两种状态

指令的危险程度不一样；

对于不同的指令，为了保证系统安全，划分了用户空间和内核空间；

linux中：0表示内核态，3表示用户态！

所以：linux在创建进程的时候就会为进程分配两块空间；

用户栈：分配变量，创建对象

内核栈：分配变量！

### 3.3 什么时候进程进行切换至内核态

硬中断；

用户态中代码出现错误也要切换！

### 3.4 进程切换时都做了什么

CPU中存在很多寄存器

![img](https://pic1.zhimg.com/80/v2-f9411d13bbc8b163ad9c50165212c14c_720w.webp)

这些寄存器保存了进程在进行运算时的一些瞬时数据；如果现在要进行进程切换了；这些数据都需要找个地方保存起来；那么保存到哪里呢？

进程PCB：在OS创建进程的时候同时也会分配一段空间存放进程的一些信息；其中就有一个字段指向一个数据结构；叫做进程控制块PCB：

用来描述和控制进程的运行的一个数据结构——进程控制块PCB(Process Control Block)，是进程实体的一部分，是操作系统中最重要的记录型数据结构。

- PCB是进程存在的唯一标志
- 系统能且只能通过PCB对进程进行控制和调度
- PCB记录了操作系统所需的、用于描述进程的当前情况以及控制进程运行的全部信息

所以：在进程进行切换的时候CPU中的数据保存到了PCB中，供CPU回来时读取恢复！

## 4.Linux select 多路复用函数

select就是一个函数：只要传入相应的参数就能获得相应的数据：

1、们所关心的文件描述符fd;

2、描述符中我们关心的状态：读事件、写事件、等

3、等待时间

调用结束后内核会返回相关信息给我们！

做好准备的个数

哪些已经做好准备；有了这些返回信息，我们就可以调用合适的IO函数！这些函数就不会再被阻塞了；-

**函数详解**

```text
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, timeval *timeout)
    
- maxfdp1 readset 和 wirteset中的最大有数据位
- readset  bitmap结构的位信息；保存我们需要读取的socket序号；
- writeset 写数据信息
- exceptset 异常信息
```

![img](https://pic2.zhimg.com/80/v2-c2de5f86c6dec2894887b03ac2073865_720w.webp)

select函数这里不再细讲，可以翻看以前的文章

![img](https://pic1.zhimg.com/80/v2-431d325945c3906ccaddbf0951267e70_720w.webp)

将函数需要的参数准备好之后调用select；

select进行80中断；将rset数据拷贝到内核中；查询对应的状态之后设置rset对应的位置值，

完成后又拷贝到用户态中的rset；这样一来rset里面的位信息就代表了哪些socket是准备好了的！

随后遍历这些位信息就可以调用read或wirte进行缓冲区的操作了！

**缺点**

可以看到，while死循环中每次执行都将rset重新置位；然后循环重新SET位信息；随后才会发起请求！过程较为繁琐且重复！

## 5.select多路复用器底层原理分析

![img](https://pic2.zhimg.com/80/v2-aeec11d654478255ad0a51a6fb961d3d_720w.webp)

![img](https://pic4.zhimg.com/80/v2-638aaed8d22eb18eb77312de7314bf4f_720w.webp)

![img](https://pic1.zhimg.com/80/v2-b8822b3ecb523af8555dfe17912f0154_720w.webp)

![img](https://pic1.zhimg.com/80/v2-4e8b9c2203d29f3b4de412689fe0b84c_720w.webp)

![img](https://pic2.zhimg.com/80/v2-ba158e38135bc9ca5a094daccc15d0dd_720w.webp)

![img](https://pic3.zhimg.com/80/v2-0bbe7c7d184546082f32bc6e9bd52c92_720w.webp)

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1321' height='931'></svg>)

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='1140' height='456'></svg>)

![img](https://pic2.zhimg.com/80/v2-1a3f5065af663ee69dbd10614c2a54e5_720w.webp)

## 6.epoll函数

了解到select的缺点后发现：select每次得到数据都要进行复位，然后又进行重复的步骤去内核中获取信息；感觉就是很多时间都花在重复的劳动上，为了解决这个问题，linux在2.6引入epoll模型，单独在内核区域开辟一块空间来做select主动去做的事，select是主动查，epoll则是准备数据，线程来了直接取就行了；大大提升了性能

既然是函数，看看相关的函数实现：

实现思路：

在内核创建一块空间；总所周知；linux下一切皆文件；所以所谓创建的空间也就是一个文件描述符fd,然后这个文件结构中有两个指针指向另外两个地址空间：事件队列、就绪队列

事件队列：存放已经建立所有socket连接

就绪队列：准备就绪的socket；也就是read或write的时候不用阻塞的socket；

其实epoll就像一个数据库；里面有两个数据表；一个放连接列表；一个放准备就绪的连接列表；

既然有这两个队列；就要涉及到增删查；这就是另外两个函数的来由；

```text
创建epoll空间
int epoll_create(int size);


int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);



对事件队列进行增删改：

epfd : epoll的文件描述符号：因为内核中可能有多个epoll

op : 参数op有以下几个值： EPOLL_CTL_ADD：注册新的fd到epfd中，并关联事件event； EPOLL_CTL_MOD：修改已经注册的fd的监听事件； EPOLL_CTL_DEL：从epfd中移除fd，并且忽略掉绑定的event，这时event可以为null；

fd : 表示socket对应的文件描述符。

![img](https://pic2.zhimg.com/80/v2-cae16b763e168bf16720da3e890eac99_720w.webp)

## 7.epoll底层原理解析

![img](https://pic3.zhimg.com/80/v2-3703c7d0d7a7ef6d8623ae8ee8dacbb2_720w.webp)

![img](https://pic1.zhimg.com/80/v2-5d3bbd408d81106f19ad52dfbcda964c_720w.webp)

![img](https://pic3.zhimg.com/80/v2-9bef7d9fde32f7e450f74a61bce2be56_720w.webp)

![img](https://pic3.zhimg.com/80/v2-50010b451e124c6c3ba699bd93a1fc46_720w.webp)

![img](https://pic1.zhimg.com/80/v2-150960e8b7a37e2745d1ce34a1db0614_720w.webp)

![img](https://pic1.zhimg.com/80/v2-8641d18bf33650a6a91bef5ea04bd360_720w.webp)

![img](https://pic2.zhimg.com/80/v2-ffed1f95b69d403ec04a9fb1f0efe5cd_720w.webp)

![img](https://pic4.zhimg.com/80/v2-4cfe93d9f02c4a5d46fa1843ea7d3ee3_720w.webp)

![img](https://pic3.zhimg.com/80/v2-68fd0316d0c532fe72ba3b6140cf57ca_720w.webp)

![img](https://pic2.zhimg.com/80/v2-68ec3366c33de62676ebeb3157feae91_720w.webp)

![img](https://pic1.zhimg.com/80/v2-ac3e968a8b0c69ed18277de06b6a2a14_720w.webp)

![img](https://pic2.zhimg.com/80/v2-a2b3a7c5c46f0a01afb065847191566d_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/579368540   

作者： linux