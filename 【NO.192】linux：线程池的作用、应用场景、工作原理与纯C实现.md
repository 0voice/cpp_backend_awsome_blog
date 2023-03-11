# 【NO.192】linux：线程池的作用、应用场景、工作原理与纯C实现

## 1.线程池的作用

**为什么会有线程池，到底解决了什么问题**

1. 减少线程的创建与销毁（线程的角度）
2. 异步解耦的作用（设计的角度）

## 2.线程池的异步处理使用场景

以日志为例，在写日志loginfo(“xxx”)，与日志落盘，是两码事，它们两之间应该是异步的。那么异步解耦就是将日志当作一个任务task，将这个任务抛给线程池去处理，由线程池去负责日志落盘。对于应用程序而言，就可以提升落盘的效率。

以nginx为例，一秒几万的请求，速度很快。如果在其中加一个日志，那么qps一下子就掉下来了，因为每请求一次就需要落盘一次，那么整个服务器的性能就下降。我们可以引入一个线程池，把日志这个任务抛给线程池，对于主循环来说，就只抛任务即可，这样就可以大大提升主线程的效率。这就是线程池异步解耦的作用

不仅仅是日志落盘，还有很多地方都可以用线程池，比较耗时的操作如数据库操作，io处理等，都可以用线程池。

线程池有必要将线程与cpu做亲和性吗？ 在注重cpu处理能力的时候，可以做黏合；如果注重的是异步解耦，那么这里更加注重的是任务，没必要将线程和cpu做绑定。

## 3.线程池工作原理

### 3.1 线程池应该提供哪些api

![img](https://pic2.zhimg.com/80/v2-749b503aacab1bc0e5b83e78e867cc51_720w.webp)

我们在使用线程池的时候，是当作一个组件去使用。所以在使用组件的时候，我们首先想到的是线程池应该提供哪些api。

1. 线程池的初始化(创建) init/create
2. 往池里面抛任务push_task
3. 线程池的销毁 deinit/destroy

这三个api是最核心的api，其他可扩展的api都是可有可无的，而这三个api是一定要有的。

### 3.2 线程池的三个组件

![img](https://pic1.zhimg.com/80/v2-ba6fec48ec6d183eeac5299809e27f30_720w.webp)

想象去银行营业厅的场景。柜员：为客户提供服务；客户：是来办业务的，对于柜员来说，这些人就是任务。那么这两个形象就构建出了pthread和task。

那么这个公示牌(xxx号来几号柜台办理业务)，是谁的属性呢？告示牌的作用是管理客户和柜员有秩序工作，它不隶属于柜员，也不隶属于客户，它是一个管理工具。

1. 柜员 ---->pthread
2. 客户 ---->task
3. 告示牌–>管理柜员和客户有秩序的工作（不会出现一个任务同时被多个线程处理的情况）

这么这就自然而然的形成了3个组件，那么它们都应该有什么属性呢：

- 对于柜员来说：工号id，停止工作标识符flag
- 对于客户来说：如果办理取款需要带银行卡，如果办贷款需要带凭证等等，所以需要一个任务func()，以及对应任务的参数arg
- 对于告示牌来说：如果没有客户，那么柜员就需要在工作中等待客户的到来，所以第一个需要条件等待cond，既然要管理有秩序的工作，肯定需要mutex来保证临界资源

下面将柜员称为执行队列，客户称为任务队列，告示牌称为池管理组件。

错误理解：要使用线程就从线程池里面拿一个线程出来使用，用完再返回给线程池。这种理解是连接池的概念。而线程池是多个线程去任务队列取任务，竞争任务。

所以线程的核心就是下面的伪代码：

```text
while(1){
    get_task();
    task->func();
}
```



## 4.代码实现

### 4.1 线程池的任务队列、执行队列、池管理组件 的定义 与 添加删除

![img](https://pic1.zhimg.com/80/v2-419d04da3719cb57884861d825d03244_720w.webp)

```text
//执行队列
typedef struct NWORKER {
    pthread_t id;
    int termination;
    struct NTHREADPOLL *thread_poll;

    struct NWORKER *prev;
    struct NWORKER *next;
} worker_t;

//任务队列
typedef struct NTASK {
    void (*task_func)(void *arg);
    void *user_data;

    struct NTASK *prev;
    struct NTASK *next;
} task_t;

//池管理组件
typedef struct NTHREADPOLL {
    worker_t *workers;
    task_t *tasks;

    pthread_cond_t cond;
    pthread_mutex_t mutex;
} thread_poll_t;
```



```text
//头插法
#define LL_ADD(item, list)do{   \
    item->prev=NULL;            \
    item->next=list;            \
    if(list!=NULL){             \
        list->prev=item;        \
    }                           \
    list=item                   \
}while(0)


#define LL_REMOVE(item, list)do{        \
    if(item->prev!=NULL){               \
        item->prev->next=item->next;    \
    }                                   \
    if(item->next!=NULL){               \
        item->next->prev=item->prev;    \
    }                                   \
    if(list==item){                     \
        list=item->next;                \
    }                                   \
    item->prev=item->next=NULL;         \
}while(0)
```

### 4.2 三个api

创建其实就是创建thread_poll_t结构体，然后按照给定的宏创建线程和worker。

push就是给task队列增加一个任务，然后用signal通知cond。

销毁将所有线程的termination置1，然后广播cond即可。

```text
//return access create thread num;
int thread_poll_create(thread_poll_t *thread_poll, int thread_num) {
    if (thread_num < 1)thread_num = 1;
    memset(thread_poll, 0, sizeof(thread_poll_t));
    //init cond
    pthread_cond_t blank_cond = PTHREAD_COND_INITIALIZER;
    memcpy(&thread_poll->cond, &blank_cond, sizeof(pthread_cond_t));
    //init mutex
    pthread_mutex_t blank_mutex = PTHREAD_MUTEX_INITIALIZER;
    memcpy(&thread_poll->mutex, &blank_mutex, sizeof(pthread_mutex_t));

    // one thread one worker
    int idx = 0;
    for (idx = 0; idx < thread_num; idx++) {
        worker_t *worker = malloc(sizeof(worker_t));
        if (worker == NULL) {
            perror("worker malloc err\n");
            return idx;
        }
        memset(worker, 0, sizeof(worker_t));
        worker->thread_poll = thread_poll;

        int ret = pthread_create(&worker->id, NULL, thread_callback, worker);
        if (ret) {
            perror("pthread_create err\n");
            free(worker);
            return idx;
        }
        LL_ADD(worker, thread_poll->workers);
    }
    return idx;
}

int thread_poll_push_task(thread_poll_t *thread_poll, task_t *task) {
    pthread_mutex_lock(&thread_poll->mutex);
    LL_ADD(task, thread_poll->tasks);
    pthread_cond_signal(&thread_poll->cond);
    pthread_mutex_unlock(&thread_poll->mutex);
}

int thread_destroy(thread_poll_t *thread_poll) {
    worker_t *worker = NULL;
    for (worker = thread_poll->workers; worker != NULL; worker = worker->next) {
        worker->termination = 1;
    }
    pthread_mutex_lock(&thread_poll->mutex);
    pthread_cond_broadcast(&thread_poll->cond);
    pthread_mutex_unlock(&thread_poll->mutex);
}
```

### 4.3 线程的回调函数

线程要做的就是取任务，执行任务。取任务从任务队列里面取。

```text
task_t *get_task(worker_t *worker) {
    while (1) {
        pthread_mutex_lock(&worker->thread_poll->mutex);
        while (worker->thread_poll->workers == NULL) {
            if (worker->termination)break;
            pthread_cond_wait(&worker->thread_poll->cond, &worker->thread_poll->mutex);
        }
        if (worker->termination) {
            pthread_mutex_unlock(&worker->thread_poll->mutex);
            return NULL;
        }
        task_t *task = worker->thread_poll->tasks;
        if (task) {
            LL_REMOVE(task, worker->thread_poll->tasks);
        }
        pthread_mutex_unlock(&worker->thread_poll->mutex);
        if (task != NULL) {
            return task;
        }
    }
};

void *thread_callback(void *arg) {
    worker_t *worker = (worker_t *) arg;
    while (1) {
        task_t *task = get_task(worker);
        if (task == NULL) {
            free(worker);
            pthread_exit("thread termination\n");
        }
        task->task_func(task);
    }
}
```

### 4.4 测试代码

这里我们创建了1000个task，开了10个thread。记住task以及task的参数，由task的func来销毁。

```text
//
// Created by 68725 on 2022/7/25.
//

#include <pthread.h>
#include <memory.h>
#include <malloc.h>
#include <stdio.h>
#include <unistd.h>


//头插法
#define LL_ADD(item, list)do{   \
    item->prev=NULL;            \
    item->next=list;            \
    if(list!=NULL){             \
        list->prev=item;        \
    }                           \
    list=item;                  \
}while(0)

#define LL_REMOVE(item, list)do{        \
    if(item->prev!=NULL){               \
        item->prev->next=item->next;    \
    }                                   \
    if(item->next!=NULL){               \
        item->next->prev=item->prev;    \
    }                                   \
    if(list==item){                     \
        list=item->next;                \
    }                                   \
    item->prev=item->next=NULL;         \
}while(0)

//执行队列
typedef struct NWORKER {
    pthread_t id;
    int termination;
    struct NTHREADPOLL *thread_poll;

    struct NWORKER *prev;
    struct NWORKER *next;
} worker_t;

//任务队列
typedef struct NTASK {
    void (*task_func)(void *arg);

    void *user_data;

    struct NTASK *prev;
    struct NTASK *next;
} task_t;

//池管理组件
typedef struct NTHREADPOLL {
    worker_t *workers;
    task_t *tasks;

    pthread_cond_t cond;
    pthread_mutex_t mutex;
} thread_poll_t;

task_t *get_task(worker_t *worker) {
    while (1) {
        pthread_mutex_lock(&worker->thread_poll->mutex);
        while (worker->thread_poll->workers == NULL) {
            if (worker->termination)break;
            pthread_cond_wait(&worker->thread_poll->cond, &worker->thread_poll->mutex);
        }
        if (worker->termination) {
            pthread_mutex_unlock(&worker->thread_poll->mutex);
            return NULL;
        }
        task_t *task = worker->thread_poll->tasks;
        if (task) {
            LL_REMOVE(task, worker->thread_poll->tasks);
        }
        pthread_mutex_unlock(&worker->thread_poll->mutex);
        if (task != NULL) {
            return task;
        }
    }
};

void *thread_callback(void *arg) {
    worker_t *worker = (worker_t *) arg;
    while (1) {
        task_t *task = get_task(worker);
        if (task == NULL) {
            free(worker);
            pthread_exit("thread termination\n");
        }
        task->task_func(task);
    }
}

//return access create thread num;
int thread_poll_create(thread_poll_t *thread_poll, int thread_num) {
    if (thread_num < 1)thread_num = 1;
    memset(thread_poll, 0, sizeof(thread_poll_t));
    //init cond
    pthread_cond_t blank_cond = PTHREAD_COND_INITIALIZER;
    memcpy(&thread_poll->cond, &blank_cond, sizeof(pthread_cond_t));
    //init mutex
    pthread_mutex_t blank_mutex = PTHREAD_MUTEX_INITIALIZER;
    memcpy(&thread_poll->mutex, &blank_mutex, sizeof(pthread_mutex_t));

    // one thread one worker
    int idx = 0;
    for (idx = 0; idx < thread_num; idx++) {
        worker_t *worker = malloc(sizeof(worker_t));
        if (worker == NULL) {
            perror("worker malloc err\n");
            return idx;
        }
        memset(worker, 0, sizeof(worker_t));
        worker->thread_poll = thread_poll;

        int ret = pthread_create(&worker->id, NULL, thread_callback, worker);
        if (ret) {
            perror("pthread_create err\n");
            free(worker);
            return idx;
        }
        LL_ADD(worker, thread_poll->workers);
    }
    return idx;
}

int thread_poll_push_task(thread_poll_t *thread_poll, task_t *task) {
    pthread_mutex_lock(&thread_poll->mutex);
    LL_ADD(task, thread_poll->tasks);
    pthread_cond_signal(&thread_poll->cond);
    pthread_mutex_unlock(&thread_poll->mutex);
}

int thread_destroy(thread_poll_t *thread_poll) {
    worker_t *worker = NULL;
    for (worker = thread_poll->workers; worker != NULL; worker = worker->next) {
        worker->termination = 1;
    }
    pthread_mutex_lock(&thread_poll->mutex);
    pthread_cond_broadcast(&thread_poll->cond);
    pthread_mutex_unlock(&thread_poll->mutex);
}

void counter(task_t *task) {
    int idx = *(int *) task->user_data;
    printf("idx:%d  pthread_id:%llu\n", idx, pthread_self());
    free(task->user_data);
    free(task);
}

#define THREAD_COUNT 10
#define TASK_COUNT 1000

int main() {
    thread_poll_t thread_poll = {0};
    int ret = thread_poll_create(&thread_poll, THREAD_COUNT);
    if (ret != THREAD_COUNT) {
        thread_destroy(&thread_poll);
    }
    int i = 0;
    for (i = 0; i < TASK_COUNT; i++) {
        //create task
        task_t *task = (task_t *) malloc(sizeof(task_t));
        if (task == NULL) {
            perror("task malloc err\n");
            exit(1);
        }
        task->task_func = counter;
        task->user_data = malloc(sizeof(int));
        *(int *) task->user_data = i;
        //push task
        thread_poll_push_task(&thread_poll, task);
    }
    getchar();
    thread_destroy(&thread_poll);
}
```

## 5.nginx线程池实现对比分祈

### 5.1 线程池初始化对比

cond初始化，mutex初始化，创建线程

![img](https://pic3.zhimg.com/80/v2-cd5ae45c487a4d7d8981aeefb5d3d526_720w.webp)

### 5.2 线程回调函数对比

取任务，执行任务

![img](https://pic3.zhimg.com/80/v2-e140f2483490bcfcade3996c1c5a8f0e_720w.webp)

### 5.3 push任务对比

nginx是将任务插到尾部，我们做的是插到头部

![img](https://pic1.zhimg.com/80/v2-3a018f7208432c888283cf2e5f527184_720w.webp)

### 5.4 线程数量的抉择

线程到底初始化多少呢？如果是计算密集型就不用太多的线程，如果是任务密集型可以多几个。以下是经验值，不一定一定按照这个来。

- 计算密集型：强计算，计算时间较长，线程数量与cpu核心数成比例即可，如1：1。
- 任务密集型：处理任务，io操作。可以开多一点，如cpu核心数的2倍。

### 5.5 线程池的动态扩缩

随着任务越来越多，线程不够用怎么办？我们可以开一个监控线程，设n=running线程 / 总线程。当n>上水位时，监控线程创建几个线程；当n<下水位时，监控线程销毁几个线程。可以设置30%和70%。

原文地址：https://zhuanlan.zhihu.com/p/546291862

作者：linux