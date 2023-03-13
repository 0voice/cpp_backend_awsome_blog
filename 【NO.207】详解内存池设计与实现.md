# 【NO.207】详解内存池设计与实现

## 1.前言

作为C++程序员，想必对于内存操作这一块是比较熟悉和操作比较频繁的；

比如申请一个对象，使用new，申请一块内存使用malloc等等；

但是，往往会有一些困扰烦恼着大家，主要体现在两部分：

- 申请内存后忘记释放，造成内存泄漏
- 内存不能循环使用，造成大量内存碎片

这两个原因会影响我们程序长期平稳的运行，也有可能会导致程序的崩溃；

## 2.内存池

> 内存池是池化技术中的一种形式。通常我们在编写程序的时候回使用 new delete 这些关键字来向操作系统申请内存，而这样造成的后果就是每次申请内存和释放内存的时候，都需要和操作系统的系统调用打交道，从堆中分配所需的内存。如果这样的操作太过频繁，就会找成大量的内存碎片进而降低内存的分配性能，甚至出现内存分配失败的情况。
> 而内存池就是为了解决这个问题而产生的一种技术。从内存分配的概念上看，内存申请无非就是向内存分配方索要一个指针，当向操作系统申请内存时，操作系统需要进行复杂的内存管理调度之后，才能正确的分配出一个相应的指针。而这个分配的过程中，我们还面临着分配失败的风险。
> 所以，每一次进行内存分配，就会消耗一次分配内存的时间，设这个时间为 T，那么进行 n 次分配总共消耗的时间就是 nT；如果我们一开始就确定好我们可能需要多少内存，那么在最初的时候就分配好这样的一块内存区域，当我们需要内存的时候，直接从这块已经分配好的内存中使用即可，那么总共需要的分配时间仅仅只有 T。当 n 越大时，节约的时间就越多。

## 3.内存池设计

![img](https://pic3.zhimg.com/80/v2-589698ace8dbf7801cb79a2c21a41762_720w.webp)

内存池设计实现中主要分为以下几部分：

- 重载new
- 创建内存节点
- 创建内存池
- 管理内存池

下面，比较详细的来说说设计细节：

重载new就不说了，直接从内存节点开始；

> 内存池节点

内存池节点需要包含以下几点元素：

1. 所属池子（pMem），因为后续在内存池管理中可以直接调用申请内存和释放内存
2. 下一个节点（pNext），这里主要是使用链表的思路，将所有的内存块关联起来；
3. 节点是否被使用（bUsed），这里保证每次使用前，该节点是没有被使用的；
4. 是否属于内存池（bBelong），主要是一般内存池维护的空间都不是特别大，但是用户申请了特别大的内存时，就走正常的申请流程，释放时也就正常释放；

> 内存池设计

内存池设计就是上面的图片类似，主要包含以下几点元素：

1. 内存首地址（_pBuffer），也就是第一块内存，这样以后方面寻找后面的内存块；
2. 内存块头（_pHeader），也就是上面说的内存池节点；
3. 内存块大小（_nSize），也就是每个节点多大；
4. 节点数（_nBlock），及时有多少个节点；

这里面需要的注意的是，申请内存块的时候，需要加上节点头，但是申请完后返回给客户使用的需要去掉头；但是释放的时候，需要前移到头，不然就会出现异常；

**释放内存：**

释放内存的时候，将使用过的内存置为false，然后指向头部，将头部作为下一个节点，这样的话，节点每次回收就可以相应的被找到；

> 内存池管理

内存池创建后，会根据节点大小和个数创建相应的内存池；

内存池管理主要就是根据不同的需求创建不同的内存池，以达到管理的目的；

这里主要有一个概念：数组映射

**数组映射**就是不同的范围内，选择不同的内存池；

添一段代码：

```
 void InitArray(int nBegin,int nEnd, MemoryPool*pMemPool)
 {
  for (int i = nBegin; i <= nEnd; i++)
  {
   _Alloc[i] = pMemPool;
  }
 }
```

根据范围进行绑定；

## 4.内存池实现

ManagerPool.hpp

```
#ifndef _MEMORYPOOL_HPP_
#define _MEMORYPOOL_HPP_
#include <iostream>
#include <mutex>
////一个内存块的最大内存大小,可以扩展
#define MAX_MEMORY_SIZE 256
class MemoryPool;
//内存块
struct MemoryBlock
{
 MemoryBlock* pNext;//下一块内存块
 bool bUsed;//是否使用
 bool bBelong;//是否属于内存池
 MemoryPool* pMem;//属于哪个池子
};
class MemoryPool
{
public:
 MemoryPool(size_t nSize=128,size_t nBlock=10)
 {
  //相当于申请10块内存，每块内存是1024
  _nSize = nSize;
  _nBlock = nBlock;
  _pHeader = NULL;
  _pBuffer = NULL;
 }
 virtual ~MemoryPool()
 {
  if (_pBuffer != NULL)
  {
   free(_pBuffer);
  }
 }
 //申请内存
 void* AllocMemory(size_t nSize)
 {
  std::lock_guard<std::mutex> lock(_mutex);
  //如果首地址为空，说明没有申请空间
  if (_pBuffer == NULL)
  {
   InitMemory();
  }
  MemoryBlock* pRes = NULL;
  //如果内存池不够用时，需要重新申请内存
  if (_pHeader == NULL)
  {
   pRes = (MemoryBlock*)malloc(nSize+sizeof(MemoryBlock));
   pRes->bBelong = false;
   pRes->bUsed = false;
   pRes->pNext = NULL;
   pRes->pMem = NULL;
  }
  else
  {
   pRes = _pHeader;
   _pHeader = _pHeader->pNext;
   pRes->bUsed = true;
  }
  //返回只返回头后面的信息
  return ((char*)pRes + sizeof(MemoryBlock));
 }
 //释放内存
 void FreeMemory(void* p)
 {
  std::lock_guard<std::mutex> lock(_mutex);
  //和申请内存刚好相反，这里需要包含头，然后全部释放
  MemoryBlock* pBlock = ((MemoryBlock*)p - sizeof(MemoryBlock));
  if (pBlock->bBelong)
  {
   pBlock->bUsed = false;
   //循环链起来
   pBlock->pNext = _pHeader;
   pBlock = _pHeader;
  }
  else
  {
   //不属于内存池直接释放就可以
   free(pBlock);
  }
 }
 //初始化内存块
 void InitMemory()
 {
  if (_pBuffer)
   return;
  //计算每块的大小
  size_t PoolSize = _nSize + sizeof(MemoryBlock);
  //计算需要申请多少内存
  size_t BuffSize = PoolSize * _nBlock;
  _pBuffer = (char*)malloc(BuffSize);
  //初始化头
  _pHeader = (MemoryBlock*)_pBuffer;
  _pHeader->bUsed = false;
  _pHeader->bBelong = true;
  _pHeader->pMem = this;
  //初始化_nBlock块，并且用链表的形式连接
  //保存头指针
  MemoryBlock* tmp1 = _pHeader;
  for (size_t i = 1; i < _nBlock; i++)
  {
   MemoryBlock* tmp2 = (MemoryBlock*)(_pBuffer + i*PoolSize);
   tmp2->bUsed = false;
   tmp2->pNext = NULL;
   tmp2->bBelong = true;
   _pHeader->pMem = this;
   tmp1->pNext = tmp2;
   tmp1 = tmp2;
  }
 }
public:
 //内存首地址（第一块内存的地址）
 char* _pBuffer;
 //内存块头
 MemoryBlock* _pHeader;
 //内存块大小
 size_t _nSize;
 //多少块
 size_t _nBlock;
 std::mutex _mutex;
};
//可以使用模板传递参数
template<size_t nSize,size_t nBlock>
class MemoryPoolor:public MemoryPool
{
public:
 MemoryPoolor()
 {
  _nSize = nSize;
  _nBlock = nBlock;
 }
};
//需要重新对内存池就行管理
class ManagerPool
{
public:
 static ManagerPool& Instance()
 {
  static ManagerPool memPool;
  return memPool;
 }
 void* AllocMemory(size_t nSize)
 {
  if (nSize < MAX_MEMORY_SIZE)
  {
   return _Alloc[nSize]->AllocMemory(nSize);
  }
  else
  {
   MemoryBlock* pRes = (MemoryBlock*)malloc(nSize + sizeof(MemoryBlock));
   pRes->bBelong = false;
   pRes->bUsed = true;
   pRes->pMem = NULL;
   pRes->pNext = NULL;
   return ((char*)pRes + sizeof(MemoryBlock));
  }
 }
 //释放内存
 void FreeMemory(void* p)
 {
  MemoryBlock* pBlock = (MemoryBlock*)((char*)p - sizeof(MemoryBlock));
  //释放内存池
  if (pBlock->bBelong)
  {
   pBlock->pMem->FreeMemory(p);
  }
  else
  {
   free(pBlock);
  }
 }
private:
 ManagerPool()
 {
  InitArray(0,128, &_memory128);
  InitArray(129, 256, &_memory256);
 }
 ~ManagerPool()
 {
 }
 void InitArray(int nBegin,int nEnd, MemoryPool*pMemPool)
 {
  for (int i = nBegin; i <= nEnd; i++)
  {
   _Alloc[i] = pMemPool;
  }
 }
 //可以根据不同内存块进行分配
 MemoryPoolor<128, 1000> _memory128;
 MemoryPoolor<256, 1000> _memory256;
 //映射数组
 MemoryPool* _Alloc[MAX_MEMORY_SIZE + 1];
};
#endif
```

OperatorMem.hpp

```
#ifndef _OPERATEMEM_HPP_
#define _OPERATEMEM_HPP_
#include <iostream>
#include <stdlib.h>
#include "MemoryPool.hpp"
void* operator new(size_t nSize)
{
 return ManagerPool::Instance().AllocMemory(nSize);
}
void operator delete(void* p)
{
 return ManagerPool::Instance().FreeMemory(p);
}
void* operator new[](size_t nSize)
{
 return ManagerPool::Instance().AllocMemory(nSize);
}
void operator delete[](void* p)
{
 return ManagerPool::Instance().FreeMemory(p);
}
#endif
```

mian.cpp

```
#include "OperateMem.hpp"
using namespace std;
int main()
{
 char* p = new char[128];
 delete[] p;
 return 0;
}
```

原文链接：https://zhuanlan.zhihu.com/p/356612864

作者：Hu先生的Linux