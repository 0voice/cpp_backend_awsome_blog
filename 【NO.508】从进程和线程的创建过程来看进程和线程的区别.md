# 【NO.508】从进程和线程的创建过程来看进程和线程的区别

在用户空间我们可以通过fork（）函数来创建一个新的进程。fork()是一个glibc标准库函数，在内核里边会有一个系统调用与之对应--sys_fork（）。同样的，我们是通过pthread_create（）来创建一个线程，内核中对应的系统调用是clone（）。现在通过分析sys_fork（）和clone（）的异同来看进程和线程的区别。本文是基于5.0版本的Linux内核和2.21版本的glibc。

**fork**

首先看一下sys_fork（），定义在kernel\fork.c文件中：

```text
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
```

可以看到sys_fork 会调用 _do_fork。

**pthread_create**

对于线程的创建，我们先来看glibc源码（glibc-2.21\sysdeps\unix\sysv\linux\createthread.c）：

```text
static int
create_thread (struct pthread *pd, const struct pthread_attr *attr,
	       bool stopped_start, STACK_VARIABLES_PARMS, bool *thread_ran)
{
  ......
 
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);
 TLS_DEFINE_INIT_TP (tp, pd);
 
  if (__glibc_unlikely (ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS,
				    clone_flags, pd, &pd->tid, tp, &pd->tid)
			== -1))
    return errno;
  ........
  
}
```

我们这里可以看到，在glibc中先设置了一个clone_flag（这个flag标记很重要，最后和进程的一个很大的区别就在这里），然后陷入到的clone（）系统调用：

```text
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
     int __user *, parent_tidptr,
     int __user *, child_tidptr,
     unsigned long, tls)
{
  return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
```

这里我们发现，创建线程时最后也会调用到 _do_fork函数！那么在_do_fork（）中，创建进程和创建线程有什么区别呢？我们继续往下看。

```text
long _do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        unsigned long stack_size,
        int __user *parent_tidptr,
        int __user *child_tidptr,
        unsigned long tls)
{
  struct task_struct *p;
  int trace = 0;
  long nr;
 
 
......
  p = copy_process(clone_flags, stack_start, stack_size,
       child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
......
  if (!IS_ERR(p)) {
    struct pid *pid;
    pid = get_task_pid(p, PIDTYPE_PID);
    nr = pid_vnr(pid);
 
 
    if (clone_flags & CLONE_PARENT_SETTID)
      put_user(nr, parent_tidptr);
 
 
......
    wake_up_new_task(p);
......
    put_pid(pid);
  } 
......
```

在_do_fork这个函数之中最重要的步骤就是 copy_process， copy_process函数中最重要的则是创建新任务的task_struct结构体。对于内核来讲进程和线程都是一个task任务，而内核用task_struct来封装一个任务。 task_struct 的结构示意图如下所示（图来源于极客时间趣谈Linux操作系统）：

![img](https://pic3.zhimg.com/80/v2-f955045e2611396e8dcf9e910bef1872_720w.webp)

```text
static __latent_entropy struct task_struct *copy_process(
          unsigned long clone_flags,
          unsigned long stack_start,
          unsigned long stack_size,
          int __user *child_tidptr,
          struct pid *pid,
          int trace,
          unsigned long tls,
          int node)
{
  int retval;
  struct task_struct *p;
......
  p = dup_task_struct(current, node);
```

dup_task_struct 主要做了下面几件事情：

- 调用 alloc_task_struct_node 分配一个 task_struct 结构；
- 调用 alloc_thread_stack_node 来创建内核栈，这里面调用 __vmalloc_node_range 分配一个连续的 THREAD_SIZE 的内存空间，赋值给 task_struct 的 void *stack 成员变量；
- 调用 arch_dup_task_struct(struct task_struct *dst, struct task_struct *src)，将 task_struct 进行复制，其实就是调用 memcpy；
- 调用 setup_thread_stack 设置 thread_info。

到这里，整个 task_struct 复制了一份，而且内核栈也创建好了。 继续看copy_process:

```text
/* copy all the process information */
	shm_init_task(p);
	retval = security_task_alloc(p, clone_flags);
	if (retval)
		goto bad_fork_cleanup_audit;
	retval = copy_semundo(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_security;
	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
	retval = copy_fs(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_files;
	retval = copy_sighand(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_fs;
	retval = copy_signal(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_sighand;
	retval = copy_mm(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_signal;
	retval = copy_namespaces(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_mm;
	retval = copy_io(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_namespaces;
	retval = copy_thread_tls(clone_flags, stack_start, stack_size, p, tls);
	if (retval)
		goto bad_fork_cleanup_io;
```

这一段代码主要是用来复制任务的信息，内容比较多，我们可以从函数名称就大概猜出具体复制些什么，这里主要挑五个比较重要的来讲。

| copy_files   | 主要用于复制一个进程打开的文件信息。这些信息用一个结构 files_struct 来维护，每个打开的文件都有一个文件描述符。 |
| ------------ | ------------------------------------------------------------ |
| copy_fs      | 主要用于复制一个进程的目录信息。这些信息用一个结构 fs_struct 来维护。 |
| copy_sighand | 分配一个新的 sighand_struct。这里最主要的是维护信号处理函数  |
| copy_signal  | 复制用于维护发给这个进程的信号的数据结构                     |
| copy_mm      | 进程都有自己的内存空间，用 mm_struct 结构来表示。copy_mm 函数中调用 dup_mm，分配一个新的 mm_struct 结构，调用 memcpy 复制这个结构。 |

接下来一个个分析这几个函数

**copy_files**

```text
static int copy_files(unsigned long clone_flags, struct task_struct *tsk)
{
  struct files_struct *oldf, *newf;
  oldf = current->files;
  if (clone_flags & CLONE_FILES) {
    atomic_inc(&oldf->count);
    goto out;
  }
  newf = dup_fd(oldf, &error);
  tsk->files = newf;
out:
  return error;
}
```

这个函数中我们看到了在glibc create_thread中设置的clone_flags,在上面的代码中可以看出，如果设置了CLONE_FLES这个位，那么只是给主线程的task_struct的files_struct的引用计数count原子地加1，然后就返回了。而如果是进程的创建，CLONE_FLES位没有被标记，则会在dup_fd里边创建一个新的 files_struct，然后将所有的文件描述符数组 fdtable 拷贝一份。dup_fd函数定义在linux-5.0\fs\file.c文件中，这里就不贴代码了。

**copy_fs**

```text
static int copy_fs(unsigned long clone_flags, struct task_struct *tsk)
{
  struct fs_struct *fs = current->fs;
  if (clone_flags & CLONE_FS) {
    fs->users++;
    return 0;
  }
  tsk->fs = copy_fs_struct(fs);
  return 0;
}
```

对于 copy_fs，在创建进程时是调用 copy_fs_struct 复制一个 fs_struct。如果是创建线程则由于 CLONE_FS 标识位已经被设置，所以也是将原来的 fs_struct 的用户数加一。这个逻辑和copy_files是一样的。

**copy_sighand**

对于和信号相关的的两个结构体，sighand_struct、signal_struct，处理过程也是类似，如果是进程创建则会复制一分，而线程创建也仅仅是将将原来的 sighand_struct 引用计数加一。

```text
static int copy_sighand(unsigned long clone_flags, struct task_struct *tsk)
{
  struct sighand_struct *sig;
 
 
  if (clone_flags & CLONE_SIGHAND) {
    atomic_inc(&current->sighand->count);
    return 0;
  }
  sig = kmem_cache_alloc(sighand_cachep, GFP_KERNEL);
  atomic_set(&sig->count, 1);
  memcpy(sig->action, current->sighand->action, sizeof(sig->action));
  return 0;
```

**copy_signal**

对于 copy_signal，进程的创建是创建一个新的 signal_struct，而线程的创建则因为 CLONE_THREAD 直接返回了。

```text
static int copy_signal(unsigned long clone_flags, struct task_struct *tsk)
{
  struct signal_struct *sig;
  if (clone_flags & CLONE_THREAD)
    return 0;
  sig = kmem_cache_zalloc(signal_cachep, GFP_KERNEL);
  tsk->signal = sig;
    init_sigpending(&sig->shared_pending);
......
}
```

**copy_mm**

对于 copy_mm，进程创建是是调用 dup_mm 复制一个 mm_struct，线程的创建则因为 CLONE_VM 标识位而直接指向了原来的 mm_struct。

```text
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
  struct mm_struct *mm, *oldmm;
  oldmm = current->mm;
  if (clone_flags & CLONE_VM) {
    mmget(oldmm);
    mm = oldmm;
    goto good_mm;
  }
  mm = dup_mm(tsk);
good_mm:
  tsk->mm = mm;
  tsk->active_mm = mm;
  return 0;
}
```

**总结**

综上所述，创建进程的时候调用的系统调用是 fork，在 copy_process 函数里面，会将五个最重要的结构体 files_struct、fs_struct、sighand_struct、signal_struct、mm_struct 都复制一遍，从此父进程和子进程各用各的数据结构，各有一个独立的内存空间。而创建线程的话，调用的是系统调用 clone，在 copy_process 函数里面， 五大结构仅仅是引用计数加一，也即线程共享进程的数据结构。

这里在次引用极客时间趣谈Linux操作系统的一张图总结一下：

![img](https://pic1.zhimg.com/80/v2-125357aa3a2f27d50a1d7eb5ce16e960_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/602672787

作者：linux