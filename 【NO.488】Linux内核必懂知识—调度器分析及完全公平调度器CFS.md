# 【NO.488】Linux内核必懂知识—调度器分析及完全公平调度器CFS

## 1.**调度器分析**

### 1.1.**调度器**

- 内核中安排进程执行的模块，用以切换进程状态。
- 做两件事：选择某些就绪进程来执行；打断某些执行的进程让其变为就绪状态。
- 分配CPU时间的基本依据：进程优先级。
  上下文切换（context switch）：将进程在CPU中切换执行的过程，内核承担 此任务，负责重建和存储被切换掉之前的CPU状态。

### **1.2.调度类sched_class结构体与调度类**

sched_class结构体表示调度类，定义在kernel/sched/sched.h。

- 成员解析
  ecqueue_task:向就绪队列添加一个进程，某个任务进入可运行状态时，该函数将会被调用，它将调度实体放入到红黑树中。
  dequeue_task:将一个进程从就绪队列中进行删除，当某个任务退出可运行状态时调用该函数，它将从红黑树中去掉对应调度实体。
  yield_task:在进程想要资源放弃对处理器的控制权时，可食用在sched_yield系统调用，会调用内核API去处理操作。
  check_preempt_curr:检查当前运行任务是否被抢占。
  pick_next_task:选在接下来要运行的最合适的实体（进程）。
  put_prev_task:用于另一个进程代替当前运行的进程。
  set_curr_task:当任务修改它的调用类或修改它的任务组时，将调用这个函数。
  task_tick:在每次激活周期调度器时，由周期性调度器调用。
- 源码注释

```
// 系统当中有多个调度类，按照调度优先级排成一个链表，下一个优先级调度类
    const struct sched_class *next;
    // 将进程加入到执行队列当中，即将调度实体（进程）存放到红黑树中，并对nr_running变量自动加1
    void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
    // 从执行队列当中删除进程，并对nr_running变量自动减1
    void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
    // 放弃CPU执行权,实际上该函数执行先出队后入队,在这种情况下,它直接将调度实体放在红黑树的最右端
    void (*yield_task) (struct rq *rq);
    bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);
    // 用于检查当前进程是否可以被新的进程抢占
    void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);
    /*
     * It is the responsibility of the pick_next_task() method that will
     * return the next task to call put_prev_task() on the @prev task or
     * something equivalent.
     *
     * May return RETRY_TASK when it finds a higher prio class has runnable
     * tasks.
     */
    // 选择下一个应该要运行的进程
    struct task_struct * (*pick_next_task) (struct rq *rq,
                        struct task_struct *prev);
    // 将进程放回到运行队列当中 
    void (*put_prev_task) (struct rq *rq, struct task_struct *p);
#ifdef CONFIG_SMP
    // 为进程选择一个合适的CPU
    int  (*select_task_rq)(struct task_struct *p, int task_cpu, int sd_flag, int flags);
    // 迁移任务到另一个CPU
    void (*migrate_task_rq)(struct task_struct *p);
    // 专门用户唤醒进程    
    void (*task_waking) (struct task_struct *task);
    void (*task_woken) (struct rq *this_rq, struct task_struct *task);
    // 修改进程在CPU的亲和力
    void (*set_cpus_allowed)(struct task_struct *p,
                 const struct cpumask *newmask);
    // 启动运行队列
    void (*rq_online)(struct rq *rq);
    // 禁止运行队列
    void (*rq_offline)(struct rq *rq);
#endif
    // 当进程改变它的调度类或进程组时被调用
    void (*set_curr_task) (struct rq *rq);
    // 调用自己time_tick函数，可能引起进程切换，将驱动运行时抢占
    void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
    // 当进程创建时候调用，不同调度策略的进程初始化不一样
    void (*task_fork) (struct task_struct *p);
    // 进程退出时会使用
    void (*task_dead) (struct task_struct *p);
    /*
     * The switched_from() call is allowed to drop rq->lock, therefore we
     * cannot assume the switched_from/switched_to pair is serliazed by
     * rq->lock. They are however serialized by p->pi_lock.
     */
    // 专门用于进程切换操作
    void (*switched_from) (struct rq *this_rq, struct task_struct *task);
    void (*switched_to) (struct rq *this_rq, struct task_struct *task);
    // 更改进程优先级 
    void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
                 int oldprio);
    unsigned int (*get_rr_interval) (struct rq *rq,
                     struct task_struct *task);
    void (*update_curr) (struct rq *rq);
#ifdef CONFIG_FAIR_GROUP_SCHED
    void (*task_move_group) (struct task_struct *p);
#endif
};
```struct sched_class {
    // 系统当中有多个调度类，按照调度优先级排成一个链表，下一个优先级调度类
    const struct sched_class *next;
    // 江金城加入到执行队列当中，即将调度实体（进程）存放到红黑树中，并对nr_running变量自动加1
    void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
    // 从执行队列当中删除进程，并对nr_running变量自动减1
    void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
    // 放弃CPU执行权,实际上该函数执行先出队后入队,在这种情况下,它直接将调度实体放在红黑树的最右端
    void (*yield_task) (struct rq *rq);
    bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);
    // 用于检查当前进程是否可以被新的进程抢占
    void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);
    /*
     * It is the responsibility of the pick_next_task() method that will
     * return the next task to call put_prev_task() on the @prev task or
     * something equivalent.
     *
     * May return RETRY_TASK when it finds a higher prio class has runnable
     * tasks.
     */
    // 选择下一个应该要运行的进程
    struct task_struct * (*pick_next_task) (struct rq *rq,
                        struct task_struct *prev);
    // 将进程放回到运行队列当中 
    void (*put_prev_task) (struct rq *rq, struct task_struct *p);
#ifdef CONFIG_SMP
    // 为进程选择一个合适的CPU
    int  (*select_task_rq)(struct task_struct *p, int task_cpu, int sd_flag, int flags);
    // 迁移任务到另一个CPU
    void (*migrate_task_rq)(struct task_struct *p);
    // 专门用户唤醒进程    
    void (*task_waking) (struct task_struct *task);
    void (*task_woken) (struct rq *this_rq, struct task_struct *task);
    // 修改进程在CPU的亲和力
    void (*set_cpus_allowed)(struct task_struct *p,
                 const struct cpumask *newmask);
    // 启动运行队列
    void (*rq_online)(struct rq *rq);
    // 禁止运行队列
    void (*rq_offline)(struct rq *rq);
#endif
    // 当进程改变它的调度类或进程组时被调用
    void (*set_curr_task) (struct rq *rq);
    // 调用自己time_tick函数，可能引起进程切换，将驱动运行时抢占
    void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
    // 当进程创建时候调用，不同调度策略的进程初始化不一样
    void (*task_fork) (struct task_struct *p);
    // 进程退出时会使用
    void (*task_dead) (struct task_struct *p);
    /*
     * The switched_from() call is allowed to drop rq->lock, therefore we
     * cannot assume the switched_from/switched_to pair is serliazed by
     * rq->lock. They are however serialized by p->pi_lock.
     */
    // 专门用于进程切换操作
    void (*switched_from) (struct rq *this_rq, struct task_struct *task);
    void (*switched_to) (struct rq *this_rq, struct task_struct *task);
    // 更改进程优先级 
    void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
                 int oldprio);
    unsigned int (*get_rr_interval) (struct rq *rq,
                     struct task_struct *task);
    void (*update_curr) (struct rq *rq);
#ifdef CONFIG_FAIR_GROUP_SCHED
    void (*task_move_group) (struct task_struct *p);
#endif
};
```

### **1.3.Linux调度类**

dl_sched_class、rt_sched_class、fair_sched_class及idle_sched_class等。每个进程都有对应一种调度策略，每一种调度策略又对应一种调度类（每一个调度类可以对应多种调度策略）。

```
extern const struct sched_class stop_sched_class;
extern const struct sched_class dl_sched_class;
extern const struct sched_class rt_sched_class;
extern const struct sched_class fair_sched_class;
extern const struct sched_class idle_sched_class;
```

rt_sched_class类 实时调度器（调度策略:SCHED_FIFO SCHED_RR）
fair_sched_class类 完全公平调度器（调度策略：SCHED_NORMAL、SCHED_BATCH）
SCHED_FIFO调度策略的实时进程永远比SCHED_NORMAL调度策略的普通进程优先运行。eg:pick_next_task函数。
调度类优先级顺序：stop_sched_class > dl_sched_class > rt_sched_class > fair_sched_class > idle_sched_class.
stop_sched_class：优先级最高的线程，会中断所有其他线程，并且不会被其他任务打断。
dl_sched_class
rt_sched_class：作用于实时线程。
fair_sched_class： 公平调度器CFS，作用： 一般常用线程。
idle_sched_class： 每个CPU的第一个PID=0线程，swapper，是一个静态线程。每个调度类属于idle_sched_class。一般运行在开机过程和CPU异常时会做dump。
SCHED_NORMAL,SCHED_BATCH,SCHED_IDLE直接被映射到fair_sched_class；
SCHED_RR,SCHED_FIFO与rt_sched_class相关联。

```
* Simple, special scheduling class for the per-CPU stop tasks:
 */
const struct sched_class stop_sched_class = {
    .next            = &dl_sched_class,
    .enqueue_task        = enqueue_task_stop,
    .dequeue_task        = dequeue_task_stop,
    .yield_task        = yield_task_stop,
    .check_preempt_curr    = check_preempt_curr_stop,
    .pick_next_task        = pick_next_task_stop,
    .put_prev_task        = put_prev_task_stop,
#ifdef CONFIG_SMP
    .select_task_rq        = select_task_rq_stop,
    .set_cpus_allowed    = set_cpus_allowed_common,
#endif
    .set_curr_task          = set_curr_task_stop,
    .task_tick        = task_tick_stop,
    .get_rr_interval    = get_rr_interval_stop,
    .prio_changed        = prio_changed_stop,
    .switched_to        = switched_to_stop,
    .update_curr        = update_curr_stop,
};
```

Linux调度核心选择下一个合适的task运行时，会按照优先级顺序遍历调度类的pick_next_task函数。

## **2.优先级**

- task_struct结构体中采用三个成员表示进程的优先级：prio和normal_prio表示动态优先级, static_prio表示进程的静态优先级。
- 内核将任务优先级划分，实时优先级范围是0到MAX_RT_PRIO-1（即99），而普通进 程的静态优先级范围是从MAX_RT_PRIO到MAX_PRIO-1（即100到139）。数字越小优先级越高。

```
/*
 * Priority of a process goes from 0..MAX_PRIO-1, valid RT
 * priority is 0..MAX_RT_PRIO-1, and SCHED_NORMAL/SCHED_BATCH
 * tasks are in the range MAX_RT_PRIO..MAX_PRIO-1. Priority
 * values are inverted: lower p->prio value means higher priority.
 *
 * The MAX_USER_RT_PRIO value allows the actual maximum
 * RT priority to be separate from the value exported to
 * user-space.  This allows kernel threads to set their
 * priority to a value higher than any user task. Note:
 * MAX_RT_PRIO must not be smaller than MAX_USER_RT_PRIO.
 */
#define MAX_USER_RT_PRIO    100
#define MAX_RT_PRIO        MAX_USER_RT_PRIO
#define MAX_PRIO        (MAX_RT_PRIO + NICE_WIDTH)
#define DEFAULT_PRIO        (MAX_RT_PRIO + NICE_WIDTH / 2)
```

- 进程分类
  实时进程（Real-Time Process）：优先级高、需要立即被执行的进程。
  普通进程（Normal Process）:优先级低、更长执行时间的进程。

## **3.调度策略**

unsigned int policy:保存进程的调度策略（5种）

- SCHED_NORMAL：用于普通进程，通过CFS调度器实现。
- SCHED_BATCH: 相当于SCHED_NORMAL分化版本，采用分时策略，根据动态优先级，分配CPU运行需要资源。用于非交互处理器消耗型进程。
- SCHED_IDLE:优先级最低，在系统空闲时才执行这类进程。系统负载很低时使用CFS。
- SCHED_FIFO:先进先出调度算法（实时调度策略），相同优先级任务先到先服务，高优先级任务可以抢占低优先级任务。
- SCHED_RR：轮流调度算法（实时调度策略）。
- SCHED_DEADLINE:新支持实时进程调度策略，针对突发性计算。

```
/*
 * Scheduling policies
 */
#define SCHED_NORMAL        0
#define SCHED_FIFO        1
#define SCHED_RR        2
#define SCHED_BATCH        3
/* SCHED_ISO: reserved but not implemented yet */
#define SCHED_IDLE        5
#define SCHED_DEADLINE        6
```

## 4.**完全公平调度算法（时间片轮询调度算法）**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216161658_67188.jpg)

实现完全公平调度算法，要为进程定义两个时间：

1. 实际运行时间
   实际运行时间=调度周期*进程权重/所有进程权重之和。*
   *调度周期：指所有进程运行一遍所需要的时间。*
   *进程权重：根据进程的重要性，分配给每个进程不同的权重。*
   *2.虚拟运行时间*
   *虚拟运行时间=实际运行时间*1024/进程权重=（调度周期*进程权重/所有进程权重之和）* 1024/进程权重=调度周期*1024/所有进程总权重。
   一个调度周期里，所有进程的虚拟运行时间相同。

- 基本原理
  CFS定义一种新调度模型，它给cfs_rq（cfs的run queue）中的每一个进程都设置一个虚 拟时钟-virtual runtime(vruntime)。如果一个进程得以执行，随着执行时间的不断增长，其 vruntime也将不断增大，没有得到执行的进程vruntime将保持不变。

## 5.**调度器结构分析**

任务：合理分配CPU时间给运行进程。
目标：有效分配CPU是时间片。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216161658_84329.jpg)

主调度器：通过schedule()函数来完成进程的选择和切换。
周期调度器：根据频率自动调用scheduler_tick函数，作用：根据进程运行时间触发调度。
上下文切换：主要做两件事：切换地址空间；切换寄存器和栈空间。

## 6.**CFS就绪队列**

调度管理是各个调度器的职责。CFS的顶级调度就队列struct cfs_rq。

```
/* CFS-related fields in a runqueue */
// CFS调度运行队列，每个CPU的rq都会包含一个cfs_rq,每个组调度的sched_entity也会有一个cfs_rq队列
struct cfs_rq {
    // CFS运行队列中所有进程总负载
    struct load_weight load;
    // nr_running::cfs_rq中调度实体数量
    // h_nr_running 只对进程有效
    unsigned int nr_running, h_nr_running;
    u64 exec_clock;
        //跟踪记录队列所有进程最小虚拟运行时间
    u64 min_vruntime;
#ifndef CONFIG_64BIT
    u64 min_vruntime_copy;
#endif
    // 红黑树的root  用于在按时间排序的红黑树中管理所有进程
    struct rb_root tasks_timeline;
    // 下一个调度结点（红黑树最左边结点就是下个调度实体）
    struct rb_node *rb_leftmost;
    /*
     * 'curr' points to currently running entity on this cfs_rq.
     * It is set to NULL otherwise (i.e when none are currently running).
     */
    struct sched_entity *curr, *next, *last, *skip;
#ifdef    CONFIG_SCHED_DEBUG
    unsigned int nr_spread_over;
#endif
#ifdef CONFIG_SMP
    /*
     * CFS load tracking
     */
    struct sched_avg avg;
    u64 runnable_load_sum;
    unsigned long runnable_load_avg;
#ifdef CONFIG_FAIR_GROUP_SCHED
    unsigned long tg_load_avg_contrib;
#endif
    atomic_long_t removed_load_avg, removed_util_avg;
#ifndef CONFIG_64BIT
    u64 load_last_update_time_copy;
#endif
#ifdef CONFIG_FAIR_GROUP_SCHED
    /*
     *   h_load = weight * f(tg)
     *
     * Where f(tg) is the recursive weight fraction assigned to
     * this group.
     */
    unsigned long h_load;
    u64 last_h_load_update;
    struct sched_entity *h_load_next;
#endif /* CONFIG_FAIR_GROUP_SCHED */
#endif /* CONFIG_SMP */
#ifdef CONFIG_FAIR_GROUP_SCHED
    struct rq *rq;    /* cpu runqueue to which this cfs_rq is attached */
    /*
     * leaf cfs_rqs are those that hold tasks (lowest schedulable entity in
     * a hierarchy). Non-leaf lrqs hold other higher schedulable entities
     * (like users, containers etc.)
     *
     * leaf_cfs_rq_list ties together list of leaf cfs_rq's in a cpu. This
     * list is used during load balance.
     */
    int on_list;
    struct list_head leaf_cfs_rq_list;
    struct task_group *tg;    /* group that "owns" this runqueue */
#ifdef CONFIG_CFS_BANDWIDTH
    int runtime_enabled;
    u64 runtime_expires;
    s64 runtime_remaining;
    u64 throttled_clock, throttled_clock_task;
    u64 throttled_clock_task_time;
    int throttled, throttle_count;
    struct list_head throttled_list;
#endif /* CONFIG_CFS_BANDWIDTH */
#endif /* CONFIG_FAIR_GROUP_SCHED */
};
```

## 7.**总结**

本文主要介绍了调度器分析：功能，调度类sched_class结构体与调度类；Linux调度类（5种）及其优先级，调度策略（5种）；完全公平调度算法，包括实际运行时间、虚拟运行时间，调度器结构分析，CFS就绪队列等内容。

原文链接：https://zhuanlan.zhihu.com/p/500583213

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)