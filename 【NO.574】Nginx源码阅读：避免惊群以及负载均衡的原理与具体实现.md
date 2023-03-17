# 【NO.574】Nginx源码阅读：避免惊群以及负载均衡的原理与具体实现

## 1.惊群效应

### 1.1 发生惊群效应的原因

#### 1.1.1 使用accept

主进程（master进程）fork出⼀批⼦进程（worker进程），⼦进程继承了⽗进程的监听端⼝（sockfd），就会出现accept惊群效应。子进程的fd属于同一个文件，若两个⼦进程同时调⽤accept进⾏阻塞监听，两个进程都会被挂起来，内核会在这个socket的等待队列wait queue链表中将两个PID记录下来以便唤醒。Linux2.6版本之后引⼊了⼀个标记为WQ_FLAG_EXCLUSIVE解决了这种惊群效应。这个在内核就已经处理了。

#### 1.1.2 使用epoll监听公共的端口

epoll与直接accept不同，epoll需要先调⽤epoll_create在内核中创建⼀个epollfd
epoll会把当前进程挂在fd的等待队列下，但是默认情况下这种挂载不会设置互斥标志，意思着当设备有事情产⽣进⾏等待队列唤醒的时候，如果当前队列有多个进程在等待，则会全部唤醒，当多个进程共享同⼀个监听端⼝并且都使⽤epoll进⾏多路复⽤的监听时，epoll将这些进程都挂在同⼀个等待队列下。

在linux的后续版本中，解决了这个问题，通过设置标志位，使得只会唤醒队列中的⼀个进程。

既然linux内核中都解决了惊群效应，为什么nginx还去实现一下？
因此内核解决比较晚，nginx很早就使用该方法去避免惊群效应了，并且为了适应不同内核版本，一直保留着，用于避免惊群效应。

### 1.2 Nginx中惊群惊群效应

每个进程都有一个reactor（fork出来的）也就是epoll，都监听着一个公共的listenfd。如果该端口有一个（连接）事件时，只有一个进程能accept成功，那么其他epoll_wait都要被唤醒，这样多个子进程在 accept 建立新连接时会有争抢，且子进程数量越多问题越明显，从而造成系统性能下降的现象。

### 1.3 Nginx中惊群效应的解决方案

多个进程的epoll都监听公共的端口会出现惊群现象，那么在nginx中有采用accept_mutex的办法，轮流去监听epoll中的建立连接的事件，保证同一时刻只有一个进程在监听listenfd（用于建立连接，进行accept）

普通事件不会出现惊群效应吗？
出现惊群效应是由于多个进程都监听同一个fd，如listenfd，在接受新的连接（accept）时候，都是监听listenfd，是同一个文件。但是对于普通的读写事件来说，就是clientfd，多个进程中的epoll不会出现重复监听同一个clientfd。也就是说，每个clientfd只可能在某个进程中的epoll中

### 1.4 Nginx中的源码

#### 1.4.1 简述避免惊群效应的源码

**1.对进程加锁**

尝试加锁，加锁成功后会附加上一个标志位NGX_POST_EVENTS

![在这里插入图片描述](https://img-blog.csdnimg.cn/81e2bef003494d749898c04a41fd5b87.png)

**2.通过标志位，将事件加入到队列中**

后续执行ngx_epoll_process_events，才会把事件（accept_events或者events）分别加入到队列中(ngx_posted_accept_events和ngx_posted_events)，只有加入到队列中的事件，在后续才能执行

![在这里插入图片描述](https://img-blog.csdnimg.cn/e2e8eaec99e34d8a92f537619b32e815.png)

**3.执行队列中的事件，并解锁**

接下去就是将队列中的事件执行了，先执行accept事件，然后解锁，再执行普通事件

![在这里插入图片描述](https://img-blog.csdnimg.cn/1f189049a5c4488894702d45a3b859a6.png)

**4.概括：**

如果进程成功上锁，那么会进入NGX_POST_EVENTS状态，那么事件会延迟执行，accept事件和普通事件都会分别加入到各自的队列中，然后再执行
如果进程没有上锁成功，如果检测到普通事件，直接执行普通事件（不可能出现accept事件，只有上锁的进程的epoll才能监听到，是因为在加锁过程中还添加了listen监听事件，没有加锁的进程epoll是没法监听到的）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3c5513fbd5504a9da819bde5cfd9303d.png)

下面是源码实现的细节

#### 1.4.2 ngx_process_events_and_timers

在ngx_worker_process_cycle调用该函数。
该部分主要是对进程之间避免惊群效应和实现负载均衡
让ngx_epoll_process_events去检测事件，加入队列中（也有直接执行的情况）
然后通过ngx_event_process_posted去执行队列中的事件

```
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;
    //timer是用于epoll_wait等待的时间，后面传入的参数
    //delta 记录处理epoll_process_events花费的时间
    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;//时间设置无穷大(实际上就是ngx_msec_t的最大值）
        flags = 0;

    } else {
        flags = NGX_UPDATE_TIME;
        timer = ngx_event_find_timer();//找到最近一个定时器触发所需要的时间

#if (NGX_WIN32)

        /* handle signals from master in case of network inactivity */
    
        if (timer == NGX_TIMER_INFINITE || timer > 500) {
            timer = 500;
        }

#endif
    }

    if (ngx_use_accept_mutex) {
        //用于负载均衡，如果当前进程中连接池中连接数量较多 或者 待连接数比较少，那么进程会采用让出，不执行任务。
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;
    
        } else {
            //尝试加锁
            //1.如果获得锁，会将所有listenfd加入到epoll
            //2.如果没有获得锁，会删除当前进程epoll中的listenfd
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {//加锁
                return;
            }
            //如果持有了锁
            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;//获得锁后的标志位NGX_POST_EVENTS（网络读写事件延迟处理标志）
    
            } else {
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }
    //之前队列中剩下的一些未完成的事件放在队列ngx_posted_next_events中，也就是要优先处理的
    //把这些事件放入ngx_posted_events队列头部，并清空ngx_posted_next_events
    if (!ngx_queue_empty(&ngx_posted_next_events)) {
        ngx_event_move_posted_next(cycle);
        timer = 0;//这些任务是上一轮的,要紧急处理，后续epoll_wait要立刻返回，不延迟
    }
    
    delta = ngx_current_msec;


    //里面执行epoll_wait
    //网络连接事件，放入ngx_posted_accept_events队列
    //网络读写事件，放入ngx_posted_events队列
    (void) ngx_process_events(cycle, timer, flags);//执行ngx_epoll_process_events
    
    delta = ngx_current_msec - delta;
    
    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "timer delta: %M", delta);
    //如果没有加锁成功，那么ngx_posted_accept_events会是空的               
    //执行accept事件（ngx_event_accept）
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    
    if (ngx_accept_mutex_held) {//解锁
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }
    
    ngx_event_expire_timers();
    //执行普通事件
    ngx_event_process_posted(cycle, &ngx_posted_events);

}
```

#### 1.4.3 ngx_trylock_accept_mutex

尝试加锁，非阻塞，立刻返回结果

如果加锁成功，那么将监听连接listenfd（多个端口）加入到epoll中
如果加锁失败，那么将epoll中的listenfd（多个端口）清空

```
ngx_int_t
ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
{
    if (ngx_shmtx_trylock(&ngx_accept_mutex)) {//尝试加锁（通过CAS操作，将mtx->lock设置为当前worker进程的pid，没有占用则mtx->lock为0）

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "accept mutex locked");
    
        //之前已经持有了锁，那么就直接返回，继续监听端口（ngx_accept_events 在 epoll 里不使用）
        //标志位 ngx_accept_mutex_held 为 1 表示当前进程已经获取了 ngx_accept_mutex 锁
        if (ngx_accept_mutex_held && ngx_accept_events == 0) {
            return NGX_OK;
        }
        
        // 注册accept事件（之前没有持有锁，需要注册 epoll 事件监听端口，遍历监听端口列表，加入 epoll 连接事件，开始接受请求）
        if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
            ngx_shmtx_unlock(&ngx_accept_mutex);// 如果监听失败就需要立即解锁，函数结束
            return NGX_ERROR;
        }
    
        // 已经成功将监听事件加入 epoll
        ngx_accept_events = 0;
        ngx_accept_mutex_held = 1;// 设置已经获得锁的标志
    
        return NGX_OK;
    }
    // trylock失败，未获得锁
    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "accept mutex lock failed: %ui", ngx_accept_mutex_held);
                   
    // 若当前进程获取 ngx_accept_mutex 锁失败，并且 ngx_accept_mutex_held 为 1，则为错误情况
    
    // 本次未获得锁，但之前持有锁，也就是说之前在监听端口
    if (ngx_accept_mutex_held) {
        // 遍历监听端口列表，删除 epoll 监听连接事件，不接受请求
        if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }
    
        ngx_accept_mutex_held = 0;//没有获得锁的标志
    }
    
    return NGX_OK;

}
```

### 1.5 补充

进程之间的锁如何实现呢？
通过CAS去设置进程之间共享内存的变量mtx->lock，如果该值是自身pid，表示锁被当前进程占有了，如果该值是0，表示没有进程占有锁.

什么时候会去争夺锁？
通过设置定时器（红黑树），获得下一个定时器触发时间，时间主要用于epoll_wait的等待，时间一到，就执行epoll_wait下面内容，执行完后，在worker process循环中又会回到ngx_process_events_and_timers，获得下一个定时器触发时间，然后去判断锁是否被占用，继续执行，以此循环。

## 2.负载均衡

在nginx不同进程之间，进程采用让出加锁机会的方式来实现负载均衡，通过当前进程拒绝监听新连接的次数ngx_accept_disabled来控制，需要让出的次数。它取决于，当前进程中的总连接数 和 待连接（空闲连接）的数量

### 2.1 实现方式

在ngx_event.c中
ngx_accept_disabled表示当前进程中拒绝accept新连接的次数，也就是说当通过定时器轮询到当前进程的时候，如果ngx_accept_disabled>0，那么就不会去获取accept_mutex锁（当前进程不会将accept_event加入epoll中去），并且ngx_accept_disabled-1

```
void
ngx_process_events_and_timers(ngx_cycle_t *cycle){
	...
		if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
        	//获取accept_mutex锁，用于后续accept连接
    	}
    ...

}
```


在ngx_event_accept.c有下面的内容，表示nginx单进程的所有连接总数的八分之一，减去剩下的空闲连接数量（还没连接的数量）。空闲连接数量小了，那么ngx_accept_disable越大，让出机会就更大了。

在ngx_event_accept.c有下面的内容，表示nginx单进程的所有连接总数的八分之一，减去剩下的空闲连接数量（还没连接的数量）。空闲连接数量小了，那么ngx_accept_disable越大，让出机会就更大了。

```
ngx_accept_disabled = ngx_cycle->connection_n / 8
                              - ngx_cycle->free_connection_n;
```


默认每个进程，每次只处理一条连接，如果想每次处理多条连接，需要开启multi_accept

实现进程间负载均衡的一些变量：
注意这些变量都是每个进程私有的，非共享内存的变量。
ngx_accept_disabled：当前进程连接拒绝的次数，先让出给其他进程
ngx_cycle->connection_n：当前进程已连接的总连接数，如果当前进程已经连接总数比较多的话，那么让出的情况会变大
ngx_cycle->free_connection_n：空闲连接数（待连接数量），还未连接的数量如果比较多，那么让出的情况就会变少

参考连接
https://wenku.baidu.com/view/83da1a1313661ed9ad51f01dc281e53a59025148.html
https://blog.csdn.net/u010285974/article/details/106643940
————————————————
版权声明：本文为CSDN博主「菊头蝙蝠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_21539375/article/details/124774136