# 【NO.431】如何减少频繁分配内存（malloc或new）造成的内存碎片

高性能之内存池（频繁使用malloc和new会降低性能）

内存池(Memory Pool)是一种内存分配方式。通常我们习惯直接使用new、malloc等API申请分配内存，这样做的缺点在于：由于所申请内存块的大小不定，当频繁使用时会造成大量的内存碎片并进而降低性能。内存池则是在真正使用内存之前，先申请分配一定数量的、大小相等(一般情况下)的内存块留作备用。当有新的内存需求时，就从内存池中分出一部分内存块，若内存块不够再继续申请新的内存。这样做的一个显著优点是尽量避免了内存碎片，使得内存分配效率得到提升。
（1）针对特殊情况，例如需要频繁分配释放固定大小的内存对象时，不需要复杂的分配算法和多线程保护。也不需要维护内存空闲表的额外开销，从而获得较高的性能。

（2）由于开辟一定数量的连续内存空间作为内存池块，因而一定程度上提高了程序局部性，提升了程序性能。

（3）比较容易控制页边界对齐和内存字节对齐，没有内存碎片的问题。

（4）当需要分配管理的内存在100M一下的时候，采用内存池会节省大量的时间，否则会耗费更多的时间。

（5）内存池可以防止更多的内存碎片的产生

（6）更方便于管理内存

利用C/C++开发大型应用程序中，内存的管理与分配是一个需要认真考虑的部分。

本文描述了内存池设计原理并给出内存池的实现代码。

内存池设计过程中需要考虑好内存的分配与释放问题，其实也就是空间和时间的矛盾。

有的内存池设计得很巧妙，内存分配与需求相当，但是会浪费过多的时间去查找分配与释放，这就得不偿失；

实际使用中，我们更多的是关心内存分配的速度，而不是内存的使用效率。基于此，本文按照如下思想设计实现内存池。

主要包含三个结构：StiaticMemory, MemoryChunk和MemoryBlock，三者之间的关系如下图所示：

![img](https://pic4.zhimg.com/80/v2-0a23bcbbb3b713ecc7fe9ae7886ec8ff_720w.webp)

1.内存的分配：

（1）如果分配大小超过1024，直接采用malloc分配，分配的时候多分配sizeof(size_t)字节，用于保存该块的大小；

（2）否则根据分配大小，查找到容纳该大小的最小size的MemoryChunk；

（3）查找MemoryChunk的链表指针pList，找到空闲的MemoryBlock返回；

（4）如果pList为NULL，临时创建MemoryBlock返回；

（5）MemoryBlock头部包含两个成员，pChunk指向的所属的MemoryChunk对象，size表明大小，其后才是给用户使用的空间；

2.内存的释放：

（1）根据释放的指针，查找器size头部，即减去sizeof(size_t)字节，判断该块的大小；

（2）如果大小超过1024，直接free;

（3）否则交给MemoryChunk处理，而块的头部保存了该指针，因此直接利用该指针就可以收回该内存。

注意的问题：

上述设计的内存池通过冗余的头部来实现内存块的分配与释放，减少了内存池的操作时间，速度上要优于原始的malloc和free操作，同时减少了内存碎片的增加。

但是该设计中没有去验证释放的块冗余头部的正确性，因此故意释放不属于内存池中的块或者修改头部信息都会导致内存池操作失败，当然这些可以由程序员来控制。

此外，内存池中分配出去的内存块如果不主动释放，内存池没有保留信息，不会自动释放，但是在退出的时候会验证验证是否完全释放，其实这个在系统测试时候就可以检测出来，我想这个缺陷也是可以弥补的，在此提出，希望使用者注意。

下面贴上源码。

MemoryChunk.h 文件，线程安全

```text
#ifndef MEMORY_CHUNK_H  
#define MEMORY_CHUNK_H  
#include <cstdio>  
#include <cassert>  
#include <cstdlib>  
  
#ifdef WIN32  
#include <windows.h>  
typedef CRITICAL_SECTION MUTEXTYPE;  
#define INITMUTEX(hMutex) InitializeCriticalSection(&hMutex)  
#define DELMUTEX(hMutex) DeleteCriticalSection(&hMutex)  
#define LOCK(hMutex) EnterCriticalSection(&hMutex)  
#define UNLOCK(hMutex) LeaveCriticalSection(&hMutex)  
#else  
#include <pthread.h>  
typedef pthread_mutex_t MUTEXTYPE;  
#define INITMUTEX(hMutex) pthread_mutex_init(&hMutex,NULL)  
#define DELMUTEX(hMutex) pthread_mutex_destroy(&hMutex)  
#define LOCK(hMutex) pthread_mutex_lock(&hMutex)  
#define UNLOCK(hMutex) pthread_mutex_unlock(&hMutex)  
#endif  
  
class MemoryChunk;  
  
/** @struct MemoryBlock  
 *  
 */  
struct BlockHeader  
{  
    MemoryChunk* pChunk;  
    size_t len;  
};  
struct MemoryBlock;  
struct BlockData  
{  
    union{  
        MemoryBlock* pNext;  
        char pBuffer;  
    };  
};  
struct MemoryBlock  
{  
    BlockHeader header;  
    BlockData data;  
};  
  
/** @class MemoryChunk  
 *  
 */  
  
class MemoryChunk  
{  
public:  
    MemoryChunk(size_t size, int count)  
    {  
        INITMUTEX(hMutex);  
        this->pFreeList=NULL;  
        this->size=size;  
        this->count=0;  
        MemoryBlock* pBlock;  
        while(count--){  
            pBlock=CreateBlock();  
            if(!pBlock)break;  
            pBlock->data.pNext=pFreeList;  
            pFreeList=pBlock;  
        }  
    }  
    ~MemoryChunk()  
    {  
        int tempcount=0;  
        MemoryBlock* pBlock;  
        while(pBlock=pFreeList){  
            pFreeList=pBlock->data.pNext;  
            DeleteBlock(pBlock);  
            ++tempcount;  
        }  
        assert(tempcount==count);//!确保释放完全  
        DELMUTEX(hMutex);  
    }  
    void* malloc()  
    {  
        MemoryBlock* pBlock;  
        LOCK(hMutex);  
        if(pFreeList){  
            pBlock=pFreeList;  
            pFreeList=pBlock->data.pNext;  
        }  
        else{  
            if(!(pBlock=CreateBlock())){  
                UNLOCK(hMutex);  
                return NULL;  
            }  
        }  
        UNLOCK(hMutex);  
        return &pBlock->data.pBuffer;  
    }  
    static void free(void* pMem)  
    {  
        MemoryBlock* pBlock=(MemoryBlock*)((char*)pMem-sizeof(BlockHeader));  
        pBlock->header.pChunk->free(pBlock);  
    }  
    void free(MemoryBlock* pBlock)  
    {  
        LOCK(hMutex);  
        pBlock->data.pNext=pFreeList;  
        pFreeList=pBlock;  
        UNLOCK(hMutex);  
    }  
      
    MemoryChunk* Next(){return pNext;}  
  
protected:  
    MemoryBlock* CreateBlock()  
    {  
        MemoryBlock* pBlock=(MemoryBlock*)::malloc(sizeof(BlockHeader)+size);  
  
        if(pBlock){  
  
            pBlock->header.pChunk=this;  
            pBlock->header.len=size;  
              
            ++count;  
        }  
        return pBlock;  
    }  
    void DeleteBlock(MemoryBlock* pBlock)  
    {  
        ::free(pBlock);  
    }  
private:  
    MemoryBlock* pFreeList;  
    size_t size;//!Block大小  
    int count;//!Block数目  
    MemoryChunk* pNext;  
    MUTEXTYPE hMutex;  
};  
#endif  
```

StaticMemory.h文件，内存池对象

```text
#ifndef STATIC_MEMORY_H  
#define STATIC_MEMORY_H  
#include "MemoryChunk.h"  
/** @ StaticMemory.h  
 * 定义实现内存池  
 * 采用固定大小策略进行内存管理与分配  
 * 减少因大量小内存分配导致的内存碎片增加  
 */  
struct HeapHeader  
{  
    size_t size;  
};  
struct MemoryHeap  
{  
    HeapHeader header;  
    char pBuffer;  
};  
  
class StaticMemory  
{  
public:  
    typedef enum{MAX_SIZE=1024,MIN_SIZE=sizeof(MemoryChunk*)};  
    StaticMemory()  
    {  
        chunkcount=0;  
        for(size_t size=MIN_SIZE; size<=MAX_SIZE; size*=2)++chunkcount;  
        //pChunkList=(MemoryChunk**)malloc(sizeof(MemoryChunk*)*chunkcount);  
        pChunkList=new MemoryChunk*[chunkcount];  
        int index=0;  
        for(size_t size=MIN_SIZE; size<=MAX_SIZE; size*=2)  
        {  
            pChunkList[index++]=new MemoryChunk(size,1000);  
        }  
    }  
    ~StaticMemory()  
    {  
        for(int index=0; index<chunkcount; ++index)  
        {  
            delete pChunkList[index];  
        }  
        //free(pChunkList);  
        delete[] pChunkList;  
    }  
    void* Malloc(size_t size)  
    {  
        if(size>MAX_SIZE){  
            return malloc(size);  
        }  
        int index=0;  
        for(size_t tsize=MIN_SIZE; tsize<=MAX_SIZE; tsize*=2){  
            if(tsize>=size)break;  
            ++index;  
        }  
        return pChunkList[index]->malloc();  
    }  
    void Free(void* pMem)  
    {  
        if(!free(pMem))MemoryChunk::free(pMem);  
    }  
protected:  
    void* malloc(size_t size)  
    {  
        MemoryHeap* pHeap=(MemoryHeap*)::malloc(sizeof(HeapHeader)+size);  
        if(pHeap){  
            pHeap->header.size=size;  
            return &pHeap->pBuffer;  
        }  
        return NULL;  
    }  
    bool free(void* pMem)  
    {  
        MemoryHeap* pHeap=(MemoryHeap*)((char*)pMem-sizeof(HeapHeader));  
        if(pHeap->header.size>MAX_SIZE){  
            ::free(pHeap);  
            return true;  
        }  
        return false;  
    }  
private:  
    MemoryChunk** pChunkList;  
    int chunkcount;  
};  
#endif  
```

ObejctManager.h文件，用于实现对象的创建与管理，比较简易。

```text
#ifndef OBJECT_MANAGER_H  
#define OBJECT_MANAGER_H  
#include "StaticMemory.h"  
/** @class ObjectManager  
 * 实现利用内存池创建对象  
 * 要求对象具有缺省构造函数  
 */  
template<typename T>  
class ObjectManager  
{  
public:  
    typedef T ObjectType;  
  
    static ObjectType* Create(StaticMemory* pool)  
    {  
        void* pobject=pool->Malloc(sizeof(T));  
        new(pobject) ObjectType();  
        return static_cast<ObjectType*>(pobject);  
    }  
    static void Delete(StaticMemory* pool, ObjectType* pobject)  
    {  
        pobject->~ObjectType();  
        pool->Free(pobject);  
    }  
};  
#endif  
```

测试结果：

分单线程和多线程进行测试，重复的内存分配与释放在实际使用中是不太可能的，为了模拟实际使用，通过随机数来确定分配内存大小，同时也通过随机数来确定分配与释放操作。在测试过程中限制最大分配大小为1024，目的是为了测试小内存块的分配情况对比。

内存池单线程测试结果

![img](https://pic3.zhimg.com/80/v2-fe9bf6eadbdbe47ddb72a26394c9d156_720w.webp)

内存池多线程测试结果

![img](https://pic4.zhimg.com/80/v2-f99b1559b4bb1efd3e259394699df237_720w.webp)

进行多线程测试主要是测试多线程运行下，加锁给内存分配带来的影响，因此为了排除CPU的影响，测试采用的机器为16盒，16G内存的Linux服务器。

具体配置如下：

Intel(R) Xeon(R) CPU E5630 @ 2.53GHz

stepping : 2
cpu MHz : 2527.084

cache size : 12288 KB

原文地址：https://zhuanlan.zhihu.com/p/375447849

作者：linux