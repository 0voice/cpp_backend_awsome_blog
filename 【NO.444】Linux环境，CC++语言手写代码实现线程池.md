# 【NO.444】Linux环境，C/C++语言手写代码实现线程池

前言

在我们日常生活中会遇到许许多多的问题，如果一个服务端要接受很多客户端的数据，该怎么办？多线程并发内存不够怎么办？所以我们需要了解线程池的相关知识。

### 1.线程池是什么？

1.线程池的简介

线程池是一种多线程处理形式，处理过程中将任务添加到队列，然后在创建线程后自动启动这些任务。线程池线程都是后台线程。每个线程都使用默认的堆栈大小，以默认的优先级运行，并处于多线程单元中。如果某个线程在托管代码中空闲（如正在等待某个事件）,则线程池将插入另一个辅助线程来使所有处理器保持繁忙。如果所有线程池线程都始终保持繁忙，但队列中包含挂起的工作，则线程池将在一段时间后创建另一个辅助线程但线程的数目永远不会超过最大值。超过最大值的线程可以排队，但他们要等到其他线程完成后才启动。

### 2.线程池的组成

1、线程池管理器（ThreadPoolManager）:用于创建并管理线程池

2、工作线程（WorkThread）: 线程池中线程

3、任务接口（Task）:每个任务必须实现的接口，以供工作线程调度任务的执行。

4、任务队列:用于存放没有处理的任务。提供一种缓冲机制。

### 3.线程池的主要优点

1.避免线程太多，使得内存耗尽

2.避免创建与销毁线程的代价

3.任务与执行分离

### 4.代码实现

1.线程池结构体定义

代码如下（示例）：

```
struct nTask {
    void (*task_func)(struct nTask* task);
    void* user_data;
    struct nTask* prev;
    struct nTask* next;
};
struct nWorker {
    pthread_t threadid;
    int terminate;
    struct nManager* manager;
    struct nWorker* prev;
    struct nWorker* next;
};
typedef struct nManager {
    struct nTask* tasks;
    struct nWorker* workers;
    pthread_mutex_t mutex;//定义互斥锁
    pthread_cond_t cond;//条件变量
}ThreadPool;
```

## 5.接口定义

代码如下（示例）：

```
//API
int nThreadPoolCreate(ThreadPool* pool, int numWorkers) {
    if (pool == NULL) return -1;
    if (numWorkers < 1) numWorkers = 1;
    memset(pool, 0, sizeof(ThreadPool));
    pthread_cond_t blank_cond = PTHREAD_COND_INITIALIZER;
    memcpy(&pool->cond, &blank_cond, sizeof(pthread_cond_t));
    pthread_mutex_t blank_mutex = PTHREAD_MUTEX_INITIALIZER;
    memcpy(&pool->mutex, &blank_mutex, sizeof(pthread_mutex_t));
    int i = 0;
    for (i = 0; i < numWorkers; i++) {
        struct nWorker* worker = (struct nWorker*)malloc(sizeof(struct nWorker));
        if (worker == NULL) {
            perror("malloc");
            return -2;
        }
        memset(worker, 0, sizeof(struct nWorker));
        worker->manager = pool;
        int ret = pthread_create(&worker->threadid, NULL, nThreadPoolCallback, worker);
        if (ret) {
            perror("pthread_create");
            free(worker);
            return -3;
        }
        LIST_INSERT(worker, pool->workers);
    }
    return 0;
}
//API
int nThreadPoolDestory(ThreadPool* pool, int nWorker) {
    struct nWorker* worker = NULL;
    for (worker = pool->workers; worker != NULL; worker = worker->next) {
        worker->terminate;
    }
    pthread_mutex_lock(&pool->mutex);
    pthread_cond_broadcast(&pool->cond);//唤醒所有线程
    pthread_mutex_unlock(&pool->mutex);
    pool->workers = NULL;
    pool->tasks = NULL;
    return 0;
}
//API
int nThreadPoolPushTask(ThreadPool* pool, struct nTask* task) {
    pthread_mutex_lock(&pool->mutex);
    LIST_INSERT(task, pool->tasks);
    pthread_cond_signal(&pool->cond);//唤醒一个线程
    pthread_mutex_unlock(&pool->mutex);
}
```

## 6.回调函数

代码如下（示例）：

```
static void* nThreadPoolCallback(void* arg) {
    struct nWorker* worker = (struct nWorker*)arg;
    while (1)
    {
        pthread_mutex_lock(&worker->manager->mutex);
        while (worker->manager->tasks == NULL) {
            if (worker->terminate)break;
            pthread_cond_wait(&worker->manager->cond, &worker->manager->mutex);
        }
        if (worker->terminate) {
            pthread_mutex_unlock(&worker->manager->mutex);
            break;
        }
        struct nTask* task = worker->manager->tasks;
        LIST_REMOVE(task, worker->manager->tasks);
        pthread_mutex_unlock(&worker->manager->mutex);
        task->task_func(task);
    }
    free(worker);
    return;
}
```

## 7.全部代码（加注释）

代码如下（示例）：

```
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<pthread.h>//Linux编程使用线程相关的就会用到这个，编译的时候还要加-lpthread，用pthread动态库
#define LIST_INSERT(item,list) do{    \            //链表的增加
        item->prev = NULL;            \
        item->next = list;            \
    if((list)!=NULL)(list)->prev=item;    \
        (list) = item;                \
}while(0)
#define LIST_REMOVE(item,list)do{    \            //链表的删除
        if (item->prev != NULL) item->prev->next = item->next;    \
        if (item->next != NULL) item->next->prev = item->prev;    \
        if (item == list) list = item->next;                    \
        item->prev = item->next = NULL;                            \
}while(0)
struct nTask {        //任务
    void (*task_func)(struct nTask* task);
    void* user_data;
    struct nTask* prev;
    struct nTask* next;
};
struct nWorker {    //工作
    pthread_t threadid;
    int terminate;    //用来判断工作是否结束，以便于终止
    struct nManager* manager;    //可以用来查看任务
    struct nWorker* prev;
    struct nWorker* next;
};
typedef struct nManager {    //管理
    struct nTask* tasks;
    struct nWorker* workers;
    pthread_mutex_t mutex;//定义互斥锁，也可以用自旋锁
    pthread_cond_t cond;//条件变量
}ThreadPool;
//callback!=task
static void* nThreadPoolCallback(void* arg) {
    struct nWorker* worker = (struct nWorker*)arg;    //接受pthread_create传来的参数
    while (1)    //默认一直执行
    {
        pthread_mutex_lock(&worker->manager->mutex);//加锁
        while (worker->manager->tasks == NULL) {
            if (worker->terminate)break;
            pthread_cond_wait(&worker->manager->cond, &worker->manager->mutex);//如果没有任务就等待
        }
        if (worker->terminate) {
            pthread_mutex_unlock(&worker->manager->mutex);//解锁
            break;
        }
        struct nTask* task = worker->manager->tasks;
        LIST_REMOVE(task, worker->manager->tasks);
        pthread_mutex_unlock(&worker->manager->mutex);
        task->task_func(task);
    }
    free(worker);
    return;
}
//API
int nThreadPoolCreate(ThreadPool* pool, int numWorkers) {
    if (pool == NULL) return -1;
    if (numWorkers < 1) numWorkers = 1;//如果任务数量小于1，就默认为1
    memset(pool, 0, sizeof(ThreadPool));
    pthread_cond_t blank_cond = PTHREAD_COND_INITIALIZER;//创建互斥量cond，静态全局
    memcpy(&pool->cond, &blank_cond, sizeof(pthread_cond_t));
    //pthread_mutex_init(&pool->mutex, NULL);
    pthread_mutex_t blank_mutex = PTHREAD_MUTEX_INITIALIZER;//创建互斥锁，静态全局
    memcpy(&pool->mutex, &blank_mutex, sizeof(pthread_mutex_t));
    int i = 0;
    for (i = 0; i < numWorkers; i++) {
        struct nWorker* worker = (struct nWorker*)malloc(sizeof(struct nWorker));
        if (worker == NULL) {
            perror("malloc");
            return -2;
        }
        memset(worker, 0, sizeof(struct nWorker));
        worker->manager = pool;
        int ret = pthread_create(&worker->threadid, NULL, nThreadPoolCallback, worker);
        if (ret) {
            perror("pthread_create");
            free(worker);
            return -3;
        }
        LIST_INSERT(worker, pool->workers);
    }
    return 0;
}
//API
int nThreadPoolDestory(ThreadPool* pool, int nWorker) {
    struct nWorker* worker = NULL;
    for (worker = pool->workers; worker != NULL; worker = worker->next) {
        worker->terminate;
    }
    pthread_mutex_lock(&pool->mutex);
    pthread_cond_broadcast(&pool->cond);//唤醒所有线程
    pthread_mutex_unlock(&pool->mutex);
    pool->workers = NULL;//置空
    pool->tasks = NULL;//置空
    return 0;
}
//API
int nThreadPoolPushTask(ThreadPool* pool, struct nTask* task) {
    pthread_mutex_lock(&pool->mutex);
    LIST_INSERT(task, pool->tasks);
    pthread_cond_signal(&pool->cond);//唤醒一个线程
    pthread_mutex_unlock(&pool->mutex);
}
#if 1
#define THREADPOOL_INIT_COUNT    20
#define TASK_INIT_SIZE            1000
void task_entry(struct nTask* task) {
    int idx = *(int*)task->user_data;
    printf("idx: %d\n", idx);
    free(task->user_data);
    free(task);
}
int main(void) {
    ThreadPool pool = { 0 };
    nThreadPoolCreate(&pool, THREADPOOL_INIT_COUNT);
    int i = 0;
    for (i = 0; i < TASK_INIT_SIZE; i++) {
        struct nTask* task = (struct nTask*)malloc(sizeof(struct nTask));
        if (task == NULL) {
            perror("malloc");
            exit(1);
        }
        memset(task, 0, sizeof(struct nTask));
        task->task_func = task_entry;
        task->user_data = malloc(sizeof(int));
        *(int*)task->user_data = i;
        nThreadPoolPushTask(&pool, task);
    }
    getchar();//防止主线程提前结束任务
}
#endif
```

## 8.总结

关于线程池是基本代码就在上面了，关于编程这一部分内容，我建议大家还是要自己去动手实现，如果只是单纯的看了一遍，知识这块可能会记住，但是操作起来可能就比较吃力，万事开头难，只要坚持下去，总会有结果的。

原文链接：https://zhuanlan.zhihu.com/p/490878260

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)