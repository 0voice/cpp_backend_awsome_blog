# 【NO.491】手写内存池以及原理代码分析【C语言】

### 1.内存池是对堆进行管理

当进程执行时，操作系统会分出0~4G的虚拟内存空间给进程，程序员可以自行管理（分配、释放）的部分就是mmap映射区、heap堆区，而内存池管理的部分就是用户进程的堆区。

### 2.为什么要用内存池？

内存池就是用来避免堆区出现碎片化

- **避免频繁地分配和释放内存（防止堆区出现碎片化）**

当客户端连接上服务端的时候，服务端会准备一部分的堆区用来做消息保留。当一个连接成功之后，服务器会在堆区为其分配一段属于这个连接的内存，当连接关闭之后，所分配的内存也随之释放。但是当连接量较大且过于频繁时，不可避免地对内存进行频繁的分配和释放。这会导致堆区出现小窗口，也就是堆区碎片化。

堆区出现碎片化会怎么样？

- 长时间工作会出现不可查的BUG
- 无法分配较大且整块的内存，malloc会返回NULL

### 3.内存池设计

场景：在一段很干净的堆区上，如何实现能避免内存碎片化的内存池？

第一：使用链表管理内存

使用链表将分配出来的一块一块内存在堆区连接起来，设置flag(是否被使用)，让链表节点上的各段内存慢慢各自扩张。

单独使用链表会出现什么问题？

- 内存块会被划分得越来越小，链表会变得越来越长，知道不能划分出更大得内存。

所以加入了固定内存的想法的设计

**对于分配小段内存时，将小段内存进行固定划分，如下**

1. 16bytes
2. 32bytes
3. 64bytes
4. 128bytes
5. 256byts
6. 512bytes

**但是不同的固定小内存的分配的话就会出现以下问题:**

1.查找慢

分配时查找

释放时候查找

2.块与块之间也会出现间隙

无法将块合并

小块回收麻烦

所以最后得出自定义固定小块和随机大块的内存池模型

### 4.内存池

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216163108_67685.jpg)

### 5.内存池工作流程

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216163109_62428.jpg)

### 6.内存池结构体

```
//内存池
struct mp_pool_s{
    size_t max;                    
    struct mp_node_s *current;        //当前指向的小块内存区，链表结构
    struct mp_large_s *large;         //大块内存
    struct mp_node_s *head[0];        //小块内存的头部
};
```

小块内存用单向链表串起来

```
//小块内存
struct mp_node_s{
    unsigned char *last;    //当前的
    unsigned char *end;   //最后
    struct mp_node_s *next; //下一个4k
    size_t failed;
};
```

大块内存也是用单向链表串起来

```
//大块内存
struct mp_large_s{
    struct mp_large_s *next;
    void *alloc;
};
```

### 7.API

创建内存池

1. 确定以size大小作为小块内存的固定大小
2. 一来是last指向所在节点的第一个字节
3. end指向最后

```
//创建内存池
struct mp_pool_s *mp_create_pool(int size){
    struct mp_pool_s *pool;
    int ret = posix_memalign((void**)&pool, ALIGNMENT, size + sizeof(struct mp_pool_s) + sizeof(struct mp_node_s));
    if(ret)return NULL;
    pool->max = (size<MP_MAX_ALLOC_FORM_POOL) ? size : MP_MAX_ALLOC_FORM_POOL;
    pool->current = pool->head;
    pool->head->last = (unsigned char*)pool + sizeof(struct mp_pool_s) + sizeof(struct mp_node_s);
    pool->head->end = pool->head->last + size;
    pool->head->failed = 0;
    pool->large = NULL;
    return pool;
}
```

销毁内存池

1. 先释放大块内存，遍历链表
2. 再释放小块，遍历链表

```
//销毁内存池
void mp_destroy_pool(struct mp_pool_s *pool){
    struct mp_large_s *large;
    for(large = pool->large; large!=NULL; large = large->next){ //释放大块
        if(large->alloc)free(large->alloc);
    }
    struct mp_node_s *h = pool->head->next;
    while(h){    //释放小块
        struct mp_pool_s *next = h->next;
        free(h);
        h = h->next;
    }
    free(pool);
}
```

**重置内存池**

1. 先释放大块内存
2. 再将小块内存的last指向内存的第一个位置

```
//重置内存池
void mp_reset_pool(struct mp_pool_s *pool){
    struct mp_node_s *head;
    struct mp_large_s *large;
    for(large = pool->large; large; large=large->next){ //将释放大块
        if(large->alloc)free(large->alloc);
    }
    pool->large = NULL;
    for(head = pool->head; head; head = head->next){    //将各个节点的头位置指向刚创建的位置
        head->last = (unsigned char*)head + sizeof(struct mp_node_s);
    }
}
```

给内存池分配一个小块内存

1. 首先分配出一整块（大小为psize）小块内存，并创建节点指向这块内存
2. 内存对齐，利用尾插法插入链表最后的位置
3. 重新调整内存池的current的指向

```
/*分配psize大小的小块内存，开始指向的位置head->last = memblk + size，链表尾插法*/
static void *mp_alloc_block(struct mp_pool_s *pool, size_t size){
    unsigned char *memblk;
    struct mp_node_s *head = pool->head;
    size_t psize = (size_t)(head->end - (unsigned char*)head);  //psize == 创建内存池时输入的参数
    int ret = posix_memalign((void*)&memblk, ALIGNMENT, psize);  //分配内存     24字节对齐
    if(ret)return NULL;
    struct mp_node_s *p, *new_node, *current;
    new_node = (struct mp_node_s*)memblk;
    new_node->end = memblk + psize;
    new_node->next = NULL;
    new_node->failed = 0;
    memblk += sizeof(struct mp_node_s);                 //跳过节点结构体做内存对齐
    memblk = mp_align_ptr(memblk, ALIGNMENT);            //内存对齐
    new_node->last = memblk + size;
    current = pool->current;
    for(p = current; p->next; p = p->next){             //尾插法
        if(p->failed++>4)current = p->next;
    }
    p->next = new_node;                            
    pool->current = current ? current : new_node;
    return memblk;
}
```

在小块内存中取出一块size大小的内存（不做字节对齐）

1. 首先获取当前内存池所指向的小块内存
2. 跟着一个节点一个节点往下面查找小块内存中是否有足够size大小的内存分配
3. 如果有就将内存做字节对齐并返回
4. 如果没有合适的小块内存则分配多一整块小块内存
5. 如果小块内存分配失败，则以创建大块内存的方式对size分配

```
//在小块内存区上取一块size大小的内存
void *mp_nalloc(struct mp_pool_s *pool, size_t size){
    unsigned char *m;
    struct mp_node_s  *p;
    if(size<=pool->max){
        p = pool->current;  //当前小块指向
        do{
            m = p->last;        
            if((size_t)(p->end - m)>=size){ //如果当前节点剩余内存比size大的话就在该节点分配
                p->last = m + size;
                return m;
            }
            p = p->last;
            //如果没有一个节的剩余大小大于size就重新分配一个block
        }while(p);
        return mp_alloc_block(pool, size);
    }
    //如果这个size超过了小块内存的限制，就以大块内容分配方式来分配
    return mp_alloc_large(pool, size);
}
```

在小块内存中取出一块size大小的内存（做字节对齐）

```
//在小块内存区上取一块size大小的内存  字节对齐
void *mp_alloc(struct mp_pool_s *pool, size_t size){
    unsigned char *memblk;
    struct mp_pool_s *p;
    if(size<=pool->max){
        p = pool->current;         //当前小块指向
        do{
            memblk = mp_align_ptr(p->last, ALIGNMENT);//做字节对齐
            if((size_t)(p->end-memblk) >= size){  //如果当前节点剩余内存比size大的话就在该节点分配
                p->last = memblk + size;
                return memblk;
            }
            p = p->next;            
        }while(p);
        //如果没有一个节的剩余大小大于size就重新分配一个block
        return mp_alloc_block(pool, size);
    }
    //如果这个size超过了小块内存的限制，就以大块内容分配方式来分配
    return mp_alloc_large(pool, size);
}
```

在小块内存中取出一段size大小的内存并初始化为0

```
//分配内存且初始化为0
void *mp_calloc(struct mp_pool_s *pool, size_t size){
    void *p = mp_alloc(pool, size);
    if(p){
        memset(p, 0 , size);
    }
    return p;
}
```

**分配size大小的大块内存（大块内存的节点放在小块内存里）**

1. 首先malloc得到内存首地址，然后将大内存挂到内存池上，如果大块内存的结构体没有创建就在小块内存中创建
2. 创建好之后挂接好并返回大块内存首地址

```
//创建大块内存
static void *mp_alloc_large(struct mp_pool_s *pool, size_t size){
    void *p = malloc(size);                //分配
    if(p==NULL)return NULL;
    size_t n = 0;
    struct mp_large_s  *large;
    for(large=pool->large; large; large = large->next){
        if(large->alloc == NULL){  
            large->alloc = p;
            return p;
        }
        if(n++>3)break;
    }
    //大内存挂接不成
    large = mp_alloc(pool, sizeof(struct mp_large_s));
    if(large==NULL){
        free(large);
        return NULL;
    }
    large->alloc = p;
    large->next = pool->large;
    pool->large = large;
    return p;
}
```

释放指定大块内存p

- 先查找再释放

```
//释放内存p
void mp_free(struct mp_pool_s *pool, void *p){
    struct mp_large_s * large;
    for(large = pool->large; large; large = large->next){
        if(p==large->alloc){
            free(large->alloc);
            large->alloc = NULL;    
            return;
        }
    }
```

重载posix_memalign

```
//重构mem_memalign
void *mp_memalign(struct mp_pool_s *pool, size_t size, size_t alignment){
    void *p;
    int ret = posix_memalign(&p, alignment, size);
    if(ret){
        return NULL;
    }
    struct mp_large_s *large = mp_alloc(pool, sizeof(struct mp_large_s ));
    if(ret)return NULL;
    struct mp_large_s *large = mp_alloc(pool, sizeof(struct mp_large_s));
    if(large==NULL){
        free(p);
        return NULL;
    }
    large->alloc = p;
    large->next = pool->large;
    pool->large = large;
    return p;
}
```

内存对齐公式

```
#define MP_ALIGNMENT               32
#define MP_PAGE_SIZE            4096
#define MP_MAX_ALLOC_FROM_POOL    (MP_PAGE_SIZE-1)
#define mp_align(n, alignment) (((n)+(alignment-1)) & ~(alignment-1))
#define mp_align_ptr(p, alignment) (void *)((((size_t)p)+(alignment-1)) & ~(alignment-1))
```

至此，内存池全部内容都在这

原文链接：https://zhuanlan.zhihu.com/p/502486049

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)