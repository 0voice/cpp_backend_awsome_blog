# 【NO.518】malloc内存分配过程详解

起malloc，但凡对C/C++有点基础的人在编写代码的时候都用过。我们调用malloc接口分配一段连续的内存空间，不使用时使用free可以释放这段内存空间。这些我们都已经比较的熟悉了。但是你知道malloc背后的调用机制吗？

C语言程序员都知道，malloc只是C语言库标准提供的一个普通函数，我们实现的malloc和库函数比起来效率要低很多，但是可以通过编写一个简单的malloc来体现C库的精髓，我们实现的malloc和库的实现原理上市一致的。

## 1.malloc的定义：

根据标准C的定义，malloc的函数原型是这样的：

```text
<span style="font-family:Microsoft YaHei;font-size:14px;">void* malloc(size_t size);</span>
```

函数要求如下：

- malloc函数分配的内存大小至少为size参数所指定的字节数。
- malloc的返回值是一个void类型的指针，我们必须强制转化为我们需要的类型的指针。
- 多次调用malloc所分配的地址不能有重叠部分，除非某次molloc所分配的地址被free释放掉了。
- malloc应该尽快的完成内存额分配并且返回。
- 实现malloc的同时实现calloc和realloc和free。

如果是子啊Linux环境下，可以使用

```text
<span style="font-size:14px;">man malloc
</span>
```

查看malloc的具体定义。

## 2.Linux的内存管理

**1.虚拟内存地址与物理内存地址的关系：**

现代操作系统在处理内存地址时普遍的采用虚拟内存地址技术，什么事虚拟内存技术呢？

这种技术使每个进程“仿佛独享”一块2N字节的内存（N是机器的位数），例如在64位的操作系统下，每个进程的虚拟内存空间是264B。这种虚拟内存空间的作用是简化程序的编写并且方便操作系统对进程之间的隔离管理。

虚拟内存技术是由MMU和页表构成的，MMU是一种映射算法，它从虚拟内存地址映射到物理内存地址上，单位是页

**2.什么是页表？**

在现代操作系统中，不管是虚拟内存地址还是物理内存地址，都是以页尾单位管理的，而不是大家以为的字节。（一个内存页是一段固定的地址，典型的内存页的大小是4K）。所以内存地址可以分为页号和页内偏移量

![img](https://pic1.zhimg.com/80/v2-b40413c04633b6ae417da1418b699414_720w.webp)

## 3.Linux进程级的内存管理

首先，我们可以了解一下一个进程的内核空间：

![img](https://pic2.zhimg.com/80/v2-fd6e55efd5e96b07e925b97ead89e729_720w.webp)

可以看到一个进程地址空间的主要成分为：

- 正文：这是整个用户空间的最低地址部分，存放的是指令（也就是程序所编译成的可执行机器码）
- 初始化数据段：这里存放的是初始化过的全局变量
- 未初始化数据段：这里存放的是未初始化的全局变量
- Heap：堆，这是我们本文重点关注的地方，堆自低地址向高地址增长，后面要讲到的brk相关的系统调用就是从这里分配内存
- Stack：这是栈区域，自高地址向低地址增长
- 命令行参数和环境变量：用户调用的最底层。

我们都知道，在malloc分配空间时是在Heap上分配的，实质上， Linux维护一个break指针，这个指针指向堆空间的某个地址。从堆起始地址到break之间的地址空间为映射好的，可以供进程访问；而从break往上，是未映射的地址空间，如果访问这段空间则程序会报错。

![img](https://pic4.zhimg.com/80/v2-35221ddb47bb148a1ffbbc914e84ea87_720w.webp)

由上文知道，要增加一个进程实际的可用堆大小，就需要将break指针向高地址移动。Linux通过brk和sbrk系统调用操作break指针。两个系统调用的原型如下：

```text
int brk(void *addr);
void *sbrk(intptr_t increment);
```

brk将break指针直接设置为某个地址，而sbrk将break从当前位置移动increment所指定的增量。brk在执行成功时返回0，否则返回-1并设置errno为ENOMEM；sbrk成功时返回break移动之前所指向的地址，否则返回(void *)-1。

一个小技巧是，如果将increment设置为0，则可以获得当前break的地址。

另外需要注意的是，由于Linux是按页进行内存映射的，所以如果break被设置为没有按页大小对齐，则系统实际上会在最后映射一个完整的页，从而实际已映射的内存空间比break指向的地方要大一些。但是使用break之后的地址是很危险的（尽管也许break之后确实有一小块可用内存地址）。



有了上面的知识，我们可以实现一个最简单的malloc（没什么用，像个玩具）

```text
/* 一个玩具malloc */
#include <sys/types.h>
#include <unistd.h>
void *malloc(size_t size)
{
    void *p;
    p = sbrk(0);
    if (sbrk(size) == (void *)-1)
        return NULL;
    return p;
}
```

这个malloc由于对所分配的内存缺乏记录，不便于内存释放，所以无法用于真实场景。

## 4.开始实现正式的malloc

一个方案是将堆内存空间以块（Block）的形式组织起来，每个块由meta区和数据区组成，meta区记录数据块的元信息（数据区大小、空闲标志位、指针等等），数据区是真实分配的内存区域，并且数据区的第一个字节地址即为malloc返回的地址。

以下是一个块的结构：

```text
typedef struct s_block *t_block;
struct s_block {
    size_t size;  /* 数据区大小 */
    t_block next; /* 指向下个块的指针 */
    int free;     /* 是否是空闲块 */
    int padding;  /* 填充4字节，保证meta块长度为8的倍数 */
    char data[1]  /* 这是一个虚拟字段，表示数据块的第一个字节，长度不应计入meta */
};
```

现在考虑如何在block链中查找合适的block。一般来说有两种查找算法：

- First fit：从头开始，使用第一个数据区大小大于要求size的块所谓此次分配的块
- Best fit：从头开始，遍历所有块，使用数据区大小大于size且差值最小的块作为此次分配的块

两种方法各有千秋，best fit具有较高的内存使用率（payload较高），而first fit具有更好的运行效率。这里我们采用first fit算法。

```text
* First fit */
t_block find_block(t_block *last, size_t size) {
    t_block b = first_block;
    while(b && !(b->free && b->size >= size)) {
        *last = b;
        b = b->next;
    }
    return b;
}
```

ind_block从frist_block开始，查找第一个符合要求的block并返回block起始地址，如果找不到这返回NULL。这里在遍历时会更新一个叫last的指针，这个指针始终指向当前遍历的block。这是为了如果找不到合适的block而开辟新block使用的。

如果现有block都不能满足size的要求，则需要在链表最后开辟一个新的block。这里关键是如何只使用sbrk创建一个struct：

```text
#define BLOCK_SIZE 24 /* 由于存在虚拟的data字段，sizeof不能正确计算meta长度，这里手工设置 */
 
t_block extend_heap(t_block last, size_t s) {
    t_block b;
    b = sbrk(0);
    if(sbrk(BLOCK_SIZE + s) == (void *)-1)
        return NULL;
    b->size = s;
    b->next = NULL;
    if(last)
        last->next = b;
    b->free = 0;
    return b;
}
```

First fit有一个比较致命的缺点，就是可能会让很小的size占据很大的一块block，此时，为了提高payload，应该在剩余数据区足够大的情况下，将其分裂为一个新的block：

```text
void split_block(t_block b, size_t s) {
t_block new;
new = b->data + s;
new->size = b->size - s - BLOCK_SIZE ;
new->next = b->next;
new->free = 1;
b->size = s;
b->next = new;
}
```

有了上面的代码，我们可以利用它们整合成一个简单但初步可用的malloc。注意首先我们要定义个block链表的头first_block，初始化为NULL；另外，我们需要剩余空间至少有BLOCK_SIZE + 8才执行分裂操作。

由于我们希望malloc分配的数据区是按8字节对齐，所以在size不为8的倍数时，我们需要将size调整为大于size的最小的8的倍数：

```text
size_t align8(size_t s) {
    if(s & 0x7 == 0)
        return s;
    return ((s >> 3) + 1) << 3;
}
#define BLOCK_SIZE 24
void *first_block=NULL;
```



```text
void *malloc(size_t size) {
    t_block b, last;
    size_t s;
    /* 对齐地址 */
    s = align8(size);
    if(first_block) {
        /* 查找合适的block */
        last = first_block;
        b = find_block(&last, s);
        if(b) {<pre name="code" class="cpp">         /* 如果可以，则分裂 */
            if ((b->size - s) >= ( BLOCK_SIZE + 8))
                split_block(b, s);
            b->free = 0;
        } else {
            /* 没有合适的block，开辟一个新的 */
            b = extend_heap(last, s);
            if(!b)
                return NULL;
        }
    } else {
        b = extend_heap(NULL, s);
        if(!b)
            return NULL;
        first_block = b;
    }
    return b->data;
}
```

原文地址：https://zhuanlan.zhihu.com/p/502710223

作者：linux