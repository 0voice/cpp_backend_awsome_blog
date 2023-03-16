# 【NO.422】linux内核vmalloc原理与实现

在 Linux 系统中的每个进程都有独立 4GB 内存空间，而 Linux 把这 4GB 内存空间划分为用户内存空间（0 ~ 3GB）和内核内存空间（3GB ~ 4GB），而内核内存空间由划分为直接内存映射区和动态内存映射区（vmalloc区）。

直接内存映射区从 3GB 开始到 3GB+896MB 处结束，直接内存映射区的特点就是物理地址与虚拟地址的关系为：虚拟地址 = 物理地址 + 3GB。而动态内存映射区不能通过这种简单的关系关联，而是需要访问动态内存映射区时，由内核动态申请物理内存并且映射到动态内存映射区中。下图是动态内存映射区在内存空间的位置：

![img](https://pic1.zhimg.com/80/v2-33c4056be2df5d21a827bff76dab9290_720w.webp)

## 1.为什么需要vmalloc区

由于直接内存映射区（3GB ~ 3GB+896MB）是直接映射到物理地址（0 ~ 896MB）的，所以内核不能通过直接内存映射区使用到超过 896MB 之外的物理内存。这时候就需要提供一个机制能够让内核使用 896MB 之外的物理内存，所以 Linux 就实现了一个 vmalloc 机制。vmalloc 机制的目的是在内核内存空间提供一个内存区，能够让这个内存区映射到 896MB 之外的物理内存。如下图：

![img](https://pic4.zhimg.com/80/v2-c3bf096397e6e70d8b648d13d5f9b123_720w.webp)

那么什么时候使用 vmalloc 呢？一般来说，如果要申请大块的内存就可以用vmalloc。

## 2.vmalloc实现

可以通过 vmalloc() 函数向内核申请一块内存，其原型如下：

```text
void * vmalloc(unsigned long size);
```

参数 size 表示要申请的内存块大小。

我们看看看 vmalloc() 函数的实现，代码如下：

```text
static inline void * vmalloc(unsigned long size)
{
    return __vmalloc(size, GFP_KERNEL|__GFP_HIGHMEM, PAGE_KERNEL);
}
```

从上面代码可以看出，vmalloc() 函数直接调用了 __vmalloc() 函数，而 __vmalloc() 函数的实现如下：

```text
void * __vmalloc(unsigned long size, int gfp_mask, pgprot_t prot)
{
    void * addr;
    struct vm_struct *area;

    size = PAGE_ALIGN(size); // 内存对齐
    if (!size || (size >> PAGE_SHIFT) > num_physpages) {
        BUG();
        return NULL;
    }

    area = get_vm_area(size, VM_ALLOC); // 申请一个合法的虚拟地址
    if (!area)
        return NULL;

    addr = area->addr;
    // 映射物理内存地址
    if (vmalloc_area_pages(VMALLOC_VMADDR(addr), size, gfp_mask, prot)) {
        vfree(addr);
        return NULL;
    }

    return addr;
}
```

__vmalloc() 函数主要工作有两点：

- 调用 get_vm_area() 函数申请一个合法的虚拟内存地址。
- 调用 vmalloc_area_pages() 函数把虚拟内存地址映射到物理内存地址。

接下来，我们看看 get_vm_area() 函数的实现，代码如下：

```text
struct vm_struct * get_vm_area(unsigned long size, unsigned long flags)
{
    unsigned long addr;
    struct vm_struct **p, *tmp, *area;

    area = (struct vm_struct *) kmalloc(sizeof(*area), GFP_KERNEL);
    if (!area)
        return NULL;
    size += PAGE_SIZE;
    addr = VMALLOC_START;
    write_lock(&vmlist_lock);
    for (p = &vmlist; (tmp = *p) ; p = &tmp->next) {
        if ((size + addr) < addr)
            goto out;
        if (size + addr <= (unsigned long) tmp->addr)
            break;
        addr = tmp->size + (unsigned long) tmp->addr;
        if (addr > VMALLOC_END-size)
            goto out;
    }
    area->flags = flags;
    area->addr = (void *)addr;
    area->size = size;
    area->next = *p;
    *p = area;
    write_unlock(&vmlist_lock);
    return area;

out:
    write_unlock(&vmlist_lock);
    kfree(area);
    return NULL;
}
```

get_vm_area() 函数比较简单，首先申请一个类型为 vm_struct 的结构 area 用于保存申请到的虚拟内存地址。然后查找可用的虚拟内存地址，如果找到，就把虚拟内存到虚拟内存地址保存到 area 变量中。最后把 area 连接到 vmalloc 虚拟内存地址管理链表 vmlist 中。vmlist 链表最终结果如下图：



![img](https://pic2.zhimg.com/80/v2-174c58ee01a475d497ae8294c7c4d839_720w.webp)



申请到虚拟内存地址后，__vmalloc() 函数会调用 vmalloc_area_pages() 函数来对虚拟内存地址与物理内存地址进行映射。

我们知道，映射过程就是对进程的 页表 进行映射。但每个进程都有一个独立 页表（内核线程除外），并且我们知道内核空间是所有进程共享的，那么就有个问题：如果只映射当前进程 页表 的内核空间，那么怎么同步到其他进程的内核空间呢？

为了解决内核空间同步问题，Linux 并不是直接对当前进程的内核空间映射的，而是对 init 进程的内核空间（init_mm）进行映射，我们来看看 vmalloc_area_pages() 函数的实现：

```text
inline int vmalloc_area_pages (unsigned long address, unsigned long size,
                               int gfp_mask, pgprot_t prot)
{
    pgd_t * dir;
    unsigned long end = address + size;
    int ret;

    dir = pgd_offset_k(address);         // 获取 address 地址在 init 进程对应的页目录项
    spin_lock(&init_mm.page_table_lock); // 对 init_mm 上锁
    do {
        pmd_t *pmd;

        pmd = pmd_alloc(&init_mm, dir, address);
        ret = -ENOMEM;
        if (!pmd)
            break;

        ret = -ENOMEM;
        if (alloc_area_pmd(pmd, address, end - address, gfp_mask, prot)) // 对页目录项进行映射
            break;

        address = (address + PGDIR_SIZE) & PGDIR_MASK;
        dir++;

        ret = 0;
    } while (address && (address < end));
    spin_unlock(&init_mm.page_table_lock);
    return ret;
}
```

从上面代码可以看出，vmalloc_area_pages() 函数映射的主体是 init 进程的内存空间。因为映射的 init 进程的内存空间，所以当前进程访问 vmalloc() 函数申请的内存时，由于没有对虚拟内存进行映射，所以会发生 缺页异常 而触发内核调用 do_page_fault() 函数来修复。我们看看 do_page_fault() 函数对 vmalloc() 申请的内存异常处理：

```text
void do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
    ...
    __asm__("movl %%cr2,%0":"=r" (address));  // 获取出错的虚拟地址
    ...

    if (address >= TASK_SIZE && !(error_code & 5))
        goto vmalloc_fault;

    ...

vmalloc_fault:
    {
        int offset = __pgd_offset(address);
        pgd_t *pgd, *pgd_k;
        pmd_t *pmd, *pmd_k;
        pte_t *pte_k;

        asm("movl %%cr3,%0":"=r" (pgd));
        pgd = offset + (pgd_t *)__va(pgd);
        pgd_k = init_mm.pgd + offset;

        if (!pgd_present(*pgd_k))
            goto no_context;
        set_pgd(pgd, *pgd_k);

        pmd = pmd_offset(pgd, address);
        pmd_k = pmd_offset(pgd_k, address);
        if (!pmd_present(*pmd_k))
            goto no_context;
        set_pmd(pmd, *pmd_k);

        pte_k = pte_offset(pmd_k, address);
        if (!pte_present(*pte_k))
            goto no_context;
        return;
    }
}
```

上面的代码就是当进程访问 vmalloc() 函数申请到的内存时，发生 缺页异常 而进行的异常修复，主要的修复过程就是把 init 进程的 页表项 复制到当前进程的 页表项 中，这样就可以实现所有进程的内核内存地址空间同步。

原文地址：https://zhuanlan.zhihu.com/p/346892153

作者：linux