# 【NO.530】Linux服务器开发,内存池原理与实现

## 0.前言

内存池顾名思义，是管理栈空间中堆的组件。有了内存池，可以避免频繁的分配和释放堆空间。从另一个角度说，内存池是为了提升变量的生存周期。

## 1.作用

频繁分配导致内存分配碎片化。
内存管理不是几分钟几天的内存问题，而是解决当程序长时间运行时，改变资源不进行管理造成被系统杀死的命运。如果运行几个月后偶现的问题较难排查，对于这种问题排查也是束手无策，所以使用内存池是一件非常有必要的事。

## 2.注意点

- 一定要用内存池。

- 不要自己实现内存池！听到这一点我就开心了，毕竟减少了时间劳动的成本。因为这种组件很大部分都工程师都没有这个能力。

- 内存池的原理一定要懂。

  jemallocl是c实现的，tcmaloc是google用c++实现的，更推荐这个，内存回收机制有些小问题，绝大部分使用没问题。用起来简单，加个宏定义，会hook住malloc。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/fcc2ca6330174b66bad1d8ad78236dd4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

  小块回收是一个极其麻烦的事情，随着时间的推移小字节内存块越来越多浪费也越来越多。解决方案是固定大小的内存。但是固定大小的也存在着不灵活的问题，很多内存用不了的情况。

  改进的方法使用梯度的固定大小，大块的内存要多少给多少，小块内存分档次进行处理。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/39d66345099548859d6016f40ab530ac.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

  但是仍存在的问题时：

- 链表查找速度慢。——>1、区分使用过否。2、红黑树，哈希。

- 出现了间隙，不利于回收，极其麻烦。——>影响小块的回收。

几种使用场景
1、全局内存池 jemmaloc/tcmalloc。
2、一个连接做一个内存池，一个连接释放一个内存池被销毁。生命周期短所以小块不用回收。
3、消息x，浪费。

## 3.实现内存池

使用大块和小块来实现。

```c
void *mp_memalign(struct mp_pool_s *pool, size_t size, size_t alignment)



 {



    void *p;



    //posix_memalign()这个函数可以分配出大空间的内存



    int ret = posix_memalign(&p, alignment, size);



    if (ret) {



        return NULL;



    }



    struct mp_large_s *large = mp_alloc(pool, sizeof(struct mp_large_s));



    if (large == NULL) 



    {



        free(p);



        return NULL;



    }



    large->alloc = p;



    large->next = pool->large;



    pool->large = large;



 



    return p;



}
struct mp_pool_s *mp_create_pool(size_t size)



 {



    struct mp_pool_s *p;



    int ret = posix_memalign((void **)&p, MP_ALIGNMENT, size + sizeof(struct mp_pool_s) + sizeof(struct mp_node_s));



    if (ret)



     {



        return NULL;



    }



    p->max = (size < MP_MAX_ALLOC_FROM_POOL) ? size : MP_MAX_ALLOC_FROM_POOL;



    p->current = p->head;



    p->large = NULL;



    p->head->last = (unsigned char *)p + sizeof(struct mp_pool_s) + sizeof(struct mp_node_s);



    p->head->end = p->head->last + size;



 



    p->head->failed = 0;



 



    return p;



}
```

posix_memallign((void **)&pool,ALIGNMENT,size+sizeof(struct mp_pool));
//分配超过4K就分配不出来，所有调用此api

注意释放顺序，必须大块先释放，再释放小块。将指向大块节点的那块内存放在小块中，可以减少释放内存时的难度系数。

## 4.总结

今天通过零声学院King老师绘声绘色的讲述内存池，让我感触颇深。
从技术知识层面，我又从新的一个角度认识了内存池。以前一直感觉内存池是一个臃肿的组件，从来没有感受到内存池存在着重要的意义。有了内存池就可以轻松避免内存管理出现的问题，有的程序工作几个月以上出现崩溃的情况，偶现的问题往往是极难排查的，用上内存池可以更加保险。尽管不需要自己手写内存池，但是我已经足够深入的了解了内存池的历史演变和工作原理，可以说这在工作中完全足够在同事面前炫耀一番了。
感觉这句话是我印象最深的：线程池、内存池其实没有标准，要根据实际情况去确定。个人理解没有标准往往比有标准更加痛苦，我只能通过自己的努力不断的探索标准，才能更快的进步。
最后，还是非常感谢零声学院的King老师，啊，有King老师的陪伴学习的路上变得异常平坦开阔。

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604988800