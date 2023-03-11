# 【NO.198】互斥锁、自旋锁、原子操作的原理、区别及应用场景

## 1.互斥锁

**原理：**

互斥锁属于sleep-waiting类型的锁，例如在一个双核的机器上有两个线程（线程A和线程B）,它们分别运行在Core0和Core1上。假设线程A想要通过pthread_mutex_lock操作去得到一个临界区的锁，而此时这个锁正被线程B所持有，那么线程A就会被阻塞，Core0会在此时进行上下文切换(Context Switch)将线程A置于等待队列中，此时Core0就可以运行其它的任务而不必进行忙等待。

**适用场景：**

因互斥锁会引起线程的切换，效率较低。使用互斥锁会引起线程阻塞等待，不会一直占用这cpu，因此当锁的内容较多，切换不频繁时，建议使用互斥锁

**使用方法：**

```text
/*初始化一个互斥锁*/
int pthread_mutex_init(pthread_mutex_t *restrict  mutex,const pthread_mutexattr_t *restrict attr);
参数：
	mutex:互斥锁地址。类型是pthread_mutex_t.
	attr:设置互斥量的属性，通常采用默认属性，即将attr设为NULL。
	可以使用宏PTHREAD_MUTEX_INITALIZER静态初始化互斥锁，如：
	pthread_mutex_t mutex=PTHREAD_MUTEX_INITIALIZER;
	这种方法等价于NULL指定的attr参数调用pthread_mutex_init()来完成动态初始化，不同之处在于PTHREAD_MUTEX_INITIALIZER宏不进行错误检查。
返回值：
	成功：0
	失败：非0
```



```text
/*销毁指定的一个互斥锁，释放资源*/
int pthread_mutex_destroy(pthread_mutex_t *mutex);
参数：
	mutex：互斥锁地址
返回值：
	成功：0
	失败：非0
```



```text
/*对互斥锁上锁，若互斥锁已经上锁，则调用这阻塞，直到互斥锁解锁后再上锁*/
int pthread_mutex_lock(pthread_mutex_t *mutex)
参数：
	mutex:互斥锁地址
返回值：
	成功：0
	失败：非0
```



```text
/*对指定的互斥锁解锁*/
int pthread_mutex_unlock(pthread_mutex_t *mutex);
参数：
	mutex:互斥锁地址
返回值：
	成功：0
	失败：非0
```

## 2.自旋锁

**原理：**

Spin lock（自旋锁）属于busy-waiting类型的锁，如果线程A是使用pthread_spin_lock操作去请求锁，那么线程A就会一直在Core0上进行忙等待并不停的进行锁请求，直到得到这个锁为止。自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁。

**使用场景：**

“自旋锁”的作用是为了解决某项资源的互斥使用。因为自旋锁不会引起调用者睡眠，所以自旋锁的效率远高于互斥锁。因此如果锁的内容较少，阻塞的时间较短，使用自旋锁比较好。

自旋锁一直占用着CPU，他在未获得锁的情况下，一直运行（自旋），所以占用着CPU，如果不能在很短的时间内获得锁，这无疑会使CPU效率降低。因此我们要慎重使用自旋锁，自旋锁只有在内核可抢占式或SMP的情况下才真正需要，在单CPU且不可抢占式的内核下，自旋锁的操作为空操作。自旋锁适用于锁使用者保持锁时间比较短的情况下。

**使用方法**

```text
/*初始化一个自旋锁*/
int pthread_spin_init(pthread_spinlock_t *lock, int pshared);
参数：
	pthread_spinlock_t ：初始化自旋锁
	pshared取值：
		PTHREAD_PROCESS_SHARED：该自旋锁可以在多个进程中的线程之间共享。（可以被其他进程中的线程看到）
		PTHREAD_PROCESS_PRIVATE:仅初始化本自旋锁的线程所在的进程内的线程才能够使用该自旋锁。
返回值：
	若成功，返回0；否则，返回错误编号
```



```text
/*销毁一个锁*/
int pthread_spin_destroy(pthread_spinlock_t *lock);
返回值：
	若成功，返回0；否则，返回错误编号
```



```text
/*用来获取（锁定）指定的自旋锁. 如果该自旋锁当前没有被其它线程所持有，则调用该函数的线程获得该自旋锁.否则该函数在获得自旋锁之前不会返回。*/
int pthread_spin_lock(pthread_spinlock_t *lock);
返回值：
	若成功，返回0；否则，返回错误编号
```



```text
/*解锁*/
int pthread_spin_unlock(pthread_spinlock_t *lock);
返回值：
	若成功，返回0；否则，返回错误编号
```



## 3.原子操作

所谓原子操作，就是该操作绝不会在执行完毕前被任何其他任务或事件打断，也就说，它的最小的执行单位，不可能有比它更小的执行单位。因此这里的原子实际是使用了物理学里的物质微粒的概念。

原子操作需要硬件的支持，因此是架构相关的，其API和原子类型的定义都定义在内核源码树的include/asm/atomic.h文件中，它们都使用汇编语言实现，因为C语言并不能实现这样的操作。

原子操作主要用于实现资源计数，很多引用计数(refcnt)就是通过原子操作实现的。

## 4.总结分析

Mutex（互斥锁）：

sleep-waiting类型的锁

与自旋锁相比它需要消耗大量的系统资源来建立锁；随后当线程被阻塞时，线程的调度状态被修改，并且线程被加入等待线程队列；最后当锁可用时，在获取锁之前，线程会被从等待队列取出并更改其调度状态；但是在线程被阻塞期间，它不消耗CPU资源。

互斥锁适用于那些可能会阻塞很长时间的场景。

1、 临界区有IO操作

2 、临界区代码复杂或者循环量大

3 、临界区竞争非常激烈

4、 单核处理器

Spin lock（自旋锁）：

busy-waiting类型的锁

对于自旋锁来说，它只需要消耗很少的资源来建立锁；随后当线程被阻塞时，它就会一直重复检查看锁是否可用了，也就是说当自旋锁处于等待状态时它会一直消耗CPU时间。

自旋锁适用于那些仅需要阻塞很短时间的场景

## 5. 代码实例

```text
/*实现count从0自增到100万*/
#include<stdio.h>
#include<pthread.h>
#include<unistd.h>
#define THREAD_COUNT 10
pthread_mutex_t mutex;
pthread_spinlock_t spinlock;
//原子操作
int inc(int *value,int add)
{
    int old;
	__asm__ volatile(
		"lock; xaddl %2, %1;"
		: "=a" (old)
		: "m" (*value), "a"(add)
		: "cc", "memory"
	);
    return old;
}
void *thread_callback(void *arg)
{
    int *pcount=(int *)arg;
    int i=0;
    while(i++<100000)
    {
    #if 0
        (*pcount)++;
    #elif 0
        pthread_mutex_lock(&mutex);
        (*pcount)++;
        pthread_mutex_unlock(&mutex);
    #elif 0
        pthread_spin_lock(&spinlock);
        (*pcount)++;
        pthread_spin_unlock(&spinlock);
    #else
        inc(pcount,1);
    #endif
        usleep(1);//休眠一微秒
    }
}
int main()
{
    
    pthread_t threadid[THREAD_COUNT]={0};
    pthread_mutex_init(&mutex,NULL);
    //自旋锁
    pthread_spin_init(&spinlock,PTHREAD_PROCESS_SHARED);
    int i=0;
    int count=0;
    for(i=0;i<THREAD_COUNT;i++)
    {
        pthread_create(&threadid[i],NULL,thread_callback,&count);
    }
    for(i=0;i<100;i++)
    {
        printf("count: %d\n",count);
        sleep(1);
    }
    return 0;
    
}
```

原文地址：https://zhuanlan.zhihu.com/p/541703486

作者：linux