# 【NO.329】如何用300行代码实现一个完整的线程池

开源项目Workflow中有个重要的基础模块：

**代码仅300行的C语言线程池**。

本文会伴随源码分析，而**逻辑完备**、**对称无差别**的特点于第3部分开始

欢迎跳阅， 或直接到Github主页上围观代码

**[https://github.com/sogou/workflow/blob/master/src/kernel/thrdpool.c](https://link.zhihu.com/?target=https%3A//github.com/sogou/workflow/blob/master/src/kernel/thrdpool.c)**

## **1.Workflow的thrdpool**

Workflow的大招：计算通信融为一体的异步调度模式，而计算的核心：Executor调度器，就是基于这个线程池实现的。可以说，一个通用而高效的线程池，是我们写C/C++代码时离不开的基础模块。

**thrdpool**代码位置在**src/kernel/**，不仅可以直接拿来使用，同时也适合阅读学习。

而更重要的，秉承Workflow项目本身一贯的严谨极简的作风，这个thrdpool代码极致简洁，**实现逻辑上亦非常完备，结构精巧，处处严谨，**复杂的并发处理依然可以**对称无差别，**不得不让我惊叹：

妙啊！！！

你可能会很好奇，线程池还能写出什么别致的新思路吗？先列出一些，你们细品：

- `特点1`：创建完线程池后，无需记录任何线程id或对象，线程池可以通过**一个等一个的方式优雅地去结束**所有线程；
  → 也就是说，每一个**线程**都是对等的
- `特点2`：**线程任务可以由另一个线程任务调起**；甚至线程池正在**被销毁时也可以提交下一个任务**；（这很重要，因为线程本身很可能是不知道线程池的状态的；
  → 即，每一个**任务**也是对等的
- `特点3`：同理，**线程任务也可以销毁这个线程池**；（非常完整～
  → 每一种**行为**也是对等的，包括**destroy**

我真的迫不及待为大家深层解读一下，这个我愿称之为**“逻辑完备”的线程池**。

## **2.前置知识**

第一部分我先从最基本的内容梳理一些个人理解，有基础的小伙伴可以直接跳过。如果有不准确的地方，欢迎大家指正交流～

为什么需要线程池？（其实思路不仅对线程池，对任何有限资源的调度管理都是类似的）

我们知道，**通过pthread**或者**std::thread**创建线程，就可以实现多线程并发执行我们的代码。

但是CPU的核数是固定的，所以真正并行执行的最大值也是固定的，**过多的线程除了频繁创建产生overhead以外，还会导致对系统资源进行争抢**，这些都是不必要的浪费。

因此我们可以管理有限个线程，**循环且合理地利用它们**。

那么线程池一般包含哪些内容呢？

- 首先是管理若干个工具人线程；
- 其次是管理交给线程去执行的任务，这个一般会有一个队列；
- 再然后线程之间需要一些同步机制，比如mutex、condition等；
- 最后就是各线程池实现上自身需要的其他内容了；

接下来我们看看**Workflow**的**thrdpool**是怎么做的。

## **3.代码概览**

以下共7步常用思路，足以让我们把代码飞快过一遍。

**第1步：先看头文件，有什么接口。**

我们打开`thrdpool.h`，只需关注这三个：

```text
// 创建线程池
thrdpool_t *thrdpool_create(size_t nthreads, size_t stacksize);
// 把任务交给线程池的入口
int thrdpool_schedule(const struct thrdpool_task *task, thrdpool_t *pool); 
// 销毁线程池
void thrdpool_destroy(void (*pending)(const struct thrdpool_task *), thrdpool_t *pool);
```

**第2步：接口上有什么数据结构。**

即，我们如何描述一个交给线程池的任务。

```text
struct thrdpool_task
{                                                                    
    void (*routine)(void *);  // 函数指针
    void *context;            // 上下文
};  
```

**第3步：再看实现.c，内部数据结构。**

```text
struct __thrdpool
{
    struct list_head task_queue;// 任务队列
    size_t nthreads;  // 线程个数
    size_t stacksize; // 构造线程时的参数
    pthread_t tid;    // 运行期间记录的是个zero值
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    pthread_key_t key;
    pthread_cond_t *terminate;
};
```

**没有一个多余，每一个成员都很到位：**

1. **tid**：线程id，**整个线程池只有一个**，它不会奇怪地去记录任何一个线程的id，这样就不完美了 ，它平时运行的时候是**空值**，退出的时候，它是用来**实现链式等待的关键**。
2. **mutex** 和 **cond**是常见的线程间同步的工具，其中这个cond是用来给**生产者和消费者**去操作任务队列用的。
3. **key**：是线程池的key，然后会赋予给每个由线程池创建的线程作为他们的thread local，**用于区分这个线程是否是线程池创建的**。
4. 一个**pthread_cond_t \*terminate**，这有两个用途：不仅是退出时的标记位 ，而且还是调用退出的那个人要等待的condition。

以上各个成员的用途，好像说了，又好像没说，是因为**几乎每一个成员都值得深挖一下**，所以我们记住它们，后面看代码的时候就会豁然开朗！

**第4步：接口都调用了什么核心函数。**

```text
thrdpool_t *thrdpool_create(size_t nthreads, size_t stacksize)
{
    thrdpool_t *pool;
    ret = pthread_key_create(&pool->key, NULL);
    if (ret == 0)
    {
        // 去掉了其他代码，但是注意到刚才的tid和terminate的赋值
        memset(&pool->tid, 0, sizeof (pthread_t));
        pool->terminate = NULL;
        if (__thrdpool_create_threads(nthreads, pool) >= 0)
            return pool;
        ...
```

这里可以看到`__thrdpool_create_threads()`里边最关键的，就是循环创建**nthreads**个线程。

```text
        while (pool->nthreads < nthreads)                                       
        {                                                                       
            ret = pthread_create(&tid, &attr, __thrdpool_routine, pool);
            ...
```

**第5步：略读核心函数的功能。**

所以我们在上一步知道了，每个线程执行的是`__thrdpool_routine()`不难想象，它会**不停从队列拿任务出来执行**：

```text
static void *__thrdpool_routine(void *arg)                                      
{                                                                               
    ...                                   
    while (1)                                                                   
    {
        // 1. 从队列里拿一个任务出来，没有就等待
        pthread_mutex_lock(&pool->mutex);
        while (!pool->terminate && list_empty(&pool->task_queue))
            pthread_cond_wait(&pool->cond, &pool->mutex);
        // 2. 线程池结束的标志位，记住它，先跳过
        if (pool->terminate) 
            break;

        // 3. 如果能走到这里，恭喜你，拿到了任务～ 
        entry = list_entry(*pos, struct __thrdpool_task_entry, list);
        list_del(*pos);
        // 4. 先解锁
        pthread_mutex_unlock(&pool->mutex); 

        task_routine = entry->task.routine;
        task_context = entry->task.context;
        free(entry); 
        // 5. 再执行
        task_routine(task_context); 

        // 6. 这里也先记住它，意思是线程池里的线程可以销毁线程池
        if (pool->nthreads == 0) 
        {                                                                       
            /* Thread pool was destroyed by the task. */
            free(pool);
            return NULL;
        }                                                                       
    }
    ... // 后面还有魔法，留下一章解读~~~
```

**第6步：把函数之间的关系联系起来。**

刚才看到的`__thrdpool_routine()`就是线程的核心函数了，它可以和谁关联起来呢？可以和接口`thrdpool_schedule()`关联上

我们说过，线程池上有个队列管理任务：

- 每个执行**routine**的线程，都是消费者
- 每个发起**schedule**的线程，都是生产者

我们已经看过消费者了，来看看生产者的代码：

```text
inline void __thrdpool_schedule(const struct thrdpool_task *task, void *buf,
                                thrdpool_t *pool)
{
    struct __thrdpool_task_entry *entry = (struct __thrdpool_task_entry *)buf;  

    entry->task = *task;
    pthread_mutex_lock(&pool->mutex);
    // 添加到队列里
    list_add_tail(&entry->list, &pool->task_queue);
    // 叫醒在等待的线程
    pthread_cond_signal(&pool->cond);
    pthread_mutex_unlock(&pool->mutex);
}
```

说到这里，`特点2`就非常清晰了：开篇说的`特点2`是说，”**线程任务可以由另一个线程任务调起**”。

只要对队列的管理做得好，显然消费者所执行的函数里也可以生产

**第7步：看其他情况的处理**

对于线程池来说就是比如销毁的情况。

接口`thrdpool_destroy()`实现非常简单：

```text
void thrdpool_destroy(void (*pending)(const struct thrdpool_task *),            
                      thrdpool_t *pool)                                         
{
    ...
    // 1. 内部会设置pool->terminate，并叫醒所有等在队列拿任务的线程
    __thrdpool_terminate(in_pool, pool);

    // 2. 把队列里还没有执行的任务都拿出来，通过pending返回给用户
    list_for_each_safe(pos, tmp, &pool->task_queue)                             
    {
        entry = list_entry(pos, struct __thrdpool_task_entry, list);            
        list_del(pos);                                                          
        if (pending)                                                            
            pending(&entry->task);                                              
        ... // 后面就是销毁各种内存，同样有魔法~
```

在退出的时候，我们那些**已经提交但是还没有被执行的任务**是绝对不能就这么扔掉了的，于是我们可以传入一个`pending()`函数，**上层可以做自己的回收、回调、或任何保证上层逻辑完备的事情。**

**设计的完整性，无处不在。**

接下来我们就可以跟着我们的核心问题，针对性地看看每个特点都是怎么实现的。

## **4.特点1: 一个等待一个的优雅退出**

这里提出一个问题：**线程池要退出，如何结束所有线程？**

一般线程池的实现都是需要记录下所有的线程id，或者thread对象，以便于我们去用**join**方法等待它们结束。

不严格地用join收拾干净会有什么问题？最直观的，模块退出时很可能会报**内存泄漏**

但是我们刚才看，**pool里并没有记录所有的tid呀**？正如开篇说的，**pool上只有一个tid，而且还是个空的值**。

而`特点1`正是**Workflow thrdpool**的答案：

**无需记录所有线程，我可以让线程挨个自动退出、且一个等待下一个，最终达到调用完thrdpool_destroy()后内存回收干净的目的。**

这里先给一个简单的图，假设发起destroy的人是main线程，我们如何做到一个等一个退出：

![img](https://pic2.zhimg.com/80/v2-67d125eef067cfc14bba9f09c4b05285_720w.webp)

外部线程，比如main，发起destroy

步骤如下：

1. 线程的退出，由thrdpool_destroy()设置**pool->terminate**开始。
2. 我们每个线程，在**while(1)** 里会第一时间发现terminate，线程池要退出了，然后会**break**出这个while循环。
3. 注意这个时候，**还持有着mutex锁**，我们拿出pool上唯一的那个tid，放到我的临时变量，我会根据拿出来的值做不同的处理。**且我会把我自己的tid放上去**，然后再解mutex锁。
4. 那么很显然，**第一个从pool上拿tid的人，会发现这是个0值，就可以直接结束了**，不用负责等待任何其他人，但我在完全结束之前需要有人负责等待我的结束，所以我会把我的id放上去。
5. 而如果发现自己从pool里拿到的tid不是0值，**说明我要负责join上一个人**，并且把我的tid放上去，**让下一个人负责我**。
6. 最后的那个人，是**那个发现pool->nthreads为0的人，那么我就可以通过这个terminate**（它本身是个condition）去通知发起destroy的人。
7. 最后发起者就可以退了。

是不是非常**有意思**！！！非常**优雅**的做法！！！

所以我们会发现，其实**大家不太需要知道太多信息，只需要知道我要负责的上一个人**。

当然每一步都是非常严谨的，结合刚才跳过的第一段魔法 感受一下：

```text
static void *__thrdpool_routine(void *arg)                                         
{
    while (1)
    {   // 1.注意这里还持有锁
        pthread_mutex_lock(&pool->mutex); 
        ... // 等着队列拿任务出来
        // 2. 这既是标识位，也是发起销毁的那个人所等待的condition
        if (pool->terminate) 
            break;
        ... // 执行拿到的任务
    }

    /* One thread joins another. Don't need to keep all thread IDs. */
    // 3. 把线程池上记录的那个tid拿下来，我来负责上一人
    tid = pool->tid;
    // 4. 把我自己记录到线程池上，下一个人来负责我
    pool->tid = pthread_self();
    // 5. 每个人都减1，最后一个人负责叫醒发起detroy的人
    if (--pool->nthreads == 0) 
        pthread_cond_signal(pool->terminate);
        
    // 6. 这里可以解锁进行等待了
    pthread_mutex_unlock(&pool->mutex);
    // 7. 只有第一个人拿到0值
    if (memcmp(&tid, &__zero_tid, sizeof (pthread_t)) != 0) 
        // 8. 只要不0值，我就要负责等上一个结束才能退
        pthread_join(tid, NULL); 
                                                                                
    return NULL; // 9. 退出，干干净净～
}
```

## **5.特点2：线程任务可以由另一个线程任务调起**

在第二部分我们看过源码，只要队列管理得好，线程任务里提交下一个任务是完全OK的。

这很合理。

那么问题来了，`特点1`又说，我们每个线程，**是不需要知道太多线程池的状态和信息的**。而线程池的销毁是个过程，如果在这个过程间提交任务会怎么样呢？

因此`特点2`的一个重要解读是：**线程池被销毁时也可以提交下一个任务**。而且刚才提过，还没有被执行的任务，可以通过我们传入的pending()函数拿回来。

简单看看销毁时的严谨做法：

```text
static void __thrdpool_terminate(int in_pool, thrdpool_t *pool)                 
{                                                                          
    pthread_cond_t term = PTHREAD_COND_INITIALIZER;
    pthread_mutex_lock(&pool->mutex);
    // 1. 加锁设置标志位，之后的添加任务不会被执行，但可以pending拿到
    pool->terminate = &term;
    // 2. 广播所有等待的消费者
    pthread_cond_broadcast(&pool->cond); 
                                                                            
    if (in_pool) // 3. 这里的魔法等下讲>_<~
    {                                                                           
        /* Thread pool destroyed in a pool thread is legal. */                  
        pthread_detach(pthread_self());                                         
        pool->nthreads--;                                                       
    }                                                                           
    // 4. 如果还有线程没有退完，我会等，注意这里是while
    while (pool->nthreads > 0) 
        pthread_cond_wait(&term, &pool->mutex);                                 

    pthread_mutex_unlock(&pool->mutex);
    
    // 5.同样地等待打算退出的上一个人                
    if (memcmp(&pool->tid, &__zero_tid, sizeof (pthread_t)) != 0)               
        pthread_join(pool->tid, NULL); 
}
```

## **6.特点3：同样可以在线程任务里销毁这个线程池**

既然线程任务可以做任何事情，理论上，**线程任务也可以销毁线程池**❓

作为一个逻辑完备的线程池，大胆一点，我们把问号去掉。

而且，**销毁并不会结束当前任务，**
**它会等这个任务执行完**。

想象一下，刚才的**__thrdpool_routine()**，**while(1)**里拿出来的那个任务，做的事情竟然是发起**thrdpool_destroy().**..

把上面的图大胆改一下：

![img](https://pic2.zhimg.com/80/v2-f92d5821fa9beb009af7b471edd865ad_720w.webp)

我们让一个routine来destroy线程池

如果发起销毁的人，**是我们自己内部的线程**，那么我们就不是等**n**个，而是等**n-1**，少了一个外部线程等待我们。如何实现才能让这些逻辑都完美融合呢？我们把刚才跳过的**三段魔法串起来**看看。

**第一段魔法，销毁的发起者。**

如果发起销毁的人是线程池内部的线程，那么它具有较强的自我管理意识

（因为前面说了，会等它这个任务执行完）而我们可以放心大胆地**pthread_detach**，无需任何人join它等待它结束。

```text
static void __thrdpool_terminate(int in_pool, thrdpool_t *pool)                 
{ 
    ...
    // 每个由线程池创建的线程都设置了一个key，由此判断是否是in_pool
    if (in_pool) 
    {                                                                           
        /* Thread pool destroyed in a pool thread is legal. */                  
        pthread_detach(pthread_self());
        pool->nthreads--;
    }        
```

**第二段魔法：线程池谁来free？**

一定是发起销毁的那个人。所以这里用**in_pool**来判断是否是外部的人：

```text
void thrdpool_destroy(void (*pending)(const struct thrdpool_task *),
                                       thrdpool_t *pool)
{
    // 已经调用完第一段，且挨个pending(未执行的task)了
    // 销毁其他内部分配的内存
    ...
    // 如果不是内部线程发起的销毁，要负责回收线程池内存
    if (!in_pool) 
        free(pool);
}
```

**那现在不是main线程发起的销毁呢**？发起的销毁的那个内部线程，怎么能保证我可以在最后关头**把所有资源回收干净、调free(pool)、最后功成身退呢**？

在前面阅读源码第5步，其实我们看过，**__thrdpool_routine()里有free的地方。**

于是现在三段魔法终于串起来了。

**第三段魔法：严谨的并发。**

```text
static void *__thrdpool_routine(void *arg)
{
    while (1) 
    {   // ...
        task_routine(task_context); // 如果routine里做的事情，是销毁线程池...
        // 注意这个时候，其他内存都已经被destroy里清掉了，万万不可以再用什么mutex、cond
        if (pool->nthreads == 0) 
        {
            /* Thread pool was destroyed by the task. */
            free(pool);
            return NULL;                                            
        }
        ...
```

非常重要的一点，**由于并发，我们是不知道谁先操作的。假设我们稍微改一改这个顺序，就又是另一番逻辑**。

比如我作为一个内部线程，在**routine()**里调用**destroy()**期间，发现还有线程没有执行完，我就要等在我的terminate上，待最后看到**nthreads==0**的那个人叫醒我。

然后，我的代码继续执行，函数栈就会从**destroy()**回到**routine()**，也就是上面那几行，再然后就可以**free(pool)**;由于这时候我已经放飞自我detach了，于是一切顺利结束。

你看，无论如何都可以完美地销毁线程池：

![img](https://pic3.zhimg.com/80/v2-91d43dbfbeff3147ac7554b54cd102e6_720w.webp)

并发是复杂多变的，代码是简洁统一的

是不是**太妙了**！我写到这里已经要感动哭了！

## **7. 简单的用法**

这个线程池只有两个文件:`thrdpool.h` 和 `thrdpool.c`，而且只依赖内核的数据结构`list.h`。我们把它拿出来玩，自己写一段代码：

```text
void my_routine(void *context)                                                   
{
   // 我们要执行的函数                    
    printf("task-%llu start.\n", reinterpret_cast<unsigned long long>(context); );
}                                                                               
                                                                                
void my_pending(const struct thrdpool_task *task) 
{
  // 线程池销毁后，没执行的任务会到这里
    printf("pending task-%llu.\n", reinterpret_cast<unsigned long long>(task->context););                                    
} 

int main()                                                                         
{
    thrdpool_t *thrd_pool = thrdpool_create(3, 1024); // 创建                            
    struct thrdpool_task task;
    unsigned long long i;
                               
    for (i = 0; i < 5; i++)
    {
        task.routine = &my_routine;                                             
        task.context = reinterpret_cast<void *>(i);                             
        thrdpool_schedule(&task, thrd_pool); // 调用
    }
    getchar(); // 卡住主线程，按回车继续
    thrdpool_destroy(&my_pending, thrd_pool); // 结束
    return 0;                                                                   
} 
```

再打印几行log，直接编译就可以跑起来：

![img](https://pic1.zhimg.com/80/v2-4f67da756812009525d50964401e67c8_720w.webp)

妈妈再也不用担心我的C语言作业

简单程度堪比大一上学期C语言作业。

## **8.并发与结构之美**

最后谈谈感受。

看完之后我很后悔为什么没有早点看为什么不早点就可以获得知识的感觉，并且在浅层看懂之际，我知道自己肯定没有完全理解到里边的精髓，毕竟我不能**深刻地理解到设计者当时对并发的构思和模型上的选择**。

我只能说，没有十多年**顶级的系统调用和并发编程的功底**写不出这样的代码，没有**极致的审美与对品控的偏执**也写不出这样的代码。

**并发编程**有很多说道，就正如退出这个这么简单的事情，想要做到退出时回收干净却很难。如果说你写业务逻辑自己管线程，退出什么的sleep(1)都无所谓，**但如果说做框架的人不能把自己的框架做得完美无暇逻辑自洽**
**就难免让人感觉差点意思**。

而这个thrdpool，它作为一个线程池，是如此的**逻辑完备**，**用最对称简洁的方式去面对复杂的并发**。

**再次让我深深地感到震撼：我们身边那些原始的、底层的、基础的代码，还有很多新思路，还可以写得如此美。**

Workflow项目源码地址**：**
**[https://github.com/sogou/workflow](https://link.zhihu.com/?target=https%3A//github.com/sogou/workflow)**

原文地址：https://zhuanlan.zhihu.com/p/515224253

作者：linux