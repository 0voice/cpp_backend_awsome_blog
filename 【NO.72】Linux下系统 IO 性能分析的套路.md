# 【NO.72】Linux下系统 I/O 性能分析的套路

## 1.如何快速分析定位 I/O 性能问题

### 1.1  文件系统 I/O性能指标

首先，想到是存储空间的使用情况，包括容量、使用量、以及剩余空间等。我们通常也称这些为磁盘空间的用量，但是这只是文件系统向外展示的空间使用，而非在磁盘空间的真实用量，因为文件系统的元数据也会占用磁盘空间。而且，如果你配置了RAID，从文件系统看到的使用量跟实际磁盘的占用空间，也会因为RAID级别不同而不一样。

除了数据本身的存储空间，还有一个容易忽略的是索引节点的使用情况，包括容量、使用量、剩余量。如果文件系统小文件数过多，可能会碰到索引节点容量已满的问题。

其次，缓存使用情况，包括页缓存、索引节点缓存、目录项缓存以及各个具体文件系统的缓存。通过使用内存，来临时缓存文件数据或者文件系统元数据，从而减少磁盘访问次数。·

最后，文件 I/O的性能指标，包括IOPS（r/s、w/s）、响应时间（延迟）、吞吐量（B/s）等。考察这类指标时，还要结合实际文件读写情况，文件大小、数量、I/O类型等，综合分析文件 I/O 的性能。

### 1.2  磁盘 I/O性能指标

磁盘 I/O的性能指标，主要由四个核心指标：使用率、IOPS、响应时间、吞吐量，还有一个前面提到过，缓冲区。

考察这些指标时，一定要注意综合 I/O的具体场景来分析，比如读写类型（顺序读写还是随机读写）、读写比例、读写大小、存储类型（有无RAID、RAID级别、本地存储还是网络存储）等。

不过考察这些指标时，有一个大忌，就是把不同场景的 I/O指标拿过来作对比。

![img](https://pic3.zhimg.com/80/v2-2ccf624bca12533c0e2df0d10c69807e_720w.webp)

### **1.3 性能工具**

一类：df、top、iostat、pidstat；

二类：/proc/meminfo、/proc/slabinfo、slabtop；

三类：strace、lsof、filetop、opensnoop

### **1.4 性能指标及性能工具之间的关系**

![img](https://pic2.zhimg.com/80/v2-13737e2711c3999167399f9bd876be09_720w.webp)

### 1.5 如何迅速分析 I/O性能瓶颈

简单来说，就是找关联。多种性能指标间，都是存在一定的关联性。想弄清楚指标之间的关联性，就要知晓各种指标的工作原理。出现性能问题，基本的分析思路是：

先用 iostat发现磁盘 I/O的性能瓶颈；

再借助 pidstat，定位导致性能瓶颈的进程；

随后分析进程 I/O的行为；

最后，结合应用程序的原理，分析这些 I/O的来源。

![img](https://pic3.zhimg.com/80/v2-44a1f3599d95380928a8088b6f59ceb2_720w.webp)

图中列出最常用的几个文件系统和磁盘 I/O的性能分析工具，及相应的分析流程。

## 2.磁盘 I/O性能优化的几个思路

### 2.1 I/O基准测试

在优化之前，我们要清楚 I/O性能优化的目标是什么？也就是说，我们观察的这些 I/O指标（IOPS、吞吐量、响应时间等），要达到多少才合适？为了更客观合理地评估优化效果，首先应该对磁盘和文件系统进行基准测试，得到它们的极限性能。

fio（Flexible I/O Tester），它是常用的文件系统和磁盘 I/O的性能基准测试工具。它提供了大量的可定制化的选项，可以用来测试，裸盘或者文件系统在各种场景下的 I/O性能，包括不同块大小、不同 I/O引擎以及是否使用缓存等场景。

fio的选项非常多，这里介绍几个常用的：

```text
# 随机读
fio -name=randread -direct=1 -iodepth=64 -rw=randread -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb
# 随机写
fio -name=randwrite -direct=1 -iodepth=64 -rw=randwrite -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb
# 顺序读
fio -name=read -direct=1 -iodepth=64 -rw=read -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb
# 顺序写
fio -name=write -direct=1 -iodepth=64 -rw=write -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=1000 -group_reporting -filename=/dev/sdb
```

重点解释几个参数：

- direct，是否跳过系统缓存，iodepth1 是跳过。
- iodepth，使用异步 I/O（AIO）时，同时发出的请求上限。
- rw，I/O模式，顺序读 / 写、随机读 / 写。
- ioengine，I/O引擎，支持同步（sync）、异步（libaio）、内存映射（mmap）、网络等各种 I/O引擎。
- bs，I/O大小。 4k，默认值。
- filename，文件路径，可以是磁盘路径，也可以是文件路径。不过要注意，用磁盘路径测试写，会破坏这个磁盘的文件系统，所以测试前，要注意备份。

下面展示， fio测试顺序读的示例：

```text
read: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=64
fio-3.1
Starting 1 process
Jobs: 1 (f=1): [R(1)][100.0%][r=16.7MiB/s,w=0KiB/s][r=4280,w=0 IOPS][eta 00m:00s]
read: (groupid=0, jobs=1): err= 0: pid=17966: Sun Dec 30 08:31:48 2018
   read: IOPS=4257, BW=16.6MiB/s (17.4MB/s)(1024MiB/61568msec)
    slat (usec): min=2, max=2566, avg= 4.29, stdev=21.76
    clat (usec): min=228, max=407360, avg=15024.30, stdev=20524.39
     lat (usec): min=243, max=407363, avg=15029.12, stdev=20524.26
    clat percentiles (usec):
     |  1.00th=[   498],  5.00th=[  1020], 10.00th=[  1319], 20.00th=[  1713],
     | 30.00th=[  1991], 40.00th=[  2212], 50.00th=[  2540], 60.00th=[  2933],
     | 70.00th=[  5407], 80.00th=[ 44303], 90.00th=[ 45351], 95.00th=[ 45876],
     | 99.00th=[ 46924], 99.50th=[ 46924], 99.90th=[ 48497], 99.95th=[ 49021],
     | 99.99th=[404751]
   bw (  KiB/s): min= 8208, max=18832, per=99.85%, avg=17005.35, stdev=998.94, samples=123
   iops        : min= 2052, max= 4708, avg=4251.30, stdev=249.74, samples=123
  lat (usec)   : 250=0.01%, 500=1.03%, 750=1.69%, 1000=2.07%
  lat (msec)   : 2=25.64%, 4=37.58%, 10=2.08%, 20=0.02%, 50=29.86%
  lat (msec)   : 100=0.01%, 500=0.02%
  cpu          : usr=1.02%, sys=2.97%, ctx=33312, majf=0, minf=75
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued rwt: total=262144,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=64
 
Run status group 0 (all jobs):
   READ: bw=16.6MiB/s (17.4MB/s), 16.6MiB/s-16.6MiB/s (17.4MB/s-17.4MB/s), io=1024MiB (1074MB), run=61568-61568msec
 
Disk stats (read/write):
  sdb: ios=261897/0, merge=0/0, ticks=3912108/0, in_queue=3474336, util=90.09% 
```

这个示例中，重点关注几行，slat、clat、lat，以及 bw和 iops。前三者，都是指 I/O延迟，但是有不同之处：

slat，是指从 I/O提交到实际执行 I/O的时长；

clat，是指从 I/O提交到 I/O完成的时长；

lat，是指从 fio创建 I/O 到 I/O完成的时长。

这里需要注意的是，对同步 I/O来说，提交和完成是一个动作，slat就是 I/O完成的时间，clat是0；使用异步 I/O时，lat 约等于 slat + clat。

再来看bw，他表示吞吐量，上面的输出中，平均吞吐量是16MB（17005/1024）。

最后的IOPS，其实是每秒 I/O的次数，上面输出的平均 IOPS是 4250.

通常情况下，应用程序的IO 读写是并行的，每次的 I/O大小也不相同。所以上面的几个场景并不能精确模拟应用程序的 I/O模式。幸运的是，fio支持 I/O 的重放，需要先用 blktrace，记录磁盘设备的 I/O访问情况，再使用 fio，重放 blktrace的记录。

```text
# 使用blktrace跟踪磁盘I/O，注意指定应用程序正在操作的磁盘
$ blktrace /dev/sdb
# 查看blktrace记录的结果
# ls
sdb.blktrace.0  sdb.blktrace.1
# 将结果转化为二进制文件
$ blkparse sdb -d sdb.bin
# 使用fio重放日志
$ fio --name=replay --filename=/dev/sdb --direct=1 --read_iolog=sdb.bin
```



### **2.2 I/O性能优化思路**

**应用程序优化**

应用程序处于 I/O栈的最上端，可以通过系统调用，来调整 I/O模式（顺序还是随机、同步还是异步），同时也是数据的最终来源。下面总结了几个方面来优化应用程序性能：

第一，可以用追加写代替随机写，减少寻址开销，加快 I/O写的速度。

第二，借助缓存 I/O，充分利用系统缓存，降低实际 I/O的次数。

第三，在应用程序内部构建自己缓存，或者使用Redis这种的外部缓存系统。这样不仅可以在内部控制缓存的数据和生命周期，而且降低其他应用程序使用缓存对自身的影响。比如，C标准库，提供的fopen、fread等库函数，都会利用标准库缓存，减少磁盘的操作。而如果直接使用open、read等系统调用时，就只能利用操作系统的页缓存和缓冲区等。

第四，在需要频繁读写同一块磁盘空间时，可以使用 mmap 代替 read/write，减少内存的拷贝次数。

第五，在需要同步写的场景中，尽量将写请求合并，而不是让每个请求都同步写磁盘，即可用fsync() 代替 O_SYNC。

第六，在多个应用程序共享相同磁盘时，为了保证 I/O不被某个应用完全占用，推荐使用 cgroups 的 I/O子系统，来限制进程/进程组的 IOPS 以及吞吐量。

最后，在使用CFQ 调度器时，可以用 ionice来调整进程的 I/O调度优先级，特别是提高核心应用的 I/O优先级，他支持三个优先级类：Idle、Best-effort 和 Realtime。其中，后两者还支持 0-7的级别，数值越小，优先级越高。

**文件系统优化**

应用程序在访问普通文件时，是通过文件系统间接负责，文件在磁盘中的读写。所以跟文件系统相关的也有很多优化方式。

第一，可以根据实际负载场景的不同，选择合适的文件系统。比如，Ubuntu默认使用ext4，Centos默认使用 xfs。相比于ext4，xfs支持更大的磁盘分区和更大的文件数量。xfs支持大于 16TB的磁盘，但它的缺点在于无法收缩，而ext4可以。

第二，在选好文件系统后，可以优化文件系统得配置选项。包括文件系统的特性（如 ext_attr、dir_index）、日志模式（如 journal、ordered、writeback等）、挂载选项（如 noatime）等等。比如在使用 tune2fs这个工具，可以调整文件系统的特性，也常用来查看文件系统超级块的内容。而通过 /etc/fstab，或者mount，来调整文件系统的日志模式和挂载选项等。

第三，优化文件系统的缓存。比如，可以优化 pdflush的脏页刷新频率（设置dirty_expire_centisecs 和 dirty_writeback_centisecs）以及脏页限额（调整 dirty_background_ratio 和 dirty_ratio）。再如，还可以优化内核回收目录项缓存和索引节点缓存的倾向，及调整 vfs_cache_pressure（/proc/sys/vm/vfs_cache_pressure，默认值100）,数值越大，表示越容易回收。

最后，在不需要持久化时，可以用内存文件系统 tmpfs 以获得更好的 I/O性能。tmpfs直接把数据保存在内存中，而不是磁盘中。比如 /dev/shm，就是大多数Linux默认配置的一个内存文件系统，它的大小默认为系统总内存的一半。

**磁盘优化**

数据的持久化，最终要落到物理磁盘上，同时磁盘也是整个 I/O栈的最底层。从磁盘角度出发，也有很多优化方法：

第一，最简单的就是SSD代替 HDD。

第二，使用 RAID把多块磁盘组合成一个逻辑磁盘，构成冗余独立的磁盘阵列，即可以提高数据的可靠性，也可以提升数据的访问性能。

第三，针对磁盘和应用程序的 I/O模式的特征，可选择最适合的 I/O调度算法。

第四，可以针对应用程序的数据，进行磁盘级别的隔离。比如，可以为日志、数据库等 I/O压力比较重的应用，配置单独的磁盘。

第五，在顺序读比较多的场景中，可以增大磁盘的预读数据，可以通过两种方法，调整 /dev/sdb的预读大小。一种，调整内核选项，/sys/block/sdb/queue/read_ahead_kb，默认大小128KB。另一种，blockdev工具，比如，blockdev --setra 8192 /dev/sdb ，注意这里的单位是 512B，所以它的数值总是 read_ahead_kb的两倍。

第六，优化内核块设备 I/O的选项。比如，调整磁盘队列的长度，/sys/block/sdb/queue/nr_requests，适当增大队列长度，可以增大磁盘的吞吐量，当然也会增大 I/O延迟。

最后，磁盘本身的硬件错误，也会导致 I/O性能急剧下降。比如，查看 dmesg中是否有硬件 I/O故障的日志，还可以使用badblocks、smartctl等工具，检测磁盘的硬件问题，或用 e2fsck等来检测文件系统错误。如果发现问题，可使用fsck 等工具修复。

原文地址：https://zhuanlan.zhihu.com/p/572855548

作者：linux