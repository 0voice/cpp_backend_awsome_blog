# 【NO.580】多线程实践概述

## 0.线程背景

        在单核CPU时代，程序的执行效率单纯的依赖更快的硬件；随着时间的变化，在2005年3月，Herb Sutter在Dobb's Journal发表了《The Free Lunch is over: A Fundamental Turn Toward Concurrency is Software》一文，文章分析处理器厂商改善CPU性能的传统方法，如提升时钟速度和指令吞吐量的方式，遇到了瓶颈，处理器发展方向趋向于超线程和多架构模式。
    
         为了让代码执行的更快、更高效，开发人员可通过编写并发代码，充分大会多核处理器的优势，使程序性能得到提升。

## 1.线程与进程

        线程与进程有着千丝万缕的联系，两者密不可分。在Linux下程序或可执行文件只是一组指令的集合，没有执行的含义。进程拥有自己独立的进程地址空间和上下文堆栈，但对一个程序本身的执行操作来说，进程不执行任何代码，只是提供一个大环境容器，进程中实际执行体是线程，因此每个进程中至少的有一个线程，即主线程。
    
        多进程、多线程都能做并发，两者的差异何在？
    
        进程之间，彼此的地址空间是独立的，但线程会共享内存地址空间。同一个进程的多个线程共享一份全局内存区域，包括初始化数据段、未初始化数据段和动态分配的堆内存。

  ![img](https://img-blog.csdnimg.cn/646178764a12489a99deed12f0c34f6b.png)


 这种共享方式给线程带来了很多优势：

创建线程花费的时间少于创建进程花费的时间
线程间的上下文切换小于进程间的上下文切换
线程间数据共享比进程之间的共享要简单
线程的缺点：

多个线程中只要有一个线程出现BUG，就会出现SegmenFault导致进程退出，相比之下进程的地址空间互相独立，多个进程通过进程间通信(IPC),一个进程的异常退出不会影响其他进程
多线程控制流、时许很难控制，每次执行都不一样
        多进程和多线程都能够很好的发挥出多核处理器的性能，两者孰优孰劣需要依赖具体的应用场景去抉择。

## 2.线程属性

### 2.1 线程ID

        在内核2.6以前的调度实体都是进程，内核并没有真正执行线程，为了遵循POSIX标准，Redhat发布了NPTL（native posix thread library）库,它的实现方式是，在内核中线程仍然被当作进程，也是用task_struct进行描述，这种线程又叫LWP（用户态线程）。

```
Struct 	task_struct {		
    …
	pid_t	pid;	//线程id
	pid_t	tgid;	//Thread Group ID(对应用户层的进程ID)

	…
	struct	task_struct	*group_leader;
	struct	list_head	thread_group;
	 
	…

}
```

        没有线程之前，一个进程对应内核的一个进程描述符，对应一个进程ID。但是引入线程概念后，情况发生了变化，一个进程下管辖的N个线程。每个线程作为一个独立的调度实体在内核中有自己的进程描述符。进程和内核的进程描述符由1：1变成1：n。POSIX标准要求所有线程调用getpid函数返回相同的进程ID。为了解决这个问题，引入了线程组的概念。
    
        线程组内的第一个线程，在用户态被称为主线程，在内核中被称为Group Leader.内核在创建第一个线程时，会将线程组ID的值设置成第一个线程ID，group_leader指针则指向自身，即主线程的进程描述符。

```
//线程组ID等于主线程ID
p->tgid = p->pid;
p->group_leader = p;
INIT_LIST_HEAD(&p->thread_group);
```

        通过group_leader指针，每个线程都能够找到主线程，主线程中存在一个链表头结构，后面创建的每一个线程都会被链入到该双向链表中。通过链表结构能偶遍历组内其他线程，也能够找到主线程。

Linux线程ID：

        用ps -eLf查询得，进程syslog（680）组内有4个线程，每个线程的ID号分别为680、703、704、705共4个线程。

```
yp@ubuntu:~$ ps -eLf
UID         PID   PPID    LWP  C NLWP STIME TTY          TIME CMD
........
syslog      680      1    680  0    4 01:02 ?        00:00:00 /usr/sbin/rsyslogd -n
syslog      680      1    703  0    4 01:02 ?        00:00:00 /usr/sbin/rsyslogd -n
syslog      680      1    704  0    4 01:02 ?        00:00:00 /usr/sbin/rsyslogd -n
syslog      680      1    705  0    4 01:02 ?        00:00:00 /usr/sbin/rsyslogd -n
```

         在已知进程ID前提下，可在/proc/PID/task目录下的子目录来查看进程内的线程个数及其线程ID。示例如下：

```
yp@ubuntu:~$ ll /proc/680/task/
total 0
dr-xr-xr-x 6 syslog syslog 0 Jul  5 03:36 ./
dr-xr-xr-x 9 syslog syslog 0 Jul  5 01:02 ../
dr-xr-xr-x 7 syslog syslog 0 Jul  5 03:36 680/
dr-xr-xr-x 7 syslog syslog 0 Jul  5 03:36 703/
dr-xr-xr-x 7 syslog syslog 0 Jul  5 03:36 704/
dr-xr-xr-x 7 syslog syslog 0 Jul  5 03:36 705/
```

### 2.2 获取线程ID

调用pthread_create函数时，函数调用成功，通过第一个参数返回

```
#include <pthread.h>

pthread_t   tid;
pthread_create(&tid, NULL, thread_proc, NULL);
```

在对应线程中调用pthread_self()

```
#include <pthread.h>
 pthread_t   tid = pthread_self();
```

系统调用

```
#include <sys/syscall.h>
int tid = syscall(SYS_gettid);
```

        pthread_t输出的是一块内存地址空间，由于进程的虚拟地址与物理地址的映射关系，pthread_t类型的线程ID可能不是全系统唯一的。通过syscall(SYS_gettid)获取的线程ID是系统范围内全局唯一的。

## 2.Linux同步对象

### 2.1 Linux互斥量

        Linux互斥量与Windows的临界区对象用法相似，也是通过限制多个线程同时访问临界资源以达到保护的目的。Linux互斥量在NPTL中实现，使用数据结构pthread_mutex_t表示一个互斥体对象。

互斥量初始化

(1) 使用PTHREAD_MUTEX_INITIALIZER直接给互斥体变量赋值

```
#include <pthread.h>
pthread_mutex_t = PTHREAD_MUTEX_INITIALIZER;
```

(2) 若互斥量是动态分配或者需要设置其属性，则使用pthread_mutex_init函数

```
int pthread_mutex_init(pthread_mutex_t* restrict_mutex, 
                       const pthread_mutexattr_init* restrict_attr)
```

        当我们不在需要一个互斥体对象时，可以使用pthread_mutex_destroy函数来销毁它，函数签名如下：

```
int pthread_mutex_lock(pthread_mutex_t* mutex);
int pthread_mutex_trylock(pthread_mutex_t* mutex);
int pthread_mutex_unlock(pthread_mutex_t* mutex);
```

设置互斥量的属性

        设置互斥体属性时需要创建一个pthread_mutexattr_t类型的对象，并用pthread_mutexattr_init进行初始化。

```
#include <pthread.h>

int main()
{
    ...

    pthread_mutexattr_t mtx_attr;
    pthread_mutexattr_init(&mtx_attr);
    pthread_mutexattr_settype(&mtx_attr, PTHREAD_MUTEX_NORMAL);  //普通锁
    pthread_mutex_init(&g_mutex, &mtx_attr);
     
    ...

}
```

互斥体的加锁解锁

        互斥量一般配合加锁、解锁操作完成对临界资源的安全访问，Linux相关函数签名如下：

```
int pthread_mutex_lock(pthread_mutex_t* mutex);
int pthread_mutex_trylock(pthread_mutex_t* mutex);
int pthread_mutex_unlock(pthread_mutex_t* mutex);
```

### 2.2 条件变量

1. **为什么使用环境变量？**

       不使用条件变量的情形，伪代码如下：

```
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;

int WaitForTrue()
{
    pthread_mutex_lock(&mtx);

    while (condition is false)          // 条件不满足
    {
        pthread_mutex_unlock(&mtx);     //解锁，等待其他线程改变共享数据
        sleep(n);                       //睡眠等待，再次加锁验证条件是否满足
        pthread_mutex_lock(&mtx);       
    }

}
```

        这段代码对共享资源进行访问时存在严重的效率问题。假设等待的条件突然满足了，但是while循环仍然会消耗一下时间，响应不及时。
    
         因此需要一种机制：线程在条件不满足的时候，主动让出互斥量，让其他线程执行，其他线程发现条件满足后向其发送信号，就可以立即唤醒它。

2. **条件变量的使用**

       环境变量的初始化和销毁，使用如下API函数：

```
int pthread_cond_init(pthread_cond_t* cond, const pthread_condattr_t* attr);
int pthread_cond_destory(pthread_cond_t* cond);
```

        可使用如下API函数等待条件变量被唤醒：

```
int pthread_cond_wait(pthread_cond_t* restrict cond, pthread_mutex_t* restrict mutex);
int pthread_cond_timedwait(pthread_cond_t* restrict cond, pthread_mutex_t* restrict mutex
                           const struct timespec* restrict abstime);
```

        如果条件变量等待的条件没有被满足，则条用pthread_cond_wait的线程会一直阻塞，pthread_cond_timedwait是带超时时间的阻塞，超过abstime设置的时间后，立即返回。
    
        因调用pthread_cond_wait而阻塞的线程可被如下API唤醒：

```
int pthread_cond_signal(pthread_cond_t* cond);
int pthread_cond_broadcast(pthread_cond_t* cond);
```

        如果多个线程阻塞在pthread_cond_wait，由于pthread_cond_signal一次只能唤醒一个线程，具体那个线程被唤醒是随机的。pthread_cond_broadcast可以唤醒所有等待的线程。

3. **条件变量虚假唤醒**

```
if (tasks.empty()) {
    pthread_cond_wait(&cv, &mutex);
}
```

```
while (tasks.empty()) {
    pthread_cond_wait(&cv, &mutex);
}
```

       没有其他线程向条件变量发送信号，但等待此条件变量的线程可能被唤醒，即操作系统唤醒pthread_cond_wait时，tasks.empty()仍然为true，这种情形称为虚假唤醒。因此需要在while循环中，当条件变量被唤醒时，还需要判断条件是否满足。

## 3.C++11同步对象

### 3.1  独占性互斥量 std::mutex

       互斥量的使用一般是通过lock（）方法来阻塞线程，直到获得互斥量的所有权为止。在线程获得互斥量并完成任务后，就必须使用unlock（）来解除对互斥量的占用，lock （）和unlock（）必须成对出现。代码示例如下：

```
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::mutex g_lock;
void ThreadProc()
{
    g_lock.lock();

    std::cout << "entered thread " << std::this_thread::get_id() << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "leaving thread " << std::this_thread::get_id() << std::endl;
     
    g_lock.unlock();

} 

int main(void)
{
    std::thread t1(ThreadProc);
    std::thread t2(ThreadProc);

    t1.join();
    t2.join();
     
    system("pause");
    return 0;

}
```



        使用lock_guard（）可以简化lock/unlock写法，避免忘记unlock（）。lock_guard（）出作用域会自动解锁。

```
void ThreadProc()
{
    std::lock_guard<std::mutex> locker(g_lock);

    std::cout << "entered thread " << std::this_thread::get_id() << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "leaving thread " << std::this_thread::get_id() << std::endl;

} 
```

### 3.2 递归互斥量 std::recursive_mutex

```
#include <iostream>
#include <thread>
#include <mutex>
#include <unistd.h>

int g_resourceID = 1;
std::recursive_mutex mtx;

void ThreadFunc_2() {
    mtx.lock();                     //第二次上锁

	g_resourceID = g_resourceID * g_resourceID ;
	printf("g_resourceID = %d \n", g_resourceID );
	mtx.unlock();

}

void ThreadFunc_1() {
	mtx.lock(); 

	g_resourceID = g_resourceID + 1;
	ThreadFunc_2();
	printf("g_resourceID = %d \n", g_resourceID );
	 
	mtx.unlock();

}

int main() {
	std::thread t1(ThreadFunc_1);
	t1.join();

	return 0;

}
```



        递归互斥锁可对同一互斥量多次加锁，可以用来解决同一线程对互斥量多次加锁导致死锁阻塞的问题。

### 3.3 条件变量

         C++11使用condition_variable类表示条件变量，与上文介绍的Linux系统原生的条件变量一样，此外还提供了等待条件满足的wait方法（wait、wait_for、wait_until）。发送信号使用notify方法（notify_one、notify_all）。与Linux系统的条件变量相比，C++11的std::condition_variable不在需要显示地初始化和销毁。

```
#include <thread>
#include <condition_variable>
#include <mutex>
#include <list>
#include <iostream>

template<typename T>
class SimpleSyncQueue
{
    public:
    SimpleSyncQueue() { }
    void Put(const T& x)
    {
        std::lock_guard<std::mutex> locker(m_mutex);
        m_queue.push_back(x);
        m_notEmpty.notify_one();
```

————————————————
版权声明：本文为CSDN博主「FooNY」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zouph/article/details/125560751