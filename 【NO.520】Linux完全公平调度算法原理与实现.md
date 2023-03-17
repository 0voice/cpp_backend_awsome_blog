# 【NO.520】Linux完全公平调度算法原理与实现

Linux 进程调度算法经历了以下几个版本的发展：

- 基于时间片轮询调度算法。(2.6之前的版本)
- O(1) 调度算法。(2.6.23之前的版本)
- 完全公平调度算法。(2.6.23以及之后的版本)

这篇主要分析 Linux 现在所使用的 完全公平调度算法。分析完全公平调度算法 前，我们先了解下 完全公平调度算法 的基本原理。

## 1.完全公平调度算法基本原理

完全公平调度算法 体现在对待每个进程都是公平的，那么怎么才能做到完全公平呢？有一个比较简单的方法就是：让每个进程都运行一段相同的时间片，这就是 基于时间片轮询调度算法。如下图所示：

![img](https://pic2.zhimg.com/80/v2-6ac0c3c0f89cb744d2cb85d83d9c64cd_720w.webp)

如上图所示，开始时进程1获得 time0 ~ time1 的CPU运行时间。当进程1的时间片使用完后，进程2获得 time1 ~ time2 的CPU运行时间。而当进程2的时间片使用完后，进程3获得 time2 ~ time3 的CPU运行时间。

如此类推，由于每个时间片都是相等的，所以理论上每个进程都能获得相同的CPU运行时间。这个算法看起来很不错，但存在两个问题：

- 不能按权重分配不同的运行时间，例如有些进程权重大的应该获得更多的运行时间。
- 每次调度时都需要遍历运行队列中的所有进程，找到优先级最大的进程运行，时间复杂度为 O(n)。对于一个高性能的操作系统来说，这是不能接受的。

为了解决上面两个问题，Linux内核的开发者创造了完全公平调度算法。

**完全公平调度的两个时间**

为了实现 完全公平调度算法，为进程定义两个时间:

**1、实际运行时间：**

> 实际运行时间 = 调度周期 * 进程权重 / 所有进程权重之和

- 调度周期：是指所有可进程运行一遍所需要的时间。
- 进程权重：依据进程的重要性，分配给每个进程不同的权重。

例如，调度周期为 30ms，进程A的权重为 1，而进程B的权重为 2。那么：

进程A的实际运行时间为：30ms * 1 / (1 + 2) = 10ms

进程B的实际运行时间为：30ms * 2 / (1 + 2) = 20ms

**2、虚拟运行时间：**

> 虚拟运行时间 = 实际运行时间 * 1024 / 进程权重 = (调度周期 * 进程权重 / 所有进程权重之和) * 1024 / 进程权重

= 调度周期 * 1024 / 所有进程总权重

从上面的公式可以看出，在一个调度周期里，所有进程的 虚拟运行时间 是相同的。所以在进程调度时，只需要找到 虚拟运行时间 最小的进程调度运行即可。

为了能够快速找到 虚拟运行时间 最小的进程，Linux 内核使用 红黑树 来保存可运行的进程，而比较的键值就是进程的 虚拟运行时间。

如果不了解 红黑树 的话，可以把它看成一个自动排序的容器即可。如下图所示：

![img](https://pic1.zhimg.com/80/v2-8e2ddac2a1783bce76975c6495b06b04_720w.webp)

如上图所示，红黑树 的左节点比父节点小，而右节点比父节点大。所以查找最小节点时，只需要获取 红黑树 的最左节点即可，时间复杂度为 O(logN)。

## 2.完全公平调度的两个对象

Linux 内核为了实现 完全公平调度算法，定义两个对象：cfs_rq (可运行进程队列) 和 sched_entity (调度实体)。

- cfs_rq (可运行进程队列)：使用 红黑树 结构来保存内核中可以运行的进程。
- sched_entity (调度实体)：可被内核调度的实体，如果忽略组调度（本文也不涉及组调度），可以把它当成是进程。

cfs_rq 对象定义

```text
struct cfs_rq {
    struct load_weight load;
    unsigned long nr_running;       // 运行队列中的进程数
    u64 exec_clock;                 // 当前时钟
    u64 min_vruntime;               // 用于修证虚拟运行时间
    struct rb_root tasks_timeline; // 红黑树的根节点
    struct rb_node *rb_leftmost;   // 缓存红黑树最左端节点, 用于快速获取最小节点
    ...
};
```

对于 cfs_rq 对象，现在主要关注的是 tasks_timeline 成员，其保存了可运行进程队列的根节点（红黑树的根节点）。

sched_entity 对象定义

```text
struct sched_entity {
  struct load_weight  load;
  struct rb_node    run_node;         // 用于连接到运行队列的红黑树中
  struct list_head  group_node;
  unsigned int    on_rq;              // 是否已经在运行队列中
  u64          exec_start;            // 开始统计运行时间的时间点
  u64          sum_exec_runtime;      // 总共运行的实际时间
  u64          vruntime;              // 虚拟运行时间(用于红黑树的键值对比)
  u64          prev_sum_exec_runtime; // 总共运行的虚拟运行时间
  u64          last_wakeup;
  u64          avg_overlap;
    ...
};
```

对于 sched_entity 对象，现在主要关注的是 run_node 成员，其主要用于把调度实体连接到可运行进程的队列中。

另外，进程描述符 task_struct 对象中，定义了一个类型为 sched_entity 的成员变量 se，如下：

```text
struct task_struct {
    ...
    struct sched_entity se;
    ...
}
```

所以，他们之间的关系如下图：

![img](https://pic2.zhimg.com/80/v2-82faf3d00ec1cff5987296f4b7ac0a89_720w.webp)

从上图可以看出，所有 sched_entity 对象通过其 run_node 成员组成了一颗红黑树，这颗红黑树的根结点保存在 cfs_rq 对象的 task_timeline 成员中。

另外，进程描述符 task_sturct 定义了一个类型为 sched_entity 的成员变量 se，所以通过进程描述符的 se 字段就可以把进程保存到可运行队列中。

## 3.完全公平调度算法实现

有了上面的基础，现在可以开始分析 Linux 内核中怎么实现 完全公平调度算法 了。

我们先来看看怎么更新一个进程的虚拟运行时间。

**1. 更新进程虚拟运行时间**

更新一个进程的虚拟运行时间是通过 __update_curr() 函数完成的，其代码如下：

/src/kernel/sched_fair.c

```text
static inline void
__update_curr(struct cfs_rq *cfs_rq, struct sched_entity *curr,
              unsigned long delta_exec)
{
    unsigned long delta_exec_weighted;
    curr->sum_exec_runtime += delta_exec; // 增加进程总实际运行的时间
    delta_exec_weighted = delta_exec;     // 初始化进程使用的虚拟运行时间
    // 根据进程的权重计算其使用的虚拟运行时间
    if (unlikely(curr->load.weight != NICE_0_LOAD)) {
        delta_exec_weighted = calc_delta_fair(delta_exec_weighted, &curr->load);
    }
    curr->vruntime += delta_exec_weighted; // 更新进程的虚拟运行时间
}
```

__update_curr() 函数各个参数意义如下：

- cfs_rq：可运行队列对象。
- curr：当前进程调度实体。
- delta_exec：实际运行的时间。

__update_curr() 函数主要完成以下几个工作：

1. 更新进程调度实体的总实际运行时间。
2. 根据进程调度实体的权重值，计算其使用的虚拟运行时间。
3. 把计算虚拟运行时间的结果添加到进程调度实体的 vruntime 字段。

我们接着分析怎么把进程添加到运行队列中。

**2. 把进程调度实体添加到运行队列中**

要将进程调度实体添加到运行队列中，可以调用 __enqueue_entity() 函数，其实现如下：

/src/kernel/sched_fair.c

```text
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_node; // 红黑树根节点
    struct rb_node *parent = NULL;
    struct sched_entity *entry;
    s64 key = entity_key(cfs_rq, se); // 当前进程调度实体的虚拟运行时间
    int leftmost = 1;
    while (*link) { // 把当前调度实体插入到运行队列的红黑树中
        parent = *link;
        entry = rb_entry(parent, struct sched_entity, run_node);
        if (key < entity_key(cfs_rq, entry)) { // 比较虚拟运行时间
            link = &parent->rb_left;
        } else {
            link = &parent->rb_right;
            leftmost = 0;
        }
    }
    if (leftmost) {
        cfs_rq->rb_leftmost = &se->run_node; // 缓存红黑树最左节点
        cfs_rq->min_vruntime = max_vruntime(cfs_rq->min_vruntime, se->vruntime);
    }
    // 这里是红黑树的平衡过程(参考红黑树数据结构的实现)
    rb_link_node(&se->run_node, parent, link);
    rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline);
}
```

__enqueue_entity() 函数的主要工作如下：

1. 获取运行队列红黑树的根节点。
2. 获取当前进程调度实体的虚拟运行时间。
3. 把当前进程调度实体添加到红黑树中（可参考红黑树的插入算法）。
4. 缓存红黑树最左端节点。
5. 对红黑树进行平衡操作（可参考红黑树的平衡算法）。

调用 __enqueue_entity() 函数后，就可以把进程调度实体插入到运行队列的红黑树中。同时会把红黑树最左端的节点缓存到运行队列的 rb_leftmost 字段中，用于快速获取下一个可运行的进程。

**3. 从可运行队列中获取下一个可运行的进程**

要获取运行队列中下一个可运行的进程可以通过调用 __pick_next_entity() 函数，其实现如下：

/src/kernel/sched_fair.c

```text
static struct sched_entity *__pick_next_entity(struct cfs_rq *cfs_rq)
{
    // 1. 先调用 first_fair() 获取红黑树最左端节点
    // 2. 调用 rb_entry() 返回节点对应的调度实体
    return rb_entry(first_fair(cfs_rq), struct sched_entity, run_node);
}
static inline struct rb_node *first_fair(struct cfs_rq *cfs_rq)
{
    return cfs_rq->rb_leftmost; // 获取红黑树最左端节点
}
```

前面介绍过，红黑树的最左端节点就是虚拟运行时间最少的进程，所以 __pick_next_entity() 函数的流程就非常简单了，如下：

- 首先调用 first_fair() 获取红黑树最左端节点。
- 然后再调用 rb_entry() 返回节点对应的调度实体。

## 4.调度时机

前面介绍了 完全公平调度算法 怎么向可运行队列添加调度的进程和怎么从可运行队列中获取下一个可运行的进程，那么 Linux 内核在什么时候会进行进程调度呢？

> 答案是由 Linux 内核的时钟中断触发的。

时钟中断 是什么？如果没了解过操作系统原理的可能不了解这个东西，但是没关系，不了解可以把他当成是定时器就好了，就是每隔固定一段时间会调用一个回调函数来处理这个事件。

时钟中断 犹如 Linux 的心脏一样，每隔一定的时间就会触发调用 scheduler_tick() 函数，其实现如下：

```text
void scheduler_tick(void)
{
    ...
    curr->sched_class->task_tick(rq, curr, 0); // 这里会调用 task_tick_fair() 函数
    ...
}
```

scheduler_tick() 函数会调用 task_tick_fair() 函数处理调度相关的工作，而 task_tick_fair() 主要通过调用 entity_tick() 来完成调度工作的，调用链如下：

```text
scheduler_tick() -> task_tick_fair() -> entity_tick()
```

entity_tick() 函数实现如下：

```text
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    // 更新当前进程的虚拟运行时间
    update_curr(cfs_rq);
    ...
    if (cfs_rq->nr_running > 1 || !sched_feat(WAKEUP_PREEMPT))
        check_preempt_tick(cfs_rq, curr); // 判断是否需要进行进程调度
}
```

entity_tick() 函数主要完成以下工作：

- 调用 update_curr() 函数更新进程的虚拟运行时间，这个前面已经介绍过。
- 调用 check_preempt_tick() 函数判断是否需要进行进程调度。

我们接着分析 check_preempt_tick() 函数的实现：

```text
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;
    // 计算当前进程可用的时间片
    ideal_runtime = sched_slice(cfs_rq, curr); 
    // 进程运行的实际时间
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    // 如果进程运行的实际时间大于其可用时间片, 那么进行调度
    if (delta_exec > ideal_runtime)
        resched_task(rq_of(cfs_rq)->curr);
}
```

check_preempt_tick() 函数主要完成以下工作：

- 通过调用 sched_slice() 计算当前进程可用的时间片。
- 获取当前进程在当前调度周期实际已运行的时间。
- 如果进程实际运行的时间大于其可用时间片, 那么调用 resched_task() 函数进行进程调度。

可以看出，在 时钟中断 的处理中，有可能会进行进程调度。除了 时钟中断 外，一些主动让出 CPU 的操作也会进行进程调度（如一些 I/O 操作），这里就不详细分析了，有兴趣可以自己研究。

原文地址：https://zhuanlan.zhihu.com/p/354028574

作者：linux