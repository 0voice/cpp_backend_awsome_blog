# 【NO.182】【源码剖析】MemoryPool —— 简单高效的内存池 allocator 实现

## 1.什么是内存池？什么是 C++ 的 allocator？

内存池简单说，是为了减少频繁使用 malloc/free new/delete 等系统调用而造成的性能损耗而设计的。当我们的程序需要频繁地申请和释放内存时，频繁地使用内存管理的系统调用可能会造成性能的瓶颈，嗯，是可能，毕竟操作系统的设计也不是盖的（麻麻说把话说太满会被打脸的(⊙v⊙)）。内存池的思想是申请较大的一块内存（不够时继续申请），之后把内存管理放在应用层执行，减少系统调用的开销。

那么，allocator 呢？它默默的工作在 C++ 所有容器的内存分配上。默默贴几个链接吧：

- [http://www.cnblogs.com/wpcockroach/archive/2012/05/10/2493564.html](https://link.zhihu.com/?target=http%3A//www.cnblogs.com/wpcockroach/archive/2012/05/10/2493564.html)
- [http://blog.csdn.net/justaipanda/article/details/7790355](https://link.zhihu.com/?target=http%3A//blog.csdn.net/justaipanda/article/details/7790355)
- [http://www.cplusplus.com/reference/memory/allocator/](https://link.zhihu.com/?target=http%3A//www.cplusplus.com/reference/memory/allocator/)
- [http://www.cplusplus.com/reference/memory/allocator_traits/](https://link.zhihu.com/?target=http%3A//www.cplusplus.com/reference/memory/allocator_traits/)

当你对 allocator 有基本的了解之后，再看这个项目应该会有恍然大悟的感觉，因为这个内存池是以一个 allocator 的标准来实现的。一开始不明白项目里很多函数的定义是为了什么，结果初步了解了 allocator 后才知道大部分是标准接口。这样一个 memory pool allocator 可以与大多数 STL 容器兼容，也可以应用于你自定义的类。像作者给出的例子 —— test.cpp， 是用一个基于自己写的 stack 来做 memory pool allocator 和 std::allocator 性能的对比 —— 最后当然是 memory pool allocator 更优。

## 2.项目

Github：[MemoryPool](https://link.zhihu.com/?target=https%3A//github.com/cacay/MemoryPool)

## 3.基本使用

因为这是一个 allocator 类，所以所有使用 std::allocator 的地方都可以使用这个 MemoryPool。在项目的 test.cpp 中，MemoryPool 作为 allocator 用于 StackAlloc（作者实现的 demo 类） 的内存管理类。定义如下：

```text
StackAlloc<int, MemoryPool<int> > stackPool;
```

其次，你也可以将其直接作为任一类型的内存池，用 newElement 创建新元素，deleteElement 释放元素，就像 new/delete 一样。用下面的例子和 new/delete 做对比：

```text
#include <iostream>
#include <cassert>
#include <time.h>
#include <vector>
#include <stack>

#include "MemoryPool.h"

using namespace std;

/* Adjust these values depending on how much you trust your computer */
#define ELEMS 1000000
#define REPS 50

int main()
{

    clock_t start;

    MemoryPool<size_t> pool;
    start = clock();
    for(int i = 0;i < REPS;++i)
    {
        for(int j = 0;j< ELEMS;++j)
        {
            // 创建元素
            size_t* x = pool.newElement();

            // 释放元素
            pool.deleteElement(x);
        }
    }
    std::cout << "MemoryPool Time: ";
    std::cout << (((double)clock() - start) / CLOCKS_PER_SEC) << "\n\n";


    start = clock();
    for(int i = 0;i < REPS;++i)
    {
        for(int j = 0;j< ELEMS;++j)
        {
            size_t* x = new size_t;

            delete x;
        }
    }
    std::cout << "new/delete Time: ";
    std::cout << (((double)clock() - start) / CLOCKS_PER_SEC) << "\n\n";

    return 0;

}
```

运行的结果是：

```text
MemoryPool Time: 1.93389

new/delete Time: 4.64903
```

嗯，内存池快了一倍多。如果是自定义的类的话这个差距应该还会更大一点。

## 4.代码分析：

项目的实现有 C++11 和 C++98 两个版本，C++11 版本似乎更加高效，不过个人 C++11 了解不多，就以 C++98 版本来分析吧。

### 4.1 主要函数

- allocate 分配一个对象所需的内存空间
- deallocate 释放一个对象的内存（归还给内存池，不是给操作系统）
- construct 在已申请的内存空间上构造对象
- destroy 析构对象
- newElement 从内存池申请一个对象所需空间，并调用对象的构造函数
- deleteElement 析构对象，将内存空间归还给内存池
- allocateBlock 从操作系统申请一整块内存放入内存池

### 4.2 关键知识点

理解项目的关键在于理解 placement new 和 union 的用法.

placement new: [http://blog.csdn.net/zhangxinrun/article/details/5940019](https://link.zhihu.com/?target=http%3A//blog.csdn.net/zhangxinrun/article/details/5940019)

union：[http://www.cnblogs.com/BeyondTechnology/archive/2010/09/19/1831293.html](https://link.zhihu.com/?target=http%3A//www.cnblogs.com/BeyondTechnology/archive/2010/09/19/1831293.html)

关于 union 的使用觉得好巧妙，这是相应的定义：

```text
    union Slot_ {
      value_type element;
      Slot_* next;
    };
```

Slot_ 在创建对象的时候存放对象的值，当这个对象被释放时这块内存作为一个 Slot_* 指针放入 free 的链表。所以 Slot_ 既可以用来存放对象，又可以用来构造链表。

### 4.3 工作原理

内存池是一个一个的 block 以链表的形式连接起来，每一个 block 是一块大的内存，当内存池的内存不足的时候，就会向操作系统申请新的 block 加入链表。还有一个 freeSlots_ 的链表，链表里面的每一项都是对象被释放后归还给内存池的空间，内存池刚创建时 freeSlots_ 是空的，之后随着用户创建对象，再将对象释放掉，这时候要把内存归还给内存池，怎么归还呢？就是把指向这个对象的内存的指针加到 freeSlots_ 链表的前面（前插）。

用户在创建对象的时候，先检查 freeSlots_ 是否为空，不为空的时候直接取出一项作为分配出的空间。否则就在当前 block 内取出一个 Slot_ 大小的内存分配出去，如果 block 里面的内存已经使用完了呢？就向操作系统申请一个新的 block。

内存池工作期间的内存只会增长，不释放给操作系统。直到内存池销毁的时候，才把所有的 block delete 掉。

### 4.4 注释源码

[点我到 Github](https://link.zhihu.com/?target=https%3A//github.com/AngryHacker/code-with-comments/tree/master/memorypool)

 **头文件**

```text
/*-
 * Copyright (c) 2013 Cosku Acay, http://www.coskuacay.com
 *
 * Permission is hereby granted, free of charge, to any person obtaining a
 * copy of this software and associated documentation files (the "Software"),
 * to deal in the Software without restriction, including without limitation
 * the rights to use, copy, modify, merge, publish, distribute, sublicense,
 * and/or sell copies of the Software, and to permit persons to whom the
 * Software is furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
 * FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
 * IN THE SOFTWARE.
 */

#ifndef MEMORY_POOL_H
#define MEMORY_POOL_H

#include <limits.h>
#include <stddef.h>

template <typename T, size_t BlockSize = 4096>
class MemoryPool
{
  public:
    /* Member types */
    typedef T               value_type;       // T 的 value 类型
    typedef T*              pointer;          // T 的 指针类型
    typedef T&              reference;        // T 的引用类型
    typedef const T*        const_pointer;    // T 的 const 指针类型
    typedef const T&        const_reference;  // T 的 const 引用类型
    typedef size_t          size_type;        // size_t 类型
    typedef ptrdiff_t       difference_type;  // 指针减法结果类型

    template <typename U> struct rebind {
      typedef MemoryPool<U> other;
    };

    /* Member functions */
    /* 构造函数 */
    MemoryPool() throw();
    MemoryPool(const MemoryPool& memoryPool) throw();
    template <class U> MemoryPool(const MemoryPool<U>& memoryPool) throw();

    /* 析构函数 */
    ~MemoryPool() throw();

    /* 元素取址 */
    pointer address(reference x) const throw();
    const_pointer address(const_reference x) const throw();

    // Can only allocate one object at a time. n and hint are ignored
    // 分配和收回一个元素的内存空间
    pointer allocate(size_type n = 1, const_pointer hint = 0);
    void deallocate(pointer p, size_type n = 1);

    // 可达到的最多元素数
    size_type max_size() const throw();

    // 基于内存池的元素构造和析构
    void construct(pointer p, const_reference val);
    void destroy(pointer p);

    // 自带申请内存和释放内存的构造和析构
    pointer newElement(const_reference val);
    void deleteElement(pointer p);

  private:
    // union 结构体,用于存放元素或 next 指针
    union Slot_ {
      value_type element;
      Slot_* next;
    };

    typedef char* data_pointer_;  // char* 指针，主要用于指向内存首地址
    typedef Slot_ slot_type_;     // Slot_ 值类型
    typedef Slot_* slot_pointer_; // Slot_* 指针类型

    slot_pointer_ currentBlock_;  // 内存块链表的头指针
    slot_pointer_ currentSlot_;   // 元素链表的头指针
    slot_pointer_ lastSlot_;      // 可存放元素的最后指针
    slot_pointer_ freeSlots_;     // 元素构造后释放掉的内存链表头指针

    size_type padPointer(data_pointer_ p, size_type align) const throw();  // 计算对齐所需空间
    void allocateBlock();  // 申请内存块放进内存池
   /*
    static_assert(BlockSize >= 2 * sizeof(slot_type_), "BlockSize too small.");
    */
};

#include "MemoryPool.tcc"

#endif // MEMORY_POOL_H
```

**实现文件**

```text
/*-
 * Copyright (c) 2013 Cosku Acay, http://www.coskuacay.com
 *
 * Permission is hereby granted, free of charge, to any person obtaining a
 * copy of this software and associated documentation files (the "Software"),
 * to deal in the Software without restriction, including without limitation
 * the rights to use, copy, modify, merge, publish, distribute, sublicense,
 * and/or sell copies of the Software, and to permit persons to whom the
 * Software is furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
 * FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
 * IN THE SOFTWARE.
 */

#ifndef MEMORY_BLOCK_TCC
#define MEMORY_BLOCK_TCC

// 计算对齐所需补的空间
template <typename T, size_t BlockSize>
inline typename MemoryPool<T, BlockSize>::size_type
MemoryPool<T, BlockSize>::padPointer(data_pointer_ p, size_type align)
const throw()
{
  size_t result = reinterpret_cast<size_t>(p);
  return ((align - result) % align);
}

/* 构造函数，所有成员初始化 */
template <typename T, size_t BlockSize>
MemoryPool<T, BlockSize>::MemoryPool()
throw()
{
  currentBlock_ = 0;
  currentSlot_ = 0;
  lastSlot_ = 0;
  freeSlots_ = 0;
}

/* 复制构造函数,调用 MemoryPool 初始化*/
template <typename T, size_t BlockSize>
MemoryPool<T, BlockSize>::MemoryPool(const MemoryPool& memoryPool)
throw()
{
  MemoryPool();
}

/* 复制构造函数,调用 MemoryPool 初始化*/
template <typename T, size_t BlockSize>
template<class U>
MemoryPool<T, BlockSize>::MemoryPool(const MemoryPool<U>& memoryPool)
throw()
{
  MemoryPool();
}

/* 析构函数，把内存池中所有 block delete 掉 */
template <typename T, size_t BlockSize>
MemoryPool<T, BlockSize>::~MemoryPool()
throw()
{
  slot_pointer_ curr = currentBlock_;
  while (curr != 0) {
    slot_pointer_ prev = curr->next;
    // 转化为 void 指针，是因为 void 类型不需要调用析构函数,只释放空间
    operator delete(reinterpret_cast<void*>(curr));
    curr = prev;
  }
}

/* 返回地址 */
template <typename T, size_t BlockSize>
inline typename MemoryPool<T, BlockSize>::pointer
MemoryPool<T, BlockSize>::address(reference x)
const throw()
{
  return &x;
}

/* 返回地址的 const 重载*/
template <typename T, size_t BlockSize>
inline typename MemoryPool<T, BlockSize>::const_pointer
MemoryPool<T, BlockSize>::address(const_reference x)
const throw()
{
  return &x;
}

// 申请一块空闲的 block 放进内存池
template <typename T, size_t BlockSize>
void
MemoryPool<T, BlockSize>::allocateBlock()
{
  // Allocate space for the new block and store a pointer to the previous one
  // operator new 申请对应大小内存，返回 void* 指针
  data_pointer_ newBlock = reinterpret_cast<data_pointer_>
                           (operator new(BlockSize));
  // 原来的 block 链头接到 newblock
  reinterpret_cast<slot_pointer_>(newBlock)->next = currentBlock_;
  // 新的 currentblock_
  currentBlock_ = reinterpret_cast<slot_pointer_>(newBlock);
  // Pad block body to staisfy the alignment requirements for elements
  data_pointer_ body = newBlock + sizeof(slot_pointer_);
  // 计算为了对齐应该空出多少位置
  size_type bodyPadding = padPointer(body, sizeof(slot_type_));
  // currentslot_ 为该 block 开始的地方加上 bodypadding 个 char* 空间
  currentSlot_ = reinterpret_cast<slot_pointer_>(body + bodyPadding);
  // 计算最后一个能放置 slot_type_ 的位置
  lastSlot_ = reinterpret_cast<slot_pointer_>
              (newBlock + BlockSize - sizeof(slot_type_) + 1);
}

// 返回指向分配新元素所需内存的指针
template <typename T, size_t BlockSize>
inline typename MemoryPool<T, BlockSize>::pointer
MemoryPool<T, BlockSize>::allocate(size_type, const_pointer)
{
  // 如果 freeSlots_ 非空，就在 freeSlots_ 中取内存
  if (freeSlots_ != 0) {
    pointer result = reinterpret_cast<pointer>(freeSlots_);
    // 更新 freeSlots_
    freeSlots_ = freeSlots_->next;
    return result;
  }
  else {
    if (currentSlot_ >= lastSlot_)
      // 之前申请的内存用完了，分配新的 block
      allocateBlock();
    // 从分配的 block 中划分出去
    return reinterpret_cast<pointer>(currentSlot_++);
  }
}

// 将元素内存归还给 free 内存链表
template <typename T, size_t BlockSize>
inline void
MemoryPool<T, BlockSize>::deallocate(pointer p, size_type)
{
  if (p != 0) {
    // 转换成 slot_pointer_ 指针，next 指向 freeSlots_ 链表
    reinterpret_cast<slot_pointer_>(p)->next = freeSlots_;
    // 新的 freeSlots_ 头为 p
    freeSlots_ = reinterpret_cast<slot_pointer_>(p);
  }
}

// 计算可达到的最大元素上限数
template <typename T, size_t BlockSize>
inline typename MemoryPool<T, BlockSize>::size_type
MemoryPool<T, BlockSize>::max_size()
const throw()
{
  size_type maxBlocks = -1 / BlockSize;
  return (BlockSize - sizeof(data_pointer_)) / sizeof(slot_type_) * maxBlocks;
}

// 在已分配内存上构造对象
template <typename T, size_t BlockSize>
inline void
MemoryPool<T, BlockSize>::construct(pointer p, const_reference val)
{
  // placement new 用法，在已有内存上构造对象，调用 T 的复制构造函数，
  new (p) value_type (val);
}

// 销毁对象
template <typename T, size_t BlockSize>
inline void
MemoryPool<T, BlockSize>::destroy(pointer p)
{
  // placement new 中需要手动调用元素 T 的析构函数
  p->~value_type();
}

// 创建新元素
template <typename T, size_t BlockSize>
inline typename MemoryPool<T, BlockSize>::pointer
MemoryPool<T, BlockSize>::newElement(const_reference val)
{
  // 申请内存
  pointer result = allocate();
  // 在内存上构造对象
  construct(result, val);
  return result;
}

// 删除元素
template <typename T, size_t BlockSize>
inline void
MemoryPool<T, BlockSize>::deleteElement(pointer p)
{
  if (p != 0) {
    // placement new 中需要手动调用元素 T 的析构函数
    p->~value_type();
    // 归还内存
    deallocate(p);
  }
}

#endif // MEMORY_BLOCK_TCC
```

原文地址：https://zhuanlan.zhihu.com/p/558263613

作者：linux