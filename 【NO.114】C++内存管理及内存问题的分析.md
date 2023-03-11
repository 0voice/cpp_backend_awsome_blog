# 【NO.114】C++内存管理及内存问题的分析

写服务端的，内存是一个绕不过的问题，而用C++写的，这个问题就显得更严重。进程的内存持续上涨，有可能是正常的内存占用，也有可能是内存碎片，而C++写的，还有可能是内存泄漏，那就需要一些方法来检测到底是哪些问题引起的

## **1. 内存占用**

首先从top这个指令说起

```text
Tasks:  80 total,   1 running,  79 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.7 sy,  0.0 ni, 92.7 id,  6.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  2052544 total,  1453600 free,   162408 used,   436536 buff/cache
KiB Swap:   782332 total,   782332 free,        0 used.  1708652 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND      
  179 root      20   0       0      0      0 S  0.3  0.0   0:00.27 [jbd2/dm-0-+ 
  493 mongodb   20   0 1102144  78548  36688 S  0.3  3.8   0:26.07 /usr/bin/mo+ 
  636 mysql     20   0  653808  75932  15548 S  0.3  3.7   0:03.55 /usr/sbin/m+ 
```

与进程内存相关的两个指标：VIRT Virtual Memory，虚拟内存、RES Resident Memory，常驻内存，通常叫物理内存。虚拟内存，是指整个进程申请的内存，包括程序本身的占内存、new或者malloc分配的内存等等。物理内存，就是这个进程在主板上内存条那里占用了多少内存。那为什么会有虚拟内存这个东西，C++不是可以操作硬件么，为什么不直接使用物理内存？这得简单了解一下操作系统的内存管理。

现代的计算机都会同时运行N个程序，有N多个进程，这些进程都是独立在运行。如果直接使用物理内存，那就会产生一个问题，进程A申请了内存，进程B也要申请一块内存，但进程B并不知道进程A的存在，就没法保证进程B使用的内存进程A没在用。因此linux下使用内核来管理这些资源，所有进程都只是向内核申请，由内核管理物理内存。而一个进程，可能多次申请、释放内存，或者程序直接当掉没有释放内存，内核为了解决这些复杂的问题，用一个列表维护了进程分配的内存，这就叫虚拟内存，然后把虚拟内存映射到物理内存，这就完成了整个内存的管理。而且，内核对内存的映射做了优化，用到时才映射，如下面的图中，进程A的new2这块内存分配了以后，一直没使用，也就不会映射到物理内存。有很多程序，利用了这个特性。例如，在socket收发时，我们可以分配很大的一块内存（比如16M），避免频繁分配缓冲区，但实际这个socket可能收到的数据块最大只有16k，那内核是不会直接映射16M物理内存的，这样既方便了我们写程序，但又没浪费物理内存。

![img](https://pic2.zhimg.com/80/v2-d519e76e268d0ee4a73ed28f8fd25b7d_720w.webp)

下面写个程序来验证这个问题

```text
#include <cstring>
#include <iostream>

int main()
{
#define PAUSE(msg) std::cout << msg << std::endl; std::cin >> p

        char p;

        size_t size = 1024 * 1024 *100;
        char *l = new char[size];

        PAUSE("new");

        memset(l, 1, size / 2);
        PAUSE("using half large");

        memset(l, 1, size);
        PAUSE("using whole large");

        delete []l;
        PAUSE("del");

        return 0;
}
```

在每次暂停时，top的输出结果(RES 1588 54328 105600 3348)，说明memset的时候，内核才会映射物理内存。

```text
new
 进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND  
  25295 root     20   0  108280   1588   1436 S   0.0   0.0   0:00.00 ./a.out


using half large
  25295 root     20   0  108280  54328   3096 S   0.0   0.7   0:00.05 ./a.out


using whole large
  25295 root     20   0  108280 105600   3156 S   0.0   1.4   0:00.12 ./a.out


del
  25295 root     20   0    5876   3348   3156 S   0.0   0.0   0:00.13 ./a.out
```

所以，通过top查看进程内存时，如果发现VIRT占用很大，说明这个程序用new或者malloc等分配了很多内存，但如果RES不是很大，那就不要慌，可能这只是程序的一个缓存优化（当然也有可能是写这个程序的人用new分配内存时很不合理，分配的值远大于使用值），实际程序运行占用的物理内存并不大。但如果RES也很高，那可能就有点慌了。

## **2. 内存泄漏**

内存泄漏是导致进程内存持续上涨最常见的原因，而这是C++中常见但不好处理的问题，一个维护多年的大项目，代码不知道由多少个人写的，想找出哪个指针的内存没释放，谈何容易。解决这个问题没有什么通用快捷的办法，只能根据实际业务处理。

第一，从业务上，能不能重现内存泄漏。例如我们做游戏的，假如玩家不停地登录，就会导致内存不断上涨，那说明问题就在登录流程，把整个流程拆分，一个个屏蔽测试，最终找出问题。

第二，从部署上，能不能定位内存泄漏。例如，最近更新了一个版本，发现内存占用变得很高，那就可以确定，是这个版本的修改出了问题。一个版本的代码量终究是有限的，查找起来也比较容易。

第三，使用valgrind memcheck。如果能够复现内存泄漏，但无法定位是哪个逻辑，那可以用valgrind memcheck。复现内存泄漏，这个通常比较难实现，一般是线下测试无法复现，线上用户量大，运行久了才会复现，而valgrind会导致程序运行很慢，无法支撑线上测试，因此这个选项通常不太适用于线上。

第四，使用Visual Leak Detector。valgrind是linux下的，如果程序可以跨平台，或者只在win下，那么可以试试这个，这个和valgrind一样，需要复现泄漏才能得到堆栈，因此也是用于线下调试比较多。

第五，重载new、delete。像我[之前的博客](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/coding-my-life/p/10125538.html)里写的，可以简单地加个计数，用于平时预防泄漏，也可更深入一些，记录内存的分配，得到内存漏泄的堆栈，但是这个是否能支撑线上debug，我持怀疑态度。

第六，使用自己的内存分配函数，每一个内存分配，都使用自己的函数，每一个STL的容器，都传入自己的分配器，然后分别记录这些内存分配的大小。这个方法看起来很不现实，但我确实见过在实际的项目中使用，对内存统计、查找有很大的帮助，而且支持在线上debug。查找内存，只需要打印下每个分配器分配的内存大小基本上可以得到结论是哪个分配器出问题。唯一的问题是它增加了开发难度，而且不能像valgrind那样不需要修改原程序即可使用。

第七，使用valgrind massif。valgrind memcheck需要复现内存泄漏，所以不容易找出问题。它会定时记录分配内存的各个堆栈以及分配内存的量，当出现内存泄漏时，根据分配内存的量检查下各个堆栈，应该是可以找到问题的。massif也会导致程序运行慢，但比memcheck要快，能不能在线上debug，这个依然得看具体情况

第八，使用第三方内存分配器，如jemalloc。并不是说使用第三方内存分配器就解决问题了，而是jemalloc自带了一大堆工具，其中jeprof可以得到内存的大小以及堆栈等信息，对查找内存泄漏有很大帮助。不过开启prof后，效率如何，能不能在线上使用，我倒是没测试过。

## **3. 内存碎片**

假如找不到内存泄漏，也许本来就没有内存泄漏，这时不妨考虑下内存碎片的问题。这里以linux下的ptmalloc为例（其他的分配器我就不懂了），说下内存分配。假如一个进程，依次分配了内存块m1(1k)、m2(10b)、

![img](https://pic2.zhimg.com/80/v2-7fb767e2bb0f024de78cf2fffe62bf51_720w.webp)

m3(1k)，然后释放了m2，那整个内存看起来是这样子的：

我们可以看到，m1、m2、m3是按顺序分配的，当m2被释放时，那中间就空了一块了。那空的这一块怎么办，是把它还给系统了吗？这个问题就很复杂了，涉及到ptmalloc的整个分配机制，这里不打算详细说，建议看[华庭(庄明强) - ptmalloc2源代码分析](https://link.zhihu.com/?target=https%3A//paper.seebug.org/papers/Archive/refs/heap/glibc%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86ptmalloc%E6%BA%90%E4%BB%A3%E7%A0%81%E5%88%86%E6%9E%90.pdf)。简单来讲，就是ptmalloc会暂把释放的内存按大小用链表存起来，比如10b的，放到fast bin那个链表，大一点的，放small bin的第一个链表，再大一点，放small bin的第二个链表，... 放进去的内存，直到第再次用到时取出。

随着程序运行，放进链表的内存可能会越来越多，但是却很少取出（可能是程序释放后没有再申请，也可能是申请的大小和链表里的大小不合适，比如链表里有个10b的，但是程序申请了1k），那这些小内存就会越来越多，进程占用的内存也会越来越多，但实际使用的内存不多。那如何检测这种情况呢？

方法一，使用

malloc_stats。malloc_stats是一个glibc的函数，因此可以在gdb调用

```text
gdb -p 16021
call malloc_stats()

Arena 0:
system bytes     =    1359872
in use bytes     =     954224
Arena 1:
system bytes     =     135168
in use bytes     =       3488
Arena 2:
system bytes     =     135168
in use bytes     =      20784
Arena 3:
system bytes     =     139264
in use bytes     =     120080
Total (incl. mmap):
system bytes     =    1769472
in use bytes     =    1098576
max mmap regions =          0
max mmap bytes   =          0
```

1. Arena N表示多个分配域，一般一个线程一个
2. system bytes 当前申请的内存总数
3. in use bytes 当前使用的内存总数
4. max mmap regions 使用mmap分配了多少块内存(大内存用mmap分配，大于128K，可由M_MMAP_THRESHOLD选项调节)
5. max mmap bytes 使用mmap分配了多少内存

这里，system bytes减去in use bytes就可以得到当前进程缓存了多少内存。不过malloc_stats是一个很老的接口了，里面的变量都是用的int，如果你的程序占用内存比较大，这里可能会溢出。

方法二，使用使用malloc_info

```text
gdb -p 16021
call malloc_info(0, stdout)

<malloc version="1">
<heap nr="0">
<sizes>
<size from="17" to="32" total="3104" count="97"/>
<size from="33" to="48" total="11136" count="232"/>
<size from="49" to="64" total="12288" count="192"/>
<size from="65" to="80" total="14640" count="183"/>
<size from="81" to="96" total="4896" count="51"/>
<size from="97" to="112" total="1232" count="11"/>
<size from="113" to="128" total="7296" count="57"/>
<size from="33" to="33" total="13299" count="403"/>
<size from="97" to="97" total="97" count="1"/>
<size from="7281" to="7281" total="7281" count="1"/>
<size from="32833" to="32833" total="32833" count="1"/>
<unsorted from="145" to="8753" total="166107" count="155"/>
</sizes>
<total type="fast" count="823" size="54592"/>
<total type="rest" count="561" size="219617"/>
<system type="current" size="1359872"/>
<system type="max" size="1376256"/>
<aspace type="total" size="1359872"/>
<aspace type="mprotect" size="1359872"/>
</heap>
<heap nr="1">
<sizes>
<size from="33" to="48" total="48" count="1"/>
<unsorted from="4673" to="4705" total="9378" count="2"/>
</sizes>
<total type="fast" count="1" size="48"/>
<total type="rest" count="2" size="9378"/>
<system type="current" size="135168"/>
<system type="max" size="135168"/>
<aspace type="total" size="135168"/>
<aspace type="mprotect" size="135168"/>
</heap>
<heap nr="2">
<sizes>
<size from="33" to="48" total="48" count="1"/>
<size from="113" to="128" total="128" count="1"/>
<size from="65" to="65" total="65" count="1"/>
<unsorted from="81" to="3233" total="10054" count="6"/>
</sizes>
<total type="fast" count="2" size="176"/>
<total type="rest" count="7" size="10119"/>
<system type="current" size="135168"/>
<system type="max" size="135168"/>
<aspace type="total" size="135168"/>
<aspace type="mprotect" size="135168"/>
</heap>
<heap nr="3">
<sizes>
<size from="65" to="80" total="80" count="1"/>
</sizes>
<total type="fast" count="1" size="80"/>
<total type="rest" count="0" size="0"/>
<system type="current" size="139264"/>
<system type="max" size="139264"/>
<aspace type="total" size="139264"/>
<aspace type="mprotect" size="139264"/>
</heap>
<total type="fast" count="827" size="54896"/>
<total type="rest" count="570" size="239114"/>
<total type="mmap" count="0" size="0"/>
<system type="current" size="1769472"/>
<system type="max" size="1785856"/>
<aspace type="total" size="1769472"/>
<aspace type="mprotect" size="1769472"/>
</malloc>
```

\1. nr即arena，通常一个线程一个

\2. <size from="17" to="32" total="3104" count="97"/>上面说了，大小在一定范围内的内存，会放到一个链表里，这就是其中一个链表。from是内存下限，to是上限，上面的意思是内存分配在 [17,32]范围内的空闲内存总共有97个，占3104字节内存。在这个区间内的内存申请都会被对齐为32，故total = to * count

\3. <total type="fast" count="2" size="176"/> 即fastbin这链表当前有2个空闲内存块，大小为176

<total type="rest" count="7" size="10119"/> 除fastbin以外，所有链表空闲的内存数量，以及内存大小。因此fast和rest加起来，应该和当前arena里所有的size一致，如

```text
<heap nr="2">
<sizes>
<size from="33" to="48" total="48" count="1"/>
<size from="113" to="128" total="128" count="1"/>
<size from="65" to="65" total="65" count="1"/>
<unsorted from="81" to="3233" total="10054" count="6"/>
</sizes>
<total type="fast" count="2" size="176"/>
<total type="rest" count="7" size="10119"/>
 
```

前两个to大小为48和128为fast bin，数量为2，剩下的都为rest，与下面的fast和reset对应。

\5. <total type="mmap" count="0" size="0"/> 使用mmap分配的当前在使用块数(count)和当前在用的内存大小(size)(低版本glibc无此字段，如centos6上的glibc 2.12)

\6. <system type="current" size="1769472"/> 当前已经申请的内存大小

\7. <system type="max" size="1785856"/> 历史上申请的内存大小（包括已经归还给操作系统的）

\8. total和mprotect看源码没看出是什么东西

到这里可以看到，假如一个进程fast和reset里的数量很多，那么说明这个进程其实缓存了很多内存。另外这里都是直接用gdb attach到一个进程直接调用函数，打印到stdout。如果需要查看的程序被关掉了stdout或者重定向了stdout（很多服务器进程都这么做），那可能看不见了，或者信息不是打印到当前终端。

## **4. 内存利用率**

如果一个进程占用的内存远高于预期，但没有持续上涨，还需要考虑下是不是内存使用率的问题。当使用new分配一块内存时，系统需要为这次分配记录大小、地址，分配的内存也需要对齐，假如分配的内存很小（比如说1b），那系统最终需要消耗的内存是远大于1b的。比如

```text
#include <cstring>
#include <iostream>

int main()
{
#define PAUSE(msg) std::cout << msg << std::endl; std::cin >> p

    char p = NULL;

    size_t total = 0;
    while (total < 1024 * 1024 * 1024)
    {
        size_t size = rand() % 16;

        total += size;
        char *p = new char[size];
    }

    PAUSE("pause");
```

这个程序每次分配小于16字节的内存，直到总分配量到1G，然而，在我的系统里（ubuntu 20.04），这个程序跑起来占用的内存就多得多

```text
进程号 USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND  
   4174 root      20   0 4479488   4.3g   1616 S   0.0  59.0   0:15.97 ./a.out 
```

已经达到了4.3G，显然内存利用率只有1/4不到。你也许会说这种分配小内存的情况不多，但其实不是的。举个例子，做关键字搜索时，会用到二叉搜索树，每一个树的节点对应一个字符，比如"abcd“就需要分配4个节点，但是每个节点其实很小。假如关键字很多（上百万还是很常见的），那这个问题就比较严重。这时候就应该使用valgrind massif来看下，到底是哪个地方分配的内存，然后根据逻辑优化即可。

原文地址：https://zhuanlan.zhihu.com/p/564309039

作者：linux