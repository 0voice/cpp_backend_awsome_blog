# 【NO.608】Linux 进程间通信：管道、共享内存、消息队列、信号量

## 1.进程间通信

因为每个进程都通过自己的页表构建物理地址和虚拟地址的映射关系，使每个进程都拥有自己的虚拟地址空间，并通过这个独立的虚拟地址空间来对物理内存进行操作，所有的进程都只能访问自己的虚拟地址，而不能直接访问物理内存，所以多个进程无法访问同一块区域，无法实现通信。

因为这种独立性，进程之间无法直接进行通信，操作系统就为了解决这种问题，提出了多重适用于不同情境下的通信方式。

**数据传输:管道、消息队列**
**数据共享：共享内存**
**进程控制：信号量**

## 2.管道

**原理：管道的本质其实就是内核中的一块缓冲区，多个进程通过访问同一个缓冲区就可以实现进程间的通信**

![img](https://pic4.zhimg.com/80/v2-05c214dbfd742e6e25f3985ca76014c7_720w.webp)

管道分为两种：匿名管道、命名管道

### 2.1 **匿名管道**

匿名 管道是内核中的一块缓冲区，因为没有具体的文件描述符，所以匿名管道只能适用于具有亲缘关系的进程间通信。父进程在创建管道的时候操作系统会返回管道的文件描述符，然后生成子进程时子进程会通过拷贝父进程的pcb来获取到这个管道的描述符，所以他们可以通过这个文件描述符来访问同一个管道，来实现进程间的通信。而不具备亲缘关系的进程则无法通过这个文件描述符来访问同一个管道。

**接口**

> \#include <unistd.h>
> 功能:创建一无名管道
> 原型
> int pipe(int fd[2]);
> 参数
> fd：文件描述符数组,其中fd[0]表示读端, fd[1]表示写端
> 返回值:成功返回0，失败返回-1

一开始父进程创建管道

![img](https://pic1.zhimg.com/80/v2-62275d9072f8c6dcbc96803d434410a0_720w.webp)

父进程fork创建子进程

![img](https://pic4.zhimg.com/80/v2-d5a31454c8ac9139b23bb7653e4c8c03_720w.webp)

关闭多余描述符

![img](https://pic1.zhimg.com/80/v2-3037f7d6b8a1af091797706690a20f70_720w.webp)

就这样，子进程通过写入端fd[1]向管道写入数据，父进程通过读入端fd[0]从管道读取数据，来实现进程间的通信。

```text
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

int main()
{
        int pipefd[2];
        pipe(pipefd);
        
        int pid = fork();
        if(pid == 0)
        {
                close(pipefd[0]);
                while(1)
                {
                        write(pipefd[1], "hello world", 12);
                        sleep(3);
                }
                close(pipefd[1]);
                exit(0);
        }
        else if(pid > 0)
        {
                close(pipefd[1]);

                char buff[1024];
                while(1)
                {
                        read(pipefd[0], buff, 12);
                        printf("%s\n", buff);
                }
                close(pipefd[0]);
        }
        return 0;
```

这是一个简单的管道

![img](https://pic3.zhimg.com/80/v2-b0ab9001a81c99d05bda7f6f490f70d2_720w.webp)

运行后每三秒写端会写入数据，然后读端立即读入数据。

因为父子进程究竟是谁先执行这一点我们无法知道，假设如果子进程还没写入，父进程却已经开始读了，这时候应该是会读不到东西的，但是这种情况并没有发生，这里就牵扯到了管道的读写特性。

**管道的读写特性：**

1. 如果管道中没有数据，则调用read读取数据会阻塞
2. 如果管道中数据满了，则调用write写入数据会阻塞
3. 如果管道的所有读端pipefd[0]被关闭，则继续调用write会因为无法读出而产生异常导致进程退出
4. 如果管道的所有写端pipefd[1]被关闭，则继续调用read，因为无法再次写入，read读完管道中的所有数据后不再阻塞，返回0退出

### 2.2 命名管道

匿名管道的限制就是只能在亲缘关系的进程间通信，如果我们想为不相关的进程交换数据，就可以使用命名管道。

**原理：命名管道也是内核中的一块缓冲区，但是它具有标识符。这个标识符是一个可见于文件系统的管道文件，能够被其他进程找到并打开管道文件来获取管道的操作句柄，多个进程可以通过打开这个管道文件来访问同一块缓冲区来实现通信**

**接口：**

> int mkfifo(const char *filename,mode_t mode);
> mode:权限掩码
> filename:管道的标识符，通过这个标识符来访问管道
> 返回值：若成功则返回0，否则返回-1

```text
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<fcntl.h>
#include<sys/types.h>
#include<sys/stat.h>

int main()
{
        mkfifo("test", 0664);
        int pid = fork();
        if(pid > 0)
        {
                int read_fd = open("test", O_RDONLY);
                char buff[1024];

                read(read_fd, buff, 12);
                printf("buff:%s\n", buff);
                close(read_fd);
        }
        else if(pid == 0)
        {
                int write_fd = open("test", O_WRONLY);

                write(write_fd, "hello world", 12);
                close(write_fd);

                exit(0);
        }
        return 0;
}
```

![img](https://pic3.zhimg.com/80/v2-047120f60559268c45700129ba1cb1b2_720w.webp)

试验一下

**open打开命名管道的特性：**

\1. 若文件以只读打开，则会阻塞，直到文件被以写的方式打开
\2. 若文件以只写打开，则会阻塞，直到文件被以读的方式打开

**管道的特性：**

\1. 管道是半双工通信（可以选择方向的单向传输），这个可以从上面的示意图看出来

\2. 管道的读写特性（无论命名匿名都一样）

若管道中没有数据则读操作堵塞，如果管道中数据满了写操作堵塞。

如果管道中所有读端关闭则写端触发异常，如果所有写端关闭则读端读完数据后不堵塞返回0

\3. 管道声明周期随进程，打开管道的所有进程退出后管道就会被释放。

\4. 管道提供字节流传输服务

\5. 命名管道额外有一个打开特性，只读打开会阻塞直到被以写打开，只写打开会阻塞直到被以读打开

\6. 管道自带同步和互斥

## 3.共享内存

**共享内存即在物理内存上开辟一块空间，然后多个进程通过页表将这同一个物理内存映射到自己的虚拟地址空间中，通过自己的虚拟地址空间来访问这块物理内存，达到了数据共享的目的。**

如图：

![img](https://pic3.zhimg.com/80/v2-9974bdf397c348b735d8be50eaa1c32e_720w.webp)

也正是因为这种特性，使得**共享内存成为了最快的进程间通信的方式**，**因为它直接通过虚拟地址来访问物理内存，比前面的管道和后面的消息队列少了内核态和用户态的几次数据拷贝和交互。**

**共享内存的使用流程：**

**1. 创建共享内存**
**2. 将共享内存映射到虚拟地址空间**
**3. 进行操作**
**4. 解除映射关系**
**5. 释放共享内存**

**接口：**

1、创建共享内存

int shmget(key_t key, size_t size, int shmflg)

> key:这个共享内存段名字
> size:共享内存大小
> shmflg:由九个权限标志构成，它们的用法和创建文件时使用的mode模式标志是一样的
> 返回值：成功返回共享内存标识符，失败返回-1
> 头文件：
> \#include <sys/ipc.h>
> \#include <sys/shm.h>

2、将共享内存映射到虚拟地址空间

void *shmat(int shmid, const void *shmaddr, int shmflg)

> shmid: 共享内存标识
> shmaddr:指定连接的地址
> shmflg:权限标志
> 返回值：成功返回指向共享内存映射在虚拟地址空间的指针（即首地址），失败返回-1
> 头文件：
> \#include <sys/types.h>
> \#include <sys/shm.h>

3、共享内存管理

int shmctl(int shmid, int cmd, struct shmid_ds *buf)

> shmid:由shmget返回的共享内存标识码
> cmd:将要采取的动作
> buf:指向一个保存着共享内存的模式状态和访问权限的数据结构
> 返回值：成功返回0，失败返回-1
> 头文件：
> \#include <sys/types.h>
> \#include <sys/shm.h>

4、解除映射关系

int shmdt(const void *shmaddr)

> shmaddr: 由shmat所返回的指针
> 返回值：成功返回0，失败返回-1
> 头文件：
> \#include <sys/types.h>
> \#include <sys/shm.h>

**特性：**

\1. 共享内存是最快的进程间通信方式
\2. 生命周期随内核
\3. 不自带同步与互斥，但可以借助信号量来实现同步与互斥

## 4.消息队列

消息队列是内核中的一个优先级队列，多个进程通过访问同一个队列，进行添加节点或者获取节点来实现通信。

**接口：**

1、创建消息队列

int msgget(key_t key, int msgflg);

> key:消息队列对象的关键字
> msgflg:消息队列的建立标志和存取权限
> 返回值：成功执行时，返回消息队列标识值。失败返回-1
> 头文件：
> \#include <sys/types.h>
> \#include <sys/ipc.h>
> \#include <sys/msg.h>

2、进程可以向队列中添加/获取节点

添加节点：

int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);

获取节点：

ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp,

int msgflg);

> msqid:消息队列对象的标识符
> msgp:消息缓冲区指针
> msgsz:消息数据的长度
> msgtyp:决定从队列中返回哪条消息
> msgflg:消息队列状态
> 返回值：成功执行时，返回0。失败返回-1
> 头文件：
> \#include <sys/types.h>
> \#include <sys/ipc.h>
> \#include <sys/msg.h>

3、删除消息队列

int msgctl(int msqid, int cmd, struct msqid_ds *buf)

> msqid:消息队列对象的标识符
> cmd:函数要对消息队列进行的操作
> buf:取出系统保存的消息队列的msqid_ds 数据，并将其存入参数buf 指向的msqid_ds 结构中
> 返回值：成功执行时，返回0。失败返回-1
> 头文件：
> \#include <sys/types.h>
> \#include <sys/ipc.h>
> \#include <sys/msg.h>

**特性：**

\1. 自带同步与互斥
\2. 生命周期随内核

## 5.信号量

信号量其实是内核中的一个计数器和阻塞队列，通过信号量来对临界资源的访问进行控制，来实现进程间的同步与互斥。

例如有一个能容纳n人的餐厅，则用一个计数器表示n，如果有人进入则n - 1，如果有人出来则n + 1，只有n > 0时才能进入，如果n <= 0时，则说明没有位置，需要将进程挂起并放入阻塞队列中，直到有人出来使资源释放时，才能将后续进程从阻塞队列中唤醒获取资源

**同步：**

通过条件判断实现临界资源访问的合理性

**互斥：**

通过同一时间的唯一访问来实现临界资源访问的安全性

**POSIX信号量**

POSIX信号量和SystemV信号量作用相同，都是用于同步操作，达到无冲突的访问共享资源目的。 但POSIX可以用于线程间同步。

```text
#include <semaphore.h>

//初始化信号量
int sem_init(sem_t *sem, int pshared, unsigned int value);
/*
参数：
 pshared:0表示线程间共享，非零表示进程间共享
 value:信号量初值
*/

//销毁信号量
int sem_destroy(sem_t *sem);

//等待信号量
int sem_wait(sem_t *sem);
//功能：等待信号量，会将信号量的值减1


//发布信号量
int sem_post(sem_t *sem);
//功能：发布信号量，表示资源使用完毕，可以归还资源了。将信号量值加1。
```

原文地址：https://zhuanlan.zhihu.com/p/611277658

作者：cpp后端技术