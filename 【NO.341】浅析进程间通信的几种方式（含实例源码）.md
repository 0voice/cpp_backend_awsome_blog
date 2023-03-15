# 【NO.341】浅析进程间通信的几种方式（含实例源码）

## 1.为什么进程间需要通信？

1).数据传输

一个进程需要将它的数据发送给另一个进程;

2).资源共享

多个进程之间共享同样的资源;

3).通知事件

一个进程需要向另一个或一组进程发送消息，通知它们发生了某种事件;

4).进程控制

有些进程希望完全控制另一个进程的执行(如Debug进程)，该控制进程希望能够拦截另一个进程的所有操作，并能够及时知道它的状态改变。

基于以上几个原因，所以就有了进程间通信的概念，那仫进程间通信的原理是什仫呢？目前有哪几种进程间通信的机制？他们是如何实现进程间通信的呢？在这篇文章中我会就这几个问题进行详细的讲解。

## 2.进程间通信的原理

每个进程各自有不同的用户地址空间,任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核,在内核中开辟一块缓冲区,进程1把数据从用户空间拷到内核缓冲区,进程2再从内核缓冲区把数据读走,内核提供的这种机制称为进程间通信机制。

主要的过程如下图所示：



![img](https://pic3.zhimg.com/80/v2-19d4ac7dccc826737fab858d8e84de52_720w.webp)

## 3.进程间通信的几种方式

### 3.1 管道(pipe)

管道又名匿名管道，这是一种最基本的IPC机制，由pipe函数创建：

> \#include <unistd.h>
> int pipe(int pipefd[2]);

返回值：成功返回0，失败返回-1；

调用pipe函数时在内核中开辟一块缓冲区用于通信,它有一个读端，一个写端：pipefd[0]指向管道的读端，pipefd[1]指向管道的写端。所以管道在用户程序看起来就像一个打开的文件,通过read(pipefd[0])或者write(pipefd[1])向这个文件读写数据，其实是在读写内核缓冲区。

使用管道的通信过程：



![img](https://pic3.zhimg.com/80/v2-1bb5c4fb3afa6e16259816dda05d756e_720w.webp)







![img](https://pic4.zhimg.com/80/v2-32ffb053af31b54b2eede40c3a34f40b_720w.webp)







![img](https://pic3.zhimg.com/80/v2-46a0bc1d51bccdeead5b65aee2a9c66e_720w.webp)









1.父进程调用pipe开辟管道,得到两个文件描述符指向管道的两端。

2.父进程调用fork创建子进程,那么子进程也有两个文件描述符指向同一管道。

3.父进程关闭管道读端,子进程关闭管道写端。父进程可以往管道里写,子进程可以从管道里读,管道是用环形队列实现的,数据从写端流入从读端流出,这样就实现了进程间通信。

管道出现的四种特殊情况：

1.写端关闭，读端不关闭；

那么管道中剩余的数据都被读取后,再次read会返回0,就像读到文件末尾一样。

2.写端不关闭，但是也不写数据，读端不关闭；

此时管道中剩余的数据都被读取之后再次read会被阻塞，直到管道中有数据可读了才重新读取数据并返回；

3.读端关闭，写端不关闭；

此时该进程会收到信号SIGPIPE，通常会导致进程异常终止。

4.读端不关闭，但是也不读取数据，写端不关闭；

此时当写端被写满之后再次write会阻塞，直到管道中有空位置了才会写入数据并重新返回。

使用管道的缺点：

1.两个进程通过一个管道只能实现单向通信，如果想双向通信必须再重新创建一个管道或者使用sockpair才可以解决这类问题；

2.只能用于具有亲缘关系的进程间通信，例如父子，兄弟进程。

一个简单的关于管道的例子：

代码实现如下：

> \#include<stdio.h>
> \#include<unistd.h>
> \#include<stdlib.h>
> \#include<string.h>
> int main()
> {
> int _pipe[2]={0,0};
> int ret=pipe(_pipe); //创建管道
> if(ret == -1)
> {
> perror("create pipe error");
> return 1;
> }
> printf("_pipe[0] is %d,_pipe[1] is %d\n",_pipe[0],_pipe[1]);
> pid_t id=fork(); //父进程fork子进程
> if(id < 0)
> {
> perror("fork error");
> return 2;
> }
> else if(id == 0) //child,写
> {
> printf("child writing\n");
> close(_pipe[0]);
> int count=5;
> const char *msg="i am from XATU";
> while(count--)
> {
> write(_pipe[1],msg,strlen(msg));
> sleep(1);
> }
> close(_pipe[1]);
> exit(1);
> }
> else //father,读
> {
> printf("father reading\n");
> close(_pipe[1]);
> char msg[1024];
> int count=5;
> while(count--)
> {
> ssize_t s=read(_pipe[0],msg,sizeof(msg)-1);
> if(s > 0){
> msg[s]='\0';
> printf("client# %s\n",msg);
> }
> else{
> perror("read error");
> exit(1);
> }
> }
> if(waitpid(id,0,NULL) != -1){
> printf("wait success\n");
> }
> }
> return 0;
> }





![img](https://pic1.zhimg.com/80/v2-58d405bcbe0ffc294033b7dbe3795c68_720w.webp)



### 3.2 命名管道(FIFO)

上一种进程间通信的方式是匿名的，所以只能用于具有亲缘关系的进程间通信，命名管道的出现正好解决了这个问题。FIFO不同于管道之处在于它提供一个路径名与之关联，以FIFO的文件形式存储文件系统中。命名管道是一个设备文件，因此即使进程与创建FIFO的进程不存在亲缘关系，只要可以访问该路径，就能够通过FIFO相互通信。

命名管道的创建与读写：

1).是在程序中使用系统函数建立命名管道；

2).是在Shell下交互地建立一个命名管道，Shell方式下可使用mknod或mkfifo命令来创建管道，两个函数均定义在头文件sys/stat.h中；

> \#include <sys/types.h>
> \#include <sys/stat.h>
> \#include <fcntl.h>
> \#include <unistd.h>
> int mknod(const char *pathname, mode_t mode, dev_t dev);
> \#include <sys/types.h>
> \#include <sys/stat.h>
> int mkfifo(const char *pathname, mode_t mode);

返回值：都是成功返回0，失败返回-1；

path为创建的命名管道的全路径名；

mod为创建的命名管道的模式，指明其存取权限；

dev为设备值，该值取决于文件创建的种类，它只在创建设备文件时才会用到；

mkfifo函数的作用：在文件系统中创建一个文件，该文件用于提供FIFO功能，即命名管道。

命名管道的特点：

1.命名管道是一个存在于硬盘上的文件，而管道是存在于内存中的特殊文件。所以当使用命名管道的时候必须先open将其打开。

2.命名管道可以用于任何两个进程之间的通信，不管这两个进程是不是父子进程，也不管这两个进程之间有没有关系。

一个简单的关于命名管道的例子：

代码实现如下：

server.c

> \#include<stdio.h>
> \#include<stdlib.h>
> \#include<sys/types.h>
> \#include<sys/stat.h>
> \#include<fcntl.h>
> void testserver()
> {
> int namepipe=mkfifo("myfifo",S_IFIFO|0666); //创建一个存取权限为0666的命名管道
> if(namepipe == -1){
> perror("mkfifo error");
> exit(1);
> }
> int fd=open("./myfifo",O_RDWR); //打开该命名管道
> if(fd == -1){
> perror("open error");
> exit(2);
> }
> char buf[1024];
> while(1)
> {
> printf("sendto# ");
> fflush(stdout);
> ssize_t s=read(0,buf,sizeof(buf)-1); //从标准输入获取消息
> if(s > 0){
> buf[s-1]='\0'; //过滤掉从标准输入中获取的换行
> if(write(fd,buf,s) == -1){ //把该消息写入到命名管道中
> perror("write error");
> exit(3);
> }
> }
> }
> close(fd);
> }
> int main()
> {
> testserver();
> return 0;
> }

client.c

> \#include<stdio.h>
> \#include<stdlib.h>
> \#include<sys/types.h>
> \#include<sys/stat.h>
> \#include<fcntl.h>
> void testclient()
> {
> int fd=open("./myfifo",O_RDWR);
> if(fd == -1){
> perror("open error");
> exit(1);
> }
> char buf[1024];
> while(1){
> ssize_t s=read(fd,buf,sizeof(buf)-1);
> if(s > 0){
> printf("client# %s\n",buf);
> }
> else{ //读失败或者是读取到字符结尾
> perror("read error");
> exit(2);
> }
> }
> close(fd);
> }
> int main()
> {
> testclient();
> return 0;
> }





![img](https://pic4.zhimg.com/80/v2-2cf9e404ef04c996b5179d83a37f962b_720w.webp)



### 3.3 消息队列(msg)

由于内容较多，以后再详细分享

### 3.4 信号量(sem)

什仫是信号量？

信号量的本质是一种数据操作锁，用来负责数据操作过程中的互斥，同步等功能。

信号量用来管理临界资源的。它本身只是一种外部资源的标识，不具有数据交换功能，而是通过控制其他的通信资源实现进程间通信。 可以这样理解，信号量就相当于是一个计数器。当有进程对它所管理的资源进行请求时，进程先要读取信号量的值：大于0，资源可以请求；等于0，资源不可以用，这时进程会进入睡眠状态直至资源可用。

当一个进程不再使用资源时，信号量+1(对应的操作称为V操作)，反之当有进程使用资源时，信号量-1(对应的操作为P操作)。对信号量的值操作均为原子操作。

为什仫要使用信号量？

为了防止出现因多个程序同时访问一个共享资源而引发的一系列问题，我们需要一种方法，它可以通过生成并使用令牌来授权，在任一时刻只能有一个执行线程访问代码的临界区域。

什仫是临界区？什仫是临界资源？

临界资源：一次只允许一个进程使用的资源。

临界区：访问临界资源的程序代码片段。

信号量的工作原理？

P(sv)：如果sv的值大于零，就给它减1；如果它的值为零，就挂起该进程的执行等待操作；

V(sv)：如果有其他进程因等待sv而被挂起，就让它恢复运行，如果没有进程因等待sv而挂起，就给它加1；

举个例子，就是两个进程共享信号量sv，一旦其中一个进程执行了P(sv)操作，它将得到信号量，并可以进入临界区，使sv减1。而第二个进程将被阻止进入临界区，因为当它试图执行P(sv)时，sv为0，它会被挂起以等待第一个进程离开临界区域并执行V(sv)释放信号量，这时第二个进程就可以恢复执行了。

与信号量有关的函数操作？

1).创建/获取一个信号量集合

> \#include <sys/types.h>
> \#include <sys/ipc.h>
> \#include <sys/sem.h>
> int semget(key_t key, int nsems, int semflg);

返回值：成功返回信号量集合的semid，失败返回-1。

key:可以用函数key_t ftok(const char *pathname, int proj_id);来获取。

nsems:这个参数表示你要创建的信号量集合中的信号量的个数。信号量只能以集合的形式创建。

semflg:同时使用IPC_CREAT和IPC_EXCL则会创建一个新的信号量集合。若已经存在的话则返回-1。单独使用IPC_CREAT的话会返回一个新的或者已经存在的信号量集合。

2).信号量结合的操作

> \#include <sys/types.h>
> \#include <sys/ipc.h>
> \#include <sys/sem.h>
> int semop(int semid, struct sembuf *sops, unsigned nsops);
> int semtimedop(int semid, struct sembuf *sops, unsigned nsops,struct timespec *timeout);

返回值：成功返回0，失败返回-1；

semid：信号量集合的id；

> struct sembuf *sops;
> struct sembuf
> {
> unsigned short sem_num; /* semaphore number */
> short sem_op; /* semaphore operation */
> short sem_flg; /* operation flags */
> }

sem_num：为信号量是以集合的形式存在的，就相当于所有信号在一个数组里面，sem_num表示信号量在集合中的编号；

sem_op：示该信号量的操作(P操作还是V操作)。如果其值为正数，该值会加到现有的信号内含值中。通常用于释放所控资源的使用权；如果sem_op的值为负数，而其绝对值又大于信号的现值，操作将会阻塞，直到信号值大于或等于sem_op的绝对值。通常用于获取资源的使用权 。

sem_flg:信号操作标志，它的取值有两种：IPC_NOWAIT和SEM_UNDO。

IPC_NOWAIT:对信号量的操作不能满足时，semop()不会阻塞，而是立即返回，同时设定错误信息；

SEM_UNDO: 程序结束时(不管是正常还是不正常)，保证信号值会被设定；

nsops:表示要操作信号量的个数。因为信号量是以集合的形式存在，所以第二个参数可以传一个数组，同时对一个集合中的多个信号量进行操作。

semop()调用之前的值。这样做的目的在于避免程序在异常的情况下结束未将锁定的资源解锁(死锁)，造成资源永远锁定。

3).int semctl(int semid,int semnum,int cmd,...);

semctl()在semid标识的信号量集合上，或者该信号量集合上第semnum个信号量上执行cmd指定的控制命令。根据cmd不同，这个函数有三个或四个参数，当有第四个参数时，第四个参数的类型是union。

> union semun{
> int val; //使用的值
> struct semid_ds *buf; //IPC_STAT、IPC_SET使用缓存区
> unsigned short *array; //GETALL、SETALL使用的缓存区
> struct seminfo *__buf; //IPC_INFO(linux特有)使用缓存区
> };

返回值：成功返回0，失败返回-1；

semid:信号量集合的编号。

semnum:信号量在集合中的标号。

4).信号量类似消息队列也是随内核的，除非用命令才可以删除该信号量

ipcs -s //查看创建的信号量集合的个数

ipcrm -s semid //删除一个信号量集合

一个简单的关于信号量的例子？

父进程中打印BB，子进程中打印AA。利用信号量机制使得AA和BB之间不出现乱序。此时的显示器就是临界资源，我们需要在父子进程的临界区进行加锁。

comm.h

> \#ifndef _COMM_H_
> \#define _COMM_H_
> \#include<stdio.h>
> \#include<unistd.h>
> \#include<stdlib.h>
> \#include<sys/types.h>
> \#include<sys/ipc.h>
> \#include<sys/sem.h>
> \#include<sys/types.h>
> \#include<sys/wait.h>
> \#define PATHNAME "."
> \#define PROJID 0x6666
> union semun{
> int val; /* Value for SETVAL */
> struct semid_ds *buf; /* Buffer for IPC_STAT, IPC_SET */
> unsigned short *array; /* Array for GETALL, SETALL */
> struct seminfo *__buf; /* Buffer for IPC_INFO(Linux-specific) */
> };
> int CreateSemSet(int num);//创建信号量
> int GetSemSet(); //获取信号量
> int InitSem(int sem_id,int which);
> int P(int sem_id,int which); //p操作
> int V(int sem_id,int which); //v操作
> int DestroySemSet(int sem_id);//销毁信号量
> \#endif //_COMM_H_
> comm.c
> \#include"comm.h"
> static commSemSet(int num,int flag)
> {
> key_t key=ftok(PATHNAME,PROJID);
> if(key == -1)
> {
> perror("ftok error");
> exit(1);
> }
> int sem_id=semget(key,num,flag);
> if(sem_id == -1)
> {
> perror("semget error");
> exit(2);
> }
> return sem_id;
> }
> int CreateSemSet(int num)
> {
> return commSemSet(num,IPC_CREAT|IPC_EXCL|0666);
> }
> int InitSem(int sem_id,int which)
> {
> union semun un;
> un.val=1;
> int ret=semctl(sem_id,which,SETVAL,un);
> if(ret < 0)
> {
> perror("semctl");
> return -1;
> }
> return 0;
> }
> int GetSemSet()
> {
> return commSemSet(0,IPC_CREAT);
> }
> static int SemOp(int sem_id,int which,int op)
> {
> struct sembuf buf;
> buf.sem_num=which;
> buf.sem_op=op;
> buf.sem_flg=0; //
> int ret=semop(sem_id,&buf,1);
> if(ret < 0)
> {
> perror("semop error");
> return -1;
> }
> return 0;
> }
> int P(int sem_id,int which)
> {
> return SemOp(sem_id,which,-1);
> }
> int V(int sem_id,int which)
> {
> return SemOp(sem_id,which,1);
> }
> int DestroySemSet(int sem_id)
> {
> int ret=semctl(sem_id,0,IPC_RMID);
> if(ret < 0)
> {
> perror("semctl error");
> return -1;
> }
> return 0;
> }
> SemSet.c
> \#include"comm.h"
> void testSemSet()
> {
> int sem_id=CreateSemSet(1); //创建信号量
> InitSem(sem_id,0);
> pid_t id=fork();
> if(id < 0){
> perror("fork error");
> exit(1);
> }
> else if(id == 0){ //child，打印AA
> printf("child is running,pid=%d,ppid=%d\n",getpid(),getppid());
> while(1)
> {
> P(sem_id,0); //p操作，信号量的值减1
> printf("A");
> usleep(10031);
> fflush(stdout);
> printf("A");
> usleep(10021);
> fflush(stdout);
> V(sem_id,0); //v操作，信号量的值加1
> }
> }
> else //father，打印BB
> {
> printf("father is running,pid=%d,ppid=%d\n",getpid(),getppid());
> while(1)
> {
> P(sem_id,0);
> printf("B");
> usleep(10051);
> fflush(stdout);
> printf("B");
> usleep(10003);
> fflush(stdout);
> V(sem_id,0);
> }
> wait(NULL);
> }
> DestroySemSet(sem_id);
> }
> int main()
> {
> testSemSet();
> return 0;
> }

### 3.5 共享内存(shm)

共享内存的原理图：



![img](https://pic3.zhimg.com/80/v2-37e557bf77de8334d999bc63bf936272_720w.webp)





与共享内存有关的函数：

1). 创建共享内存

> \#include <sys/ipc.h>
> \#include <sys/shm.h>
> int shmget(key_t key, size_t size, int shmflg);

返回值：成功返回共享内存的id，失败返回-1；

key：和上面介绍的信号量的semget函数的参数key一样；

size：表示要申请的共享内存的大小，一般是4k的整数倍；

flags：IPC_CREAT和IPC_EXCL一起使用，则创建一个新的共享内存，否则返回-1。IPC_CREAT单独使用时返回一个共享内存，有就直接返回，没有就创建。

2).挂接函数

> void *shmat(int shmid);

返回值：返回这块内存的虚拟地址；

shmat的作用是将申请的共享内存挂接在该进程的页表上，是将虚拟内存和物理内存相对应；

3).去挂接函数

> int shmdt(const void *shmaddr);

返回值：失败返回-1；

shmdt的作用是去挂接，将这块共享内存从页表上剥离下来，去除两者的映射关系；

shmaddr:表示这块物理内存的虚拟地址。

4).int shmctl(int shmid,int cmd,const void* addr);

shmctl用来设置共享内存的属性。当cmd是IPC_RMID时可以用来删除一块共享内存。

5).共享内存类似消息队列和信号量，它的生命周期也是随内核的，除非用命令才可以删除该共享内存；

ipcs -m //查看创建的共享内存的个数

ipcrm -m shm_id //删除共享内存

一个简单的关于共享内存的例子：

利用共享内存实现在serve这个进程中向共享内存中写入数据A，从client读出数据。

comm.h

> \#ifndef __COMM__
> \#define __COMM__
> \#include<stdio.h>
> \#include<sys/types.h>
> \#include<sys/ipc.h>
> \#include<sys/shm.h>
> \#include<unistd.h>
> \#define PATHNAME "."
> \#define PROCID 0x6666
> \#define SIZE 4096*1
> int CreatShm();
> int GetShm();
> //int AtShm();
> //int DtShm();
> int DestroyShm(int shm_id);
> \#endif
> comm.c
> \#include"comm.h"
> static int CommShm(int flag)
> {
> key_t key=ftok(PATHNAME,PROCID);
> if(key < 0)
> {
> perror("ftok");
> return -1;
> }
> int shm_id=shmget(key,SIZE,flag);
> if(shm_id < 0)
> {
> perror("shmget");
> return -2;
> }
> return shm_id;
> }
> int CreatShm()
> {
> return CommShm(IPC_CREAT|IPC_EXCL|0666);
> }
> int GetShm()
> {
> return CommShm(IPC_CREAT);
> }
> //int AtShm();
> //int DtShm();
> int DestroyShm(int shm_id)
> {
> int ret=shmctl(shm_id,IPC_RMID,NULL);
> if(ret < 0)
> {
> perror("shmctl");
> return -1;
> }
> return 0;
> }
> server.c
> \#include"comm.h"
> void testserver()
> {
> int shm_id=CreatShm();
> printf("shm_id=%d\n",shm_id);
> char *mem=(char *)shmat(shm_id,NULL,0);
> while(1)
> {
> sleep(1);
> printf("%s\n",mem);
> }
> shmdt(mem);
> DestroyShm(shm_id);
> }
> int main()
> {
> testserver();
> return 0;
> }
> client.c
> \#include"comm.h"
> void testclient()
> {
> int shm_id=GetShm();
> char *mem=(char *)shmat(shm_id,NULL,0);
> int index=0;
> while(1)
> {
> sleep(1);
> mem[index++]='A';
> index %= (SIZE-1);
> mem[index]='\0';
> }
> shmdt(mem);
> DestroyShm(shm_id);
> }
> int main()
> {
> testclient();
> return 0;
> }

共享内存的特点：

共享内存是这五种进程间通信方式中效率最高的。但是因为共享内存没有提供相应的互斥机制，所以一般共享内存都和信号量配合起来使用。

为什仫共享内存的方式比其他进程间通信的方式效率高？

消息队列，FIFO，管道的消息传递方式一般为 ：

1).服务器获取输入的信息；

2).通过管道，消息队列等写入数据至内存中，通常需要将该数据拷贝到内核中；

3).客户从内核中将数据拷贝到自己的客户端进程中；

4).然后再从进程中拷贝到输出文件；

上述过程通常要经过4次拷贝，才能完成文件的传递。

而共享内存只需要：

1).输入内容到共享内存区

2).从共享内存输出到文件

上述过程不涉及到内核的拷贝，这些进程间数据的传递就不再通过执行任何进入内核的系统调用来传递彼此的数据，节省了时间，所以共享内存是这五种进程间通信方式中效率最高的。

原文地址：https://zhuanlan.zhihu.com/p/94856678

作者：linux