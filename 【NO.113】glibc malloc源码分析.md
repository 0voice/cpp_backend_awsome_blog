# 【NO.113】glibc malloc源码分析

## 1.**malloc**

本文梳理了一下malloc跟free的源码。malloc()函数在源代码中使用宏定义为public_mALLOc()。public_mALLOc()函数只是简单的封装_int_malloc()函数，_int_malloc()函数才是内存分配的核心实现。

### 1.1**public_mALLOc()**

```text
Void_t* public_mALLOc(size_t bytes) 
{   
    mstate ar_ptr;   
    Void_t *victim; 
    __malloc_ptr_t (*hook) (size_t, __const __malloc_ptr_t)    
    = force_reg (__malloc_hook);   
    if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));
```

首先检查是否存在__malloc_hook。如果存在，则调用hook函数。注意hook函数的传参为请求分配的内存大小。

```text
 arena_lookup(ar_ptr);   
    arena_lock(ar_ptr, bytes);   
    if(!ar_ptr)     
        return 0;   
    victim = _int_malloc(ar_ptr, bytes);
```

获取分配区指针，如果获取分配区失败，返回退出，否则，调用_int_malloc()函数分配内存。

```text
    if(!victim) 
    {     
        /* Maybe the failure is due to running out of mmapped areas. */
        if(ar_ptr != &main_arena)
        {       
            (void)mutex_unlock(&ar_ptr->mutex);       
            ar_ptr = &main_arena;       
            (void)mutex_lock(&ar_ptr->mutex);       
            victim = _int_malloc(ar_ptr, bytes);                      
            (void)mutex_unlock(&ar_ptr->mutex);
        }
```

如果_int_malloc()函数分配内存失败，并且使用的分配区不是主分配区，这种情况可能是mmap区域的内存被用光了。如果目前主分配区还可以从堆中分配内存，则需要再尝试从主分配区中分配内存。首先释放所使用分配区的锁，然后获得主分配区的锁，并调用_int_malloc()函数分配内存，最后释放主分配区的锁。

```text
        else 
        { 
#if USE_ARENAS       
            /* ... or sbrk() has failed and there is still a chance to mmap() */
            ar_ptr = arena_get2(ar_ptr->next ? ar_ptr : 0, bytes);                   
            (void)mutex_unlock(&main_arena.mutex);       
            if(ar_ptr) 
            {         
                victim = _int_malloc(ar_ptr, bytes);                     
                (void)mutex_unlock(&ar_ptr->mutex);       
            }
#endif
        }
    }
```

如果_int_malloc()函数分配内存失败，并且使用的分配区是主分配区，查看是否有非主分配区，如果有，调用arena_get2()获取分配区，然后对主分配区解锁，如果arena_get2()返回一个非分配区，尝试调用_int_malloc()函数从该非主分配区分配内存，最后释放该非主分配区的锁。

```text
    else 
        (void)mutex_unlock(&ar_ptr->mutex);
    assert(!victim || chunk_is_mmapped(mem2chunk(victim)) || ar_ptr ==      
    arena_for_chunk(mem2chunk(victim)));   
    return victim; 
} 
```

如果_int_malloc()函数分配内存成功，释放所使用的分配区的锁。返回分配的chunk。

### 1.2**_int_malloc**

_int_malloc函数是内存分配的核心，根据分配的内存块的大小，该函数中实现了了四种分配内存的路径。先给出_int_malloc()函数的定义及临时变量的定义：

```text
static Void_t* _int_malloc(mstate av, size_t bytes) 
{   
    INTERNAL_SIZE_T nb;               /* normalized request size */
    unsigned int    idx;              /* associated bin index */
    mbinptr         bin;              /* associated bin */

    mchunkptr       victim;           /* inspected/selected chunk */
    INTERNAL_SIZE_T size;             /* its size */
    int             victim_index;     /* its bin index */
 
    mchunkptr       remainder;        /* remainder from a split */
    unsigned long   remainder_size;   /* its size */

    unsigned int    block;            /* bit map traverser */
    unsigned int    bit;              /* bit map traverser */
    unsigned int    map;              /* current word of binmap */
 
    mchunkptr       fwd;              /* misc temp for linking */
    mchunkptr       bck;              /* misc temp for linking */
 
 
  const char *errstr = NULL; 
 
  /*
     Convert request size to internal form by adding SIZE_SZ bytes
     overhead plus possibly more to obtain necessary alignment and/or
     to obtain a size of at least MINSIZE, the smallest allocatable
     size. Also, checked_request 2size traps (returning 0) request sizes
     that are so large that they wrap around zero when padded and
     aligned.
   */
```

checked_request2size()函数将需要分配的内存大小bytes转换为需要分配的chunk大小nb，Ptmalloc内部分配都是以chunk为单位，根据chunk的大小，决定如何获得满足条件的chunk。

### 1.3**分配fast bin chunk**

```text
 /*
     If the size qualifies as a fastbin, first check corresponding bin.
     This code is safe to execute even if av is not yet initialized, so we
     can try it without checking, which saves some time on this fast path.
   */
    if ((unsigned long)(nb) <= (unsigned long)(get_max_fast ())) 
    {     
        idx = fastbin_index(nb);     
        mfastbinptr* fb = &fastbin (av, idx); 
#ifdef ATOMIC_FASTBINS     
        mchunkptr pp = *fb;     
        do       
        {         
            victim = pp;         
            if (victim == NULL)           
            break;       
        } while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim))            != victim); 
#else     
        victim = *fb; 
#endif     
        if (victim != 0) 
        {       
            if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))                 
            {           
                errstr = "malloc(): memory corruption (fast)";         
            errout:           
                malloc_printerr (check_action, errstr, chunk2mem (victim));                   
                return NULL;         
            } 
#ifndef ATOMIC_FASTBINS       
            *fb = victim->fd; 
#endif       
            check_remalloced_chunk(av, victim, nb);       
            void *p = chunk2mem(victim);       
            if (__builtin_expect (perturb_byte, 0))         
                alloc_perturb (p, bytes);       
            return p;     
        }       
    } 
```

如果所需的chunk大小小于等于fast bins中的最大chunk大小，首先尝试从fast bin中分配chunk。
1.根据所需的chunk大小获得该chunk所属的fast bin的index，根据该index获得所属fast bin的空闲chunk链表头指针。
如果没有开启ATOMIC_FASTBINS优化，则按以下步骤：
2.将头指针的下一个chunk作为空闲chunk链表的头部。
3.取出第一个chunk，并调用chunk2mem()函数返回用户所需的内存块。
如果开启了ATOMIC_FASTBINS优化，则步骤与上述类似，只是在删除fastbin头节点的时候使用了lock-free技术，加快了分配速度。

**check**

fastbin分配对size做了检查，如果分配chunk的size不等于分配时的idx，就会报错。使用chunksize()和fastbin_index函数计算chunk的size大小，所以我们无需管size的后三位(size_sz=8的情况下无需管后四位)，只需保证前几位与idx相同即可。

### 1.4**分配small bin chunk**

```text
 /*
     If a small request, check regular bin.  Since these "smallbins"
     hold one size each, no search ing within bins is necessary.
     (For a large request, we need to wait until unsorted chunks are
     processed to find best fit. But for small ones, fits are exact
     anyway, so we can check now, which is faster.)
   */
    if (in_smallbin_range(nb)) 
    {     
        idx = smallbin_index(nb);     
        bin = bin_at(av,idx); 
 
        if ( (victim = last(bin)) != bin) 
        {       
            if (victim == 0) /* initialization check */
                malloc_consolidate(av);       
            else 
            {         
                bck = victim->bk;         
                if (__builtin_expect (bck->fd != victim, 0))           
                {             
                    errstr = "malloc(): smallbin double linked list corrupted";                     
                    goto errout;           
                }         
                set_inuse_bit_at_offset(victim, nb);         
                bin->bk = bck;         
                bck->fd = bin; 
 
                if (av != &main_arena)           
                    victim->size |= NON_MAIN_ARENA; 
                check_malloced_chunk(av, victim, nb);         
                void *p = chunk2mem(victim);         
                if (__builtin_expect (perturb_byte, 0))           
                    alloc_perturb (p, bytes);         
                return p;       
            }     
        }   
    } 
```

如果所需的chunk大小属于small bin，则会执行如下步骤：
1.查找chunk对应small bins数组的index，根据index获得某个small bin的空闲chunk双向循环链表表头。
2.将最后一个chunk赋值给victim，如果victim与表头相同，表示该链表为空，不能从small bin的空闲chunk链表中分配，这里不做处理，等后面的步骤来处理。
3.victim与表头不同有两种情况。

- victim为0
  1.表示small bin还没有初始化为双向循环链表，调用malloc_consolidete()函数将fast bins中的chunk合并。
- victim不为0
  1.设置victim chunk的inuse标志，该标志处于vimctim chunk的下一个相邻chunk的size字段的第一个bit。
  2.做与unlink类似的操作将chunk从small bin中脱链。

4.判断当前分配区是否属于非主分配区，如果是，将victim chunk的size字段中的标志非主分配区的标志bit清零。
5.调用chunk2mem()函数获得chunk的实际可用的内存指针，将该内存指针返回给应用层。

**check**

申请的chunk需满足chunk->bk->fd = chunk

### 1.6**分配large bin chunk**

```text
 /*
      If this is a large request, consolidate fastbins before continuing.
      While it might look excessive to kill all fastbins before
      even seeing if there is space available, this avoids
      fragmentation problems normally associated with fastbins.
      Also, in practice, programs tend to have runs of either small or
      large requests, but less often mixtures, so consolidation is not
      invoked all that often in most programs. And the programs that
      it is called frequently in otherwise tend to fragment.
   */
 
    else 
    {     
        idx = largebin_index(nb);     
        if (have_fastchunks(av))       
        malloc_consolidate(av);   
    } 
```

所需chunk不属于small bins，那么就在large bins的范围内，则

1.根据chunk的大小获得对应large bin的index

2.判断当前分配区的fast bins中是否包含chunk，如果存在，调用malloc_consolidate()函数合并fast bins中的chunk，并将这些空闲chunk加入unsorted bin中。

下面的源代码实现从last remainder chunk，large bins和top chunk中分配所需的chunk，这里包含了多个多层循环，在这些循环中，主要工作是分配前两步都未分配成功的small bin chunk、large bin chunk和large chunk。最外层的循环用于重新尝试分配small bin chunk，因为如果在前一步分配smallbin chunk不成功，并没有调用malloc_consolidate()函数合并fast bins中的chunk，将空闲chunk加入unsorted bin中，如果第一尝试从last remainder chunk、top chunk中分配small bin chunk都失败以后，如果fast bins中存在空闲chunk，会调用malloc_consolidate()函数，那么在usorted bin中就可能存在合适的small bin chunk供分配，所以需要再次尝试。

```text
 /*
     Process recently freed or remaindered chunks, taking one only if
     it is exact fit, or, if this a small request, the chunk is remainder from
     the most recent non - exact fit.  Place other traversed chunks in
     bins.  Note that this step is the onl y place in any routine where
     chunks are placed in bins.
 
 
    The outer loop here is needed because we might not realize until
     near the end of malloc that we should have consolidated, so must
     do so and retry. This happens at most once, and only when we would
     otherwise need to expand memory to service a "small" request.
   */
    for(;;) 
    {     
        int iters = 0; 
        while ( (victim = unsorted_chunks(av)->bk) != unsorted_chunks(av)) 
        { 
```

反向遍历unsorted bin的双向链表，遍历结束的条件是循环链表中只剩一个头节点。

```text
            bck = victim->bk;       
            if (__builtin_expect (victim->size <= 2 * SIZE_SZ, 0)           
            || __builtin_expect (victim->size > av->system_mem, 0))             
                malloc_printerr (check_action, "malloc(): memory corruption",                         
                chunk2mem (victim));       
            size = chunksize(victim);
```

1.检查当前遍历的chunk是否合法，chunk的大小不能小于等于2*SIZE_SZ，也不能超过该分配区总的内存分配量。

2.获取chunk的大小并赋值给size。

```text
          if (in_smallbin_range(nb) &&           
                bck == unsorted_chunks(av) &&           
                victim == av->last_remainder &&           
                (unsigned long)(size) > (unsigned long)(nb + MINSIZE)) 
            { 
```

如果需要分配一个small bin chunk，并且unsorted bin中只有一个chunk，并且这个chunk为last remainder chunk，并且这个chunk的大小大于所需chunk的大小加上MINSIZE，在满足这些条件的情况下，可以使用这个chunk切分出需要的small bin chunk。这是唯一的从unsorted bin中分配small bin chunk的情况。

```text
                /* split and reattach remainder */
                remainder_size = size - nb;         
                remainder = chunk_at_offset(victim, nb);         
                unsorted_chunks(av)->bk = unsorted_chunks(av)->fd = remainder;             
                av->last_remainder = remainder;         
                remainder->bk = remainder->fd = unsorted_chunks(av);         
                if (!in_smallbin_range(remainder_size))           
                {             
                    remainder->fd_nextsize = NULL;             
                    remainder->bk_nextsize = NULL;           
                } 
```

1.从chunk中切分出所需大小的chunk。

2.计算切分后剩下chunk的大小，将剩下的chunk加入unsorted bin的链表中，并将剩下的chunk作为分配区的last remainder chunk。

3.如果剩下的chunk属于large bin chunk，将该chunk的fd_nextsize和bk_nextsize设置为NULL，因为这个chunk仅仅存在于unsorted bin中，并且unsorted bin中有且仅有这一个chunk。

```text
                set_head(victim, nb | PREV_INUSE |                  
                (av != &main_arena ? NON_MAIN_ARENA : 0));         
                set_head(remainder, remainder_size | PREV_INUSE);         
                set_foot(remainder, remainder_size); 
                check_malloced_chunk(av, victim, nb);         
                void *p = chunk2mem(victim);         
                if (__builtin_expect (perturb_byte, 0))           
                    alloc_perturb (p, bytes);         
                return p；
            }
```

设置分配出的chunk和last remainder chunk的相关信息。，调用chunk2mem()获得chunk中可用的内存指针，返回给应用层，退出。

```text
        /* remove from unsorted list */
            unsorted_chunks(av)->bk = bck;       
            bck->fd = unsorted_chunks(av); 
```

将双向循环链表中的最后一个chunk移除。

```text
if (size == nb) 
            {         
                set_inuse_bit_at_offset(victim, size);         
                if (av != &main_arena)           
                    victim->size |= NON_MAIN_ARENA;         
                check_malloced_chunk(av, victim, nb);         
                void *p = chunk2mem(victim);         
                if (__builtin_expect (perturb_byte, 0))           
                    alloc_perturb (p, bytes);         
                return p;       
             }
```

如果当前chunk与所需的chunk大小一致
1.设置当前chunk处于inuse状态
2.设置分配区标志位
3.调用chunk2mem()获得chunk中的可用内存指针，返回给应用层。

```text
/* place chunk in bin */
            if (in_smallbin_range(size)) 
            {         
                victim_index = smallbin_index(size);         
                bck = bin_at(av, victim_index);         
                fwd = bck->fd;
            }
```

如果当前chunk属于small bins，获得当前chunk所属small bin的index，并将该small bin的链表表头复制给bck，第一个chunk赋值给fwd，也就是当前的chunk会插入到bck和fwd之间，作为small bin比链表的第一个chunk。

```text
else 
             {         
                victim_index = largebin_index(size);         
                bck = bin_at(av, victim_index);         
                fwd = bck->fd;
```

如果当前chunk属于large bins，获得当前chunk所属large bin的index，并将该large bin的链表表头赋值给bck，第一个chunk赋值给fwd，也就是当前的chunk会插入到bck和fwd之间，作为large bin链表的第一个chunk。

```text
/* maintain large bins in sorted order */
                 if (fwd != bck) 
                 {           
                    /* Or with inuse bit to speed comparisons */
                     size |= PREV_INUSE;           
                    /* if smalle r than smallest, bypass loop below */
                     assert((bck->bk->size & NON_MAIN_ARENA) == 0);
```

如果fwd不等于bck，意味着large bin中有空闲chunk存在，由于large bin中的空闲chunk是按照大小排序的，需要将当前从unsorted bin中取出的chunk插入到large bin中合适的位置。将当前的chunk的size的inuse标志bit置位，相当于加1，便于加快chunk大小的比较，找到合适的地方插入当前chunk。

```text
if ((unsigned long)(size) < (unsigned long)(bck->bk->size)) 
                    {             
                        fwd = bck;             
                        bck = bck->bk;             
                        victim->fd_nextsize = fwd->fd;             
                        victim->bk_nextsize = fwd->fd->bk_nextsize;             
                        fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
                    }
```

如果当前的chunk比large bin的最后一个chunk的大小还小，那么当前chunk1就插入到large bin的链表的最后，作为最后一个chunk。可以看出large bin中的chunk是按照从大到小的顺序排序的，同时一个chunk存在于两个双向循环链表中，一个链表包含了large bin中所有的chunk，另一个链表为chunk size链表，该链表从每个相同大小的chunk中取除第一个chunk按照大小顺序链接在一起，便于一次跨域多个相同大小的chunk遍历下一个不同大小的chunk，这样可以加快在large bin链表中的遍历速度。

```text
else
                    {
                        assert((fwd->size & NON_MAIN_ARENA) == 0);            
                        while ((unsigned long) size < fwd->size)               
                        {                 
                            fwd = fwd->fd_nextsize;                 
                            assert((fwd->size & NON_MAIN_ARENA) == 0);               
                        }
```

正向遍历chunk size链表，直到找到第一个chunk大小小于等于当前chunk大小的chunk退出循环。

```text
if ((unsigned long) size == (unsigned long) fwd->size)               
                            /* Always insert in the second position.  */
                            fwd = fwd->fd;
```

如果从large bin链表中找到了与当前chunk大小相同的chunk，则统一大小的chunk已经存在，那么chunk size一定包含了fwd指向的chunk，为了不久改chunk size链表，当前chunk只能插入fwd之后。

```text
else               
                        {                 
                            victim->fd_nextsize = fwd; 
                            victim->bk_nextsize = fwd->bk_nextsize;                 
                            fwd->bk_nextsize = victim;                                 
                            victim->bk_nextsize->fd_nextsize = victim;
                        }
```

如果chunk size链表中还没有包含当前chunk大小的chunk，也就是说当前chunk的大小大于fwd的大小，则将当前chunk作为该chunk size的代表加入chunk size链表，chunk size链表也是按照由大到小的顺序排序。

```text
bck = fwd->bk;           
                    }         
                }
                else           
                    victim->fd_nextsize = victim->bk_nextsize = victim;
            }
```

如果large bin链表中没有chunk，直接将当前chunk加入chunk size链表。

```text
mark_bin(av, victim_index);       
            victim->bk = bck;       
            victim->fd = fwd;       
            fwd->bk = victim;       
            bck->fd = victim;
```

将当前chunk插入到对应的空闲的chunk链表中，并将large bin所对应binmap的相应bit置位。

```text
#define MAX_ITERS    10000       
            if (++iters >= MAX_ITERS)         
                break；
        }
```

如果unsorted bin中的chunk超过了10000个，最多遍历一万个就退出，避免长时间处理unsorted bin影响内存分配的效率。
当unsorted bin中的空闲chunk加入到相应的small bins和large bins后，将使用最佳匹配法分配large bin chunk。

```text
/*
           If a large request, scan through the chunks of current bin in
           sorted order to find smallest that fits.  Use the skip list for this.
         */
         if (!in_smallbin_range(nb)) 
        {       
            bin = bin_at(av, idx); 
 
           /* skip scan if empty or largest chunk is too small */
            if ((victim = first(bin)) != bin &&           
                (unsigned long)(victim->size) >= (unsigned long)(nb)) 
            {
```

如果所需分配的chunk为large bin chunk，查询对应的large bin链表，如果large bin链表为空，或者链表中最大的chunk也不能满足要求，则不能从large bin中分配。否则，遍历large bin链表，找到合适的chunk。

```text
victim = victim->bk_nextsize;         
                while (((unsigned long)(size = chunksize(victim)) <                 
                        (unsigned long)(nb)))           
                    victim = victim->bk_nextsize;
```

反向遍历chunk size链表，直到找到第一个大于等于所需chunk大小的chunk退出循环。

```text
/* Avoid removing the first entry for a size so that the skip
                list does not have to be rerouted.  */
                if (victim != last(bin) && victim->size == victim->fd->size)             
                    victim = victim->fd;
```

如果large bin链表中选取的chunk civtim不是链表中的最后一个chunk，并且与victim大小相同的chunk不止一个，那么意味着victim为chunk size链表中的节点，为了不调整chunk size链表，需要避免将chunk size链表中取出，所以取victim->fd节点对应的chunk作为候选chunk。由于large bin链表中的chunk也是按照大小排序，同一大小的chunk有多个时，这些chunk必定排在一起，所以victim->fd节点对应的chunk的大小必定与victim的大小一样。

```text
remainder_size = size - nb;         
                unlink(victim, bck, fwd);
```

计算将victim切分后剩余大小，并调用unlink()宏函数将victim从large bin链表中取出。

```text
if (remainder_size < MINSIZE)  
                {           
                    set_inuse_bit_at_offset(victim, size);           
                    if (av != &main_arena)             
                        victim->size |= NON_MAIN_ARENA; 
                }
```

如果将victim切分后剩余大小小于MINSIZE，则将整个victim分配给应用层，设置victim的inuse标志，inuse标志位于下一个相邻的chunk的size字段中。如果当前分配区不是主分配区，将victim的size字段中的非主分配区标志置位。

```text
/* Split */
                else 
                {           
                    remainder = chunk_at_offset(victim, nb);           
                    /* We cannot assume the unsorted list is empty and therefore
                    have to perform a complete insert here.  */
                    bck = unsorted_chunks(av);           
                    fwd = bck->fd;           
                    if (__builtin_expect (fwd->bk != bck, 0)) 
                    {               
                        errstr = "malloc(): corrupted unsorted chunks";               
                        goto errout;             
                    }           
                    remainder->bk = bck;           
                    remainder->fd = fwd;           
                    bck->fd = remainder;           
                    fwd->bk = remainder;           
                    if (!in_smallbin_range(remainder_size))             
                    {               
                        remainder->fd_nextsize = NULL;               
                        remainder->bk_nextsize = NULL;             
                    }
```

从victim中切分出所需的chunk，剩余部分作为一个新的chunk加入到unsorted bin中。 如果剩余部分chunk属于large bins，将剩余部分chunk的chunk size链表指针设置为NULL，因为unsorted bin中的chunk是不排序的，这两个指针无用，必须清零。

```text
set_head(victim, nb | PREV_INUSE |                    
                            (av != &main_arena ? NON_MAIN_ARENA : 0));                   
                   set_head(remainder, remainder_size | PREV_INUSE);                   
                   set_foot(remainder, remainder_size);
                }
```

设置victim和remainder的状态，由于remainder为空闲chunk，所以需要设置该chunk的foot。

```text
check_malloced_chunk(av, victim, nb);         
                void *p = chunk2mem(victim);         
                if (__builtin_expect (perturb_byte, 0))           
                    alloc_perturb (p, bytes);         
                return p;
            }
        }
```

从large bin中使用最佳匹配法找到了合适的chunk，调用chunk2mem()获得chunk中可用的内存指针，返回给应用层，退出。
如果通过上面的方式从最合适的small bin或large bin中都没有分配到需要的chunk，则查看比当前bin的index大的small bin或large bin是否有空闲chunk可利用来分配所需的chunk。源代码实现如下：

```text
++idx;     
        bin = bin_at(av,idx);     
        block = idx2block(idx);     
        map = av->binmap[block];     
        bit = idx2bit(idx);
```

获取下一个相邻bin的空闲chunk链表，并获取该bin对于binmap中的bit位的值。Binmap中的标识了相应的bin中是否有空闲 chunk 存在。Binmap按block管理，每个block为一个int，共32个bit，可以表示32个bin中是否有空闲chunk存在。使用binmap可以加快查找bin是否包含空闲chunk。这里只查询比所需chunk大的bin中是否有空闲chunk 可用。

```text
for (;;) 
        {       
            /* Skip rest of block if there are no more set bits in this block.  */
            if (bit > map || bit == 0) 
            {         
                do 
                    {           
                        if (++block >= BINMAPSIZE)  /* out of bins */
                        goto use_top;         
                    }while ( (map = av->binmap[block]) == 0); 
                bin = bin_at(av, (block << BINMAPSHIFT));         
                bit = 1;       
            }
```

Idx2bit()宏将idx指定的位设置为1，其它位清零，map表示一个block值，如果bit大于map，意味着map为0，该block所对应的所有bins中都没有空闲chunk，于是遍历binmap的下一个block，直到找到一个不为0的block或者遍历完所有的 block。 退出循环遍历后，设置 bin 指向 block 的第一个 bit 对应的 bin，并将 bit 置为 1，表示该 block 中 bit 1 对应的 bin，这个 bin 中如果有空闲 chunk，该 chunk 的大小一定满足要求。

```text
/* Advance to bin with set bit. There must be one. */
            while ((bit & map) == 0) 
            {         
                bin = next_bin(bin);         
                bit <<= 1;         
                assert(bit != 0);       
            }
            /* Inspect the bin. It is likely to be non - empty */
            victim = last(bin); 
```

在一个block遍历对应的bin，直到找到一个bit不为0退出遍历，则该bit对于的bin中有空闲chunk存在。将bin链表中的最后一个chunk赋值为victim。

```text
/*  If a false alarm (empty bin), clear the bit. */
            if (victim == bin) 
            {         
                av->binmap[block] = map &= ~bit; 
                /* Write through */
                bin = next_bin(bin);         
                bit <<= 1;
            }
```

如果victim与bin链表头指针相同，表示该bin中没有空闲chunk，binmap中的相应位设置不准确，将binmap的相应bit位清零，获取当前bin下一个bin，将bit移到下一个bit位，即乘以2。

```text
else 
            {         
                size = chunksize(victim);         
                /*  We know the first chunk in this bin is big enough to use. */
                assert((unsigned long)(size) >= (unsigned long)(nb));         
                remainder_size = size - nb;         
                /* unlink */
                unlink(victim, bck, fwd);
```

当前bin中的最后一个chunk满足要求，获取该chunk的大小，计算切分出所需chunk后剩余部分的大小，然后将victim从bin的链表中取出。

```text
/* Exhaust */
                if (remainder_size < MINSIZE) 
                {           
                    set_inuse_bit_at_offset(victim, size);           
                    if (av != &main_arena)             
                        victim->size |= NON_MAIN_ARENA
                }
```

如果剩余部分的大小小于MINSIZE，将整个chunk分配给应用层(代码在后面)，设置victim的状态为inuse，如果当前分配区为非分配区，设置victim的非主分配区标志位。

```text
/* Split */
                else 
                {           
                    remainder = chunk_at_offset(victim, nb); 
                    /* We cannot assume the unsorted list is empty and therefore
                    have to perform a complete insert here.  */
                    bck = unsorted_chunks(av);           
                    fwd = bck->fd;           
                    if (__builtin_expect (fwd->bk != bck, 0))             
                    {               
                        errstr = "malloc(): corrupted unsorted chunks 2";               
                        goto errout;             
                    }           
                    remainder->bk = bck;           
                    remainder->fd = fwd;
                    bck->fd = remainder;           
                    fwd->bk = remainder; 
                    /* advertise as last remainder */
                    if (in_smallbin_range(nb))             
                    av->last_remainder = remainder;           
                    if (!in_smallbin_range(remainder_size))             
                    {               
                        remainder->fd_nextsize = NULL;               
                        remainder->bk_nextsize = NULL;
                    }
```

从victim中切分出所需的chunk，剩余部分作为一个新的chunk加入到unsorted bin中。如果剩余部分chunk属于large bins，将剩余部分chunk的chunk size链表指针设置为NULL，因为unsorted bin中的chunk是不排序的，这两个指针无用，必须清零。

```text
set_head(victim, nb | PREV_INUSE |                    
                             (av != &main_arena ? NON_MAIN_ARENA : 0));                       
                     set_head(remainder, remainder_size | PREV_INUSE);                      
                     set_foot(remainder, remainder_size);
                }
```

设置victim和remainder的状态，由于remainder为空闲chunk，所以需要设置该chunk的foot。

```text
check_malloced_chunk(av, victim, nb);         
                void *p = chunk2mem(victim);         
                if (__builtin_expect (perturb_byte, 0))           
                    alloc_perturb (p, bytes);         
                return p;
            }
        }
```

调用chunk2mem()获得chunk中可用的内存指针，返回给应用层，退出。
如果从所有的bins中都没有获得所需的chunk，可能的情况为bins中没有空闲chunk，或者所需的chunk大小很大，下一步将尝试从top chunk中分配所需chunk。源代码实现如下：

```text
use_top:
        victim = av->top;     
        size = chunksize(victim);
```

将当前分配区的top chunk赋值给victim，并获得victim的大小。

```text
if ((unsigned long)(size) >= (unsigned long)(nb + MINSIZE)) 
        {       
            remainder_size = size - nb;       
            remainder = chunk_at_offset(victim, nb);       
            av->top = remainder;       
            set_head(victim, nb | PREV_INUSE |                
                    (av != &main_arena ? NON_MAIN_ARENA : 0));       
            set_head(remainder, remainder_size | PREV_INUSE); 
            check_malloced_chunk(av, victim, nb);       
            void *p = chunk2mem(victim);       
            if (__builtin_expect (perturb_byte, 0))         
                alloc_perturb (p, bytes);       
            return p;     
        }
```

由于top chunk切分出所需chunk后，还需要MINSIZE的空间来作为fencepost，所需必须满足top chunk的大小大于所需chunk的大小加上MINSIZE这个条件，才能从top chunk中分配所需chunk。从top chunk切分出所需chunk的处理过程跟前面的chunk切分类似，不同的是，原top chunk切分后的剩余部分将作为新的top chunk，原top chunk的fencepost仍然作为新的top chunk的fencepost，所以切分之后剩余的chunk不用set_foot。

```text
#ifdef ATOMIC_FASTBINS     
        /* When we are using atomic ops to free fast chunks we can get
        here for all block sizes.  */
        else if (have_fastchunks(av)) 
        {       
            malloc_consolidate(av);       
            /* restore original bin index */
            if (in_smallbin_range(nb))         
                idx = smallbin_index(nb);       
            else         
                idx = largebin_index(nb); 
        }
```

如果top chunk也不能满足要求，查看fast bins中是否有空闲chunk存在，由于开启了ATOMIC_FASTBINS优化情况下，free属于fast bins的chunk时不需要获得分配区的锁，所以在调用_int_malloc()函数时，有可能有其它线程已经向fast bins中加入了新的空闲chunk，也有可能是所需的chunk属于small bins，但通过前面的步骤都没有分配到所需的chunk，由于分配small bin chunk时在前面的步骤都不会调用malloc_consolidate()函数将fast bins中的 chunk合并加入到unsorted bin中。所在这里如果fast bin中有chunk存在，调用malloc_consolidate()函数，并重新设置当前bin的index。并转到最外层的循环，尝试重新分配small bin chunk或是large bin chunk。如果开启了ATOMIC_FASTBINS优化，有可能在由其它线程加入到fast bins中的chunk被合并后加入unsorted bin中，从unsorted bin中就可以分配出所需的large bin chunk了，所以对没有成功分配的large bin chunk也需要重试。

```text
#else
        else if (have_fastchunks(av)) 
        {       
            assert(in_smallbin_range(nb));       
            malloc_consolidate(av);       
            idx = smallbin_index(nb); /* restore original bin index */
        }
```

如果top chunk也不能满足要求，查看fast bins中是否有空闲chunk存在，如果fast bins中有空闲chunk存在，在没有开启ATOMIC_FASTBINS优化的情况下，只有一种可能，那就是所需的chunk属于small bins，但通过前面的步骤都没有分配到所需的small bin chunk，由于分配small bin chunk时在前面的步骤都不会调用malloc_consolidate()函数将fast bins中的空闲chunk合并加入到unsorted bin中。所在这里如果fast bins中有空闲chunk存在，调用 malloc_consolidate()函数，并重新设置当前bin的index。并转到最外层的循环，尝试重新分配small bin chunk。

```text
else 
       {       
            void *p = sYSMALLOc(nb, av);       
            if (p != NULL && __builtin_expect (perturb_byte, 0))         
            alloc_perturb (p, bytes);       
            return p; 
        }
    }
}
```

山穷水尽了，只能想系统申请内存了。sYSMALLOc()函数可能分配的chun 包括small bin chunk，large bin chunk 和large chunk。

**check**

1.fastbin头部的chunk的idx与fastbin的idx要一致。
2.unsorted bin中的chunk1大小不能小于等于2*SIZE_SZ，也不能超过该分配区总的内存分配量。
3.将chunk从unsorted bin取出放入small bin和large bin时用到了unlink()宏，注意绕过unlink()宏中的检测。
4.切出的remainder在重新放入unsorted bin时需要满足 unsorted_chunks(av)->fd->bk = unsorted_chunks(av)。

## 2.**总结**

malloc分配步骤大致如下：
1.检查有没有_malloc_hook，有则调用hook函数。
2.获得分配区的锁，调用函数_int_malloc()分配内存。
3.如果申请大小在fast bin范围内，则从fast bin分配chunk，成功则返回用户指针，否则进行下一步。(当对应的bin为空时，就会跳过第5步操作)
4.如果申请大小在small bin范围内，则从small bin中分配chunk，成功则返回用户指针，否则进行下一步。
5.调用malloc_consolidate()函数合并fast bin，并链接进unsorted bin中。
6.如果申请大小在small bin范围内，且此时unsorted bin只有一个chunk，并且这个chunk为last remainder chunk且大小够大，则从这个chunk中切分出需要的大小，成功则返回用户指针，否则进行下一步。
7.反向遍历unsorted bin，如果当前chunk与所需chunk大小一致，则分配，成功则返回用户指针，否则将当前chunk放入small bin或者large bin中合适的位置。
8.使用最佳匹配算法在large bin中找到合适的chunk进行分配，成功则返回用户指针，否则进行下一步。
9.到了这一步，说明没有大小正好合适的chunk，则看看比当前bin的index大的small bin或者large bin中有没有空闲chunk可用来分配。成功则返回用户指针，否则进行下一步。
10.尝试从top chunk中分配，成功则返回用户指针，否则进行下一步。
11.如果fast bin中还有chunk，调用malloc_consolidate()回到第6步(因为第3步对应bin为空时会跳过第五步，而fast bin合并之后有可能出现能够分配的small bin)。
12.到了这步还不行，则调用sYSMALLOc()函数向系统申请内存。

原文地址：https://zhuanlan.zhihu.com/p/564448120

作者：linux