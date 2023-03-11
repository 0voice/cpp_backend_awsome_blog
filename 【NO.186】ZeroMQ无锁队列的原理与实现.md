# 【NO.186】ZeroMQ无锁队列的原理与实现

## 1.前言

无锁队列用在什么地方？每秒几十万的元素时再考虑使用无锁队列，比如股票行情这种。如果队列里一秒就几千几万的元素，那就不需要使用无锁队列，性能没有太大的提高。

源码：[ypipe.hpp](https://link.zhihu.com/?target=https%3A//github.com/gopherWxf/c-c-linux-LearningCode/tree/master/3.2.4%E6%97%A0%E9%94%81%E9%98%9F%E5%88%97freequeue)

## 2. 为什么需要无锁队列

锁是解决并发问题的万能钥匙，可是并发问题只有锁能解决吗？锁引起的问题：

- Cache损坏(Cache trashing)

线程间频繁切换的时候会导致 Cache 中数据的丢失，Cache中的数据会失效,因为它缓存的是将被换出任务的数据,这些数据对于新换进的任务是没⽤的。处理器的运⾏速度⽐主存快N倍,所以⼤量的处理器时间被浪费在处理器与主存的数据传输上。这就是在处理器和主存之间引⼊Cache的原因。Cache是⼀种速度更快但容量更⼩的内存(也更加昂贵),当处理器要访问主存中的数据时,这些数据⾸先被拷⻉到Cache中，因为这些数据在不久的将来可能⼜会被处理器访问。Cache misses对性能有⾮常⼤的影响,因为处理器访问Cache中的数据将⽐直接访问主存快得多。在保存和恢复上下⽂的过程中还隐藏了额外的开销。

- 在同步机制上争抢队列

阻塞不是微不⾜道的操作。它导致操作系统暂停当前的任务或使其进⼊睡眠状态(等待，不占⽤任何的处理器)。直到资源(例如互斥锁)可⽤，被阻塞的任务才可以解除阻塞状态(唤醒)。在⼀个负载较重的应⽤程序中使⽤这样的阻塞队列来在线程之间传递消息会导致严重的争⽤问题。也就是说，任务将⼤量的时间(睡眠，等待，唤醒)浪费在获得保护队列数据的互斥锁，⽽不是处理队列中的数据上。

⾮阻塞机制⼤展伸⼿的机会到了。任务之间不争抢任何资源，在队列中预定⼀个位置，然后在这个位置上插⼊或提取数据。这中机制使⽤了⼀种被称之为CAS(⽐较和交换)的特殊操作，这个特殊操作是⼀种特殊的指令，它可以原⼦的完成以下操作:它需要3个操作数m，A，B，其中m是⼀个内存地址，操作将m指向的内存中的内容与A⽐较，如果相等则将B写⼊到m指向的内存中并返回true，如果不相等则什么也不做返回false。简而言之非阻塞的机制使用了 CAS 的特殊操作，使得任务之间可以不争抢任何资源，然后在队列中预定的位置上，插入或者提取数据。[CAS底层实现](https://link.zhihu.com/?target=https%3A//blog.csdn.net/qq_42956653/article/details/126141284)

- 多线程动态内存分配malloc性能下降

在多线程系统中,需要仔细的考虑动态内存分配。当⼀个任务从堆中分配内存时，标准的内存分配机制会阻塞所有与这个任务共享地址空间的其它任务(进程中的所有线程)。这样做的原因是让处理更简单，且它⼯作得很好。两个线程不会被分配到⼀块相同的地址的内存，因为它们没办法同时执⾏分配请求。显然线程频繁分配内存会导致应⽤程序性能下降(必须注意,向标准队列或map插⼊数据的时候都会导致堆上的动态内存分配)

## 3. 无锁队列的实现(参考zmq，只支持一写一读的场景)

### 3.1 无锁队列前言

//TODO git地址补充 源码的ypipe.hpp、yqueue.hpp，这些源码可以在⼯程项⽬使⽤，但要注意，这⾥只⽀持单写单读的场景。 其中yqueue 是用来设计队列，ypipe 用来设计队列的写入/读取时机、回滚以及 flush，首先我们来看 yqueue 的设计。

### 3.2 原子操作函数介绍

```text
template<typename T>
class atomic_ptr_t {
public:
    void set(T *ptr_); //⾮原⼦操作
    T *xchg(T *val_); //原⼦操作，设置⼀个新的值，然后返回旧的值
    T *cas(T *cmp_, T *val_);//原⼦操作
private:
    volatile T *ptr;
};
```

- set函数，把私有成员ptr指针设置成参数ptr_的值，不是⼀个原⼦操作，需要使⽤者确保执⾏set过程没有其他线程使⽤ptr的值。
- xchg函数，把私有成员ptr指针设置成参数val_的值，并返回ptr设置之前的值。原⼦操作，线程安全。
- cas函数，原⼦操作，线程安全，把私有成员ptr指针与参数cmp_指针⽐较：如果相等返回ptr设置之前的值，并把ptr更新为参数val_的值，如果不相等直接返回ptr值。

### 3.3 yqueue_t的chunk块机制

#### **3.3.1 chunk块机制 一次分配多个元素**

首先我们需要考虑元素的分配，元素存在哪里？yqueue 中的数据结构使用的 chunk 块机制，每次批量分配一批元素，这样可以减少内存的分配和释放yqueue_t内部由⼀个⼀个chunk组成，每个chunk保存N个元素：spare_chunk⾥⾯，当再次需要分配chunk_t的时候从spare_chunk中获取。

当队列空间不⾜时每次分配⼀个chunk_t，每个chunk_t能存储N个元素。在数据出队列后，队列有多余空间的时候，回收的chunk也不是⻢上释放，⽽是根据局部性原理先回收到

```text
struct chunk_t {
   T values[N]; //每个chunk_t可以容纳N个T类型的元素，以后就以一个chunk_t为单位申请内存
   chunk_t *prev;
   chunk_t *next;
};
```

![img](https://pic3.zhimg.com/80/v2-490a9cc9ec8a235c769115da7e5158ea_720w.webp)

#### **3.3.2 chunk块机制 局部性原理**

程序局部性原理：是指程序在执行时呈现出局部性规律，即在一段时间内，整个程序的执行仅限于程序中的某一部分。相应地，执行所访问的存储空间也局限于某个内存区域，具体来说，局部性通常有两种形式：时间局部性和空间局部性。

时间局部性：被引用过一次的存储器位置在未来会被多次引用（通常在循环中）。

空间局部性：如果一个存储器的位置被引用，那么将来他附近的位置也会被引用。

在yqueue_t类中有一个spare_chunk用于保存最近的空闲块 。也就是说，在将一个chunk中的所有元素都pop掉了，那么我们可以free这个chunk。但是我们可以保存一块最近的空闲块，以后如果chunk不够用时，扩容chunk就不用malloc，直接复用该spare_chunk即可。根据局部性原理，这个spare_chunk的地址或者内存页很有可能还在cache里，那么这样的机制就可以减少一次malloc以及存入cache的操作。

```text
//  class yqueue_t
//  People are likely to produce and consume at similar rates.  In
//  this scenario holding onto the most recently freed chunk saves
//  us from having to call malloc/free.
atomic_ptr_t<chunk_t> spare_chunk; //空闲块（把所有元素都已经出队的块称为空闲块），读写线程的共享变量
```

可以看到在pop的时候，如果删除满一格chunk，就把这个chunk放到spare_chunk里。

```text
//  Removes an element from the front end of the queue.
inline void pop() {
    if (++begin_pos == N) // 删除满一个chunk才回收chunk
    {
        chunk_t *o = begin_chunk;
        begin_chunk = begin_chunk->next;
        begin_chunk->prev = NULL;
        begin_pos = 0;

        //  'o' has been more recently used than spare_chunk,
        //  so for cache reasons we'll get rid of the spare and
        //  use 'o' as the spare.
        chunk_t *cs = spare_chunk.xchg(o); //由于局部性原理，总是保存最新的空闲块而释放先前的空闲快
        free(cs);
    }
}
```

在push的时候，如果chunk满了，说明要发生扩容，那么我们优先从spare_chunk取出最近的空闲块当新的chunk来使用

```text
//  Adds an element to the back end of the queue.
inline void push() {
    back_chunk = end_chunk;
    back_pos = end_pos; //

    if (++end_pos != N) //end_pos!=N表明这个chunk节点还没有满
        return;

    chunk_t *sc = spare_chunk.xchg(NULL); // 为什么设置为NULL？ 因为如果把之前值取出来了则没有spare chunk了，所以设置为NULL
    if (sc)                               // 如果有spare chunk则继续复用它
    {
        end_chunk->next = sc;
        sc->prev = end_chunk;
    }
    else // 没有则重新分配
    {
        // static int s_cout = 0;
        // printf("s_cout:%d\n", ++s_cout);
        end_chunk->next = (chunk_t *) malloc(sizeof(chunk_t)); // 分配一个chunk
        alloc_assert(end_chunk->next);
        end_chunk->next->prev = end_chunk;
    }
    end_chunk = end_chunk->next;
    end_pos = 0;
}
```

![img](https://pic1.zhimg.com/80/v2-f1ffe0391fd90f4ff72a5807704ccbb0_720w.webp)

### 3.4 yqueue_t成员和接口介绍

yqueue_t的作用就是管理元素、管理chunk。chunk和spare_chunk上文已经说过了，这里不再赘述。

```text
// T is the type of the object in the queue.队列中元素的类型
// N is granularity(粒度) of the queue，简单来说就是chunk_t ⼀个结点可以装载N个T类型的元素
template<typename T, int N>
class yqueue_t {
public:
    inline yqueue_t();// 创建队列.
    inline ~yqueue_t();// 销毁队列.
    inline T &front();// Returns reference to the front element of the queue. If the queue is empty, behaviour is undefined.
    inline T &back();// Returns reference to the back element of the queue.If the queue is empty, behaviour is undefined.
    inline void push();// Adds an element to the back end of the queue.
    inline void pop();// Removes an element from the front of the queue.
    inline void unpush()// Removes element from the back end of the queue。 回滚时使⽤
private:
// Individual memory chunk to hold N elements.
    struct chunk_t {
        T values[N];
        chunk_t *prev;
        chunk_t *next;
    };
    chunk_t *begin_chunk;
    int begin_pos;
    chunk_t *back_chunk;
    int back_pos;
    chunk_t *end_chunk;
    int end_pos;
    atomic_ptr_t<chunk_t> spare_chunk; //空闲块（我把所有元素都已经出队的块称为空闲块），读写线程的共享变量
};
```

**2.4.1 begin/back/end_chunk 与 begin/back/end_pos 成员介绍**

```text
chunk_t *begin_chunk;
int begin_pos;
chunk_t *back_chunk;
int back_pos;
chunk_t *end_chunk;
int end_pos;
```

yqueue_t内部有三个chunk_t类型指针以及对应的索引位置：

- begin_chunk/begin_pos：begin_chunk用于指向队列的第一个chunk，begin_pos用于指向第一个chunk的第一个元素的索引位置，因为pop()，所以第一个元素不可能永远是0，会随着pop而改变。同理第一个chunk也会被回收，也需要记录第一个chunk的位置。
- back_chunk/back_pos：begin_chunk用于指向队列的最后一个chunk，back_pos用于指向最后一个chunk的最后一个元素的索引位置。
- end_chunk/end_pos：在最后一个chunk未满的情况下，end_chunk和back_chunk是相同的，back_pos的下一个就是end_pos。在最后一个chunk满的情况下，end_chunk指向新分配的chunk，end_pos=0。也就是说end_chunk和end_pos是辅助back_chunk/back_pos的，可以理解为探测。

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='720' height='528'></svg>)

**2.4.2 函数介绍**

frount和pop连用。back和push连用。

**构造函数yqueue_t**

预先分配⼀个chunk。

```text
//  创建队列.
inline yqueue_t() {
    begin_chunk = (chunk_t *) malloc(sizeof(chunk_t));
    alloc_assert(begin_chunk);
    begin_pos = 0;
    back_chunk = NULL; //back_chunk总是指向队列中最后一个元素所在的chunk，现在还没有元素，所以初始为空
    back_pos = 0;
    end_chunk = begin_chunk; //end_chunk总是指向链表的最后一个chunk
    end_pos = 0;
}
```

![img](data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='301' height='283'></svg>)

**稀构函数~yqueue_t**

销毁所有的chunk

```text
//  销毁队列.
inline ~yqueue_t() {
    while (true) {
        if (begin_chunk == end_chunk) {
            free(begin_chunk);
            break;
        }
        chunk_t *o = begin_chunk;
        begin_chunk = begin_chunk->next;
        free(o);
    }

    chunk_t *sc = spare_chunk.xchg(NULL);
    free(sc);
}
```

**front、back函数**

这⾥的front()或者back()函数，需要注意的返回的是左值引⽤，我们可以修改其值。

对于先进后出队列⽽⾔：

- begin_chunk->values[begin_pos]代表队列头可读元素， 读取队列头元素即是读取begin_pos位置的元素；
- back_chunk->values[back_pos]代表队列尾可写元素，写⼊元素时则是更新back_pos位置的元素，要确保元素真正⽣效，还需要调⽤push函数更新back_pos的位置，避免下次更新的时候⼜是更新当前back_pos位置对应的元素。

```text
//  Returns reference to the front element of the queue.
//  If the queue is empty, behaviour is undefined.
// 返回队列头部元素的引用，调用者可以通过该引用更新元素，结合pop实现出队列操作。
inline T &front() // 返回的是引用，是个左值，调用者可以通过其修改容器的值
{
    return begin_chunk->values[begin_pos];
}

//  Returns reference to the back element of the queue.
//  If the queue is empty, behaviour is undefined.
// 返回队列尾部元素的引用，调用者可以通过该引用更新元素，结合push实现插入操作。
// 如果队列为空，该函数是不允许被调用。
inline T &back() // 返回的是引用，是个左值，调用者可以通过其修改容器的值
{
    return back_chunk->values[back_pos];
}
```

**push函数**

- 当++end_pos != N 时，说明当前的chunk还有空余位置可以插入，则不需要扩容
- 当++end_pos == N时，说明当前的chunk已经插入满了，下一次插入就要插入到新的chunk了，所以需要发生扩容

需要新分配chunk时，先尝试从spare_chunk获取，如果获取到则直接使⽤，如果spare_chunk为NULL则需要重新分配chunk。最终都是要更新end_chunk和end_pos。

```text
//  Adds an element to the back end of the queue.
    inline void push() {
        back_chunk = end_chunk;
        back_pos = end_pos; //

        if (++end_pos != N) //end_pos!=N表明这个chunk节点还没有满
            return;

        chunk_t *sc = spare_chunk.xchg(NULL); // 为什么设置为NULL？ 因为如果把之前值取出来了则没有spare chunk了，所以设置为NULL
        if (sc)                               // 如果有spare chunk则继续复用它
        {
            end_chunk->next = sc;
            sc->prev = end_chunk;
        }
        else // 没有则重新分配
        {
            // static int s_cout = 0;
            // printf("s_cout:%d\n", ++s_cout);
            end_chunk->next = (chunk_t *) malloc(sizeof(chunk_t)); // 分配一个chunk
            alloc_assert(end_chunk->next);
            end_chunk->next->prev = end_chunk;
        }
        end_chunk = end_chunk->next;
        end_pos = 0;
    }
```

![img](https://pic1.zhimg.com/80/v2-23afa5ab083f07310f0832ae9d5b8568_720w.webp)

**unpush函数**

unpush函数没什么好说的，也是考虑有没有发生扩容的情况，然后分两种情况回退即可。

```text
//  Removes element from the back end of the queue. In other words
 //  it rollbacks last push to the queue. Take care: Caller is
 //  responsible for destroying the object being unpushed.
 //  The caller must also guarantee that the queue isn't empty when
 //  unpush is called. It cannot be done automatically as the read
 //  side of the queue can be managed by different, completely
 //  unsynchronised thread.
 // 必须要保证队列不为空，参考ypipe_t的uwrite
 inline void unpush() {
     //  First, move 'back' one position backwards.
     if (back_pos) // 从尾部删除元素
         --back_pos;
     else {
         back_pos = N - 1; // 回退到前一个chunk
         back_chunk = back_chunk->prev;
     }

     //  Now, move 'end' position backwards. Note that obsolete end chunk
     //  is not used as a spare chunk. The analysis shows that doing so
     //  would require free and atomic operation per chunk deallocated
     //  instead of a simple free.
     if (end_pos) // 意味着当前的chunk还有其他元素占有
         --end_pos;
     else {
         end_pos = N - 1; // 当前chunk没有元素占用，则需要将整个chunk释放
         end_chunk = end_chunk->prev;
         free(end_chunk->next);
         end_chunk->next = NULL;
     }
 }
```

**pop函数**

- ++begin_pos != N，说明当前chunk还有元素没被取出，该chunk还要继续被使⽤；
- ++end_pos == N，说明该chunk的所有元素已经被取出，所以该chunk要被回收。把最后回收的chunk保存到spare_chunk，然后释放之前spare_chunk保存的chunk。

这⾥有两个点需要注意：

1. pop掉的元素，其销毁⼯作交给调⽤者完成，即是pop前调⽤者需要通过front()接⼝读取并进⾏销毁
2. 空闲块的保存，要求是原⼦操作。因为闲块是读写线程的共享变量，因为在push中也使⽤了spare_chunk。

```text
//  Removes an element from the front end of the queue.
inline void pop() {
    if (++begin_pos == N) // 删除满一个chunk才回收chunk
    {
        chunk_t *o = begin_chunk;
        begin_chunk = begin_chunk->next;
        begin_chunk->prev = NULL;
        begin_pos = 0;

        //  'o' has been more recently used than spare_chunk,
        //  so for cache reasons we'll get rid of the spare and
        //  use 'o' as the spare.
        chunk_t *cs = spare_chunk.xchg(o); //由于局部性原理，总是保存最新的空闲块而释放先前的空闲快
        free(cs);
    }
}
```

### 3.5 ypipe—> yqueue的封装

yqueue 负责元素内存的分配与释放，入队以及出队列；ypipe 负责 yqueue 读写指针的变化。ypipe_t在yqueue_t的基础上构建⼀个单写单读的⽆锁队列

```text
template<typename T, int N>
class ypipe_t {
public:
    // Initialises the pipe.
    inline ypipe_t();

    // The destructor doesn't have to be virtual. It is mad virtual
    // just to keep ICC and code checking tools from complaining.
    inline virtual ~ypipe_t();

    // Write an item to the pipe. Don't flush it yet. If incomplete is
    // set to true the item is assumed to be continued by items
    // subsequently written to the pipe. Incomplete items are never flushed down the stream.
    // 写⼊数据，incomplete参数表示写⼊是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
    inline void write(const T &value_, bool incomplete_);

    // Pop an incomplete item from the pipe. Returns true is such
    // item exists, false otherwise.
    inline bool unwrite(T *value_);

    // Flush all the completed items into the pipe. Returns false if
    // the reader thread is sleeping. In that case, caller is obliged to
    // wake the reader up before using the pipe again.
    // 刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调⽤者需要唤醒读线程。
    inline bool flush();

    // Check whether item is available for reading.
    // 这⾥⾯有两个点，⼀个是检查是否有数据可读，⼀个是预取
    inline bool check_read();

    // Reads an item from the pipe. Returns false if there is no value.
    // available.
    inline bool read(T *value_);

    // Applies the function fn to the first elemenent in the pipe
    // and returns the value returned by the fn.
    // The pipe mustn't be empty or the function crashes.
    inline bool probe(bool (*fn)(T &));

protected:
    // Allocation-efficient queue to store pipe items.
    // Front of the queue points to the first prefetched item, back of
    // the pipe points to last un-flushed item. Front is used only by
    // reader thread, while back is used only by writer thread.
    yqueue_t<T, N> queue;
    // Points to the first un-flushed item. This variable is used
    // exclusively by writer thread.
    T *w;//指向第⼀个未刷新的元素,只被写线程使⽤
    // Points to the first un-prefetched item. This variable is used
    // exclusively by reader thread.
    T *r;//指向第⼀个还没预提取的元素，只被读线程使⽤
    // Points to the first item to be flushed in the future.
    T *f;//指向下⼀轮要被刷新的⼀批元素中的第⼀个
    // The single point of contention between writer and reader thread.
    // Points past the last flushed item. If it is NULL,
    // reader is asleep. This pointer should be always accessed using
    // atomic operations.
    atomic_ptr_t<T> c;//读写线程共享的指针，指向每⼀轮刷新的起点（看代码的时候会详细说）。当c为空时，表示读线程睡眠（只会在读线程中被设置为空）
    // Disable copying of ypipe object.
    ypipe_t(const ypipe_t &);
    const ypipe_t &operator=(const ypipe_t &);
};
```

#### **3.5.1 如何写入和读取**

这一节的目的是找出改变wrfc这四个指针的的函数，至于函数的具体作用会放下下面写。

写入可以单独写，也可以批量写，先来看看write函数。可以看到如果incomplete_=true，则说明在批量写，直到incomplete_=false时，进行写提交。

```text
//  Write an item to the pipe.  Don't flush it yet. If incomplete is
//  set to true the item is assumed to be continued by items
//  subsequently written to the pipe. Incomplete items are never flushed down the stream.
// 写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
inline void write(const T &value_, bool incomplete_) {
    //  Place the value to the queue, add new terminator element.
    queue.back() = value_;
    queue.push();

    //  Move the "flush up to here" poiter.
    if (!incomplete_) {
        f = &queue.back(); // 记录要刷新的位置
    }
}
```



```text
//1. 单次写
yquque.write(count,false);
yquque.flush();
//2. 批量写
yquque.write(count,true);
yquque.write(count,true);
yquque.write(count,false);
yquque.flush();
```

上面两种方式最后都用到了flush，下面来看看flush。

```text
//  Flush all the completed items into the pipe. Returns false if
//  the reader thread is sleeping. In that case, caller is obliged to
//  wake the reader up before using the pipe again.
// 刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调用者需要唤醒读线程。
// 批量刷新的机制， 写入批量后唤醒读线程；
// 反悔机制 unwrite
inline bool flush() {
    //  If there are no un-flushed items, do nothing.
    if (w == f) // 不需要刷新，即是还没有新元素加入
        return true;

    //  Try to set 'c' to 'f'.
    // read时如果没有数据可以读取则c的值会被置为NULL
    if (c.cas(w, f) != w) // 尝试将c设置为f，即是准备更新w的位置
    {

        //  Compare-and-swap was unseccessful because 'c' is NULL.
        //  This means that the reader is asleep. Therefore we don't
        //  care about thread-safeness and update c in non-atomic
        //  manner. We'll return false to let the caller know
        //  that reader is sleeping.
        c.set(f); // 更新为新的f位置
        w = f;
        return false; //线程看到flush返回false之后会发送一个消息给读线程，这需要写业务去做处理
    }
    else  // 读端还有数据可读取
    {
        //  Reader is alive. Nothing special to do now. Just move
        //  the 'first un-flushed item' pointer to 'f'.
        w = f;             // 更新f的位置
        return true;
    }
}
```

写入分析完了，来看看如何读。

```text
//  Check whether item is available for reading.
// 这里面有两个点，一个是检查是否有数据可读，一个是预取
inline bool check_read() {
    //  Was the value prefetched already? If so, return.
    if (&queue.front() != r && r) //判断是否在前几次调用read函数时已经预取数据了return true;
        return true;

    //  There's no prefetched value, so let us prefetch more values.
    //  Prefetching is to simply retrieve the
    //  pointer from c in atomic fashion. If there are no
    //  items to prefetch, set c to NULL (using compare-and-swap).
    // 两种情况
    // 1. 如果c值和queue.front()， 返回c值并将c值置为NULL，此时没有数据可读
    // 2. 如果c值和queue.front()， 返回c值，此时可能有数据度的去
    r = c.cas(&queue.front(), NULL); //尝试预取数据

    //  If there are no elements prefetched, exit.
    //  During pipe's lifetime r should never be NULL, however,
    //  it can happen during pipe shutdown when items are being deallocated.
    if (&queue.front() == r || !r) //判断是否成功预取数据
        return false;

    //  There was at least one value prefetched.
    return true;
}

//  Reads an item from the pipe. Returns false if there is no value.
//  available.
inline bool read(T *value_) {
    //  Try to prefetch a value.
    if (!check_read())
        return false;

    //  There was at least one value prefetched.
    //  Return it to the caller.
    *value_ = queue.front();
    queue.pop();
    return true;
}
```

下面来多分析一下，如果read返回false，那么我们应该怎么做？读失败意味着管道内没有可读的数据，所以我们可以休眠，可以让出cpu，也可以条件等待。

这里最正确的做法是用条件等待。上面的flush返回false代表着读端在等待，那么flush返回false后我们应该通知读端。

```text
//读端
if (yqueue.read(&value)) {
    //数据处理
}
else {
    // usleep(100);
    std::unique_lock<std::mutex> lock(ypipe_mutex_);
    ypipe_cond_.wait(lock);
    // sched_yield();
}

//写端
yqueue.write(count, false);
if (!yqueue.flush()) {
    // printf("notify_one\n");
    std::unique_lock<std::mutex> lock(ypipe_mutex_);
    ypipe_cond_.notify_one();
}
```

其实我们初略的观察这些函数，就能发现，这几个函数改变的是w,r,f,c这四个指针，下面来看看这四个指针的具体作用吧。

#### **3.5.2 w,r,f,c图文结合详解（重点理解）**

这里这几个变量非常抽象，要结合着函数来讲

- T *f：指向下一轮要被刷新的一批元素的第一个。
- T *w：指向第一个未刷新的元素，只被写线程使用；
- T *r：指向第一个没有被预提取的元素，只被读线程使用；
- atomic_ptr_t c：读写线程共享的指针，指向每⼀轮刷新的起点。当c为空时，表示读线程睡眠（只会在读线程中被设置为空）
- write()：写⼊数据，incomplete参数表示写⼊是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。完成后会将f = &queue.back();
- unwrite()：在数据没有flush之前可以运⾏反悔 Pop an incomplete item from the pipe. Returns true is such item exists, false otherwise.
- bool flush()：将write的元素真正刷新到队列，使读端可以访问对应的数据。返回false意味着读线程在休眠，在这种情况下调⽤者需要唤醒读线程。如果读端阻塞，则c=f;w=f;否则w=f;
- bool check_read()：检测是否有数据可读，如果c==queue.front则c=NULL,否则r=c
- bool read (T *value_)：读数据，将读出的数据写⼊value指针中，返回false意味着没有数据可读

这样写感觉还是非常抽象，下面结合着函数和图来讲这些函数与四个变量的关系吧。

**构造函数ypipe_t()**

在构造函数里面，下一轮要被刷新的元素的第一个(f)，必然是第一个位置；第一个未刷新的元素(w)，也是第一个位置；第一个没有被预读取的元素( r )，也是第一个位置；每一轮刷新的起点，也是第一个位置( c );

![img](https://pic1.zhimg.com/80/v2-516e4a2a7d857674e813c7f8c8e05c98_720w.webp)

```text
inline ypipe_t() {
	//  Insert terminator element into the queue.
	queue.push(); //yqueue_t的尾指针加1，开始back_chunk为空，现在back_chunk指向第一个chunk_t块的第一个位置
	
	//  Let all the pointers to point to the terminator.
	//  (unless pipe is dead, in which case c is set to NULL).
	r = w = f = &queue.back(); //就是让r、w、f、c四个指针都指向这个end迭代器
	c.set(&queue.back());
}
```

**写入函数write(const T &value_, bool incomplete_)**

第二个参数决定是否要刷新一批元素，false时，刷新一批元素，那么下一轮要被刷新的元素的第一个( f ) 就要改变了。

![img](https://pic3.zhimg.com/80/v2-d71f7eb1deaf536d9bfd74e107edfbb6_720w.webp)

~~~text
```cpp
//  Write an item to the pipe.  Don't flush it yet. If incomplete is
//  set to true the item is assumed to be continued by items
//  subsequently written to the pipe. Incomplete items are never flushed down the stream.
// 写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
inline void write(const T &value_, bool incomplete_) {
    //  Place the value to the queue, add new terminator element.
    queue.back() = value_;
    queue.push();

    //  Move the "flush up to here" poiter.
    if (!incomplete_) {
        f = &queue.back(); // 记录要刷新的位置
    }
}
~~~

**刷新元素使元素对读线程可见 bool flush()**

还记得c吗？指向每一轮刷新的起点。如果c和w一样，则尝试将c置为f。刷新元素，指向第一个未刷新的元素( w )，那么必然w=f了。此时前面的元素都可以被读线程可见。

我们来看看什么情况下c != w。

在未更新前队列没有数据可读，没有数据可读的时候，check_read将c⾥⾯的ptr置为NULL。所以会走下面的流程。返回false的⽬的是告诉调⽤者数据读端(接收端)没有数据可读，可能处于休眠的状态，可以结合condition机制，发送⼀个notify唤醒读端继续读取数据。

```text
//  Try to set 'c' to 'f'.
// read时如果没有数据可以读取则c的值会被置为NULL
if (c.cas(w, f) != w) // 尝试将c设置为f，即是准备更新w的位置
{

    //  Compare-and-swap was unseccessful because 'c' is NULL.
    //  This means that the reader is asleep. Therefore we don't
    //  care about thread-safeness and update c in non-atomic
    //  manner. We'll return false to let the caller know
    //  that reader is sleeping.
    c.set(f); // 更新为新的f位置
    w = f;
    return false; //线程看到flush返回false之后会发送一个消息给读线程，这需要写业务去做处理
}
```

未更新前队列有数据可读，此时只需要更新w即可，但此时c值不去更新。

```text
else  // 读端还有数据可读取
{
    //  Reader is alive. Nothing special to do now. Just move
    //  the 'first un-flushed item' pointer to 'f'.
    w = f;             // 更新f的位置
    return true;
}
```

![img](https://pic4.zhimg.com/80/v2-b50779c840bba5a4fe0c72ab66eefd1b_720w.webp)

从write和flush我们也可以看出来，在更新w和f的时候并没有互斥的保护，所以此程序插⼊数据的时候不适合⽤于多线程场景。

flush函数主要是将w更新到f的位置，说明已经写到的位置。

```text
//  Flush all the completed items into the pipe. Returns false if
//  the reader thread is sleeping. In that case, caller is obliged to
//  wake the reader up before using the pipe again.
// 刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调用者需要唤醒读线程。
// 批量刷新的机制， 写入批量后唤醒读线程；
// 反悔机制 unwrite
inline bool flush() {
    //  If there are no un-flushed items, do nothing.
    if (w == f) // 不需要刷新，即是还没有新元素加入
        return true;

    //  Try to set 'c' to 'f'.
    // read时如果没有数据可以读取则c的值会被置为NULL
    if (c.cas(w, f) != w) // 尝试将c设置为f，即是准备更新w的位置
    {

        //  Compare-and-swap was unseccessful because 'c' is NULL.
        //  This means that the reader is asleep. Therefore we don't
        //  care about thread-safeness and update c in non-atomic
        //  manner. We'll return false to let the caller know
        //  that reader is sleeping.
        c.set(f); // 更新为新的f位置
        w = f;
        return false; //线程看到flush返回false之后会发送一个消息给读线程，这需要写业务去做处理
    }
    else  // 读端还有数据可读取
    {
        //  Reader is alive. Nothing special to do now. Just move
        //  the 'first un-flushed item' pointer to 'f'.
        w = f;             // 更新f的位置
        return true;
    }
}
```

**预取读取函数ckeck_read()**

如果指针r指向的是队头元素（r==&queue.front()）或者r没有指向任何元素（NULL）则说明队列中并没有可读的数据，这个时候check_read尝试去预取数据。所谓的预取就是令 r=c (cas函数就是返回c本身的值，看上⾯关于cas的实现)， ⽽c在write中被指向f（⻅上图），这时从queue.front()到f这个位置的数据都被预取出来了，然后每次调⽤read都能取出⼀段。

![img](https://pic2.zhimg.com/80/v2-a22ff17566ec1087aab1910b08724f35_720w.webp)

值得注意的是，当c==&queue.front()时，代表数据被取完了，这时把c指向NULL，接着读线程会睡眠，这也是给写线程 检查 读线程是否睡眠的标志（c指向NULL）。

继续上⾯写⼊AB数据的场景，第⼀次调⽤read时，会先check_read，把指针r指向指针c的位置（所谓的预取），这时r,c,w,f的关系如下：

![img](https://pic2.zhimg.com/80/v2-40f88f65c5ac33c430209c51eca71475_720w.webp)

为什么要预读取？当front()和r相等时：

- r = c.cas(&queue.front(), NULL);执行之前，如果写端没有flush，那么c置为NULL，说明没有数据可读，返回false。
- r = c.cas(&queue.front(), NULL);执行之前，如果写端调用flush，那么c就不等于front()，则r返回了新的f值，最终返回true。

```text
//  Check whether item is available for reading.
// 这里面有两个点，一个是检查是否有数据可读，一个是预取
inline bool check_read() {
    //  Was the value prefetched already? If so, return.
    if (&queue.front() != r && r) //判断是否在前几次调用read函数时已经预取数据了return true;
        return true;

    //  There's no prefetched value, so let us prefetch more values.
    //  Prefetching is to simply retrieve the
    //  pointer from c in atomic fashion. If there are no
    //  items to prefetch, set c to NULL (using compare-and-swap).
    // 两种情况
    // 1. 如果c值和queue.front()， 返回c值并将c值置为NULL，此时没有数据可读
    // 2. 如果c值和queue.front()， 返回c值，此时可能有数据度的去
    r = c.cas(&queue.front(), NULL); //尝试预取数据

    //  If there are no elements prefetched, exit.
    //  During pipe's lifetime r should never be NULL, however,
    //  it can happen during pipe shutdown when items are being deallocated.
    if (&queue.front() == r || !r) //判断是否成功预取数据
        return false;

    //  There was at least one value prefetched.
    return true;
}

//  Reads an item from the pipe. Returns false if there is no value.
//  available.
inline bool read(T *value_) {
    //  Try to prefetch a value.
    if (!check_read())
        return false;

    //  There was at least one value prefetched.
    //  Return it to the caller.
    *value_ = queue.front();
    queue.pop();
    return true;
}
```

### 3.6总结

_c指针，则是读写线程都可以操作，因此需要使⽤原⼦的CAS操作来修改，它的可能值有以下⼏种：

- NULL：读线程设置，此时意味着已经没有数据可读，读线程在休眠。
- ⾮零：写线程设置，这⾥⼜区分两种情况：

```text
- 旧值为_w的情况下，cas(_w,_f)操作修改为_f，意味着如果原先的值为_w，则原⼦性的修改为_f，表示有更多已被刷新的数据可读。
- 在旧值为NULL的情况下，此时读线程休眠，因此可以安全的设置为当前_f指针的位置。
```



```text
- 写端yquque.write(count,false)；将f = &queue.back();
- 写端yquque.flush();如果c==w，则c=f;w=f;否则w=f;
- 读端check_read();如果c==queue.front则c=NULL否则r更新为f。
```

## 4. ZMQ无锁队列1写1读性能测试

这里分三种测试情况：

- 一次写就提交，read失败就usleep
- 10次写才提交，read失败就yield
- flush失败就notify，read失败就wait

可以看到用cond是效率是最高的，usleep的情况和yield的情况类似，实时性没有cond高。并且按照道理来说，正确的使用方法也是用cond

![img](https://pic3.zhimg.com/80/v2-c3e7b784caf442341fabf01e254c86da_720w.webp)

下面来看一看互斥锁队列 vs 互斥锁+条件变量队列 vs 内存屏障链表 vs RingBuffer CAS 实现。可以看到在一个写线程一个读线程的情况下，我们的ZMQ无锁队列是最快的。

![img](https://pic3.zhimg.com/80/v2-38f5ea4a32847da786d66b318501b13e_720w.webp)

那么在一写一读的场景下，我们就优先选用ZMQ无锁队列即可

![img](https://pic3.zhimg.com/80/v2-e3264df82fa2d8474023a83964599962_720w.webp)

## 5. 如何实现多写多读的无锁队列？

后续的多写多读的无锁队列由下一篇文章再来介绍。

原文地址：https://zhuanlan.zhihu.com/p/552982779

作者：linux