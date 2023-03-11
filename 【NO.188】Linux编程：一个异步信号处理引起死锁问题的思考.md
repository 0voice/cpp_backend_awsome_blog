# 【NO.188】Linux编程：一个异步信号处理引起死锁问题的思考

## 1.前言

最近在维护别人的代码时，遇到一个线程死锁问题，一番折腾，最终定位到的是“信号异步处理引发死锁问题”。“信号异步处理死锁问题”是一个老生常谈的问题了，虽然问题简单，但定位起来仍需花费点时间；如果代码量大、复现概率低，还需花费更多的人力。因此，有必要回顾下这个问题，避免踩坑，包括新手和老手都可能踩坑。

死锁问题原型伪代码：

```text
void signal_handle(int signo) 
{
	pthread_mutex_lock(&mutex);
	/* todo */
	pthread_mutex_unlock(&mutex);
	return; 
}

int main(int argc, char *argv)
{
	signal(SIGALRM, signal_handle);
	pthread_mutex_init(&mutex, NULL);
	printf("Main thread id:0x%lx\n", syscall(SYS_gettid));
	for(;;)
	{
		pthread_mutex_lock(&mutex);
		/* todo */
		pthread_mutex_unlock(&mutex);
		usleep(10);
	}
	return 0; 
}
```

## 2.为什么会产生死锁

### 2.1 死锁

死锁（DeadLock) 是指两个或者两个以上的进程（线程）在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程（线程）称为死锁进程(线程）。

根据死锁的定义，死锁产生的条件是：

- 两个进程（线程）以上
- 有资源竞争

### 2.2 分析

对于信号回调函数，与主线程是同一线程（线程ID相同），不满足两个进程（线程）的条件，为什么会发生死锁呢？下面我们先通过一段代码验证信号回调函数与主线程是否为同一个线程。

```text
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <sys/syscall.h>

void signal_handle(int signo) 
{
	printf("Signal thread id:0x%lx\n", syscall(SYS_gettid));
	return; 
}

int main(int argc, char *argv)
{
	signal(SIGALRM, signal_handle);
	printf("Main thread id:0x%lx\n", syscall(SYS_gettid));
	alarm(1);
	sleep(1);
	return 0; 
}
```

编译执行：

```text
acuity@ubuntu:/mnt/hgfs/LSW/temp$ gcc signal.c -o signal 
acuity@ubuntu:/mnt/hgfs/LSW/temp$ ./signal
Main thread id:0x1509
Signal thread id:0x1509
```

通过测试代码可以知道，信号回调函数与主线程的ID一致，说明两者实质是同一线程。

### 2.3 结论

虽然信号回调函数与主线程是同一线程，但当主线程捕捉到信号时，主线程的执行任务会被挂起，转而去执行处理信号，并执行信号回调函数。即是产生了软中断。因此，如果主线程持有锁，此时有信号进来，CPU去处理信号回调函数；函数中由于申请不到锁资源，处于等待状态；主线程因为软中断（信号回调函数）未退出，而一直处于上锁状态。两者一直在等待资源，形成了死锁。

## 3.避免死锁

既然我们知道这种情况易产生死锁，避免死锁才是我们的根本目的。而避免死锁方式是回调函数中禁止使用锁，或者以其他方式替换处理。参考以下方式。

- 使用自旋锁（ spinlock ）代替互斥锁（mutex），程序进入了spinlock临界区，中断是会被关闭的；即是spinunlock后才会捕捉到信号，避免了死锁；
- 新建一个信号处理线程，把信号回调任务由该线程处理；信号处理线程循环调用sigwait（sigtimedwait）同步信号；
- 创建线程时，调用 pthread_sigmask 设置本线程的信号掩码，屏蔽信号捕捉。

处理复杂的任务，建议使用第二种方式。

**除此之外，一个严谨的信号处理回调函数，应该遵循以下基本原则：**

- 信号回调函数尽可能简单，确保尽快退出函数，与中断处理函数原则一样；
- 信号回调函数不能调用不可重入函数和线程不安全函数，如`malloc`、`free`、`printf`、标准I/O函数；
- 信号回调函数访问全局变量时，变量需加`volatile`修饰，避免编译器优化。

## 4.举一反三

通过上面的分析，回调函数禁止使用互斥锁。但是，一些库函数、第三方SDK、开发成员写的模块等，函数内部可能使用了互斥锁，使用时需格外注意，因为这些函数没有显式申请互斥锁，如果出现问题，将会增加查找问题的难度，无法直接通过审查代码初步发现<。比如，C库中常见的线程安全函数（内部加锁），这些函数在回调函数中使用时需格外注意：

- localtime_r，时间转换函数，localtime返回的是静态变量，不是线程安全函数，多线程访问时，值可能会被修改；后C库提供localtime_r线程安全函数，实际是内部加了锁；
- rand_r，随机数生成函数；
- strtok_r，字符串分割函数；
- asctime_r、ctime_r，时间格式化为字符函数；
- gethostbyaddr_r、gethostbyname_r，主机名称和地址转换函数。

## 5.死锁例子代码

```text
#include <stdio.h>
#include <stdint.h>
#include <unistd.h>
#include <signal.h>
#include <sys/syscall.h>
#include "pthread.h" 

pthread_mutex_t mutex;

void signal_handle(int signo) 
{
	pthread_mutex_lock(&mutex);
	printf("Signal thread id:0x%lx\n", syscall(SYS_gettid));
	pthread_mutex_unlock(&mutex);
	return; 
}

int main(int argc, char *argv)
{
	uint32_t sys = 0;
	
	signal(SIGALRM, signal_handle);
	pthread_mutex_init(&mutex, NULL);
	printf("Main thread id:0x%lx\n", syscall(SYS_gettid));
	for(;;)
	{
		pthread_mutex_lock(&mutex);
		printf("sys [%d]\n", sys++);
		if (sys == 3)
		{
			alarm(1);
			sleep(1);	/* 故意延迟，产生死锁 */
		}
		if (sys == 5)
		{
			pthread_mutex_unlock(&mutex);
			break;
		}
		pthread_mutex_unlock(&mutex);
		sleep(1);
	}
	return 0; 
}
```

原文地址：https://zhuanlan.zhihu.com/p/550607034

作者：Linux