# 【NO.222】Linux基础组件之无锁消息队列ypipe/yqueue详解

## 1.CAS定义

比较并交换（compare and swap,CAS），是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新值。

```text
bool CAS( int * pAddr, int nExpected, int nNew ) 
atomically 
{ 
	if ( *pAddr == nExpected ) 
	{ 
		*pAddr = nNew ; 
		return true ; 
	}
	else
		return false ; 
// 返回bool告知原子性交换是否成功
}
```

## 2.为什么需要无锁队列

锁引起的问题：

（1）cache损坏 / 失效
（2）在同步机制上的争抢队列
（3）动态内存分配

![img](https://pic4.zhimg.com/80/v2-d25627fddb63327da1711e0ea8fc880b_720w.webp)

## 3.有锁导致线程切换引发cache损坏

在保存和恢复上下午的过程中还隐藏了额外的开销：Cache中的数据会失效，因为它缓存的是将被换出的任务数据，这些数据对于新换进的任务是没有用的。

CPU的运行速度比主存快很多，所以大量的处理器时间被浪费在处理器与主存的数据传输上，因此，在处理器和主存之间引入Cache。Cache是一种速度更快但容量更小的内存(也更加昂贵),当处理器要访问主存中的数据时，这些数据首先被拷贝到Cache

中，因为这些数据在不久的将来可能又会被处理器访问。Cache misses对性能有非常大的影响，因为处理器访问Cache中的数据将比直接访问主存快得多。线程被频繁抢占产生的Cache损坏将导致应用程序性能下降。

![img](https://pic4.zhimg.com/80/v2-54716fc65cb07e0a55c37cb617511c3b_720w.webp)

## 4.在同步机制上的争抢队列

阻塞导致系统暂停当前的任务或使其进入睡眠状态（等待，不占用CPU资源），直到资源（例如锁机制）可用，被阻塞的任务才能解除阻塞状态（唤醒）。在一个负载较重的应用程序中使用这样的阻塞队列来在线程之间传递消息会导致严重的争用问题。也就是说，任务将大量的时间(睡眠，等待，唤醒)浪费在获得保护队列数据的互斥锁，而不是处理队列中的数据上。

非阻塞机制大展伸手的机会到了。任务之间不争抢任何资源，在队列中预定一个位置，然后在这个位置上插入或提取数据。这中机制使用了一种被称之为CAS(比较和交换)的特殊操作，这个特殊操作是一种特殊的指令，它可以原子的完成以下操作:它需要3个操作数m，A，B，其中m是一个内存地址，操作将m指向的内存中的内容与A比较，如果相等则将B写入到m指向的内存中并返回true，如果不相等则直接返回false。

## 5.动态内存分配

在多线程中，需要仔细考虑动态内存分配。当一个任务从堆中分配内存时，标准的内存分配机制会阻塞所有与这个任务共享地址空间的其他任务（进程中的其他线程）。这样做的原因是让处理更简单，且其工作很好。两个线程不会被分配到一块相同的地址的内存，因为它们没有办法同时执行分配请求。显然线程频繁分配内存会导致应用程序性能下降(必须注意,向标准队列或map插入数据的时候都会导致堆上的动态内存分配)。

## 6.无锁队列的实现

无锁队列由两个类构成：ypipe_t和yqueue_t。zeromq，最快的消息队列。

（1）适用于一读一写的应用场景，比如一个epool+线程池中每个线程绑定一个唯一的队列。

![img](https://pic1.zhimg.com/80/v2-47d6a9f4bd43b90bf62ac68f3a49bcd4_720w.webp)

（2）通过chunk模式批量分配结点，减少因为动态内存分配线程之间的互斥。写线程申请内存、读线程释放内存也会导致动态内存的互斥。

批量分配结点数量没有固定的，需要根据业务场景进行调节；一般设置比较大没有什么问题，设置小了相对容易会产生问题而已。

（3）通过spare_chunk的作用（消息队列水位局部性原理，一般消息数量在一个位置上下波动）来降低chunk的频繁分配和释放。

消息数量在一个位置上下波动时，已经读取元素的chunk不立即释放，而是放在spare_chunk存储，当下一次需要分配chunk时，检查spare_chunk（如果有保存chunk就复用，没有再执行分配）。

![img](https://pic1.zhimg.com/80/v2-3f3e5e9f5f53a766536eec12739ef3a0_720w.webp)

（4）通过预写机制，批量更新写入位置，减少CAS的调用（同时读写消息队列对于CAS是有竞争的）。

![img](https://pic1.zhimg.com/80/v2-90225c4681559fe3788ac2e6874e8cec_720w.webp)

（5）巧妙的唤醒机制。读端没有数据可读时可以进行wait状态；写端在写入数据时可以根据返回值获知写入数据前消息队列是否为空，如果写入之前为空则可以唤醒读端。注意wait是业务层的，无锁消息队列本身没有wait / notify机制。

## 7.ypipe_t无锁队列的使用

![img](https://pic4.zhimg.com/80/v2-0f0677c73bfdd0245fa0cbabbbac3cdb_720w.webp)

yqueue.write(count,false)，写入元素为count，false代表这次已经写完数据，true表示还没写完数据。

yquue.flush()使读端能看到更新后的数据；返回false表示刷新之前队列为空，可notify唤醒读端；返回true说明队列本身有数据。flush才真正调用CAS。

[yqueue.read](https://link.zhihu.com/?target=http%3A//yqueue.read)(&value)读取元素，返回true表示读到元素；返回false表示消息队列为空，可以让出CPU或者进入wait状态等待写端唤醒。

示例1：

```text
// ...

static int s_queue_item_num = 2000000; // 每个线程插入的元素个数
ypipe_t<int, 100> yqueue;
void *yqueue_producer_thread(void *argv)
{
	int count=0;
	for(int i=0;i<s_queue_item_num;)
	{
		yqueue.write(count,false);// write
		count=lxx_atomic_add(&s_count_push,1);//线程安全，原子操作
		i++;
		yqueue.flush();// 刷新
	}
	return NULL;
}

// ...
```

示例2：

```text
void *yqueue_producer_thread_batch(void *argv)
{
  int count = 0;
  int item_num = s_queue_item_num / 10;
  for (int i = 0; i < item_num;)
  {
    yqueue.write(count, true);  // 写true
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, true);
    count = lxx_atomic_add(&s_count_push, 1);
    yqueue.write(count, false);   //最后一个元素 写false
    count = lxx_atomic_add(&s_count_push, 1);
    i++;
    yqueue.flush();//刷新
  }
  return NULL;
}
```

示例3：

```text
// ...

static int s_queue_item_num = 2000000; // 每个线程插入的元素个数
std::mutex ypipe_mutex_;
std::condition_variable ypipe_cond_;
ypipe_t<int, 100> yqueue;
void *yqueue_producer_thread(void *argv)
{
	int count=0;
	for(int i=0;i<s_queue_item_num;)
	{
		yqueue.write(count,false);// write
		count=lxx_atomic_add(&s_count_push,1);//线程安全，原子操作
		i++;
		if(!yqueue.flush())// 返回false，说明读端没有读到数据
		{
			std::unique_lock<std::mutex> lock(ypipe_mutex);
			// 注意，业务层自己实现notify，yqueue本身没有notyfy机制的
			ypipe_cond_.notify_one();
		}
	}
	std::unique_lock<std::mutex> lock(ypipe_mutex);
	ypipe_cond_.notify_one();
	return NULL;
}

// ...
```

## 8.源码分析原子操作函数

![img](https://pic2.zhimg.com/80/v2-14319e6a20998276d132beaabd18b449_720w.webp)

set函数，把私有成员ptr指针设置成参数ptr_的值，不是一个原子操作，需要使用者确保执行set过程没有其他线程使用ptr的值。

```text
// This class encapsulates several atomic operations on pointers. 
template <typename T> class atomic_ptr_t 
{
public: 
	inline void set (T *ptr_); //非原子操作 
	inline T *xchg (T *val_); //原子操作，设置一个新的值，然后返回旧的值 
	inline T *cas (T *cmp_, T *val_);//原子操作 
private: 
	volatile T *ptr; 
}
```

## 9.源码分析yqueue_t

yqueue_t是比ypipe_t更底层的类。用于消息队列结点元素存储；不涉及CAS。

### 9.1类接口和变量

```text
#ifndef __ZMQ_YQUEUE_HPP_INCLUDED__
#define __ZMQ_YQUEUE_HPP_INCLUDED__

#include <stdlib.h>
#include <stddef.h>

// #include "err.hpp"
#include "atomic_ptr.hpp"

//  即是yqueue_t一个结点可以装载N个T类型的元素， yqueue_t的一个结点是一个数组
template <typename T, int N>
class yqueue_t
{
public:
    //  创建队列.
    inline yqueue_t();

    //  销毁队列.
    inline ~yqueue_t();

    // 返回队列头部元素的引用，调用者可以通过该引用更新元素，结合pop实现出队列操作。
    inline T &front(); // 返回的是引用，是个左值，调用者可以通过其修改容器的值
    
    // 返回队列尾部元素的引用，调用者可以通过该引用更新元素，结合push实现插入操作。
    // 如果队列为空，该函数是不允许被调用。
    inline T &back(); // 返回的是引用，是个左值，调用者可以通过其修改容器的值
    
    //  Adds an element to the back end of the queue.
    inline void push();

    // 必须要保证队列不为空，参考ypipe_t的uwrite
    inline void unpush();

    //  Removes an element from the front end of the queue.
    inline void pop();

private:
    //  Individual memory chunk to hold N elements.
    // 链表结点称之为chunk_t
    struct chunk_t
    {
        T values[N]; //每个chunk_t可以容纳N个T类型的元素，以后就以一个chunk_t为单位申请内存
        chunk_t *prev;
        chunk_t *next;
    };

    //  Back position may point to invalid memory if the queue is empty,
    //  while begin & end positions are always valid. Begin position is
    //  accessed exclusively be queue reader (front/pop), while back and
    //  end positions are accessed exclusively by queue writer (back/push).
    chunk_t *begin_chunk; // 链表头结点
    int begin_pos;        // 起始点
    chunk_t *back_chunk;  // 队列中最后一个元素所在的链表结点
    int back_pos;         // 尾部
    chunk_t *end_chunk;   // 拿来扩容的，总是指向链表的最后一个结点
    int end_pos;

    //  People are likely to produce and consume at similar rates.  In
    //  this scenario holding onto the most recently freed chunk saves
    //  us from having to call malloc/free.
    atomic_ptr_t<chunk_t> spare_chunk; //空闲块（把所有元素都已经出队的块称为空闲块），读写线程的共享变量

    //  Disable copying of yqueue.
    yqueue_t(const yqueue_t &);
    const yqueue_t &operator=(const yqueue_t &);
};

#endif
```

### 9.2 数据结构和逻辑

```text
// 链表结点称之为chunk_t
struct chunk_t
{
    T values[N]; //每个chunk_t可以容纳N个T类型的元素，以后就以一个chunk_t为单位申请内存
    chunk_t *prev;
    chunk_t *next;
};
```

yqueue_t的实现，每次批量分配一批元素，减少内存的分配和释放，解决不断动态内存分配的问题。yqueue_t内部由一个个chunk构成，每个chunk保存N个元素；chunk_t是一个双向链表。

![img](https://pic4.zhimg.com/80/v2-205a7b33e506e6cca5511a9b813f18cb_720w.webp)

当队列空间不足时每次分配一个chunk_t，每个chunk_t能存储N个元素。

在数据出队列后，队列有多余空间的时候，回收的chunk也不是马上释放，而是根据局部性原理先回收到spare_chunk里面，当再次需要分配chunk_t的时候从spare_chunk中获取。

![img](https://pic2.zhimg.com/80/v2-466847b514e239c06376bb249b099981_720w.webp)

yqueue_t内部有三个chunk_t类型指针以及对应的索引位置：

begin_chunk/begin_pos：begin_chunk用于指向队列头的chunk，begin_pos用于指向队列第一个元素在当前chunk中的位置。

back_chunk/back_pos：back_chunk用于指向队列尾的chunk，back_pos用于指向队列最后一个元素在当前chunk的位置。

end_chunk/end_pos：由于chunk是批量分配的，所以end_chunk用于指向分配的最后一个chunk位置。



这里特别需要注意区分back_chunk/back_pos和end_chunk/end_pos的作用：

back_chunk/back_pos：对应的是元素存储位置。

end_chunk/end_pos：决定是否要分配chunk或者回收chunk。



示例：

![img](https://pic1.zhimg.com/80/v2-de2a91a1a10c4b9bba8191b2646814f4_720w.webp)

另外还有一个spare_chunk指针，用于保存释放的chunk指针，当需要再次分配chunk的时候，会首先查看这里，从这里分配chunk。这里使用了原子的cas操作来完成，利用了操作系统的局部性原理。

### 9.3 yqueue_t构造函数

```text
//  创建队列.
inline yqueue_t()
{
    begin_chunk = (chunk_t *)malloc(sizeof(chunk_t));
    alloc_assert(begin_chunk);
    begin_pos = 0;
    back_chunk = NULL; //back_chunk总是指向队列中最后一个元素所在的chunk，现在还没有元素，所以初始为空
    back_pos = 0;
    end_chunk = begin_chunk; //end_chunk总是指向链表的最后一个chunk
    end_pos = 0;
}
```

![img](https://pic4.zhimg.com/80/v2-8589d70f626be7770c88f77850c77e8b_720w.webp)

end_chunk总是指向最后分配的chunk，刚分配出来的chunk，end_pos也总是为0。

back_chunk需要chunk有元素插入的时候才指向对应的chunk。

### 9.4 front()和back()函数

这两个函数的作用与C++ STL queue的front和back函数相同的效果。

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

这里的front()或者back()函数，需要注意的返回的是左值引用，我们可以修改其值。

对于先进后出队列而言：

begin_chunk->values[begin_pos]代表队列头可读元素， 读取队列头元素即是读取begin_pos位置的元素；

back_chunk->values[back_pos]代表队列尾可写元素，写入元素时则是更新back_pos位置的元素，要确保元素真正生效，还需要调用push函数更新back_pos的位置，避免下次更新的时候又是更新当前back_pos位置对应的元素。

### 9.5 push()函数

更新下一个元素写入位置，如果end_pos超过chunk的索引位置(==N)则申请一个chunk（先尝试从spare_chunk获取，如果为空再申请分配全新的chunk）。

**最终都是要更新end_chunk和end_pos。**

```text
//  Adds an element to the back end of the queue.
inline void push()
{
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
        end_chunk->next = (chunk_t *)malloc(sizeof(chunk_t)); // 分配一个chunk
        alloc_assert(end_chunk->next);
        end_chunk->next->prev = end_chunk;  
    }
    end_chunk = end_chunk->next;
    end_pos = 0;
}
```

push()函数的使用：

（1）通过back()获取可写入位置，写入数据；

（2）**通过push()更新下一个可写位置**。

```text
// 写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
inline void write(const T &value_, bool incomplete_)
{
    //  Place the value to the queue, add new terminator element.
    queue.back() = value_;
    queue.push();

    //  Move the "flush up to here" poiter.
    if (!incomplete_)
    {
        f = &queue.back(); // 记录要刷新的位置
    }
    else
    {
        //  ...
    }
}
```

### 9.6 pop()函数

这里主要更新下一次读取的位置，并检测是否需要释放chunk（先保存到spare_chunk，然后检测spare_chunk返回值是否为空，如果返回值不为空说明之前有保存chunk，但我们只能保存一个chunk，所以把之前的chunk释放掉）

```text
//  Removes an element from the front end of the queue.
inline void pop()
{
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

整个chunk的元素都被取出队列才去回收chunk，而且是把最后回收的chunk保存到spare_chunk，然后释放之前保存的chunk。

需要注意：

（1）pop掉的元素，其销毁工作交给调用者完成，即是pop前调用者需要通过front()接口读取并进行销毁（比如动态分配的对象）。

（2）空闲块的保存，要求是原子操作。因为闲块是读写线程的共享变量，因为在push中也使用了spare_chunk。

push()函数的使用：

（1）通过front()读取数据；

（2）读完数据后通过pop()更新下一个可读位置。

### 9.7 源码

```text
/*
    Copyright (c) 2007-2013 Contributors as noted in the AUTHORS file

    This file is part of 0MQ.

    0MQ is free software; you can redistribute it and/or modify it under
    the terms of the GNU Lesser General Public License as published by
    the Free Software Foundation; either version 3 of the License, or
    (at your option) any later version.

    0MQ is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU Lesser General Public License for more details.

    You should have received a copy of the GNU Lesser General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef __ZMQ_YQUEUE_HPP_INCLUDED__
#define __ZMQ_YQUEUE_HPP_INCLUDED__

#include <stdlib.h>
#include <stddef.h>

// #include "err.hpp"
#include "atomic_ptr.hpp"

//  yqueue is an efficient queue implementation. The main goal is
//  to minimise number of allocations/deallocations needed. Thus yqueue
//  allocates/deallocates elements in batches of N.
//
//  yqueue allows one thread to use push/back function and another one
//  to use pop/front functions. However, user must ensure that there's no
//  pop on the empty queue and that both threads don't access the same
//  element in unsynchronised manner.
//
//  T is the type of the object in the queue. 队列中元素的类型
//  N is granularity(粒度) of the queue (how many pushes have to be done till  actual memory allocation is required).
//  即是yqueue_t一个结点可以装载N个T类型的元素， yqueue_t的一个结点是一个数组
template <typename T, int N>
class yqueue_t
{
public:
    //  创建队列.
    inline yqueue_t()
    {
        begin_chunk = (chunk_t *)malloc(sizeof(chunk_t));
        alloc_assert(begin_chunk);
        begin_pos = 0;
        back_chunk = NULL; //back_chunk总是指向队列中最后一个元素所在的chunk，现在还没有元素，所以初始为空
        back_pos = 0;
        end_chunk = begin_chunk; //end_chunk总是指向链表的最后一个chunk
        end_pos = 0;
    }

    //  销毁队列.
    inline ~yqueue_t()
    {
        while (true)
        {
            if (begin_chunk == end_chunk)
            {
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

    //  Adds an element to the back end of the queue.
    inline void push()
    {
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
            end_chunk->next = (chunk_t *)malloc(sizeof(chunk_t)); // 分配一个chunk
            alloc_assert(end_chunk->next);
            end_chunk->next->prev = end_chunk;  
        }
        end_chunk = end_chunk->next;
        end_pos = 0;
    }

    //  Removes element from the back end of the queue. In other words
    //  it rollbacks last push to the queue. Take care: Caller is
    //  responsible for destroying the object being unpushed.
    //  The caller must also guarantee that the queue isn't empty when
    //  unpush is called. It cannot be done automatically as the read
    //  side of the queue can be managed by different, completely
    //  unsynchronised thread.
    // 必须要保证队列不为空，参考ypipe_t的uwrite
    inline void unpush()
    {
        //  First, move 'back' one position backwards.
        if (back_pos) // 从尾部删除元素
            --back_pos;
        else
        {
            back_pos = N - 1; // 回退到前一个chunk
            back_chunk = back_chunk->prev;
        }

        //  Now, move 'end' position backwards. Note that obsolete end chunk
        //  is not used as a spare chunk. The analysis shows that doing so
        //  would require free and atomic operation per chunk deallocated
        //  instead of a simple free.
        if (end_pos) // 意味着当前的chunk还有其他元素占有
            --end_pos;
        else
        {
            end_pos = N - 1; // 当前chunk没有元素占用，则需要将整个chunk释放
            end_chunk = end_chunk->prev;
            free(end_chunk->next);
            end_chunk->next = NULL;
        }
    }

    //  Removes an element from the front end of the queue.
    inline void pop()
    {
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

private:
    //  Individual memory chunk to hold N elements.
    // 链表结点称之为chunk_t
    struct chunk_t
    {
        T values[N]; //每个chunk_t可以容纳N个T类型的元素，以后就以一个chunk_t为单位申请内存
        chunk_t *prev;
        chunk_t *next;
    };

    //  Back position may point to invalid memory if the queue is empty,
    //  while begin & end positions are always valid. Begin position is
    //  accessed exclusively be queue reader (front/pop), while back and
    //  end positions are accessed exclusively by queue writer (back/push).
    chunk_t *begin_chunk; // 链表头结点
    int begin_pos;        // 起始点
    chunk_t *back_chunk;  // 队列中最后一个元素所在的链表结点
    int back_pos;         // 尾部
    chunk_t *end_chunk;   // 拿来扩容的，总是指向链表的最后一个结点
    int end_pos;

    //  People are likely to produce and consume at similar rates.  In
    //  this scenario holding onto the most recently freed chunk saves
    //  us from having to call malloc/free.
    atomic_ptr_t<chunk_t> spare_chunk; //空闲块（把所有元素都已经出队的块称为空闲块），读写线程的共享变量

    //  Disable copying of yqueue.
    yqueue_t(const yqueue_t &);
    const yqueue_t &operator=(const yqueue_t &);
};

#endif
```

## 10.源码分析ypipe_t

ypipe_t相对yqueue难理解。ypipe_t用于控制读写位置；这涉及到CAS的问题；读写存在临界点。ypipe_t在yqueue_t的基础上构建一个单写单读的无锁队列。

### 10.1类接口和变量

```text
#ifndef __ZMQ_YPIPE_HPP_INCLUDED__
#define __ZMQ_YPIPE_HPP_INCLUDED__

#include "atomic_ptr.hpp"
#include "yqueue.hpp"

//  Lock-free queue implementation.
//  Only a single thread can read from the pipe at any specific moment.
//  Only a single thread can write to the pipe at any specific moment.
//  T is the type of the object in the queue.
//  N is granularity of the pipe, i.e. how many items are needed to
//  perform next memory allocation.

template <typename T, int N>
class ypipe_t
{
public:
    //  Initialises the pipe.
    inline ypipe_t();

    //  The destructor doesn't have to be virtual. It is mad virtual
    //  just to keep ICC and code checking tools from complaining.
    inline virtual ~ypipe_t()
    {
    }

    //  Following function (write) deliberately copies uninitialised data
    //  when used with zmq_msg. Initialising the VSM body for
    //  non-VSM messages won't be good for performance.

#ifdef ZMQ_HAVE_OPENVMS
#pragma message save
#pragma message disable(UNINIT)
#endif

    //  Write an item to the pipe.  Don't flush it yet. If incomplete is
    //  set to true the item is assumed to be continued by items
    //  subsequently written to the pipe. Incomplete items are neverflushed down the stream.
    // 写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
    inline void write(const T &value_, bool incomplete_);

#ifdef ZMQ_HAVE_OPENVMS
#pragma message restore
#endif

    //  Pop an incomplete item from the pipe. Returns true is such
    //  item exists, false otherwise.
    inline bool unwrite(T *value_);

    //  Flush all the completed items into the pipe. Returns false if
    //  the reader thread is sleeping. In that case, caller is obliged to
    //  wake the reader up before using the pipe again.
    // 刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调用者需要唤醒读线程。
    // 批量刷新的机制， 写入批量后唤醒读线程；
    // 反悔机制 unwrite
    inline bool flush();

    //  Check whether item is available for reading.
    // 这里面有两个点，一个是检查是否有数据可读，一个是预取
    inline bool check_read();
    //  Reads an item from the pipe. Returns false if there is no value.
    //  available.
    inline bool read(T *value_);

    //  Applies the function fn to the first elemenent in the pipe
    //  and returns the value returned by the fn.
    //  The pipe mustn't be empty or the function crashes.
    inline bool probe(bool (*fn)(T &));

protected:
    //  Allocation-efficient queue to store pipe items.
    //  Front of the queue points to the first prefetched item, back of
    //  the pipe points to last un-flushed item. Front is used only by
    //  reader thread, while back is used only by writer thread.
    yqueue_t<T, N> queue;

    //  Points to the first un-flushed item. This variable is used
    //  exclusively by writer thread.
    T *w; //指向第一个未刷新的元素,只被写线程使用

    //  Points to the first un-prefetched item. This variable is used
    //  exclusively by reader thread.
    T *r; //指向第一个还没预提取的元素，只被读线程使用

    //  Points to the first item to be flushed in the future.
    T *f; //指向下一轮要被刷新的一批元素中的第一个

    //  The single point of contention between writer and reader thread.
    //  Points past the last flushed item. If it is NULL,
    //  reader is asleep. This pointer should be always accessed using
    //  atomic operations.
    atomic_ptr_t<T> c; //读写线程共享的指针，指向每一轮刷新的起点（看代码的时候会详细说）。当c为空时，表示读线程睡眠（只会在读线程中被设置为空）

    //  Disable copying of ypipe object.
    ypipe_t(const ypipe_t &);
    const ypipe_t &operator=(const ypipe_t &);
};

#endif
```

**核心要点：**

（1）T *w：指向第一个未刷新的元素，只被写线程使用；用来控制是否需要唤醒读端，当读端没有数据可以读取的时候，将c变量设为NULL。

（2）T *r：指向第一个还没有预取的元素，只被读线程使用；用来控制可读位置，（注意）这个r不是读位置的索引（读位置索引是begin_pos，可写位置索引是back_pos），而是读索引的位置等于r的时候说明队列已经为空。

（3）T *f：指向下一轮要被刷新的一批元素中的第一个；用来控制写入位置，当f被更新到c的时候读端才能看到写入的数据。

（4）atomic_ptr_t c：读写线程共享的指针，指向每一轮刷新的起点；当c为空时，表示读线程睡眠（只会在读线程中被设置为空）。

**主要接口：**

（1）void write (const T &value, bool incomplete)：写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。

（2）bool flush ()：刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调用者需要唤醒读线程。

（3）bool read (T *value_):读数据，将读出的数据写入value指针中，返回false意味着没有数据可读。

### 10.2 ypipe_t()初始化

```text
//  Initialises the pipe.
inline ypipe_t()
{
    //  Insert terminator element into the queue.
    queue.push(); //yqueue_t的尾指针加1，开始back_chunk为空，现在back_chunk指向第一个chunk_t块的第一个位置

    //  Let all the pointers to point to the terminator.
    //  (unless pipe is dead, in which case c is set to NULL).
    r = w = f = &queue.back(); //就是让r、w、f、c四个指针都指向这个end迭代器
    c.set(&queue.back());
}
```

![img](https://pic1.zhimg.com/80/v2-73d4bae28965c1a4e16d77575173ec90_720w.webp)

### 10.3 write()函数

```text
// 写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
inline void write(const T &value_, bool incomplete_)
{
    //  Place the value to the queue, add new terminator element.
    queue.back() = value_;
    queue.push();//更新下一次写的位置

    //  Move the "flush up to here" poiter.
    if (!incomplete_)// 如果f不更新，flush的时候 read也是没有数据
    {
        f = &queue.back(); // 记录要刷新的位置
        // printf("1 f:%p, w:%p\n", f, w);
    }
    else
    {
        //  printf("0 f:%p, w:%p\n", f, w);
    }
}
```

![img](https://pic2.zhimg.com/80/v2-48c8725e3a9789ff9d840cdc8f4eb811_720w.webp)

write(val, false); 触发更新f的位置。f实际是back_pos的位置，即是下一次可以写入的位置。

### 10.4 cas()函数

cas函数，原子操作，线程安全，把私有成员ptr指针与参数cmp_指针比较：

（1）如果相等，就把ptr设置为参数val_的值，返回ptr设置之前的值；

（2）如果不相等直接返回ptr值。

```text
        //  Perform atomic 'compare and swap' operation on the pointer.
        //  The pointer is compared to 'cmp' argument and if they are
        //  equal, its value is set to 'val'. Old value of the pointer
        //  is returned.
        // 原来的值(ptr指向)如果和 comp_的值相同则更新为val_,并返回原来的ptr
        //   ○ 如果相等返回ptr设置之前的值，并把ptr更新为参数val_的值，；
        //   ○ 如果不相等直接返回ptr值。
        inline T *cas (T *cmp_, T *val_)//原子操作
        {
#if defined ZMQ_ATOMIC_PTR_ATOMIC_H
            return (T*) atomic_cas_ptr (&ptr, cmp_, val_);
#elif defined ZMQ_ATOMIC_PTR_TILE
            return (T*) arch_atomic_val_compare_and_exchange (&ptr, cmp_, val_);
#elif defined ZMQ_ATOMIC_PTR_X86
            T *old;
            __asm__ volatile (
                "lock; cmpxchg %2, %3"
                : "=a" (old), "=m" (ptr)
                : "r" (val_), "m" (ptr), "0" (cmp_)
                : "cc");
            return old;
#else
#error atomic_ptr is not implemented for this platform
#endif
        }
```

### 10.5 flush()函数

主要是将w更新到f位置（伴随着是否更新c），说明已经写到的位置。

```text
// 刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调用者需要唤醒读线程。
// 批量刷新的机制， 写入批量后唤醒读线程；
// 反悔机制 unwrite
inline bool flush()
{
    //  If there are no un-flushed items, do nothing.
    if (w == f) // 不需要刷新，即是还没有新元素加入
        return true;

    //  Try to set 'c' to 'f'.
    // read时如果没有数据可以读取则c的值会被置为NULL
    if (c.cas(w, f) != w) // 尝试将c设置为f，即是准备更新w的位置
    {
		// - 如果c==w，则更新f->c，并返回原来的c； 
		// - 如果c!=w， 则返回原来的c；
        //  Compare-and-swap was unseccessful because 'c' is NULL.
        //  This means that the reader is asleep. Therefore we don't
        //  care about thread-safeness and update c in non-atomic
        //  manner. We'll return false to let the caller know
        //  that reader is sleeping.
        c.set(f); // 更新w的位置
        w = f;
        return false; //线程看到flush返回false之后会发送一个消息给读线程，这需要写业务去做处理
    }
    else  // 读端还有数据可读取
    {
        //  Reader is alive. Nothing special to do now. Just move
        //  the 'first un-flushed item' pointer to 'f'.
        w = f;             // 只需要更新w的位置
        return true;
    }
}
```

flush()后w一定为f，w的作用主要是用来控制return false/true。

刷新之后，w、f、c、r的关系：

![img](https://pic4.zhimg.com/80/v2-1ce2f1922721c872f53c3178e1db48ff_720w.webp)

### 10.6 read()函数

r实际上是用来控制可以读取到的位置（**注意不是读到r，而是r的前一位置可以读取，r位置是不可以读取的**），当front和r重叠的时候说明没有数据可以读取。以此来检测是否有数据可以读取。

```text
//  Check whether item is available for reading.
// 这里面有两个点，一个是检查是否有数据可读，一个是预取
inline bool check_read()
{
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
inline bool read(T *value_)
{
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

指针r指向队头元素【r==&queue.front()】或者r不指向任何数据（即NULL），说明队列中没有可读的数据；这个时候check_read()会尝试去预取数据（就是令 r=c）。而c在write中被指向f（见上图），这时从queue.front()到f这个位置的数据都被预取出来了，然后每次调用read都能取出一段。值得注意的是，当c==&queue.front()时，代表数据被取完了，这时把c指向NULL，接着读线程会睡眠，这也是给写线程检查读线程是否睡眠的标志。

继续上图的场景：

![img](https://pic1.zhimg.com/80/v2-bf75d1a04f107f8f9f6840a8a30d5308_720w.webp)

在 【[7.read](https://link.zhihu.com/?target=http%3A//7.read)(&ret),函数返回false,ret没有获取到值】的时候，front()和r相等。

（1）如果此时在r = c.cas(&queue.front(), NULL); 执行时没有flush的操作。则说明没有数据可以读取，最终返回false；

（2）如果在r = c.cas(&queue.front(), NULL); 之前写入方write新数据后并调用了flush，则r被更新，最终返回true。

### 10.7 源码

```text
/*
    Copyright (c) 2007-2013 Contributors as noted in the AUTHORS file

    This file is part of 0MQ.

    0MQ is free software; you can redistribute it and/or modify it under
    the terms of the GNU Lesser General Public License as published by
    the Free Software Foundation; either version 3 of the License, or
    (at your option) any later version.

    0MQ is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU Lesser General Public License for more details.

    You should have received a copy of the GNU Lesser General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
*/

#ifndef __ZMQ_YPIPE_HPP_INCLUDED__
#define __ZMQ_YPIPE_HPP_INCLUDED__

#include "atomic_ptr.hpp"
#include "yqueue.hpp"

//  Lock-free queue implementation.
//  Only a single thread can read from the pipe at any specific moment.
//  Only a single thread can write to the pipe at any specific moment.
//  T is the type of the object in the queue.
//  N is granularity of the pipe, i.e. how many items are needed to
//  perform next memory allocation.

template <typename T, int N>
class ypipe_t
{
public:
    //  Initialises the pipe.
    inline ypipe_t()
    {
        //  Insert terminator element into the queue.
        queue.push(); //yqueue_t的尾指针加1，开始back_chunk为空，现在back_chunk指向第一个chunk_t块的第一个位置

        //  Let all the pointers to point to the terminator.
        //  (unless pipe is dead, in which case c is set to NULL).
        r = w = f = &queue.back(); //就是让r、w、f、c四个指针都指向这个end迭代器
        c.set(&queue.back());
    }

    //  The destructor doesn't have to be virtual. It is mad virtual
    //  just to keep ICC and code checking tools from complaining.
    inline virtual ~ypipe_t()
    {
    }

    //  Following function (write) deliberately copies uninitialised data
    //  when used with zmq_msg. Initialising the VSM body for
    //  non-VSM messages won't be good for performance.

#ifdef ZMQ_HAVE_OPENVMS
#pragma message save
#pragma message disable(UNINIT)
#endif

    //  Write an item to the pipe.  Don't flush it yet. If incomplete is
    //  set to true the item is assumed to be continued by items
    //  subsequently written to the pipe. Incomplete items are neverflushed down the stream.
    // 写入数据，incomplete参数表示写入是否还没完成，在没完成的时候不会修改flush指针，即这部分数据不会让读线程看到。
    inline void write(const T &value_, bool incomplete_)
    {
        //  Place the value to the queue, add new terminator element.
        queue.back() = value_;
        queue.push();

        //  Move the "flush up to here" poiter.
        if (!incomplete_)
        {
            f = &queue.back(); // 记录要刷新的位置
            // printf("1 f:%p, w:%p\n", f, w);
        }
        else
        {
            //  printf("0 f:%p, w:%p\n", f, w);
        }
    }

#ifdef ZMQ_HAVE_OPENVMS
#pragma message restore
#endif

    //  Pop an incomplete item from the pipe. Returns true is such
    //  item exists, false otherwise.
    inline bool unwrite(T *value_)
    {
        if (f == &queue.back())
            return false;
        queue.unpush();
        *value_ = queue.back();
        return true;
    }

    //  Flush all the completed items into the pipe. Returns false if
    //  the reader thread is sleeping. In that case, caller is obliged to
    //  wake the reader up before using the pipe again.
    // 刷新所有已经完成的数据到管道，返回false意味着读线程在休眠，在这种情况下调用者需要唤醒读线程。
    // 批量刷新的机制， 写入批量后唤醒读线程；
    // 反悔机制 unwrite
    inline bool flush()
    {
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
            c.set(f); // 更新w的位置
            w = f;
            return false; //线程看到flush返回false之后会发送一个消息给读线程，这需要写业务去做处理
        }
        else  // 读端还有数据可读取
        {
            //  Reader is alive. Nothing special to do now. Just move
            //  the 'first un-flushed item' pointer to 'f'.
            w = f;             // 只需要更新w的位置
            return true;
        }
    }

    //  Check whether item is available for reading.
    // 这里面有两个点，一个是检查是否有数据可读，一个是预取
    inline bool check_read()
    {
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
    inline bool read(T *value_)
    {
        //  Try to prefetch a value.
        if (!check_read())
            return false;

        //  There was at least one value prefetched.
        //  Return it to the caller.
        *value_ = queue.front();
        queue.pop();
        return true;
    }

    //  Applies the function fn to the first elemenent in the pipe
    //  and returns the value returned by the fn.
    //  The pipe mustn't be empty or the function crashes.
    inline bool probe(bool (*fn)(T &))
    {
        bool rc = check_read();
        // zmq_assert(rc);

        return (*fn)(queue.front());
    }

protected:
    //  Allocation-efficient queue to store pipe items.
    //  Front of the queue points to the first prefetched item, back of
    //  the pipe points to last un-flushed item. Front is used only by
    //  reader thread, while back is used only by writer thread.
    yqueue_t<T, N> queue;

    //  Points to the first un-flushed item. This variable is used
    //  exclusively by writer thread.
    T *w; //指向第一个未刷新的元素,只被写线程使用

    //  Points to the first un-prefetched item. This variable is used
    //  exclusively by reader thread.
    T *r; //指向第一个还没预提取的元素，只被读线程使用

    //  Points to the first item to be flushed in the future.
    T *f; //指向下一轮要被刷新的一批元素中的第一个

    //  The single point of contention between writer and reader thread.
    //  Points past the last flushed item. If it is NULL,
    //  reader is asleep. This pointer should be always accessed using
    //  atomic operations.
    atomic_ptr_t<T> c; //读写线程共享的指针，指向每一轮刷新的起点（看代码的时候会详细说）。当c为空时，表示读线程睡眠（只会在读线程中被设置为空）

    //  Disable copying of ypipe object.
    ypipe_t(const ypipe_t &);
    const ypipe_t &operator=(const ypipe_t &);
};

#endif
```

## 11.总结

ypipe_t / yqueue_t无锁队列是单写单读，通过chunk机制避免频繁内存动态分配（内存分配或释放时，多个线程之间存在锁的竞争）。

ypipe_t / yqueue_t局部性原理，消息队列（概率性）在某一段时间（时间极短）可能存在波动，复用最近回收的chunk，提升效率。

flush()函数可以检测队列之前是否为空（目的是通知对端唤醒），flush()后就有数据可读了。

原文地址：https://zhuanlan.zhihu.com/p/596885618

作者：linux