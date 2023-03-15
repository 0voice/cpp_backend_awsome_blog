# 【NO.315】linux c/c++开发：多线程并发锁：互斥锁、自旋锁、原子操作、CAS

## 1.多线程计数



![img](https://pic2.zhimg.com/80/v2-3bf36685131097c0d734fbfdb996c165_720w.webp)



背景：

火车抢票，总共10个窗口，每个窗口都同时进行10w张抢票

可以采用多线程的方式，火车票计数是公共的任务

```text
#include<pthread.h>//posix线程 
#include<stdio.h>
#include<unistd.h>

#define THREAD_COUNT 10  //定义线程数10


//线程入口函数
void* thread_callback(void* arg){ 
    int* pcount=(int*)arg;
    int i=0;
    while(i++<100000){
        (*pcount)++;
        usleep(1);//单位微秒
    }
}

//10个窗口，同时对count进行++操作
int main(){

    pthread_t threadid[THREAD_COUNT]={0};//初始化线程id

    int count=0;
    for(int i=0;i<THREAD_COUNT;i++){//创建10个线程
        //第一个参数：返回线程。 第二个参数：线程的属性（堆栈）。第三个：线程的入口函数。第四个：主线程往子线程传的参数
        pthread_create(&threadid[i],NULL,thread_callback,&count);//count是传入thread_callback内的
    }

    for(int i=0;i<100;i++){
        printf("count: %d\n",count);
        sleep(1);//单位秒
    }
}
```

虽然包含了线程的头文件<pthread.h>，可是编译的时候却报错“对pthread_create未定义的引用“，原来时因为 pthread库不是Linux系统默认的库，连接时需要使用库libpthread.a,所以在使用pthread_create创建线程时，在编译中要加-lpthread参数

可以使用下面方法编译

```text
g++ thread_count.cpp -o thread_count -lpthread
```

执行后，发现，按理说要执行到100w,可是停到99w多就结束了。



![img](https://pic3.zhimg.com/80/v2-0cda2303c652427d1b4f7365b9f83636_720w.webp)



## 2.发现问题

理想状态，线程应该是这样的



![img](https://pic2.zhimg.com/80/v2-e1b1af397983a3b35508f5598dc26881_720w.webp)



但实际上存在，执行完线程1MOV操作后，线程1切换到线程2。导致两个线程的操作，本应该50->52，但是结果确实50->51



![img](https://pic1.zhimg.com/80/v2-b906f842d2827de23bc26d0dd64d27f8_720w.webp)



count是一个临界资源（两个线程共享一个变量），因此为了避免上述这种情况发生，要加锁



## 3.互斥锁

当一个线程在执行一个指令的时候，另一个线程进不来。

相当于把count++转化为汇编的3行命令给打包在一起。

定义互斥锁

```text
pthread_mutex_t mutex;//定义互斥锁
```

初始化互斥锁

```text
pthread_mutex_init(&mutex,NULL);//互斥锁初始化（第二个参数是 锁的属性）
```

加锁/解锁

```text
//加了互斥锁
 pthread_mutex_lock(&mutex);
 (*pcount)++;
 pthread_mutex_unlock(&mutex);
```

完整代码

```text
#include<pthread.h>
#include<stdio.h>
#include<unistd.h>

#define THREAD_COUNT 10

pthread_mutex_t mutex;//定义互斥锁

void* thread_callback(void* arg){
    int* pcount=(int*)arg;
    int i=0;
    while(i++<100000){
#if 0
        (*pcount)++;
#else   
        //加了互斥锁
        pthread_mutex_lock(&mutex);
        (*pcount)++;
        pthread_mutex_unlock(&mutex);
#endif
        usleep(1);//单位微秒

    }
}

//10个窗口，同时对count进行++操作
int main(){

    pthread_t threadid[THREAD_COUNT]={0};//初始化线程id
    pthread_mutex_init(&mutex,NULL);//互斥锁初始化（第二个参数是 锁的属性）

    int count=0;
    for(int i=0;i<THREAD_COUNT;i++){//创建10个线程
        //第一个参数：返回线程。 第二个参数：线程的属性（堆栈）。第三个：线程的入口函数。第四个：主线程往子线程传的参数
        pthread_create(&threadid[i],NULL,thread_callback,&count);//count是传入thread_callback内的
    }

    for(int i=0;i<100;i++){
        printf("count: %d\n",count);
        sleep(1);//单位秒
    }
}
```

## 4.自旋锁

在写法上和互斥锁基本上没有差别

定义自旋锁

```text
pthread_spinlock_t spinlock;//定义自旋锁
```

初始化自旋锁

```text
pthread_spin_init(&spinlock,PTHREAD_PROCESS_SHARED);//自旋锁初始化（第二个参数是 进程共享）
```

自旋锁（加锁/解锁）

```text
//加了自旋锁
pthread_spin_lock(&spinlock);
(*pcount)++;
pthread_spin_unlock(&spinlock);
```

完整代码

```text
#include<pthread.h>
#include<stdio.h>
#include<unistd.h>

#define THREAD_COUNT 10
// pthread_mutex_t mutex;
pthread_spinlock_t spinlock;//定义自旋锁
void* thread_callback(void* arg){
    int* pcount=(int*)arg;
    int i=0;
    while(i++<100000){
#if 0
        (*pcount)++;
#elif 0 
        //加了互斥锁
        pthread_mutex_lock(&mutex);
        (*pcount)++;
        pthread_mutex_unlock(&mutex);
#else 
        //加了自旋锁
        pthread_spin_lock(&spinlock);
        (*pcount)++;
        pthread_spin_unlock(&spinlock);
#endif
        usleep(1);

    }
}

int main(){

    pthread_t threadid[THREAD_COUNT]={0};
    // pthread_mutex_init(&mutex,NULL);
    pthread_spin_init(&spinlock,PTHREAD_PROCESS_SHARED);//自旋锁初始化（第二个参数是 进程共享）
    int count=0;
    for(int i=0;i<THREAD_COUNT;i++){
        pthread_create(&threadid[i],NULL,thread_callback,&count);
    }

    for(int i=0;i<100;i++){
        printf("count: %d\n",count);
        sleep(1);//单位秒
    }
}
```

## 5.互斥锁和自旋锁的对比

使用场景：

当锁的内容很少的时候，继续等待的时间代价比 线程切换的时间代价更小的 时候，选择使用自旋锁（因为互斥锁会切换线程，等待重新调度请求，判断锁是否被占用，如果占用，继续阻塞，并切换到其他线程。如果切换线程的代价比 等待的代价大，可以使用自旋锁。否则使用互斥锁）。

锁的内容比较多的时候，使用互斥锁。（比如，线程安全的红黑树，可以使用mutex）

也就是说：

锁的内容少/没有系统调用，等待的时间代价少-》用自旋锁

锁的内容多，等待时间代价大-》用互斥锁

## 6.原子操作

如果把这三条指令变成一条，那么就不会出现，这种问题了。



![img](https://pic2.zhimg.com/80/v2-9d67d540d39f8546df58047b8cc9082d_720w.webp)



原子操作：单条cpu指令实现（因此使用范围有限，必须要是cpu指令中有的指令）



![img](https://pic3.zhimg.com/80/v2-3bde1236456587c396114c80d8311a3e_720w.webp)



完整代码

```text
#include<pthread.h>
#include<stdio.h>
#include<unistd.h>

#define THREAD_COUNT 10
// pthread_mutex_t mutex;
// pthread_spinlock_t spinlock;


//用一个函数去实现
int increase(int *value,int add){
    int old;
    __asm__ volatile(
        "lock;xaddl %2,%1;"
        :"=a"(old)
        :"m"(*value),"a"(add)
        :"cc","memory"
    );

    return old;
}

void* thread_callback(void* arg){
    int* pcount=(int*)arg;
    int i=0;
    while(i++<100000000){
#if 0
        (*pcount)++;
#elif 0 
        //加了互斥锁
        pthread_mutex_lock(&mutex);
        (*pcount)++;
        pthread_mutex_unlock(&mutex);
#elif 0 
        //加了自旋锁
        pthread_spin_lock(&spinlock);
        (*pcount)++;
        pthread_spin_unlock(&spinlock);
#else
		//原子操作
        increase(pcount,1);
#endif
        usleep(1);

    }
}

int main(){

    pthread_t threadid[THREAD_COUNT]={0};
    // pthread_mutex_init(&mutex,NULL);
    // pthread_spin_init(&spinlock,PTHREAD_PROCESS_SHARED);//自旋锁初始化（第二个参数是 进程共享）
    int count=0;
    for(int i=0;i<THREAD_COUNT;i++){
        pthread_create(&threadid[i],NULL,thread_callback,&count);
    }

    for(int i=0;i<100;i++){
        printf("count: %d\n",count);
        sleep(1);//单位秒
    }
}
```

## 7.其他：CAS

CAS：compare and swap

cpu有这样一条指令cmpxchg(a,b,c)，(其实就是原子操作的原理)。

它的意思是

```text
if(a==b){
	a=c;
}
```

下面例子中，instance就是a，NULL是b，c是malloc(sizeof(object))

```text
if(instance==NULL){
	instance=malloc(sizeof(object));
}
```

使用cas实现求和

```text
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<pthread.h>


#define CAS(a_ptr, a_oldVal, a_newVal) __sync_bool_compare_and_swap(a_ptr, a_oldVal, a_newVal)
// #define AtomicAdd(a_ptr,a_count) __sync_fetch_and_add (a_ptr, a_count)
#define THREAD_COUNT 10


void* callback(void* arg){
    int* pcount=(int*)arg;
    for(int i=0;i<100000;i++){
        while(true){
            int current=(*pcount);
            if(CAS(pcount,current,current+1)) break;
        }
        usleep(1);
    }
    return (void*)nullptr;
}

int main(int argc,char** argv){
    pthread_t threadid[THREAD_COUNT]={0};
    int count=0;
    for(int i=0;i<THREAD_COUNT;i++){//创建10个线程
        pthread_create(&threadid[i],NULL,callback,&count);
    }
    for(int i=0;i<100;i++){
        printf("count: %d\n",count);
        sleep(1);//单位秒
    }
}
```

原文地址：https://zhuanlan.zhihu.com/p/527010381

作者：linux