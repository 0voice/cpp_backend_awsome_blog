# 【NO.42】这是我见过最详细的Nginx 内存池分析

# **1.为什么要使用内存池**

　　大多数的解释不外乎提升程序的处理性能及减小内存中的碎片，对于性能优化这点主要体现在：
　　（1）系统的malloc/free等内存申请函数涉及到较多的处理，如申请时合适空间的查找，释放时的空间合并。
　　（2）默认的内存管理函数还会考虑多线程的应用，加锁操作会增加开销。
　　（3）每次申请内存的系统态与用户态的切换也及为的消耗性能。
　　对于由于应用的频繁的在堆上分配及释放空间所带来的内存碎片化，其实主流的思想是认为存在的，不过也有人认为这种考虑其实是多余的，在“内存池到底为我们解决了什么问题”一文中则认为，大量的内存申请与释放仅会造成短暂的内存碎片化的产生，并不会引起大量内存的长久碎片化，从而导致最后申请大内存时的完全不可用性。文中认为对于确定的应用均是“有限对象需求”，即在任一程序中申请与释放的对象种类总是有限的，大小也总是有一定重复性的，这样在碎片产生一段时间后，会因为同样的对象申请而消除内存的临时碎片化。

　　不过，综上，内存池有利于提升程序在申请及释放内存时的处理性能这点是确定的。内存池的主要优点有：
　　（1）特殊情况的频繁的较小的内存空间的释放与申请不需要考虑复杂的分配释放方法，有较高的性能。
　　（2）初始申请时通常申请一块较大的连续空间的内存区域，因此进行管理及内存地址对齐时处理非常方便。
　　（3）小块内存的申请通常不用考虑实际的释放操作。

# **2.内存池的原理及实现方法**

内存池的原理基本是内存的提前申请，重复利用。其中主要需要关注的是内存池的初始化，内存分配及内存释放。
内存池的实现方法主要分两种：
一种是固定式，即提前申请的内存空间大小固定，空间划分成固定大小的内存单元以供使用如下图示：

![动图封面](https://pic1.zhimg.com/v2-86aaf64fd29706e5cbb5e02f875af482_720w.jpg?source=d16d100b)



另一种是动态式，即初始申请固定大小的单元，空间不做明确的大小划分，而是根据需要提供合适大小的空间以供使用。
不过，这两种方式在处理大空间的内存申请时的处理方法通常都是采用系统的内存申请调用。

# **3.Nginx的内存池实现**

nginx的内存池设计得非常精妙，它在满足小块内存申请的同时，也处理大块内存的申请请求，同时还允许挂载自己的数据区域
及对应的数据清楚操作。
nginx内存池实现主要是在core/ngx_palloc.{h,c}中，一些支持函数位于os/unix/ngx_alloc.{h,c}中，支持函数主要是对原有的
malloc/free/memalign等函数的封装，对就的函数为：
》ngx_alloc 完成malloc的封装
》ngx_calloc 使用malloc分配空间，同时使用memset完成初始化
》ngx_memalign 会根据系统不同而调用不一样的函数处理，如posix系列使用posix_memalign,windows则不考虑对齐。主要作用
　　　　　　　　　是申请指定的alignment对齐的起始地址的内存空间。
》ngx_free 完成free的封装

nginx内存池中有两个非常重要的结构，一个是ngx_pool_s,主要是作为整个内存池的头部，管理内存池结点链表，大内存链表，
cleanup链表等，具体结构如下：

```text
//该结构维护整个内存池的头部信息

struct ngx_pool_s {
ngx_pool_data_t d; //数据块
size_t max; //数据块大小，即小块内存的最大值
ngx_pool_t *current; //保存当前内存值
ngx_chain_t *chain; //可以挂一个chain结构
ngx_pool_large_t *large; //分配大块内存用，即超过max的内存请求
ngx_pool_cleanup_t *cleanup; //挂载一些内存池释放的时候，同时释放的资源
ngx_log_t *log;
};
```

另一重要的结构为ngx_pool_data_s,这个是用来连接具体的内存池结点的，具体如下：

```text
//该结构用来维护内存池的数据块，供用户分配之用
typedef struct {
u_char *last; //当前内存分配结束位置，即下一段可分配内存的起始位置
u_char *end; //内存池结束位置
ngx_pool_t *next; //链接到下一个内存池
ngx_uint_t failed;//统计该内存池不能满足分配请求的次数
} ngx_pool_data_t;
```

还有另两个结构ngx_pool_large_t,ngx_pool_cleanup_t，如下示：

```text
//大内存结构
struct ngx_pool_large_s {
ngx_pool_large_t *next; //下一个大块内存
void *alloc;//nginx分配的大块内存空间
};

struct ngx_pool_cleanup_s {
ngx_pool_cleanup_pt handler; //数据清理的函数句柄
void *data; //要清理的数据
ngx_pool_cleanup_t *next; //连接至下一个
};
```

然后我们具体看一下nginx内存池的组成结构，如下图示：

![img](https://pic3.zhimg.com/80/v2-14986487e035e3d72b43ffff116c313e_720w.webp)

上面的图中，current指针是指向的首结点，在具体的运行过程中是会根据failed值进行调整的。还有就是
ngx_pool_cleanup_s与ngx_pool_large_s的结构空间均来自内存池结点。

**然后看nginx相关的操作：** 

## **3.1创建内存池**

内存池的创建是在ngx_create_pool函数中完成的，实现如下：

```text
//创建内存池
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;
    //ngx_memalign实际上会依据os不用，分情况处理，在os不支持memalign情况的分配时，选择直接分配内存
    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);  // 分配内存函数，uinx,windows分开走
    if (p == NULL) {
        return NULL;
    }

    p->d.last = (u_char *) p + sizeof(ngx_pool_t); //初始指向 ngx_pool_t 结构体后面
    p->d.end = (u_char *) p + size; //整个结构的结尾后面
    p->d.next = NULL;
    p->d.failed = 0;

    size = size - sizeof(ngx_pool_t);
    //实际上pool数据区的大小与系统页的大小有关的
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    //最大不超过 NGX_MAX_ALLOC_FROM_POOL,也就是getpagesize()-1 大小
    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```

## **3.2销毁内存池**

内存池的销毁位于ngx_destroy_pool(ngx_pool_t *pool)中，此函数会清理所有的内存池结点，同时清理large链表的内存
并且对于注册的cleanup链表的清理操作也会进行。具体实现如下：

```text
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;
    //会先调用cleanup函数进行清理操作,不过这儿是对自己指向的数据进行清理
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);
        }
    }
    //这儿是作大数据块的清除
    for (l = pool->large; l; l = l->next) {

        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);

        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }

#if (NGX_DEBUG)

    /*
     * we could allocate the pool->log from this pool
     * so we cannot use this log while free()ing the pool
     */

    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: %p, unused: %uz", p, p->d.end - p->d.last);

        if (n == NULL) {
            break;
        }
    }

#endif

    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);

        if (n == NULL) {
            break;
        }
    }
}
```

## **3.3分配内存**

从内存池中分配内存涉及到几个函数，如下：

```text
void *ngx_palloc(ngx_pool_t *pool, size_t size); //palloc取得的内存是对齐的
void *ngx_pnalloc(ngx_pool_t *pool, size_t size); //pnalloc取得的内存是不对齐的
void *ngx_pcalloc(ngx_pool_t *pool, size_t size); //pcalloc直接调用palloc分配好内存，然后进行一次0初始化操作
void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment); //在分配size大小的内存，并按照alignment对齐，然后挂到large字段下
static void *ngx_palloc_block(ngx_pool_t *pool, size_t size); //申请新的内存池结点
static void *ngx_palloc_large(ngx_pool_t *pool, size_t size); //申请大的内存块
```

下面仅对部分函数进行源码的分析：
首先来看ngx_palloc()函数，其源码为：

```text
//有内存对齐的空间申请
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    ngx_pool_t  *p;

    if (size <= pool->max) {
        //从current遍历到链表末尾,不找前面原因其实是因为failed的控制机制，保证前面的节点
        //基本处于満的状态。仅剩余部分小块区域。
        p = pool->current;

        do {
            m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT); // 对齐内存指针，加快存取速度

            if ((size_t) (p->d.end - m) >= size) {
                p->d.last = m + size;

                return m;
            }

            p = p->d.next;

        } while (p);
        //遍历结束也不能找到合适的可以满足申请要求的结点则新建结点
        return ngx_palloc_block(pool, size);
    }
    //申请大内存时的处理
    return ngx_palloc_large(pool, size);
}
```

其中涉及到两个函数，分别为ngx_palloc_block，ngx_palloc_large先来看ngx_palloc_block，其源码如下：

```text
//申请新的内存池块
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new, *current;

    psize = (size_t) (pool->d.end - (u_char *) pool);

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;
    //这儿有个细节，新的节点可以用ngx_pool_t指针表示，但具体的数据存储则是ngx_pool_data_t.
    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;
    //这儿是调整current指针，每一次空间申请失败都会导致current至内存池链表结尾的
    //结点的failed次数加1，这样在连续分配时，当前current其后的几个结点，其实也差不多
    //处于饱和状态，然后这时将current一次调至失败次数较小的结点是合理的，不过判断跳转时机
    //是依据经验值的。
    current = pool->current;

    for (p = current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            current = p->d.next;
        }
    }

    p->d.next = new;

    pool->current = current ? current : new;

    return m;
}
```

然后是ngx_palloc_large,其源码如下：

```text
//控制大块内存的申请
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;
    //ngx_alloc仅是对alloc的简单封装
    p = ngx_alloc(size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    n = 0;

    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }

        if (n++ > 3) {
            break;
        }
    }

    large = ngx_palloc(pool, sizeof(ngx_pool_large_t));
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```

## **3.4其余的函数**

内存池中还一些其它支持函数，这里不细说了：

```text
ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size);
void ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd);
void ngx_pool_cleanup_file(void *data);
void ngx_pool_delete_file(void *data);
```

# 4.下面是一全例子

这个例子主要是演示下nginx内存池的使用，代码如下：

```text
/*
 * author:doop-ymc
 * date:2013-11-11
 * version:1.0
 */

#include <stdio.h>
#include "ngx_config.h"
#include "ngx_conf_file.h"
#include "nginx.h"
#include "ngx_core.h"
#include "ngx_string.h"
#include "ngx_palloc.h"

#define MY_POOL_SIZE 5000

volatile ngx_cycle_t  *ngx_cycle;

void 
ngx_log_error_core(ngx_uint_t level, ngx_log_t *log, ngx_err_t err,
    const char *fmt, ...)
{

}

void 
echo_pool(ngx_pool_t* pool)
{
    int                  n_index;
    ngx_pool_t          *p_pool;
    ngx_pool_large_t    *p_pool_large;

    n_index = 0;
    p_pool = pool;
    p_pool_large = pool->large;

    printf("------------------------------\n");
    printf("pool begin at: 0x%x\n", pool);

    do{
        printf("->d         :0x%x\n", p_pool);
        printf("        last = 0x%x\n", p_pool->d.last);
        printf("        end  = 0x%x\n", p_pool->d.end);
        printf("        next = 0x%x\n", p_pool->d.next);
        printf("      failed = %d\n", p_pool->d.failed);
        p_pool = p_pool->d.next;
    }while(p_pool);
    printf("->max       :%d\n", pool->max);
    printf("->current   :0x%x\n", pool->current);
    printf("->chain     :0x%x\n", pool->chain);
    
    if(NULL == p_pool_large){
        printf("->large     :0x%x\n", p_pool_large);
    }else{
        do{
            printf("->large     :0x%x\n", p_pool_large);
            printf("        next = 0x%x\n", p_pool_large->next);
            printf("       alloc = 0x%x\n", p_pool_large->alloc);
            p_pool_large = p_pool_large->next;
        }while(p_pool_large);
    }
    
    printf("->cleanup   :0x%x\n", pool->cleanup);
    printf("->log       :0x%x\n\n\n", pool->log);
    
}

int main()
{
    ngx_pool_t *my_pool;

    /*create pool size:5000*/
    my_pool = ngx_create_pool(MY_POOL_SIZE, NULL);
    if(NULL == my_pool){
        printf("create nginx pool error,size %d\n.", MY_POOL_SIZE);
        return 0;
    }
    printf("+++++++++++CREATE NEW POOL++++++++++++\n");
    echo_pool(my_pool);

    printf("+++++++++++ALLOC 2500+++++++++++++++++\n");
    ngx_palloc(my_pool, 2500);
    echo_pool(my_pool);

    printf("+++++++++++ALLOC 2500+++++++++++++++++\n");
    ngx_palloc(my_pool, 2500);
    echo_pool(my_pool);

    printf("+++++++++++ALLOC LARGE 5000+++++++++++\n");
    ngx_palloc(my_pool, 5000);
    echo_pool(my_pool);

    printf("+++++++++++ALLOC LARGE 5000+++++++++++\n");
    ngx_palloc(my_pool, 5000);
    echo_pool(my_pool);

    ngx_destroy_pool(my_pool);
    return 0;

}
```

Makefile文件：

```text
CC = gcc
CFLAGS += -W -Wall -g 

NGX_ROOT_PATH = /home/doop-ymc/nginx/nginx-1.0.14

TARGETS = pool_t
TARGETS_C_FILE = $(TARGETS).c

all: $(TARGETS)

.PHONY:clean

clean:
    rm -f $(TARGETS) *.o

INCLUDE_PATH = -I. \
            -I$(NGX_ROOT_PATH)/src/core \
            -I$(NGX_ROOT_PATH)/src/event \
            -I$(NGX_ROOT_PATH)/src/event/modules \
            -I$(NGX_ROOT_PATH)/src/os/unix \
            -I$(NGX_ROOT_PATH)/objs \

CORE_DEPS = $(NGX_ROOT_PATH)/src/core/nginx.h \
    $(NGX_ROOT_PATH)/src/core/ngx_config.h \
    $(NGX_ROOT_PATH)/src/core/ngx_core.h \
    $(NGX_ROOT_PATH)/src/core/ngx_log.h \
    $(NGX_ROOT_PATH)/src/core/ngx_palloc.h \
    $(NGX_ROOT_PATH)/src/core/ngx_array.h \
    $(NGX_ROOT_PATH)/src/core/ngx_list.h \
    $(NGX_ROOT_PATH)/src/core/ngx_hash.h \
    $(NGX_ROOT_PATH)/src/core/ngx_buf.h \
    $(NGX_ROOT_PATH)/src/core/ngx_queue.h \
    $(NGX_ROOT_PATH)/src/core/ngx_string.h \
    $(NGX_ROOT_PATH)/src/core/ngx_parse.h \
    $(NGX_ROOT_PATH)/src/core/ngx_inet.h \
    $(NGX_ROOT_PATH)/src/core/ngx_file.h \
    $(NGX_ROOT_PATH)/src/core/ngx_crc.h \
    $(NGX_ROOT_PATH)/src/core/ngx_crc32.h \
    $(NGX_ROOT_PATH)/src/core/ngx_murmurhash.h \
    $(NGX_ROOT_PATH)/src/core/ngx_md5.h \
    $(NGX_ROOT_PATH)/src/core/ngx_sha1.h \
    $(NGX_ROOT_PATH)/src/core/ngx_rbtree.h \
    $(NGX_ROOT_PATH)/src/core/ngx_radix_tree.h \
    $(NGX_ROOT_PATH)/src/core/ngx_slab.h \
    $(NGX_ROOT_PATH)/src/core/ngx_times.h \
    $(NGX_ROOT_PATH)/src/core/ngx_shmtx.h \
    $(NGX_ROOT_PATH)/src/core/ngx_connection.h \
    $(NGX_ROOT_PATH)/src/core/ngx_cycle.h \
    $(NGX_ROOT_PATH)/src/core/ngx_conf_file.h \
    $(NGX_ROOT_PATH)/src/core/ngx_resolver.h \
    $(NGX_ROOT_PATH)/src/core/ngx_open_file_cache.h \
    $(NGX_ROOT_PATH)/src/core/ngx_crypt.h \
    $(NGX_ROOT_PATH)/src/event/ngx_event.h \
    $(NGX_ROOT_PATH)/src/event/ngx_event_timer.h \
    $(NGX_ROOT_PATH)/src/event/ngx_event_posted.h \
    $(NGX_ROOT_PATH)/src/event/ngx_event_busy_lock.h \
    $(NGX_ROOT_PATH)/src/event/ngx_event_connect.h \
    $(NGX_ROOT_PATH)/src/event/ngx_event_pipe.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_time.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_errno.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_alloc.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_files.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_channel.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_shmem.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_process.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_setproctitle.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_atomic.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_gcc_atomic_x86.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_thread.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_socket.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_os.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_user.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_process_cycle.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_linux_config.h \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_linux.h \
    $(NGX_ROOT_PATH)/src/core/ngx_regex.h \
    $(NGX_ROOT_PATH)/objs/ngx_auto_config.h


NGX_PALLOC = $(NGX_ROOT_PATH)/objs/src/core/ngx_palloc.o
NGX_STRING = $(NGX_ROOT_PATH)/objs/src/core/ngx_string.o
NGX_ALLOC = $(NGX_ROOT_PATH)/objs/src/os/unix/ngx_alloc.o

$(TARGETS): $(TARGETS_C_FILE) $(NGX_PALLOC) $(NGX_STRING) $(NGX_ALLOC)
    $(CC) $(CFLAGS) $(INCLUDE_PATH)  $^ -o $@

$(NGX_PALLOC):$(CORE_DEPS) \
    $(NGX_ROOT_PATH)/src/core/ngx_palloc.c
    $(CC) -c -g -o0 $(INCLUDE_PATH) -o $(NGX_PALLOC) $(NGX_ROOT_PATH)/src/core/ngx_palloc.c

$(NGX_ALLOC):$(CORE_DEPS) \
    $(NGX_ROOT_PATH)/src/os/unix/ngx_alloc.c
    $(CC) -c -g -o0 $(INCLUDE_PATH) -o $(NGX_ALLOC) $(NGX_ROOT_PATH)/src/os/unix/ngx_alloc.c

$(NGX_STRING):$(CORE_DEPS) \
    $(NGX_ROOT_PATH)/src/core/ngx_string.c
    $(CC) -c -g -o0 $(INCLUDE_PATH) -o $(NGX_STRING) $(NGX_ROOT_PATH)/src/core/ngx_string.c
```

运行结果：

```text
[root@localhost pool]# ./pool_t 
+++++++++++CREATE NEW POOL++++++++++++
------------------------------
pool begin at: 0x8f33020
->d         :0x8f33020
        last = 0x8f33048
        end  = 0x8f343a8
        next = 0x0
      failed = 0
->max       :4960
->current   :0x8f33020
->chain     :0x0
->large     :0x0
->cleanup   :0x0
->log       :0x0


+++++++++++ALLOC 2500+++++++++++++++++
------------------------------
pool begin at: 0x8f33020
->d         :0x8f33020
        last = 0x8f33a0c
        end  = 0x8f343a8
        next = 0x0
      failed = 0
->max       :4960
->current   :0x8f33020
->chain     :0x0
->large     :0x0
->cleanup   :0x0
->log       :0x0


+++++++++++ALLOC 2500+++++++++++++++++
------------------------------
pool begin at: 0x8f33020
->d         :0x8f33020
        last = 0x8f33a0c
        end  = 0x8f343a8
        next = 0x8f343c0
      failed = 0
->d         :0x8f343c0
        last = 0x8f34d94
        end  = 0x8f35748
        next = 0x0
      failed = 0
->max       :4960
->current   :0x8f33020
->chain     :0x0
->large     :0x0
->cleanup   :0x0
->log       :0x0


+++++++++++ALLOC LARGE 5000+++++++++++
------------------------------
pool begin at: 0x8f33020
->d         :0x8f33020
        last = 0x8f33a14
        end  = 0x8f343a8
        next = 0x8f343c0
      failed = 0
->d         :0x8f343c0
        last = 0x8f34d94
        end  = 0x8f35748
        next = 0x0
      failed = 0
->max       :4960
->current   :0x8f33020
->chain     :0x0
->large     :0x8f33a0c
        next = 0x0
       alloc = 0x8f35750
->cleanup   :0x0
->log       :0x0


+++++++++++ALLOC LARGE 5000+++++++++++
------------------------------
pool begin at: 0x8f33020
->d         :0x8f33020
        last = 0x8f33a1c
        end  = 0x8f343a8
        next = 0x8f343c0
      failed = 0
->d         :0x8f343c0
        last = 0x8f34d94
        end  = 0x8f35748
        next = 0x0
      failed = 0
->max       :4960
->current   :0x8f33020
->chain     :0x0
->large     :0x8f33a14
        next = 0x8f33a0c
       alloc = 0x8f36ae0
->large     :0x8f33a0c
        next = 0x0
       alloc = 0x8f35750
->cleanup   :0x0
->log       :0x0
```

总结：从上面的例子初步能看出一些nginx pool使用的轮廓，不过这儿没有涉及到failed的处理。

# 5.内存池的释放

这部分在nginx中主要是利用其自己web server的特性来完成的；web server总是不停的接受连接及
请求，nginx中有不同等级的内存池，有进程级的，连接级的及请求级的，这样内存总会在对应的进程，
连接，或者请求终止时进行内存池的销毁。

原文链接：https://zhuanlan.zhihu.com/p/481302777

原文作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)