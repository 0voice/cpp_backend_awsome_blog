# 【NO.45】Linux性能优化—内存实战篇

## **1.Linux内存工作原理**

### 1.1内存映射

　　Linux内核给每个进程都提供了一个独立的虚拟空间，并且这个地址空间是连续的。这样，进程就可以很方便地访问内存，更确切地说是访问虚拟内存。

　　虚拟地址空间的内部又被分为**内核空间**和**用户空间**两部分，不同字长（也就是单个CPU指令可以处理数据的最大长度）的处理器，地址空间的范围也不同。比如常见的32位和64位系统

![img](https://pic4.zhimg.com/80/v2-4f0f7b568e2aca27835dc71d4720ba23_720w.webp)

- 32位系统的内核空间占用1G，位于最高处，剩下的3G是用户空间
- 64位系统的内核空间和用户空间都是128T，分别占据整个内存空间的最高处和最低处，剩下的中间部分是未定义的。

　　进程在用户态时，只能访问用户空间内存；只有进入内核态后，才可以访问内核空间内存。虽然每个进程的地址空间都包含了内核空间，但这些内核空间其实关联的都是相同的物理内存。这样，进程切换到内核态后，就可以很方便地访问内核空间内存。既然每个进程都有一个这么大的地址空间，那么**所有的虚拟内存加起来，自然要比实际的物理内存大很多**。所以，并不是所有的虚拟内存都会分配物理内存，只有那些实际使用的虚拟内存才分配物理内存，并且分配后的物理内存，是通过**内存映射**来管理的。

　　内存映射，其实就是将虚拟内存地址映射到物理内存地址。为了完成内存映射，内核为每个进程都维护了一张页表，记录虚拟地址与物理地址的映射关系：

![img](https://pic2.zhimg.com/80/v2-e15fddafc54630799a1aaff3e4449ae1_720w.webp)

　　页表实际上存储在CPU的内存管理单元MMU中，正常情况下，处理器就可以直接通过硬件找出访问的内存。而当进程访问的内存地址在页表中查不到时，系统会产生一个缺页异常，进入内核空间分配物理内存、更新进程页表，最后在返回用户空间，恢复进程的运行。

　　TLB（Translation Lookaside Buffer，后备缓冲器）会影响CPU的内存访问性能：TLB其实就是MMU中页表的高速缓存。由于进程的虚拟地址空间是独立的，而TLB的访问速度又比MMU快很多，所以，通过减少上下文切换，减少TLB的刷新次数，就可以提高TLB缓存的使用率，进而提高CPU的内存访问性能。

　　MMU规定了一个内存映射的最小单位，也就是页，通常是4KB。每次内存映射，都需要关联4KB或者4KB整数倍的内存空间。由于页的大小只有4KB，导致的整个页表会变的非常大。例如：32位系统就需要100多万个页表项（4GB/4KB），才可以实现整个地址空间的映射。为了解决页表项过多的问题，Linux提供了两种机制：多级页表 和 大页（HugePage）

**多级页表**：把整个内存分成区块来管理，将原来的映射关系改成区块索引和区块内的偏移。由于虚拟内存空间通常只用了很少一部分，那么多级页表就只保存这些使用中的区块，这样就可以大大减少页表的选项。Linux用的正是四级页表来管理内存页，如图，虚拟地址被分为5个部分，前4个表项用于选择页，最后一个索引表示页内偏移。

![img](https://pic1.zhimg.com/80/v2-97039bffe2c96a9e03d297f4c9014870_720w.webp)

**大页**：顾名思义就是比普通也更大的内存块，常见的大小有2MB和1GB。大页通常用在使用大量内存的进程上，比如Oracle、DPDK等。

### **1.2虚拟内存空间分布**

　　粗略绘制32位系统的虚拟内存空间分布图

![img](https://pic2.zhimg.com/80/v2-24b9746ef1b977d712ed47148510f6e9_720w.webp)

　　从此图可以看出，用户空间内存从低到高分别是五种不同的内存段。

- **只读段**，包括代码和常量等
- **数据段**，包括全局变量等
- **堆**，包括动态分配的内存，从低地址开始向上增长
- **文件映射段**，包括动态库、共享空间，从高地址开始向下增长
- **栈**，包括局部变量和函数调用的上下文等。栈的大小固定，一般为8MB

　　在这五个内存段中，**堆和文件映射段的内存是动态分配的**。比如说，使用C标准库的malloc()或mmap()，就可以分别在堆和文件映射段动态分配内存。

### **1.3内存分配与回收**

**a>内存分配**

　　malloc()是C标准库提供的内存分配函数，对应到系统调用上，有两种实现方式，即**brk**()和**mmap**()。

- **对小块内存（小于128K）**，C标准库使用**brk**()来分配，也就是通过移动堆顶的位置来分配内存。这些**内存释放后并不会立即归还系统，而是被缓存起来，这样就可以重复使用**。

- - brk()方式的缓存，可以减少缺页异常的发生，提高内存访问效率。不过由于这些内存没有归还系统，在内存工作繁忙是，频繁的内存分配和释放会造成内存碎片



- **对大块内存（大于128K）**，则直接使用内存映射mmap()来分配，也就是在文件映射段找一块空闲内存分配出去。

- - mmap()方式分配的内存**，会在释放时直接归还系统**，所以每次mmap都会发生缺页异常。在内存工作繁忙时，频繁的内存分配会造成大量的缺页异常，使内核的管理负担增大。这也是malloc只对大块内存使用mmap的原因



　　当这两种调用发生后，其实并没有真正分配内存。这些**内存都只在首次访问时才分配**，也就是**通过缺页异常进入内核**中，再**由内核来分配内存**。Linux使用伙伴系统来管理内存分配，与MMU的页管理一样，伙伴系统也是以页为单位来管理内存的，并通过相邻页的合并，较少内存碎片化（比如brk方式造成的内存碎片）。

**b>内存回收**

　　对于内存来说，如果只分配而不释放，就会造成内存泄露，甚至会耗尽系统内存。所以，在应用程序用完内存后，需要调用 free() 或 unmap() ，来释放不用的内存。在发现内存紧张时，系统就会通过一系列机制来回收内存：

- **回收缓存**，比如使用 **LRU**(Least Recently Used)算法，回收最近最少使用的内存页面
- **回收不常访问的内存**，把不常用的内存**通过交换分区直接写到磁盘**中
- **杀死进程**，内存紧张时系统会通过 OOM（Out Of Memory），直接杀掉占用大量内存的进程

　　其中，第二种方式回收不常访问的内存时，会用到**交换分区（Swap）**。Swap其实就是吧一块磁盘空间当成内存来用。它可以吧进程暂时不用的数据存储到磁盘中（这个过程称为换出），当进程访问这些内存时，在从磁盘读取这些数据到内存（这个过程称为换入）。所以Swap把系统的可用内存变大了。不过通常只在内存不足时，才会发生Swap交换，并且由于**磁盘读写的速度远比内存慢，Swap会导致严重的内存性能问题**。

　　第三种方式提到的OOM，其实时内核的一种保护机制。它监控进程的内存使用情况，并且使用 oom_score 为每个进程的内存使用情况进行评分：

- 一个进程消耗的内存越大，oom_score就越大
- 一个进程运行占用的CPU越多，oom_score就越小

　　进程的o**om_score越大，代表消耗的内存越多，也就越容易被OOM杀死**，从而可以更好保护系统。结合实际需求，可以通过 /proc 文件系统，手动设置进程的 oom_adj，从而调整oom_score。oom_adj的范围是[-17,15]，数值越大表示进程越容易被OOM杀死；数值越小表示进程越不容易被OOM杀死，其中 -17 表示禁止OOM。例如：手动调整sshd进程的oom_adj 为 -16，保障sshd进程不容易被OOM杀死

echo -16 > /proc/$(pidof sshd)/oom_adj

### **1.4查看内存使用情况**

**a>free**

![img](https://pic4.zhimg.com/80/v2-4fc4b42f081ecb9818626c30b13f15ab_720w.webp)

- **total**：总内存大小
- **used**：已使用内存大小，包含了共享内存
- **free**：未使用内存大小
- **shared**：共享内存大小
- **buff/cache**：缓存和缓冲区大小
- **available**：新进程可用内存大小。不仅包含未使用内存，还包括了可回收的内存，一般比未使用内存更大（但并不是所有缓存都可以回收，因为有的缓存可能正在使用中）

**b>top**

**![img](https://pic3.zhimg.com/80/v2-dcc74390f455c7a33064231fc1cd8c72_720w.webp)**

- **VIRT**：进程的虚拟内存大小，只要是进程申请过的内存，即便还没有真正分配物理内存，也会计算在内。
- **RES**：常驻内存的大小，也就是进**程实际使用的物理内存大小**，但不包括Swap和共享内存。
- **SHR**：共享内存的大小，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等。
- **%MEM**：进程使用物理内存占系统总内存的百分比

## **2.Buffer/Cache**

### **2.1定义**

　　使用man free查看

```text
buffers     Memory used by kernel buffers (Buffers in /proc/meminfo)

cache       Memory used by the page cache and slabs (Cached and SReclaimable in /proc/meminfo)

buff/cache  Sum of buffers and cache
```

- **Buffers** 是内核缓冲区用到的内存，对应的是 /proc/meminfo 中的Buffers值
- **Cache** 是内核页缓存和 Slab 用到的内存，对应的是 /proc/meminfo 中的Cached 与 SReclaimable之和

　　使用man proc 查看

```text
Buffers %lu
     Relatively temporary storage for raw disk blocks that shouldn't get tremendously large (20MB or so).

Cached %lu
     In-memory cache for files read from the disk (the page cache).  Doesn't include SwapCached.

SReclaimable %lu (since Linux 2.6.19)
     Part of Slab, that might be reclaimed, such as caches.

SUnreclaim %lu (since Linux 2.6.19)
     Part of Slab, that cannot be reclaimed on memory pressure.
```

- Buffers 是对原始磁盘块的临时存储，也就是用来**缓存磁盘的数据**，通常不会特别大（20MB左右）。这样，内核就可以把分散的写集中起来，统一优化磁盘的写入，比如可以把多次小的写合并成单次大的写等。
- Cached 是从磁盘读取文件的页缓存，也就是用来**缓存从文件读取的数据**。这样，下次访问这些文件数据时，就可以直接从内存中快速获取，而不需要再次访问缓慢的磁盘。
- SReclaimable 是Slab 的一部分。Slab 包括两个部分，其中的可回收部分用 SReclaimable 记录；不可回收部分用 SUnreclaim 记录

### **2.2案例**

清理系统缓存

\#清理文件页、目录项、Inodes 等各种缓存 echo 3 > /proc/sys/vm/drop_caches

**a>磁盘和文件写案例**

终端1：首先输出vmstat

![img](https://pic3.zhimg.com/80/v2-2f7783eccefce31bc80e4f17a8a3a01e_720w.webp)

- buff 和 cache就是前面说的Buffers 和 Cache，单位是KB
- bi 和 bo 则分别表示块设备读取和写入的大小，单位为 块/秒。因为Linux中块的大小是1KB，所以等价于KB/s

终端2：执行dd命令，通过读取随机设备，生成一个500MB大小的文件

dd if=/dev/urandom of=/tmp/file bs=1M count=500

终端1：继续观察vmstat中的Buffer 和 Cache。

![img](https://pic3.zhimg.com/80/v2-383edcbcc44b067cfe198092288d3eae_720w.webp)

　　发现在dd命令运行时，Cache 在不停地增长，而Buffer 基本保持不变：

- 在cache刚开始增长时，块设备I/O很少，bi值 只出现了一次 488KB/s，bo 则只有一次4KB，过一段时间后，才会出现大量的块设备写，bo甚至高达12880 KB/s
- 当dd命令结束后，Cache不在增长，但是块设备写还会持续一段时间，并且多次I/O写的结果加起来，才是dd要写的500M数据

终端2：清理缓存后，向磁盘/dev/sdb1 写入2GB的随机数据

```text
#清理文件页、目录项、Inodes 等各种缓存
echo 3 > /proc/sys/vm/drop_caches
#运行dd 命令向磁盘分区 /dev/sdb1 写入2G数据
dd if=/dev/urandom of=/dev/sdb1 bs=1M count=2048
```

终端1：观察内存和I/O变化

![img](https://pic1.zhimg.com/80/v2-2fd316f62bde98d3530d809a87d85e60_720w.webp)

　　此时可以看出，虽然都是写数据，但是写磁盘和写文件的现象不太一样。写磁盘时（也就是bo大于0时），Buffer和Cache都在增长，但是显然Buffer增长的快很多。这说明，写磁盘用到了大量的Buffer。

　　关于Cache，在写**文件时会用到Cache缓存数据，而写磁盘则会用到Buffer缓存数据**。所以，**Cache是文件读的缓存，实际Cache也会缓存写文件时的数据**。

**b>磁盘和文件读案例**

终端2：从文件/tmp/file中读取数据写入空设备

```text
#清理文件页、目录项、Inodes 等各种缓存
echo 3 > /proc/sys/vm/drop_caches
#运行dd 命令读取文件数据
dd if=/tmp/file of=/dev/null
```

终端1：vmstat观察内存和I/O变化情况

![img](https://pic1.zhimg.com/80/v2-8223dac466338ed087d857c8c635d99c_720w.webp)

　　观察vmstat输出，发现**读取文件时**（bi大于）），Buffer保持不变，而**Cache在不停增长**。

终端2：清理缓存，从磁盘分区 /dev/sda1中读取数据，写入空设备

```text
#清理文件页、目录项、Inodes 等各种缓存
echo 3 > /proc/sys/vm/drop_caches
#运行dd 命令读取文件数据
dd if=/dev/sda1 of=/dev/null bs=1M count=1024
```

终端1：vmstat观察内存和I/O变化情况

![img](https://pic2.zhimg.com/80/v2-c077749e79ee6dc0869e0489fc47b17d_720w.webp)

　　发现在读磁盘时（bi大于0），Buffer和Cache都在增长，但显然Buffer增长的快很多**。说明读磁盘时，数据缓存到了Buffer中**。

c>总结

- Buffer 既可以用作写入磁盘数据的缓存，也可以用作从磁盘读取数据的缓存
- Cache 既可以用作从文件读取数据的也缓存，也可以用作写文件的页缓存

**Buffer是对磁盘数据的缓存，而Cache是文件数据的缓存，它们既会用在读请求中，也会用在写请求中**。



原文链接：https://zhuanlan.zhihu.com/p/483177281

原文作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)