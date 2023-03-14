# 【NO.223】Nginx 的异步非阻塞体现在哪里？从理论分析到源码验证

## 1.**理论分析**

1、首先要明确一点，这里讲的 “异步” 是业务层面上的。

2、那业务层面的异步是怎么个异步法？同步异步的概念我就不说了，前面文章有。异步最重要的标志就是通知，通知，通知！！！

这两天很累，不想多说话，长话短说吧： 以epoll为例，(nginx有提供select和poll的代码)，你可以同时监控很多个文件描述符，调用epoll是阻塞的，但是真实场景下不会让你有那个机会阻塞的。当有事件可读，就处理它。它准备了多少，就处理多少，当读写返回EAGAIN时，我们将它再次加入到epoll里面。等下次再可读了再出来被处理。只有当所有事件都没准备好时，才在epoll里面等着。

切换也是因为异步事件未准备好，而主动让出的。这里的切换是没有任何代价，你可以理解为循环处理多个准备好的事件，事实上就是这样的。

就这么个异步法，很高效。

定时器：nginx 借助 epoll_wait 的 timewait 设置超时时间，nginx 里面的定时器事件放在一颗维护定时器的红黑树里面，每次在进入epoll_wait 前，先从该红黑树里面拿到所有定时器事件的最小时间，在计算出 epoll_wait 的超时时间后进入 epoll_wait。所以，当没有事件产生，也没有中断信号时，epoll_wait会超时，也就是说，定时器事件到了。这时，nginx会检查所有的超时事件，将他们的状态设置为超时，然后再去处理网络事件。

## **2.源码体现**

（看了半天，感觉是 reactor 模型，下一个项目我要用这个！）

### 2.1 **worker进程对事件模块的初始化：**

事件模块的初始化就发生在ngx_worker_process_init函数中。

其调用关系：

```text
main()->
ngx_master_process_cycle()->
ngx_start_worker_processes()->
ngx_spawn_process()->
ngx_worker_process_cycle()->
ngx_worker_process_init()
```

在这个函数里面，调用了各个模块的启动方法：

```text
for (i = 0; ngx_modules[i]; i++) {
    if (ngx_modules[i]->init_process) {
        if (ngx_modules[i]->init_process(cycle) == NGX_ERROR) {
            /* fatal */
            exit(2);
        }
    }
}
```

在此处，会调用ngx_event_core_module的ngx_event_process_init函数。

这个函数（很长，很长）主要做了这么几件事情：

1、处理惊群问题（回头会专门来一篇讲惊群）

2、初始化两个队列，一个用于存放不能及时处理的建立连接事件，一个用于存储不能及时处理的读写事件。

```text
ngx_queue_init(&ngx_posted_accept_events);
ngx_queue_init(&ngx_posted_events);
```

3、初始化定时器。

4、调用 ngx_epoll_init 。

5、分配连接池空间、读事件结构体数组、写事件结构体数组。

6、为每个监听端口分配连接。

7、为每个监听端口的连接的读事件设置handler，并将每个监听端口的连接的读事件添加到epoll中。

### 2.2 **worker 开始循环干活了**

ngx_worker_process_cycle 函数（也不是很长，但是我不想放）。

不过这个我可以压缩一下，因为看了好几遍了，没上面那个那么恐怖：

```text
for ( ;; ) {
    ......
 
    /* 处理IO事件和时间事件 */
    ngx_process_events_and_timers(cycle);
    ......
}
```

短哈。

在worker的主循环中，所有的事件都是通过函数ngx_process_events_and_timers处理的，那我们自然就要再往下走了嘛，今天我还非要看看它到底是怎么吃一半了再塞回去的！

### **2.3 ngx_process_events_and_timers**

这个函数又干了些什么好事儿呢？

1、配置更新时间方式。是吧，原先我是不把定时器当回事儿的，但是这代码到处都是定时器，所以我就把定时器当回事儿了。

2、处理了一下惊群锁的事情。主要思想就是：负载过高咱就不抢，不然赶紧的冲上去。后面讲惊群的时候放这个代码。

3、调用事件处理函数ngx_process_events，epoll使用的是ngx_epoll_process_events函数。这个咱一会儿还得进去。

4、计算ngx_process_events函数的调用时间。

5、处理ngx_posted_accept_events队列的连接事件。这里accept事件的handler为ngx_event_accept。

6、处理定时器事件，具体操作是在定时器红黑树中查找过期的事件，调用其handler方法。

7、处理ngx_posted_events队列的读写事件，即遍历ngx_posted_events队列，调用事件的handler方法。

### **2.4 ngx_epoll_process_events**

```text
static ngx_int_t ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
······

	ngx_event_t *rev, *wev;

    /* 调用epoll_wait，从epoll中获取发生的事件 */
    events = epoll_wait(ep, event_list, (int) nevents, timer);
 
······
    /* 处理epoll_wait返回为-1的情况 */
    if (err) {
······
    }
 
    /* 若events返回为0，判断是因为epoll_wait超时还是其他原因 */
    if (events == 0) {
        if (timer != NGX_TIMER_INFINITE) {
            return NGX_OK;
        }
 
        ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                      "epoll_wait() returned no events without timeout");
        return NGX_ERROR;
    }
 
    /* 对epoll_wait返回的链表进行遍历 */
    for (i = 0; i < events; i++) {
        c = event_list[i].data.ptr;
 
        /* 从data中获取connection & instance的值，并解析出instance和connection */
        instance = (uintptr_t) c & 1;
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
 
        /* 取出connection的read事件 */
        rev = c->read;
 
        /* 判断读事件是否过期 */
        if (c->fd == -1 || rev->instance != instance) {
            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }
 
        /* 取出事件的类型 */
        revents = event_list[i].events;
 
        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "epoll: fd:%d ev:%04XD d:%p",
                       c->fd, revents, event_list[i].data.ptr);
 
        /* 若连接发生错误，则将EPOLLIN、EPOLLOUT添加到revents中，在调用读写事件时能够处理连接的错误 */
        if (revents & (EPOLLERR|EPOLLHUP)) {
            ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll_wait() error on fd:%d ev:%04XD",
                           c->fd, revents);
 
            revents |= EPOLLIN|EPOLLOUT;
        }
 
        /* 事件为读事件且读事件在epoll中 */
        if ((revents & EPOLLIN) && rev->active) {
 ······
 
            rev->ready = 1;
 
            /* 事件是否需要延迟处理？对于抢到锁监听端口的worker，会将事件延迟处理 */
            if (flags & NGX_POST_EVENTS) {
                /* 根据事件的是否是accept事件，加到不同的队列中 */
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;
 
                ngx_post_event(rev, queue);
 
            } else {
                /* 若不需要延迟处理，直接调用read事件的handler */
                rev->handler(rev);
            }
        }
 
        /* 取出connection的write事件 */
        wev = c->write;
 
        /* 事件为写事件且写事件在epoll中 */
        if ((revents & EPOLLOUT) && wev->active) {
 
            /* 判断写事件是否过期 */
            if (c->fd == -1 || wev->instance != instance) {
                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                               "epoll: stale event %p", c);
                continue;
            }
 
            wev->ready = 1;
······
 
            /* 事件是否需要延迟处理？对于抢到锁监听端口的worker，会将事件延迟处理 */
            if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);
 
            } else {
                /* 若不需要延迟处理，直接调用write事件的handler */
                wev->handler(wev);
            }
        }
    }
 
    return NGX_OK;
}
```

代码中可以看出，其中accept事件（即监听端口上的可读事件）会被缓存到队列ngx_posted_accept_events，普通事件会被缓存到队列ngx_posted_events。

缓存完事件，接下来就是处理新建连接事件（accept事件），因为当前进程已经监听了某个客户端的端口，该端口的请求中的可读事件先要处理下，该读的数据读完，即处理队列ngx_posted_accept_events中的新建连接事件，如果在处理新建连接期间还有新的请求连接事件，会阻塞，等待下次进程获取锁后读取。读完可读事件后就执行解锁操作ngx_shmtx_unlock。

锁释放完之后就处理连接套接口之后的连接事件了，即保存在队列ngx_posted_events中的事件。

### **2.5 ngx_event_process_posted**

```text
void ngx_event_process_posted(ngx_cycle_t *cycle, ngx_queue_t *posted)
{
    ngx_queue_t  *q;
    ngx_event_t  *ev;

    while (!ngx_queue_empty(posted)) {

        q = ngx_queue_head(posted);
        ev = ngx_queue_data(q, ngx_event_t, queue);

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                      "posted event %p", ev);

        ngx_delete_posted_event(ev);

        ev->handler(ev);
    }
}
```

可以看出，就是不断遍历队列，调用对应的handler处理事件。

原文地址：https://zhuanlan.zhihu.com/p/596770134

作者：linux