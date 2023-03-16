# 【NO.475】nginx 中数据结构讲解

nginx为了做到跨平台， 定义、封装了一些基本的数据结构。由于nginx 对内存分配比较“吝啬”（当然咯，只有保证低内存消耗，才可能实现十万甚至百万级别的同时并发连接数），所以nginx 的数据结构天生都是尽可能少占用内存。学习优秀的代码，未来才能设计好整个系统平台。
可重点看ngx_pool_t 的设计思想

ngx_array_t 数据结构
src/core

在 Nginx 数组中，内存分配是基于内存池的，并不是固定不变的，也不是需要多少内存就申请多少，若当前内存不足以存储所需元素时，按照当前数组的两倍内存大小进行申请，这样做减少内存分配的次数，提高效率。

数据结构定义：

```
typedef struct {
    void        *elts;      // 指向数组数据区域的首地址
    ngx_uint_t   nelts;     // 数组实际数据的个数
    size_t       size;      // 单个元素所占据的字节大小
    ngx_uint_t   nalloc;    // 数组容量 
    ngx_pool_t  *pool;      // 数组对象所在的内存池 
} ngx_array_t;
```


数据结构图：

```
基本操作
/* 创建新的动态数组 */
ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
/* 销毁数组对象，内存被内存池回收 */
void ngx_array_destroy(ngx_array_t *a);
/* 在现有数组中增加一个新的元素 */
void *ngx_array_push(ngx_array_t *a);
/* 在现有数组中增加 n 个新的元素 */
void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);
```


示例代码：

```
#include <stdio.h>
#include <string.h>
#include "ngx_config.h"
#include "ngx_core.h"
#include "ngx_list.h"
#include "ngx_palloc.h"
#include "ngx_string.h"

#define N 10

typedef struct Key {
    int id;
    char name[32];
}Key;

volatile ngx_cycle_t *ngx_cycle;
void ngx_log_error_core(ngx_uint_t level, ngx_log_t *log,
			ngx_err_t err, const char *fmt, ...)
{
}

void print_array(ngx_array_t* arr)
{
    Key* key = arr->elts;

    int i = 0;
    for(i = 0; i < arr->nelts; i ++)
    {
        printf("%s.\n", key[i].name);
    }

}

int main()
{
    printf("Key = %d\n", sizeof(Key));   // 36
    printf("ngx_variable_value_t = %d\n", sizeof(ngx_variable_value_t));   // 16 = 8 + 8
    ngx_pool_t* pool = ngx_create_pool(1024, NULL);
    ngx_array_t* array = ngx_array_create(pool, N, sizeof(Key));

    int i = 0;
    Key* key = NULL;
    for(i = 0; i < 24; i ++) 
    {
        key = ngx_array_push(array);
        key->id = i + 1;
        sprintf(key->name, "Test %d", key->id);
    }
    
    key = ngx_array_push_n(array, 10);
    for(i = 0; i < 10; i ++)
    {
        key[i].id = 24 + i + 1;
        sprintf(key[i].name, "Other Test %d", key[i].id);
    }
    print_array(array);
    return 0;

}

// 编译命令
// gcc -o ngx_array_main ngx_array_main.c  -I ../nginx_folds/nginx/src/core/  -I ../nginx_folds/nginx/objs/ -I ../nginx_folds/nginx/src/os/unix/ -I ../nginx_folds/pcre-8.41/ -I ../nginx_folds/nginx/src/event/  ../nginx_folds/nginx/objs/src/core/ngx_list.o ../nginx_folds/nginx/objs/src/core/ngx_string.o ../nginx_folds/nginx/objs/src/core/ngx_palloc.o ../nginx_folds/nginx/objs/src/os/unix/ngx_alloc.o ../nginx_folds/nginx/objs/src/core/ngx_array.o
```

ngx_list_t 数据结构
ngx_list_t 是 Nginx 封装的链表容器，链表容器内存分配是基于内存池进行的，操作方便，效率高。Nginx 链表容器和普通链表类似，均有链表表头和链表节点，通过节点指针组成链表。

数据结构定义如下

```
/* 链表结构 */
typedef struct ngx_list_part_s  ngx_list_part_t;

/* 链表中的节点结构 */
struct ngx_list_part_s {
    void             *elts; /* 指向该节点数据区的首地址 */
    ngx_uint_t        nelts;/* 该节点数据区实际存放的元素个数 */
    ngx_list_part_t  *next; /* 指向链表的下一个节点 */
};

/* 链表表头结构 */
typedef struct {
    ngx_list_part_t  *last; /* 指向链表中最后一个节点 */
    ngx_list_part_t   part; /* 链表中表头包含的第一个节点 */
    size_t            size; /* 元素的字节大小 */
    ngx_uint_t        nalloc;/* 链表中每个节点所能容纳元素的个数 */
    ngx_pool_t       *pool; /* 该链表节点空间的内存池对象 */
} ngx_list_t;
```

数据结构图：

```
基本操作
/* 创建链表 */
ngx_list_t * ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);

/* 初始化链表 */
static ngx_inline ngx_int_t ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size);

/* 添加一个元素 */
void *ngx_list_push(ngx_list_t *l);
```



```
示例代码
#include <stdio.h>
#include <string.h>
#include "ngx_config.h"
#include "ngx_core.h"
#include "ngx_list.h"
#include "ngx_palloc.h"
#include "ngx_string.h"

#define N	10
volatile ngx_cycle_t *ngx_cycle;

void ngx_log_error_core(ngx_uint_t level, ngx_log_t *log,
			ngx_err_t err, const char *fmt, ...)
{

}


void print_list(ngx_list_t *l) {
	ngx_list_part_t *p = &(l->part);
	

	while (p) {
	
		int i = 0;
		for (i = 0;i < p->nelts;i ++) {
			printf("%s\n", (char*)(((ngx_str_t*)p->elts + i)->data));
		}
		p = p->next;
		printf(" -------------------------- \n");
	}

}

// typedef struct {
//     size_t      len;  // long unsigned int
//     u_char     *data;
// } ngx_str_t;

int main() {

	// 4, 4, 8, 8, 16
	printf("int = %ld\n", sizeof(int));
	printf("unsigned int = %ld\n", sizeof(unsigned int));
	printf("long = %ld\n", sizeof(long));
	printf("long unsigned int = %ld\n", sizeof(long unsigned int));
	printf("sizeof(ngx_str_t) = %d\n", sizeof(ngx_str_t));   // 16
	
	ngx_pool_t *pool = ngx_create_pool(1024, NULL);
	
	ngx_list_t *l = ngx_list_create(pool, N, sizeof(ngx_str_t));
	
	int i = 0;
	for (i = 0;i < 24;i ++) {
	
		ngx_str_t *ptr = ngx_list_push(l);
		
		char *buf = ngx_palloc(pool, 32);
		sprintf(buf, "MyList %d node", i+1);
	
		ptr->len = strlen(buf);
		ptr->data = buf;
	}
	
	print_list(l);
	return 0;

}

// 编译命令
// gcc -o ngx_list_main ngx_list_main.c  -I ../nginx_folds/nginx/src/core/  -I ../nginx_folds/nginx/objs/ -I ../nginx_folds/nginx/src/os/unix/ -I ../nginx_folds/pcre-8.41/ -I ../nginx_folds/nginx/src/event/  ../nginx_folds/nginx/objs/src/core/ngx_list.o ../nginx_folds/nginx/objs/src/core/ngx_string.o ../nginx_folds/nginx/objs/src/core/ngx_palloc.o ../nginx_folds/nginx/objs/src/os/unix/ngx_alloc.o 
```

首先在说明ngx_pool_t内存池前，先介绍相关的15个方法

内存池操作
ngx_create_pool
ngx_destroy_pool
ngx_reset_pool

基于内存池的分配、释放内存操作
ngx_palloc : 分配地址对齐的内存。按总线的长度（例sizeof（unsigned long）对齐地址后，可以减少CPU读取内存的次数，当然代价是有一些内存浪费）
ngx_pnalloc ： 分配内存时不进行地址对齐
ngx_pcalloc ： 分配出地址对齐的内存后，在调用memset 将这些内存全部清0
ngx_pmemalign ： 按照alignment进行地址对齐来分配内存。注意，这样分配出的内存不管申请的size大小，都是不会使用小块内存池的，它会从进程的堆中分配内存，并挂在大块内存组成的large单链表中
ngx_pfree ： 提前释放大块内存。它的效率不高，其实就是遍历large链表，寻找ngx_pool_large_t 的alloc 成员等于待释放地址，找到后释放内存给操作系统，将ngx_pool_large_t 移出链表并删除

随着内存池释放同步释放资源的操作
ngx_pool_cleanup_add : 添加一个需要在内存池释放时同步释放的资源。
ngx_pool_run_cleanup_file ： 在内存池释放前，如果需要提前关闭文件（当然是调用过ngx_pool_cleanup_add 添加的文件，同时ngx_pool_cleanup_t 的handle成员被设为ngx_pool_cleanup_file）, 则调用该方法
ngx_pool_cleanup_file ： 以关闭文件来释放资源的方法，可以设置到ngx_pool_cleanup_t 的handle 成员
ngx_pool_delete_file ： 以删除文件来释放资源的方法，可以设置到ngx_pool_cleanup_t 的handle 成员

与内存池无关的分配、释放操作
ngx_alloc ： 从操作系统中分配内存
ngx_calloc ： 从操作系统中分配内存，在调用memset 把内存清0
ngx_free ： 释放内存到操作系统

Nginx 已经提供封装了malloc、free的ngx_alloc 、ngx_free 方法，那为什么还需要一个复杂的内存池呢？对于没有垃圾回收机制的c语言编写引用来说，最容易犯的错就是内存泄露。当分配内存与释放内存的逻辑相距遥远时，还很容易发生同一块内存被释放两次。内存池就是来降低程序员犯错误的机率的。模块开发者只需要关心内存的分配，而释放内存则交个内存池来释放。

ngx_pool_t内存池的设计上还考虑了小块内存的频繁分配在效率上有提升空间，以及内存碎片还可以在减少些。不过在这里需要定义一下什么叫小块内存，NGX_MAX_ALLOC_FROM_POOL宏是一个很重要的标准：
#NGX_MAX_ALLOC_FROM_POOL (ngx_pageisze -1)
可见，在x86架构上就是4095 字节，通常，小于等于NGX_MAX_ALLOC_FROM_POOL 就意味这小块内存。这也不是绝对的，当调用ngx_create_pool 创建内存池时，如果传递的size参数小于NGX_MAX_ALLOC_FROM_POOL + sizeof(ngx_pool_t)， 则对于这个内存池来说，size - sizeof(ngx_pool_t) 字节就是小块内存的标准。大块内存和小块内存的处理规则不同，这个在源码中看处理逻辑就知道了。

下面是有关ngx_pool_t 结构的一些数据结构：

```
// src/score/ngx_core.h
// 首先说一下这个变量问题，这里_s 和 _t 为什么这样定义，我搜到的答案给我的解释其中合理的是：
// _s 指的是struct 变量;  _t 指的是 某一个type类型
#include <ngx_config.h>

typedef struct ngx_module_s          ngx_module_t;
typedef struct ngx_conf_s            ngx_conf_t;
typedef struct ngx_cycle_s           ngx_cycle_t;
typedef struct ngx_pool_s            ngx_pool_t;
typedef struct ngx_chain_s           ngx_chain_t;
typedef struct ngx_log_s             ngx_log_t;
typedef struct ngx_open_file_s       ngx_open_file_t;
typedef struct ngx_command_s         ngx_command_t;
typedef struct ngx_file_s            ngx_file_t;
typedef struct ngx_event_s           ngx_event_t;
```



```
src/core/ngx_palloc.{h,c}

typedef struct {
	// 当前内存分配结束位置，即下一段可分配内存的位置 (指向未分配的空闲内存的首地址)
    u_char               *last;	
    u_char               *end;	// 小块内存池结束位置
    ngx_pool_t           *next;	// 指向下一内存的指针，内存池是通过链表连接的
    ngx_uint_t            failed;	//记录内存池分配失败的次数（4.4之后移向下一个小块内存池）
} ngx_pool_data_t;   // 内存池的数据结构模块
```



```
struct ngx_pool_s {
	// 内存池的数据块，结构如上，描述是小块内存池。当分配小块内存，剩余的空间不足时，会再分配1个ngx_pool_t， 它们会通过d中next成员构成单链表
    ngx_pool_data_t       d;		
    size_t                max;		// 评估申请内存属于小块内存还是大块内存的标准（=小块内存的最大值）
	

	// 多个小块内存池构成链表时，current 指向分配内存时遍历的第1个小块内存池
	ngx_pool_t           *current;
	
	// 与内存池关系不大
	ngx_chain_t          *chain;	// 指针指向chain结构
	
	// 大块内存都直接从进程的堆中分配，为了能够在销毁内存池时同时释放大块内存，就把
	// 每一次分配的大块内存通过ngx_pool_large_t 组成单链表挂在large成员上
	ngx_pool_large_t     *large;	// 指向大块内存链表，超过max的内存分配是不同规则的
	
	// 所有待清理资源（例如需要关闭或者删除的文件）以ngx_pool_cleanup_t 对象构成单链表挂在cleanup 成员上
	ngx_pool_cleanup_t   *cleanup;	// 释放内存池指针，内部包含函数指针结构，类似析构函数
	// 内存池执行中输出日志的对象
	ngx_log_t            *log;		// 内存分配的日志指针

};

// src/core/ngx_buf.h
struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};

// ngx_palloc.h
struct ngx_pool_large_s {
	// 所有大块内存通过next 指针连在一起
    ngx_pool_large_t     *next;
    void                 *alloc;
};


struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;
    void                 *data;
    ngx_pool_cleanup_t   *next;
};
```


从ngx_pool_s 结构中，可以知道，当申请的内存算是大块内存时(大于ngx_pool_t 的max成员)，是直接调用ngx_alloc 从进程的堆中分配的，同时会再分配一个ngx_pool_large_t 结构体挂在large链表中，其定义如上面的 ngx_pool_large_s ：

对于非常大的内存，如果它的生命周期远远的短于所述的内存池，那么在内存池销毁前提前的释放它就变得有意义了。而ngx_free 方法就是提前释放大块内存的，需要注意，它的实现是遍历large 链表，找打alloc 等于待师范地址的ngx_pool_large_t 后，调用ngx_free释放大块内存，但不是房ngx_pool_large_t结构体，而是把alloc 置为NULL。如此实现的意义是下次分配大块内存时，能复用这个结构体。所以可以想见，如果large 链表中的元素很多，那么ngx_free 的遍历损耗是很大，因此，最好不要调用ngx_pfree。

在看看小块内存，通过从进程的堆中预分配更过的内存（ngx_create_pool 的size参数决定分配的大小），而后直接使用这块内存的一部分作为小块内存返回给申请者，以此实现减少碎片和调用malloc 的次数。它们是放在成员d中维护管理的，看看ngx_pool_data_t 是如何定义。

```
//src/core/ngx_palloc.h  
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p; // 这里就占80字节，里面有8个指针，两个long int 变量

    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL) {
        return NULL;
    }
    
    p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    p->d.end = (u_char *) p + size;
    p->d.next = NULL;
    p->d.failed = 0;
    
    size = size - sizeof(ngx_pool_t);
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    
    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;
    
    return p;

}
```



```
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;

    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);
        }
    }

#if (NGX_DEBUG)

    /*
     * we could allocate the pool->log from this pool
     * so we cannot use this log while free()ing the pool
     */
    
    for (l = pool->large; l; l = l->next) {
        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);
    }
    
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: %p, unused: %uz", p, p->d.end - p->d.last);
    
        if (n == NULL) {
            break;
        }
    }

#endif

    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }
    
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);
    
        if (n == NULL) {
            break;
        }
    }

}
```


下图是将ngx_pool_t 的内存逻辑画出来，可以直观地参考下图进行学习：

下图是将ngx_pool_t 的内存逻辑画出来，可以直观地参考下图进行学习：

下面写了一个演示代码：
附上运行结果，可结合上面的逻辑图来看，还是比较清楚的。

```
pool_test.cpp


#include <iostream>
#include <string>

extern "C" {
    #include "ngx_config.h"
    #include "ngx_conf_file.h"
    #include "nginx.h"
    #include "ngx_core.h"
    #include "ngx_string.h"
    #include "ngx_palloc.h"
}
using namespace std;

volatile ngx_cycle_t  *ngx_cycle;

void ngx_log_error_core(ngx_uint_t level, ngx_log_t *log, ngx_err_t err,
            const char *fmt, ...)
{
}

void dump_pool(ngx_pool_t* pool)
{
    printf("------------start--------------\n");
    while (pool)
    {
        printf("pool = 0x%x\n", pool);
        printf("  .d\n");
        printf("    .last = 0x%x\n", pool->d.last);
        printf("    .end = 0x%x\n", pool->d.end);
        printf("    .next = 0x%x\n", pool->d.next);
        printf("    .failed = %d\n", pool->d.failed);
        printf("  .max = %d\n", pool->max);
        printf("  .current = 0x%x\n", pool->current);
        printf("  .chain = 0x%x\n", pool->chain);
        printf("  .large = 0x%x\n", pool->large);
        printf("  .cleanup = 0x%x\n", pool->cleanup);
        printf("  .log = 0x%x\n", pool->log);
        printf("available pool memory = %d\n", pool->d.end - pool->d.last);
        pool = pool->d.next;
    }
    printf("------------end--------------\n");
}

int main()
{
    ngx_pool_t *pool;

    printf("--------------------------------\n");
    printf("create a new pool:\n");
    pool = ngx_create_pool(1024, NULL);
    dump_pool(pool);


    

    printf("alloc block 1 from the pool:\n");
    // printf("--------------------------------\n");
    ngx_palloc(pool, 512);
    dump_pool(pool);
     
    // printf("--------------------------------\n");
    printf("alloc block 2 from the pool:\n");
    // printf("--------------------------------\n");
    ngx_palloc(pool, 512);
    dump_pool(pool);
     
    printf("alloc block 3 from the pool :\n");
    ngx_palloc(pool, 512);
    dump_pool(pool);
    
    printf("alloc block 4 from the pool :\n");
    ngx_palloc(pool, 512);
    dump_pool(pool);
     
    ngx_destroy_pool(pool);
    return 0;

}
```




--------------------------------

```
create a new pool:
------------start--------------
pool = 0x3f1da2c0
  .d
    .last = 0x3f1da310
    .end = 0x3f1da6c0
    .next = 0x0
    .failed = 0
  .max = 944
  .current = 0x3f1da2c0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 944

------------end--------------
alloc block 1 from the pool:
------------start--------------
pool = 0x3f1da2c0
  .d
    .last = 0x3f1da510
    .end = 0x3f1da6c0
    .next = 0x0
    .failed = 0
  .max = 944
  .current = 0x3f1da2c0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 432

------------end--------------
alloc block 2 from the pool:
------------start--------------
pool = 0x3f1da2c0
  .d
    .last = 0x3f1da510
    .end = 0x3f1da6c0
    .next = 0x3f1da6d0
    .failed = 0
  .max = 944
  .current = 0x3f1da2c0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 432

pool = 0x3f1da6d0
  .d
    .last = 0x3f1da8f0
    .end = 0x3f1daad0
    .next = 0x0
    .failed = 0
  .max = 0
  .current = 0x0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 480

------------end--------------
alloc block 3 from the pool :
------------start--------------
pool = 0x3f1da2c0
  .d
    .last = 0x3f1da510
    .end = 0x3f1da6c0
    .next = 0x3f1da6d0
    .failed = 1                 // 开始只分配了1024个B空间， 分了两次512B空间后，再次分配的就会失败
  .max = 944
  .current = 0x3f1da2c0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 432     // 需要减去pool 表头占的前80 B, 1024 - 512 - 80 

pool = 0x3f1da6d0
  .d
    .last = 0x3f1da8f0
    .end = 0x3f1daad0
    .next = 0x3f1daae0
    .failed = 0
  .max = 0
  .current = 0x0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 480   // 第二个节点只用减去4 * 8 B,  1024 - 512 - 32

pool = 0x3f1daae0
  .d
    .last = 0x3f1dad00
    .end = 0x3f1daee0
    .next = 0x0
    .failed = 0
  .max = 0
  .current = 0x0
  .chain = 0x0
  .large = 0x0
  .cleanup = 0x0
  .log = 0x0
available pool memory = 480     // 1024 - 512 - 32 , 后面每次都是这个数

------------end--------------
```


最后我制作了一个里面包含了我学习nginx的一个镜像， 现在你们只需要执行下面命令就能拉取学习需要的环境了

最后我制作了一个里面包含了我学习nginx的一个镜像， 现在你们只需要执行下面命令就能拉取学习需要的环境了

docker pull registry.cn-hangzhou.aliyuncs.com/aclj/nginx:1.0.1

如果需要项目环境源代码，可以参考此github， 后续可根据需求增加东西，欢迎star~
————————————————
版权声明：本文为CSDN博主「_刘小雨」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_39486027/article/details/124728540