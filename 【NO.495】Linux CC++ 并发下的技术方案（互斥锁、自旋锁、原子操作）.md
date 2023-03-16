# 【NO.495】Linux C/C++ 并发下的技术方案（互斥锁、自旋锁、原子操作）

前言

线程：是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。在Unix System V及SunOS中也被称为轻量进程，但轻量进程更多指内核线程，而把用户线程称为线程。

## 1.为什么要使用锁？

下面是一个主线程count计数的方案，分别通过它的10个子线程进行count的自增。

```
mkdir LOCK    //创建LOCK文件夹
cd LOCK        //转到LOCK目录下
touch Lock.c    //创建Lock.c文件
gcc -o lock Lock.c -lpthread//编译的时候需要用pthread动态库
```

然后这里是Lock.c的代码

```
#include<stdio.h>
#include<pthread.h>
#define THREAD_COUNT   10
void* thread_callback(void* arg) {
    int* pcount = (int*)arg;
    int i = 0;
    while (i++ < 1000000) {
        (*pcount)++;
        usleep(1);
    }
}
int main() {
    pthread_t threadid[THREAD_COUNT] = { 0 };
    int i = 0;
    int count = 0;
    for (i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threadid[i], NULL, thread_callback, &count);//创建线程（线程id，线程属性，线程入口函数，主线程往子线程传的参数）
    }
    for (i = 0; i < 100; i++) {
        printf("count: %d\n", count);
        sleep(1);
    }
    getchar();
}
```

因为电脑配置不同，所以CPU的处理不同。

如果每个子线程使count自增10万次

最后count值可能等于100万（或者小于100w），

由于我的电脑每个子线程使count自增100万次，才会小于1000万，所以我以每个子线程自增100万次作为演示。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165028_37903.jpg)

在这里可以发现，明明count应该达到1000万才对，但是由于操作系统进程的切换导致count++这条语句出现了问题。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165028_74288.jpg)

这是count++转化为汇编的语句，分为三条，在正常情况下，线程一执行完这三条语句再执行线程二.

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165029_69604.jpg)

但是异常的情况下，线程一可能三条语句执行不完，就会进行线程二的执行。

导致的结果就是明明应该使count自增2次，但实践上只自增了1次，这样的结果就会导致1000万条数据有所衰减。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165030_54870.jpg)

所以为了避免这种情况的出现，我们可以使用，互斥锁、自旋锁、原子操作等方法解决这个问题。

## 2.解决方法（互斥锁、自旋锁、原子操作）

### 2.1.互斥锁

代码如下（示例）：

```
#include<stdio.h>
#include<pthread.h>
#define THREAD_COUNT   10
pthread_mutex_t mutex;//添加一个互斥锁
void* thread_callback(void* arg) {
    int* pcount = (int*)arg;
    int i = 0;
    while (i++ < 1000000) {
#if 0
#else
        pthread_mutex_lock(&mutex);//线程调用函数让互斥锁上锁
        (*pcount)++;
        pthread_mutex_unlock(&mutex);//解除互斥锁的锁定
#endif
        usleep(1);
    }
}
int main() {
    pthread_t threadid[THREAD_COUNT] = { 0 };
    pthread_mutex_init(&mutex, NULL);//线程初始化
    int i = 0;
    int count = 0;
    for (i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threadid[i], NULL, thread_callback, &count);//创建线程（线程id，线程属性，线程入口函数，主线程往子线程传的参数）
    }
    for (i = 0; i < 100; i++) {
        printf("count: %d\n", count);
        sleep(1);
    }
    getchar();
}
```

### 2.2.自旋锁

代码如下（示例）：

```
#include<stdio.h>
#include<pthread.h>
#define THREAD_COUNT   10
pthread_spinlock_t spinlock;//添加一个自旋锁
void* thread_callback(void* arg) {
    int* pcount = (int*)arg;
    int i = 0;
    while (i++ < 1000000) {
#if 0
#else
        pthread_spin_lock(&spinlock);//线程调用函数
        (*pcount)++;
        pthread_spin_unlock(&spinlock);
#endif
        usleep(1);
    }
}
int main() {
    pthread_t threadid[THREAD_COUNT] = { 0 };
    pthread_spin_init(&spinlock, PTHREAD_PROCESS_SHARED);
    int i = 0;
    int count = 0;
    for (i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threadid[i], NULL, thread_callback, &count);//创建线程（线程id，线程属性，线程入口函数，主线程往子线程传的参数）
    }
    for (i = 0; i < 100; i++) {
        printf("count: %d\n", count);
        sleep(1);
    }
    getchar();
}
```

### 2.3.原子操作

代码如下（示例）：

```
#include<stdio.h>
#include<pthread.h>
#define THREAD_COUNT   10
int inc(int* value, int add) {
    int old;
    __asm__ volatile(        //汇编语句
        "lock; xaddl %2,%1;"    //使 %1的结果为%2+%1
        : "=a" (old)
        : "m" (*value), "a"(add)
        : "cc", "memory"
    );
    return old;
}
void* thread_callback(void* arg) {
    int* pcount = (int*)arg;
    int i = 0;
    while (i++ < 1000000) {
#if 0
#else
        inc(pcount, 1);
#endif
        usleep(1);
    }
}
int main() {
    pthread_t threadid[THREAD_COUNT] = { 0 };
    int i = 0;
    int count = 0;
    for (i = 0; i < THREAD_COUNT; i++) {
        pthread_create(&threadid[i], NULL, thread_callback, &count);//创建线程（线程id，线程属性，线程入口函数，主线程往子线程传的参数）
    }
    for (i = 0; i < 100; i++) {
        printf("count: %d\n", count);
        sleep(1);
    }
    getchar();
}
```

### 2.4.三种方法的比较（个人理解）

自旋锁：线程调用时相当于执行while（1）语句，直到获取锁内内容执行完，才进行下一个线程。

互斥锁：如果当前线程没有执行完，引起线程切换，就会执行下一条线程，当再次回来的时候重新执行。

原子操作：把多条语句合并成一条语句。

自旋锁适合锁的内容很少的时候使用，而互斥锁适合锁的内容较多的时候使用。

## 3.总结

今天通过这个例子，我让大家理解了，主线程和子线程之间可能会发生一些事情，导致程序最后不能达到我们预期的想法，因此就需要通过锁定语句的方法来使程序正常执行。

原文链接：https://zhuanlan.zhihu.com/p/505964220

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)