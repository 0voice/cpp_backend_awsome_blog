# 【NO.546】互斥锁、读写锁、自旋锁，以及原子操作指令xaddl、cmpxchg的使用场景剖析

## 1.前言

  本文介绍锁与临界资源与原子操作的的使用场景。

  本专栏知识点是通过零声教育的线上课学习，进行梳理总结写下文章，对c/c++linux课程感兴趣的读者，可以点击链接 C/C++后台高级服务器课程介绍 详细查看课程的服务。

## 2.临界资源

### 2.1 什么是临界资源

  临界资源就是被多个线程/进程共享，但在某一时刻只能被一个线程/进程所使用的资源。
  下文以一个经典案例(多线程同时进行i++)介绍三种锁，以及cpu指令集支持的原子操作和CAS。

  主线程启动后创建十个线程，并将主线程中的count变量当作参数传入子线程中，也就是说十个线程同时操作一个共享资源count，子线程执行10w次count++ 操作，主线程每隔两秒打印一次count的值。下面来看看加锁与不加锁的区别。

### 2.2 多线程操作临界资源且不加锁

```
//
// Created by 68725 on 2022/8/3.
//

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/mman.h>

#define THREAD_SIZE     10


//callback
void *func(void *arg) {
    int *pcount = (int *) arg;
    int i = 0;
    while (i++ < 100000) {
        (*pcount)++;
        usleep(1);
    }
}


int main() {
    pthread_t th_id[THREAD_SIZE] = {0};

    int i = 0;
    int count = 0;
    
    for (i = 0; i < THREAD_SIZE; i++) {
        int ret = pthread_create(&th_id[i], NULL, func, &count);
        if (ret) {
            break;
        }
    }
    for (i = 0; i < 100; i++) {
        printf("count --> %d\n", count);
        sleep(2);
    }

}
```


  我们预期count最终是达到100w，为什么在不加锁的时候没有达到预期效果？很明显，count++不是原子操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/89e8bc5c2c284e6e9c4c28a79dcee5b3.png)

### 2.3 i++不是原子操作,i++对应三条汇编指令

  如果i++是原子操作，那么必然会累加到100w，那么i++到底对应着那几步呢？

  下面以idx++举例，idx的值是存储在内存里面，首先从内存MOV到eax寄存器里面，然后通过寄存器进行自增，最后再从eax写回到内存中。在编译器不做任何优化的情况下，idx++就对应这三个步骤。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/4661d51cd58b49b2be658a9da18ded83.png)

在大多数情况下，程序是这样执行的

![在这里插入图片描述](https://img-blog.csdnimg.cn/7b1600f44c3440888d0e08dcc37ba3ac.png)

  但是也会存在下面这两种情况。线程1首先将idx的值读到了寄存器，然后cpu执行线程2，线程2执行完三步骤后，又回到线程1，线程1接着执行剩下的两步。有人可能会想，两个线程不都执行完了吗？有什么不同？

![在这里插入图片描述](https://img-blog.csdnimg.cn/28c4e3f1b92147979a6db1d74b9c4d76.png) 

 首先，在线程1让出后，线程1的上下文(比如这里的eax)，是存储到线程1里面的，线程1恢复后，又将上下文load回去。这里就涉及到yield和resume了，详细介绍看纯c协程框架NtyCo实现与原理的第二节与第三节。 理解了上下文的切换后，就容易理解了，有没有发现，两次++操作，最终会漏加。

![在这里插入图片描述](https://img-blog.csdnimg.cn/cedefbea49504537962be1ead0dd4b7c.png)  

所以在多线程中，操作临界资源时，那么这个临界资源是原子的，那么就不用加锁，要么就必须加锁，否在就会出现上述问题！

  那么所谓加锁是什么意思？就是将这三条汇编指令变成一个原子操作，只要有一个线程lock加锁了，别的线程就不能执行进来，直到加锁的线程解锁，别的线程才能加锁。那么这三条汇编指令就是原子的了。下面将介绍3中锁以及2个原子操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6ef7a59ea5d64c45b90edac6b5b000c9.png)

### 2.4 多线程操作临界资源且加互斥锁

  下面来看看两种加锁方式，这两种都可以跑了100w，但是这两种加锁的粒度是不一样的，在这个程序中，谁是临界资源？是pcount，而不是while，所以第二种加锁虽然可以跑通，但是它加锁的粒度太大了，就本程序而言，第二种加锁方式这和单线程跑有什么区别？所以我们要对临界资源加锁,不是临界资源的不加锁，掌控好锁的粒度

```
//正确加锁
while (i++ < 100000) {
    pthread_mutex_lock(&mutex);
    (*pcount)++;
    pthread_mutex_unlock(&mutex);
}
//错误加锁
pthread_mutex_lock(&mutex);
while (i++ < 100000) {
    (*pcount)++;
}
pthread_mutex_unlock(&mutex);

```



```
//
// Created by 68725 on 2022/8/3.
//

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/mman.h>

#define THREAD_SIZE     10
pthread_mutex_t mutex;


//callback
void *func(void *arg) {
    int *pcount = (int *) arg;
    int i = 0;
    while (i++ < 100000) {
        pthread_mutex_lock(&mutex);
        (*pcount)++;
        pthread_mutex_unlock(&mutex);


        usleep(1);
    }

}


int main() {
    pthread_t th_id[THREAD_SIZE] = {0};
    pthread_mutex_init(&mutex, NULL);

    int i = 0;
    int count = 0;
    
    for (i = 0; i < THREAD_SIZE; i++) {
        int ret = pthread_create(&th_id[i], NULL, func, &count);
        if (ret) {
            break;
        }
    }
    for (i = 0; i < 100; i++) {
        printf("count --> %d\n", count);
        sleep(2);
    }

}
```


  可以看到加锁之后，成功达到我们的预期

![在这里插入图片描述](https://img-blog.csdnimg.cn/452374f996034d739e4bef0d41da1cab.png)

### 2.5 多线程操作临界资源且加读写锁

  读写锁，顾名思义，读临界资源的时候加读锁，写临界资源的时候加写锁。适用于读多写少的场景。

A线程加了读锁，B线程可以继续加读锁，但是不能加写锁。
A线程加了写锁，B线程不能加读锁，也不能加写锁。

```
//
// Created by 68725 on 2022/8/3.
//

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/mman.h>

#define THREAD_SIZE     10
pthread_rwlock_t rwlock;


//callback
void *func(void *arg) {
    int *pcount = (int *) arg;
    int i = 0;
    while (i++ < 100000) {
        pthread_rwlock_wrlock(&rwlock);
        (*pcount)++;
        pthread_rwlock_unlock(&rwlock);


        usleep(1);
    }

}


int main() {
    pthread_t th_id[THREAD_SIZE] = {0};
    pthread_rwlock_init(&rwlock, NULL);


    int i = 0;
    int count = 0;
    
    for (i = 0; i < THREAD_SIZE; i++) {
        int ret = pthread_create(&th_id[i], NULL, func, &count);
        if (ret) {
            break;
        }
    }
    for (i = 0; i < 100; i++) {
        pthread_rwlock_rdlock(&rwlock);
        printf("count --> %d\n", count);
        pthread_rwlock_unlock(&rwlock);
    
        sleep(2);
    }

}



```

![在这里插入图片描述](https://img-blog.csdnimg.cn/89a73d861af543ce8462f5bbea3b7f1b.png)

### 2.6 多线程操作临界资源且加自旋锁

  spinlock与mutex一样，mutex在哪里加锁，spinlock就在哪加锁，使用方法是一样的，但是其内部行为不一样。那么mutex和spinlock的区别在哪呢？

互斥锁在获取不到锁时，会进入休眠，等待释放时被唤醒。会让出CPU。
自旋锁在获取不到锁时，一直等待，在等待过程种不会有进程，线程切换。只会一直等，死等。
  互斥锁与自旋锁的使用场景下文介绍。

```
//
// Created by 68725 on 2022/8/3.
//

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/mman.h>

#define THREAD_SIZE     10
pthread_spinlock_t spinlock;


//callback
void *func(void *arg) {
    int *pcount = (int *) arg;
    int i = 0;
    while (i++ < 100000) {
        pthread_spin_lock(&spinlock);
        (*pcount)++;
        pthread_spin_unlock(&spinlock);

        usleep(1);
    }

}


int main() {
    pthread_t th_id[THREAD_SIZE] = {0};
    pthread_spin_init(&spinlock, PTHREAD_PROCESS_SHARED);


    int i = 0;
    int count = 0;
    
    for (i = 0; i < THREAD_SIZE; i++) {
        int ret = pthread_create(&th_id[i], NULL, func, &count);
        if (ret) {
            break;
        }
    }
    for (i = 0; i < 100; i++) {
        printf("count --> %d\n", count);
    
        sleep(2);
    }

}
```


![在这里插入图片描述](https://img-blog.csdnimg.cn/fa940a2de3df421badc9e2b8fd362763.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/36e70b5ff42f46a380333c76b23351fb.png)

### 2.7 原子操作

  我们发现加锁，都是将i++对应的汇编的三个步骤，变成原子性。那么我们有没有办法直接将i++对应的汇编指令，变成一条指令？可以，我们使用xaddl这条指令。

  Intel X86指令集提供了指令前缀lock⽤于锁定前端串⾏总线FSB，保证了指令执⾏时不会收到其他处理器的⼲扰。


  所谓原子操作，它不是某条具体的指令，它是CPU支持的指令集，都是原子操作。比如说CAS，CAS是原子操作的一种，而不能说原子操作就是CAS。

```
xaddl -----> Inc
xaddl：第二个参数加第一个参数，并把值存储到第一个参数里
//
// Created by 68725 on 2022/8/3.
//

#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/mman.h>

#define THREAD_SIZE     10

int inc(int *value, int add) {
    int old;
    __asm__ volatile (
    "lock; xaddl %2, %1;" 
    : "=a" (old)
    : "m" (*value), "a" (add)
    : "cc", "memory"
    );
    return old;
}

//callback
void *func(void *arg) {
    int *pcount = (int *) arg;
    int i = 0;
    while (i++ < 100000) {
        inc(pcount, 1);

        usleep(1);
    }

}


int main() {
    pthread_t th_id[THREAD_SIZE] = {0};

    int i = 0;
    int count = 0;
    
    for (i = 0; i < THREAD_SIZE; i++) {
        int ret = pthread_create(&th_id[i], NULL, func, &count);
        if (ret) {
            break;
        }
    }
    for (i = 0; i < 100; i++) {
        printf("count --> %d\n", count);
    
        sleep(2);
    }

}
```




![在这里插入图片描述](https://img-blog.csdnimg.cn/41965f2cf3424a12b58c9e381365b26f.png)

cmpxchg-----> CAS
CAS：Compare And Swap，先比较，再赋值，翻译成代码就是下面

```
if(a==b){//Compare
	a=c;//Swap
}
```



  CPU的指令集支持了先比较后赋值的指令，叫cmpxchg。正因为CPU执行了这个指令，它才是原子操作。

```
//  Perform atomic 'compare and swap' operation on the pointer.
//  The pointer is compared to 'cmp' argument and if they are
//  equal, its value is set to 'val'. Old value of the pointer is returned.
inline T *cas (T *cmp_, T *val_)
{
	T *old;
	__asm__ volatile (
		"lock; cmpxchg %2, %3"
		: "=a" (old), "=m" (ptr)
		: "r" (val_), "m" (ptr), "0" (cmp_)
		: "cc");
	return old;
}
```



## 3.三种锁的api介绍

### 3.1 互斥锁 mutex

  有两个特殊的api，pthread_mutex_trylock 尝试加锁，如果没有获取到锁则返回，而不是休眠。pthread_mutex_timedlock 等待一段时间，超时了还没获取倒锁则返回。

```
/* Mutex handling.  */

/* Initialize a mutex.  */
extern int pthread_mutex_init (pthread_mutex_t *__mutex,
			       const pthread_mutexattr_t *__mutexattr)
     __THROW __nonnull ((1));

/* Destroy a mutex.  */
extern int pthread_mutex_destroy (pthread_mutex_t *__mutex)
     __THROW __nonnull ((1));

/* Try locking a mutex.  */
extern int pthread_mutex_trylock (pthread_mutex_t *__mutex)
     __THROWNL __nonnull ((1));

/* Lock a mutex.  */
extern int pthread_mutex_lock (pthread_mutex_t *__mutex)
     __THROWNL __nonnull ((1));

#ifdef __USE_XOPEN2K
/* Wait until lock becomes available, or specified time passes. */
extern int pthread_mutex_timedlock (pthread_mutex_t *__restrict __mutex,
				    const struct timespec *__restrict
				    __abstime) __THROWNL __nonnull ((1, 2));
#endif

/* Unlock a mutex.  */
extern int pthread_mutex_unlock (pthread_mutex_t *__mutex)
     __THROWNL __nonnull ((1));
```



### 3.2 读写锁 rdlock

  读写锁适用于多读少写的情况，否则还是用互斥锁。

A线程加了读锁，B线程可以继续加读锁，但是不能加写锁。
A线程加了写锁，B线程不能加读锁，也不能加写锁。

```
/* Functions for handling read-write locks.  */

/* Initialize read-write lock RWLOCK using attributes ATTR, or use
   the default values if later is NULL.  */
extern int pthread_rwlock_init (pthread_rwlock_t *__restrict __rwlock,
				const pthread_rwlockattr_t *__restrict
				__attr) __THROW __nonnull ((1));

/* Destroy read-write lock RWLOCK.  */
extern int pthread_rwlock_destroy (pthread_rwlock_t *__rwlock)
     __THROW __nonnull ((1));

/* Acquire read lock for RWLOCK.  */
extern int pthread_rwlock_rdlock (pthread_rwlock_t *__rwlock)
     __THROWNL __nonnull ((1));

/* Try to acquire read lock for RWLOCK.  */
extern int pthread_rwlock_tryrdlock (pthread_rwlock_t *__rwlock)
  __THROWNL __nonnull ((1));

# ifdef __USE_XOPEN2K

/* Try to acquire read lock for RWLOCK or return after specfied time.  */
extern int pthread_rwlock_timedrdlock (pthread_rwlock_t *__restrict __rwlock,
				       const struct timespec *__restrict
				       __abstime) __THROWNL __nonnull ((1, 2));

# endif

/* Acquire write lock for RWLOCK.  */
extern int pthread_rwlock_wrlock (pthread_rwlock_t *__rwlock)
     __THROWNL __nonnull ((1));

/* Try to acquire write lock for RWLOCK.  */
extern int pthread_rwlock_trywrlock (pthread_rwlock_t *__rwlock)
     __THROWNL __nonnull ((1));

# ifdef __USE_XOPEN2K

/* Try to acquire write lock for RWLOCK or return after specfied time.  */
extern int pthread_rwlock_timedwrlock (pthread_rwlock_t *__restrict __rwlock,
				       const struct timespec *__restrict
				       __abstime) __THROWNL __nonnull ((1, 2));

# endif

/* Unlock RWLOCK.  */
extern int pthread_rwlock_unlock (pthread_rwlock_t *__rwlock)
     __THROWNL __nonnull ((1));
```





### 3.3 自旋锁 spinlock

  自旋锁最大的特点是，获取不到锁就一直等待，即使CPU时间片用完了也不会发生切换，死等。而上面两种锁不一样，获取不到就会休眠，让出CPU时间片，切换到其他线程或进程执行。

```
/* Functions to handle spinlocks.  */

/* Initialize the spinlock LOCK.  If PSHARED is nonzero the spinlock can
   be shared between different processes.  */
extern int pthread_spin_init (pthread_spinlock_t *__lock, int __pshared)
     __THROW __nonnull ((1));

/* Destroy the spinlock LOCK.  */
extern int pthread_spin_destroy (pthread_spinlock_t *__lock)
     __THROW __nonnull ((1));

/* Wait until spinlock LOCK is retrieved.  */
extern int pthread_spin_lock (pthread_spinlock_t *__lock)
     __THROWNL __nonnull ((1));

/* Try to lock spinlock LOCK.  */
extern int pthread_spin_trylock (pthread_spinlock_t *__lock)
     __THROWNL __nonnull ((1));

/* Release spinlock LOCK.  */
extern int pthread_spin_unlock (pthread_spinlock_t *__lock)
     __THROWNL __nonnull ((1));
```



## 4.三种锁的使用场景

  比如说读一个文件，就使用mutex。而如果是简单的加加减减操作，就是用spinlock。如果系统提供了原子操作的接口，对于i++这种操作来说，用原子操作更合适。

spinlock：临界资源操作简单/没有发生系统调用/持续时间较短（自旋锁就主要用在临界区持锁时间非常短且CPU资源不紧张的情况下，等待时消耗cpu资源较多，自旋锁一般用于多核的服务器。）

mutex：临界资源操作复杂/发生系统调用/持续时间比较长

- 临界区有IO操作
- 临界区代码复杂或者循环量大
- 临界区竞争非常激烈
- 单核处理器

原子操作：使用场景很小，必须需要CPU的指令集支持才行。（原子操作适用于简单的加减等数学运算，属于粒度最小的操作。比如往链表里增加一个结点，可以做出原子操作吗？不行，因为CPU指令集没有同时多个赋值的指令。cas 多线程同时竞争的时候效率并不会特别高，如果互斥锁和自旋锁能满足要求了尽量不要用cas）

## 5.原子操作的接口

## c

  对于gcc、g++编译器来讲，它们提供了⼀组API来做原⼦操作：

```
type __sync_fetch_and_add (type *ptr, type value, ...)
type __sync_fetch_and_sub (type *ptr, type value, ...)
type __sync_fetch_and_or (type *ptr, type value, ...)
type __sync_fetch_and_and (type *ptr, type value, ...)
type __sync_fetch_and_xor (type *ptr, type value, ...)
type __sync_fetch_and_nand (type *ptr, type value, ...)

bool __sync_bool_compare_and_swap (type *ptr, type oldval type newval, ...)
type __sync_val_compare_and_swap (type *ptr, type oldval type newval, ...)
type __sync_lock_test_and_set (type *ptr, type value, ...)
void __sync_lock_release (type *ptr, ...)
```


  详细⽂档⻅：https://gcc.gnu.org/onlinedocs/gcc-4.1.1/gcc/Atomic-Builtins.html#AtomicBuiltins

  详细⽂档⻅：https://gcc.gnu.org/onlinedocs/gcc-4.1.1/gcc/Atomic-Builtins.html#AtomicBuiltins

c++
  对于c++11来讲，也有⼀组atomic的接⼝：https://en.cppreference.com/w/cpp/atomic
————————————————
版权声明：本文为CSDN博主「cheems~」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_42956653/article/details/126141284