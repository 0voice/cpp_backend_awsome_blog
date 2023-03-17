# 【NO.539】Linux服务器开发,线程池原理与实现

## 0.前言

## 1.线程池究竟解决了什么问题？

- 可能第一感觉是线程池减少线程创建销毁，但是这是站在线程的角度去思考的。
- 异步解耦的作用 主循环只做一个抛任务，写日志交给线程池，提高了程序运行效率。

**注重cpu的处理能力的时候才会去粘合，注重的任务的话还是不要粘合为好。**

## 2.API

- create、init
- push_task 返回的结果对于业务来说作用不大
- destroy、deinit
  锦上添花的api：task_count,free_thread。

## 3.线程池的作用

- 线程池是管理任务队列和工作队列的管理组件。

## 4.工作原理

- 线程池并非内存池那般，从池子里面取用完归还，而是线程之间争夺资源。

- 通过线程池的入口函数去获取任务，循环往复，再取任务再循环。

- 生产者消费者模型：任务队列是生产者，工作队列是消费者，加锁获取资源。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/720ae11ba22c4e8182e488396a6d1f32.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

## 5.手写线程池

- 可以对任务进行区分，加优先级，更优秀的做法可以创建多个线程池对象。
- 条件等待，进来时候解锁，出来加锁。
- 当往任务队列中添加任务，应该先加锁锁住资源，中间发送条件变量的信号。

```c
#define LL_ADD(item, list) do {     \



    item->prev = NULL;                \



    item->next = list;                \



    if(list !< NULL) list *prev =item  \



    list = item;                    \



} while(0)
```

- 利用宏向链表中添加数据,这个感觉奇妙啊~

  ```c
  while (1) {
  
  
  
        pthread_mutex_lock(&worker->workqueue->jobs_mtx);
  
  
  
        while (worker->workqueue->waiting_jobs == NULL) {
  
  
  
            if (worker->terminate) break;
  
  
  
            pthread_cond_wait(&worker->workqueue->jobs_cond, &worker->workqueue->jobs_mtx);
  
  
  
        }
  
  
  
        if (worker->terminate) {
  
  
  
            pthread_mutex_unlock(&worker->workqueue->jobs_mtx);
  
  
  
            break;}
  
  
  
        nJob *job = worker->workqueue->waiting_jobs;
  
  
  
        if (job != NULL) {
  
  
  
            LL_REMOVE(job, worker->workqueue->waiting_jobs);}
  
  
  
        pthread_mutex_unlock(&worker->workqueue->jobs_mtx);
  
  
  
        if (job == NULL) continue;
  
  
  
        job->job_function(job);
  
  
  
    }
  ```

- 条件变量这块，是先解锁再加锁，King老师反复强调。不停的判断job为空，需要注意一下，担心万一被别的线程占用了怎么办。
  pthread_mutex_lock(&workqueue->jobs_mtx);

```c
workqueue->workers = NULL;



workqueue->waiting_jobs = NULL;



pthread_cond_broadcast(&workqueue->jobs_cond);



pthread_mutex_unlock(&workqueue->jobs_mtx);
```

- 通知等待着的线程，不折不扣的生产者，不过发送信号也需要加锁。防止竞争。

## 6.对比nginx线程池

我们的线程池为双链表，nginx的队列为什么使用二级指针。因为一级指针能指向struct ngx_thread_task_s的第一项ngx_thread_task_t *next的位置，这也正是ngx的巧妙之处。

## 7.总结

通过今天的学习，不仅学习到了有关线程池的知识，更重要的King老师讲述了学习的三个过程：1、逻辑想通；2、代码调通；3、对照学习。目测小生只是刚刚能做好第一步，接下来还是要付出更多的努力才能将这些知识驾驭娴熟。
这节课确实是比较轻松的，不过需要注意的是King老师在添加删除链表时使用的宏方法，这个也不是第一次见到了，希望能掌握好成为自己的东西。线程之间的条件等待也绝不是第一次听说，wait使得所有线程都爬在此处，等待任务队列中有数据，随时触发线程之前抢占资源。最后就是不停的判断job是否为空，因为担心被其他线程抢走。
在未来的时间里，我会利用C++11的std::thread函数从新改造这个线程池，希望达到异曲同工之妙。

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：hhttps://bbs.csdn.net/topics/604976777