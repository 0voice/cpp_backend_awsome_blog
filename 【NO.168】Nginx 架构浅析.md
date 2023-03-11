# 【NO.168】Nginx 架构浅析

## **1.Nginx 基础架构**

nginx 启动后以 daemon 形式在后台运行，后台进程包含一个 master 进程和多个 worker 进程。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA05XyKrmjqibXn76CzlYj1wA1GUmjCCw0tuWAMNI2oT6kfMfGOkTDFxbQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)master与worker

nginx 是由一个 master 管理进程，多个 worker 进程处理工作的多进程模型。基础架构设计，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA0rJpg7py6IQko9XQuAKEnib9xHlkSzSElcKBvQ2265cOyvABicfuk9cPg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)基础架构设计

master 负责管理 worker 进程，worker 进程负责处理网络事件。整个框架被设计为一种依赖事件驱动、异步、非阻塞的模式。

如此设计的优点：

- 1.可以充分利用多核机器，增强并发处理能力。
- 2.多 worker 间可以实现负载均衡。
- 3.Master 监控并统一管理 worker 行为。在 worker 异常后，可以主动拉起 worker 进程，从而提升了系统的可靠性。并且由 Master 进程控制服务运行中的程序升级、配置项修改等操作，从而增强了整体的动态可扩展与热更的能力。

## **2.Master 进程**

### 2.1 核心逻辑

master 进程的主逻辑在`ngx_master_process_cycle`，核心关注源码：

```
ngx_master_process_cycle(ngx_cycle_t *cycle)
{
    ...
    ngx_start_worker_processes(cycle, ccf->worker_processes,
                                        NGX_PROCESS_RESPAWN);
    ...


    for ( ;; ) {
        if (delay) {...}

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "sigsuspend");

        sigsuspend(&set);

        ngx_time_update();

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                             "wake up, sigio %i", sigio);

        if (ngx_reap) {
            ngx_reap = 0;
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "reap children");
            live = ngx_reap_children(cycle);
        }

        if (!live && (ngx_terminate || ngx_quit)) {...}

        if (ngx_terminate) {...}

        if (ngx_quit) {...}

        if (ngx_reconfigure) {...}

        if (ngx_restart) {...}

        if (ngx_reopen) {...}

        if (ngx_change_binary) {...}

        if (ngx_noaccept) {
            ngx_noaccept = 0;
            ngx_noaccepting = 1;
            ngx_signal_worker_processes(cycle,
                                                  ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }
    }
 }
```

由上述代码，可以理解，master 进程主要用来管理 worker 进程，具体包括如下 4 个主要功能：

- 1.接受来自外界的信号。其中 master 循环中的各项标志位就对应着各种信号，如：`ngx_quit`代表`QUIT`信号，表示优雅的关闭整个服务。
- 2.向各个 worker 进程发送信。比如`ngx_noaccept`代表`WINCH`信号，表示所有子进程不再接受处理新的连接，由 master 向所有的子进程发送 QUIT 信号量。
- 3.监控 worker 进程的运行状态。比如`ngx_reap`代表`CHILD`信号，表示有子进程意外结束，这时需要监控所有子进程的运行状态，主要由`ngx_reap_children`完成。
- 4.当 woker 进程退出后（异常情况下），会自动重新启动新的 woker 进程。主要也是在`ngx_reap_children`

### 2.2 热更

#### **2.2.1 热重载-配置热更**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA0XPu1mQ1Ev3VQ2g3dlXVSXy6Ws6hzepxaToYibSBtZ5Z4dF5uEoIas7Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)热重载

nginx 热更配置时，可以保持运行中平滑更新配置，具体流程如下：

- 1.更新 nginx.conf 配置文件，向 master 发送 SIGHUP 信号或执行 nginx -s reload
- 2.master 进程使用新配置，启动新的 worker 进程
- 3.使用旧配置的 worker 进程，不再接受新的连接请求，并在完成已存在的连接后退出

#### **2.2.2 热升级-程序热更**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA0bicSzcPO3JTr5aib6NkRibicuesib0JAlqt70dgvlsvYW8ubgdibB8L76tHQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)热升级

nginx 热升级过程如下：

- 1.将旧 Nginx 文件换成新 Nginx 文件（注意备份）
- 2.向 master 进程发送 USR2 信号（平滑升级到新版本的 Nginx 程序）
- 3.master 进程修改 pid 文件号，加后缀.oldbin
- 4.master 进程用新 Nginx 文件启动新 master 进程，此时新老 master/worker 同时存在。
- 5.向老 master 发送 WINCH 信号，关闭旧 worker 进程，观察新 worker 进程工作情况。若升级成功，则向老 master 进程发送 QUIT 信号，关闭老 master 进程；若升级失败，则需要回滚，向老 master 发送 HUP 信号（重读配置文件），向新 master 发送 QUIT 信号，关闭新 master 及 worker。

## **3.Worker 进程**

### 3.1 核心逻辑

worker 进程的主逻辑在`ngx_worker_process_cycle`，核心关注源码：

```
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ngx_int_t worker = (intptr_t) data;

    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;

    ngx_worker_process_init(cycle, worker);

    ngx_setproctitle("worker process");

    for ( ;; ) {

        if (ngx_exiting) {...}

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");

        ngx_process_events_and_timers(cycle);

        if (ngx_terminate) {...}

        if (ngx_quit) {...}

        if (ngx_reopen) {...}
    }
}
```

由上述代码，可以理解，worker 进程主要在处理网络事件，通过`ngx_process_events_and_timers`方法实现，其中事件主要包括：网络事件、定时器事件。

### 3.2 事件驱动-epoll

worker 进程在处理网络事件时，依靠 epoll 模型，来管理并发连接，实现了事件驱动、异步、非阻塞等特性。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA08AoXwjpIHibw3gKsqUwx5OFSe0gyp3TJibdK2W9VeGm7iaEhTN4Cjx2YA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)infographic-Inside-NGINX_nonblocking

通常海量并发连接过程中，每一时刻（相对较短的一段时间），往往只需要处理一小部分有事件的连接即`活跃连接`。基于以上现象，epoll 通过将`连接管理`与`活跃连接管理`进行分离，实现了高效、稳定的网络 IO 处理能力。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA0Te31anxlCPmr6BfiatKnFvK2yELrhnd91jPInoTmibicpq7zMnOwiaPlQA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)网络模型对比

其中，epoll 利用红黑树高效的增删查效率来管理`连接`，利用一个双向链表来维护`活跃连接`。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA0wafvhzDibWWdTmAMgq9EfP0X3bREwPsE57pucUg919pcZIWEXKpOWsQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)epoll数据结构

### 3.3 惊群

由于 worker 都是由 master 进程 fork 产生，所以 worker 都会监听相同端口。这样多个子进程在 accept 建立连接时会发生争抢，带来著名的“惊群”问题。worker 核心处理逻辑`ngx_process_events_and_timers`核心代码如下：

```
void ngx_process_events_and_timers(ngx_cycle_t *cycle){
    //这里面会对监听socket处理
    ...

    if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;
    } else {
        //获得锁则加入wait集合,
        if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
            return;
        }
        ...
        //设置网络读写事件延迟处理标志，即在释放锁后处理
        if (ngx_accept_mutex_held) {
            flags |= NGX_POST_EVENTS;
        }
    }
    ...
    //这里面epollwait等待网络事件
    //网络连接事件，放入ngx_posted_accept_events队列
    //网络读写事件，放入ngx_posted_events队列
    (void) ngx_process_events(cycle, timer, flags);
    ...
    //先处理网络连接事件，只有获取到锁，这里才会有连接事件
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    //释放锁，让其他进程也能够拿到
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }
    //处理网络读写事件
    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

由上述代码可知，Nginx 解决惊群的方法：

- 1.将连接事件与读写事件进行分离。连接事件存放为`ngx_posted_accept_events`，读写事件存放为`ngx_posted_events`。
- 2.设置`ngx_accept_mutex`锁，只有获得锁的进程，才可以处理连接事件。

### 3.4 负载均衡

worker 间的负载关键在于各自接入了多少连接，其中接入连接抢锁的前置条件是`ngx_accept_disabled > 0`，所以`ngx_accept_disabled`就是负载均衡机制实现的关键阈值。

```
ngx_int_t             ngx_accept_disabled;
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```

因此，在 nginx 启动时，`ngx_accept_disabled`的值就是一个负数，其值为连接总数的 7/8。当该进程的连接数达到总连接数的 7/8 时，该进程就不会再处理新的连接了，同时每次调用'ngx_process_events_and_timers'时，将`ngx_accept_disabled`减 1，直到其值低于阈值时，才试图重新处理新的连接。因此，nginx 各 worker 子进程间的负载均衡仅在某个 worker 进程处理的连接数达到它最大处理总数的 7/8 时才会触发，其负载均衡并不是在任意条件都满足。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavkzcOqibK2l3xbf5wHIhibA0ib3xvf51icrkAibq9skzqfCCtXFjTaibUqKeicoVP88dfltKmW2nxSZYKeA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)实际工作情况

其中'pid'为 1211 的进程为 master 进程，其余为 worker 进程

## **4.思考**

### 4.1 为什么不采用多线程模型管理连接？

- 1.无状态服务，无需共享进程内存
- 2.采用独立的进程，可以让互相之间不会影响。一个进程异常崩溃，其他进程的服务不会中断，提升了架构的可靠性。
- 3.进程之间不共享资源，不需要加锁，所以省掉了锁带来的开销。

### 4.2 为什么不采用多线程处理逻辑业务？

- 1.进程数已经等于核心数，再新建线程处理任务，只会抢占现有进程，增加切换代价。
- 2.作为接入层，基本上都是数据转发业务，网络 IO 任务的等待耗时部分，已经被处理为非阻塞/全异步/事件驱动模式，在没有更多 CPU 的情况下，再利用多线程处理，意义不大。并且如果进程中有阻塞的处理逻辑，应该由各个业务进行解决，比如 openResty 中利用了 Lua 协程，对阻塞业务进行了优化。

原文作者：handsomeli，腾讯 IEG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/gd59U50tlJPa4B4avXRG1Q