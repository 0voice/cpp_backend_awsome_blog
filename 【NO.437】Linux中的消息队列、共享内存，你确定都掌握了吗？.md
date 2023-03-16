# 【NO.437】Linux中的消息队列、共享内存，你确定都掌握了吗？

## 1.消息队列(message queue)

> 消息队列是消息的链表，存放在内存中，由内核维护

***消息队列的特点\***
1、消息队列中的消息是有类型的。
2、消息队列中的消息是有格式的。
3、消息队列可以实现消息的随机查询。消息不一定要以先进先出的次序读取，编程时可以按消息的类型读取。
4、消息队列允许一个或多个进程向它写入或者读取消息。
5、与无名管道、命名管道一样，从消息队列中读出消息，消息队列中对应的数据都会被删除。
6、每个消息队列都有消息队列标识符，消息队列的标识符在整个系统中是唯一的。
7、只有内核重启或人工删除消息队列时，该消息队列才会被删除。若不人工删除消息队列，消息队列会一直存在于系统中。
8、*消息队列可以独立于进程存在，可以无阻塞收发，可以选择性的接收消息
》 在ubuntu 12.04中消息队列限制值如下:

- 每个消息内容最多为8K字节
- 每个消息队列容量最多为16K字节
- 系统中消息队列个数最多为1609个
- 系统中消息个数最多为16384个

System V提供的IPC通信机制需要一个key值，通过key值就可在系统内获得一个唯一的消息队列标识符。
key值可以是人为指定的，也可以通过ftok函数获得。

```
#include <sys/types.h>
#include <sys/ipc.h>
key_t ftok(const char *pathname, int proj_id)；
功能：
    获得项目相关的唯一的IPC键值。
参数：
    pathname：路径名
    proj_id：项目ID，非0整数(只有低8位有效)
返回值：
    成功返回key值
    失败返回 -1
```

## 2.创建消息队列：

```
#include <sys/msg.h>
int msgget(key_t key, int msgflg)；
功能：
创建一个新的或打开一个已经存在的消息队列。不同的进程调用此函数，
只要用相同的key值就能得到同一个消息队列的标识符。
参数：
key：IPC键值。
msgflg：标识函数的行为及消息队列的权限。
返回值：
成功：消息队列的标识符，失败：返回-1。
》msgflg的取值：
IPC_CREAT：创建消息队列。
IPC_EXCL：检测消息队列是否存在。
位或权限位：消息队列位或权限位后可以设置消息队列的访问权限，
格式和open函数的mode_t一样，但可执行权限未使用。
```

> 使用shell命令操作消息队列:
> 查看消息队列
> ipcs -q
> 删除消息队列
> ipcrm -q msqid

## 3.消息队列的消息的格式：

```
typedef struct _msg
{
    long mtype; /*消息类型*/
    char mtext[100]; /*消息正文*/
    ... /*消息的正文可以有多个成员*/
}MSG;
消息类型必须是长整型的，而且必须是结构体类型的第一个成员，
类型下面是消息正文，正文可以有多个成员（正文成员可以是任意数据类型的）。
```

## 4.发送消息：

```
#include <sys/msg.h>
int msgsnd(int msqid, const void *msgp,size_t msgsz, int msgflg);
功能：
    将新消息添加到消息队列。
参数：
    msqid：消息队列的标识符。
    msgp：待发送消息结构体的地址。
    msgsz：消息正文的字节数。
    msgflg：函数的控制属性
        0：msgsnd调用阻塞直到条件满足为止。
        IPC_NOWAIT: 若消息没有立即发送则调用该函数的进程会立即返回。
返回值：
    成功：0
    失败：返回-1
```

## 5.接收消息：

```
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz,
long msgtyp, int msgflg);
功能：
从标识符为msqid的消息队列中接收一个消息。一旦接收消息成功，
则消息在消息队列中被删除。
参数：
msqid：消息队列的标识符，代表要从哪个消息列中获取消息。
msgp： 存放消息结构体的地址。
msgsz：消息正文的字节数。
msgtyp：消息的类型、可以有以下几种类型
    msgtyp = 0：返回队列中的第一个消息
    msgtyp > 0：返回队列中消息类型为msgtyp的消息
    msgtyp < 0：返回队列中消息类型值小于或等于
    msgtyp绝对值的消息，如果这种消息有若干个，则取类型值最小的消息。
    注意：
    若消息队列中有多种类型的消息，msgrcv获取消息的时候按消息类型获取，
    不是先进先出的。在获取某类型消息的时候，若队列中有多条此类型的消息，
    则获取最先添加的消息，即先进先出原则。
msgflg：函数的控制属性
    0：msgrcv调用阻塞直到接收消息成功为止。
    MSG_NOERROR:若返回的消息字节数比nbytes字节数多,则消息就会截短到nbytes字节,且不通知消息发送进程。
    IPC_NOWAIT:调用进程会立即返回。若没有收到消息则立即返回-1。
返回值：
    成功返回读取消息的长度
    失败返回-1。
```

## 6.消息队列的控制：

```
#include <sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
功能：
    对消息队列进行各种控制，如修改消息队列的属性，或删除消息消息队列。
参数：
    msqid：消息队列的标识符。
    cmd：函数功能的控制。
    buf：msqid_ds数据类型的地址，用来存放或更改消息队列的属性。
返回值：
    成功：返回 0
    失败：返回 -1
》cmd：函数功能的控制
    IPC_RMID：删除由msqid指示的消息队列，将它从系统中删除并破坏相关数据结构。
    IPC_STAT：将msqid相关的数据结构中各个元素的当前值存入到由buf指向的结构中。
    IPC_SET：将msqid相关的数据结构中的元素设置为由buf指向的结构中的对应值。
```

例：01_message_queue_**write**.c 01_message_queue_**read**.c

![img](https://pic4.zhimg.com/80/v2-b5fb7ba4aff8b1afcb9bb42c57513f47_720w.webp)

![img](https://pic1.zhimg.com/80/v2-f8eb15b8b8a3aaf56faa85a44aa95718_720w.webp)

## 7.总结：

```
》每个程序都有两个任务，一个任务是负责接收消息，一个任务是负责发送消息，
通过fork创建-子进程实现多任务。
》一个进程负责接收信息，它只接收某种类型的消息，只要别的进程发送此类型的消息，
此进程-就能收到。收到后通过消息的name成员就可知道是谁发送的消息。
》另一个进程负责发信息，可以通过输入来决定发送消息的类型。
》设计程序的时候，接收消息的进程接收消息的类型不一样，这样就实现了发送的消息
只被接收-此类型消息的人收到，其它人收不到。这样就是实现了给特定的人发送消息。
```

## 8.共享内存(shared memory)

> 共享内存允许两个或者多个进程共享给定的存储区域。

**共享内存的特点**
1、共享内存是进程间共享数据的一种最快的方法。一个进程向共享的内存区域写入了数据，共享这个内存区域的所有进程就可以立刻看到其中的内容。
2、使用共享内存要注意的是多个进程之间对一个给定存储区访问的互斥。若一个进程正在向共享内存区写数据，则在它做完这一步操作前，别的进程不应当去读、写这些数据。

![img](https://pic2.zhimg.com/80/v2-7fd8e629e2de2d1855b068d4f389e141_720w.webp)

在ubuntu 12.04中共享内存限制值如下：
-共享存储区的最小字节数：1
-共享存储区的最大字节数：32M
-共享存储区的最大个数：4096
-每个进程最多能映射的共享存储区的个数：4096

## 9.获得一个共享存储标识符

```
#include <sys/ipc.h>
#include <sys/shm.h>
int shmget(key_t key, size_t size,int shmflg);
功能:创建或打开一块共享内存区
参数：
key：IPC键值
size：该共享存储段的长度(字节)
shmflg：标识函数的行为及共享内存的权限。
返回值:
成功：返回共享内存标识符。
失败：返回－1。
》参数属性shmflg：
IPC_CREAT：如果不存在就创建
IPC_EXCL：如果已经存在则返回失败
位或权限位：共享内存位或权限位后可以设置共享内存的访问权限，格式和open函数的mode_t一样，但可执行权限未使用。
```

> 使用shell命令操作共享内存:
> 查看共享内存
> ipcs -m
> 删除共享内存
> ipcrm -m shmid

## 10.共享内存映射(attach)

```
#include <sys/types.h>
#include <sys/shm.h>
void *shmat(int shmid, const void *shmaddr,int shmflg);
功能：
将一个共享内存段映射到调用进程的数据段中。
参数：
shmid：共享内存标识符。
shmaddr：共享内存映射地址(若为NULL则由系统自动指定)，推荐使用NULL。
shmflg：共享内存段的访问权限和映射条件。
返回值：
成功：返回共享内存段映射地址
失败：返回 -1
》shmflg：共享内存段的访问权限和映射条件
    0：共享内存具有可读可写权限。
    SHM_RDONLY：只读。
    SHM_RND：（shmaddr非空时才有效）
    没有指定SHM_RND则此段连接到shmaddr所指定的地址上(shmaddr必需页对齐)。
    指定了SHM_RND则此段连接到shmaddr-shmaddr%SHMLBA 所表示的地址上。
》注：
shmat函数使用的时候第二个和第三个参数一般设为NULL和0，
即系统自动指定共享内存地址，并且共享内存可读可写。
```

## 11.解除共享内存映射(detach)

```
#include <sys/types.h>
#include <sys/shm.h>
int shmdt(const void *shmaddr);
功能：
将共享内存和当前进程分离(仅仅是断开联系并不删除共享内存)。
参数：
shmaddr：共享内存映射地址。
返回值：
成功返回 0，
失败返回 -1。
```

## 12.共享内存控制

```
#include <sys/ipc.h>
#include <sys/shm.h>
int shmctl(int shmid, int cmd,struct shmid_ds *buf);
功能：共享内存空间的控制。
参数：
shmid：共享内存标识符。
cmd：函数功能的控制。
buf：shmid_ds数据类型的地址，用来存放或修改共享内存的属性。
返回值：
成功返回 0，
失败返回 -1。
》cmd：函数功能的控制
    IPC_RMID：删除。
    IPC_SET：设置shmid_ds参数。
    IPC_STAT：保存shmid_ds参数。
    SHM_LOCK：锁定共享内存段(超级用户)。
    SHM_UNLOCK：解锁共享内存段。
```

例：02_shared_memory_**write**.c 02_shared_memory_**read**.c

![img](https://pic3.zhimg.com/80/v2-2102704c6fb5ded6b4d73ad30841d9ce_720w.webp)

![img](https://pic1.zhimg.com/80/v2-de902402d906112813abd946c5e20540_720w.webp)

注意：
SHM_LOCK用于锁定内存，禁止内存交换。并不代表共享内存被锁定后禁止其它进程访问。其真正的意义是：被锁定的内存不允许被交换到虚拟内存中。这样做的优势在于让共享内存一直处于内存中，从而提高程序性能。

原文链接：https://zhuanlan.zhihu.com/p/411043847

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)