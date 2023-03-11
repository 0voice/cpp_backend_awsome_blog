# 【NO.116】线程池的优点及其原理，简单、明了。

## 1.使用线程池的好处

池化技术应用：线程池、数据库连接池、http连接池等等。

池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。

**线程池提供了一种限制、管理资源的策略。** 每个**线程池**还维护一些基本统计信息，例如已完成任务的数量。

**使用线程池的好处：**

- **降低资源消耗**：通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度**：当任务到达时，可以不需要等待线程创建就能立即执行。
- **提高线程的可管理性**：线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，监控和调优。

## 2.Executor框架

Executor框架不仅包括了线程池的管理，还提供了线程工厂、队列以及拒绝策略等，让并发编程变得更加简单。

**Executor框架的使用示意图**

![img](https://pic2.zhimg.com/80/v2-dff549e683cdef4a7025c42b9f431e5d_720w.webp)

1. **主线程首先要创建实现Runnable或Callable接口的任务对象。**
2. **把创建完成的实现Runnable/Callable接口的对象直接交给ExecutorService执行：**

```
ExecutorService.execute(Runnable command)或者ExecutorService.sumbit(Runnable command)或ExecutorService.sumbit(Callable <T> task).
```

1. **如果执行ExecutorService.submit(…)，ExecutorService将返回一个实现Future接口的对象。最后，主线程可以执行FutureTask.get()方法来等待任务执行完成。主线程也可以执行FutureTask.cancel()来取消次任务的执行。**

```
import java.util.concurrent.ArrayBlockingQueue;import java.util.concurrent.ThreadPoolExecutor;import java.util.concurrent.TimeUnit;public class ThreadPoolExecutorDemo {    private static final int CORE_POOL_SIZE = 5;    private static final int MAX_POOL_SIZE = 10;    private static final int QUEUE_CAPACITY = 100;    private static final Long KEEP_ALIVE_TIME = 1L;    public static void main(String[] args) {        ThreadPoolExecutor executor = new ThreadPoolExecutor(                CORE_POOL_SIZE,                MAX_POOL_SIZE,                KEEP_ALIVE_TIME,                TimeUnit.SECONDS,                new ArrayBlockingQueue<>(QUEUE_CAPACITY),                new ThreadPoolExecutor.CallerRunsPolicy());        //执行线程代码        executor.shutdown();    }} 
```

**CORE_POOL_SIZE:**核心线程数定义了最小可以同时运行的线程数量。

**MAX_POOL_SIZE：**当队列中存放的任务到达队列容量的时候，当前可以同时运行的线程数量变为最大线程数。

**QUEUE_CAPACITY：**当新任务加入是会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，任务就会被存放到队列中。

**KEEP_ALIVE_TIME：**当线程池中的线程数量大于核心线程数时，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过**KEEP_ALIVE_TIME**才会被回收销毁。

**ThreadPoolExecutor.CallerRunsPolicy()：**调用执行自己的线程运行任务，也就是直接在调用execute方法的线程运行（run）被拒绝的任务，如果执行程序已关闭，则会丢弃任务。因此这种策略会降低新任务的提交速度，影响程序的整体性能。另外，这个策略喜欢增加队列容量。如果应用程序可以承受此延迟并且不能任务丢弃一个任务请求的话，可以选择这个策略。

**线程池分析原理**

![img](https://pic1.zhimg.com/80/v2-965f8856f55a687ef34664b2c1a7086c_720w.webp)

## 3.线程池大小确定

有一个简单且使用面比较广的公式：

- **CPU密集型任务（N+1）：**这种任务消耗的主要是CPU资源，可以将线程数设置为N（CPU核心数）+1，比CPU核心数多出来一个线程是为了防止线程偶发的缺页中断，或者其他原因导致的任务暂停而带来的影响。一旦任务停止，CPU就会处于空闲状态，而这种情况下多出来一个线程就可以充分利用CPU的空闲时间。
- **I/O密集型（2N）：**这种任务应用起来，系统大部分时间用来处理I/O交互，而线程在处理I/O的是时间段内不会占用CPU来处理，这时就可以将CPU交出给其他线程使用。因此在I/O密集型任务的应用中，可以配置多一些线程，具体计算方是2N。

原文链接：https://zhuanlan.zhihu.com/p/330504183

作者：Linux服务器研究 