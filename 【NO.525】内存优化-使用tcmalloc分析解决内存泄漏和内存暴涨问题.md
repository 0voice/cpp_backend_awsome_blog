# 【NO.525】内存优化-使用tcmalloc分析解决内存泄漏和内存暴涨问题

## 1.下载安装tcmalloc

\#1、到google下载代码：

当然你最好下载最新或者最稳定版本，这里比如下载2.1版本：

wget [https://gperftools.googlecode.com/files/gperftools-2.1.tar.gz](https://link.zhihu.com/?target=https%3A//gperftools.googlecode.com/files/gperftools-2.1.tar.gz)

\#解压

tar -zxvf google-perftools-2.1.tar.gz

\#看看说明

cd google-perftools-2.1

./configure -h

./configure

make && make install

## 2.代码中使用tcmalloc替换malloc

我们如何使用tcmalloc来替换glibc的malloc呢？

在链接tcmalloc的时候我们可以使用以下任意一种方式：

1.启动程序之前，预先加载tcmalloc动态库的环境变量设置： exportLD_PRELOAD="
/usr/local/lib/libtcmalloc.so"

2.在你的动态库链接的地方加入：-ltcmalloc

## 3.检测内存泄漏

**測试代码1：**

```text
#include <iostream>
using namespace std;
int main()
{
        int *p = new int();
        return 0;
}
```

编译：g++ t.cpp -o main -ltcmalloc -g -O0

内存泄漏检查： env HEAPCHECK=normal ./main

结果：

```text
root@ubuntu:/home/gaoke/test# env HEAPCHECK=normal ./main
WARNING: Perftools heap leak checker is active -- Performance may suffer
Have memory regions w/o callers: might report false leaks
Leak check _main_ detected leaks of 4 bytes in 1 objects
The 1 largest leaks:
*** WARNING: Cannot convert addresses to symbols in output below.
*** Reason: Cannot find 'pprof' (is PPROF_PATH set correctly?)
*** If you cannot fix this, try running pprof directly.
Leak of 4 bytes in 1 objects allocated from:
  @ 4007ef 
  @ 7f7895a64f45 
  @ 400719 


If the preceding stack traces are not enough to find the leaks, try running THIS shell command:

pprof ./main "/tmp/main.6712._main_-end.heap" --inuse_objects --lines --heapcheck  --edgefraction=1e-10 --nodefraction=1e-10 --gv

If you are still puzzled about why the leaks are there, try rerunning this program with HEAP_CHECK_TEST_POINTER_ALIGNMENT=1 and/or with HEAP_CHECK_MAX_POINTER_OFFSET=-1
If the leak report occurs in a small fraction of runs, try running with TCMALLOC_MAX_FREE_QUEUE_SIZE of few hundred MB or with TCMALLOC_RECLAIM_MEMORY=false, it might help find leaks more repeatably
Exiting with error code (instead of crashing) because of whole-program memory leaks
```

大家注意，这里有关键字Leak，你就得当心这里可能存在内存泄漏，提示

Leak of 4 bytes in 1 objects allocated from

对，是有四字节的内存泄漏，虽然你看代码能看到指针p未释放，但是这里你需要掌握的是在你无法直观的通过阅读代码来找到内存泄漏点的情况下，如何用tcmalloc工具来分析问题。

相信细心的你会注意到运行输出的这一行

pprof ./main "/tmp/main.6712._main_-end.heap" --inuse_objects --lines --heapcheck --edgefraction=1e-10 --nodefraction=1e-10 --gv

这里就是我要重点讲的**pprof工具**

google-perftool提供了一个叫pprof的工具，它是一个perl的脚本，通过这个工具，可以将google-perftool的输出结果分析得更为直观，输出为text、图片、pdf等格式。

这里我们把结果通过text的方式输出：你只需要把刚才的--gv换成--text

pprof ./main "/tmp/main.6712._main_-end.heap" --inuse_objects --lines --heapcheck --edgefraction=1e-10 --nodefraction=1e-10 --text

![img](https://pic3.zhimg.com/80/v2-2c76472bc6570da8926d7ec55bd43472_720w.webp)

好了，你可以看到这里已经很明显了，给你提示了t.cpp文件的第五行代码存在内存泄漏（当然你也可以输出其他格式，raw，png，pdf等等，whatever，只要可以帮助你去分析问题解决问题）。

实际上项目中遇到的内存泄漏问题是异常复杂的，我给的这个示例只是小试牛刀。项目常见的内存泄漏点大家都清楚，new了但是没有得到delete，但是要根据pprof工具对应的函数，代码行找到对应的泄漏点你可能需要花费点功夫。

实际上你的大多数应用都是以服务的方式启动，长时间处于作业/工作状态。你需要定期来检测下内存泄漏情况，那么这时你需要显示的调用接口来输出leak情况，

**示例代码2**

```text
bool memory_check(void* arg)
{
    HeapLeakChecker::NoGlobalLeaks();
    return TRUE;
}
```

将上面的代码加到你的定时检测逻辑里，或者需要观察的点，那么他就会输出示例1中的内容，动态的帮助你分析内存泄漏点。

## 4.分析使用tcmalloc后内存暴涨不降问题

记得几年前我开始推广大家使用tcmalloc后，一些同事做压测过程中也遇到了不少麻烦。比如当有大量数据过来，new出来很多的大块内存，突然发现有时候内存增长到几个G，开始以为是内存泄露的问题。

![img](https://pic1.zhimg.com/80/v2-34354b03b8124b88c621dba3749b2e58_720w.webp)

先是用tcmalloc环境变量来检查内存泄漏没有找到泄漏的报告，用valgrind也做了大量的测试，但是valgrind显示没有内存泄露。 实际上遇到这种问题不要慌，基本上是对tcmalloc使用上的问题，你要知道默认情况下，tcmalloc会将长时间未用的内存交还系统。tcmalloc_release_rate这个flag控制了这个交回频率。你可以在运行时通过这个语句强制这个release发生：

MallocExtension::instance()->ReleaseFreeMemory();

当然了，你可以通过 SetMemoryReleaseRate() 来设置这个tcmalloc_release_rate. 如果设置为0，代表永远不交回。数字越大代表交回的频率越大。一般合理的值就是设置一个0 - 10 之间的一个数。也可以通过设置环境变量 TCMALLOC_RELEASE_RATE来设置这个rate。

实际上我估计很多人看了官网说的

MallocExtension::instance()->SetMemoryReleaseRate(7.0);

很疑惑，我曾经带着疑惑做了测试，发现SetMemoryReleaseRate设置9，10回收的内存仍然是很慢的，所以后来我索性在进程启动的开始设置SetMemoryReleaseRate为9，然后在new对象的时候ReleaseFreeMemory，在new对象析构的时候ReleaseFreeMemory一次 （new出来的对象可能从new到delete的生命周期是不确定的，可能存在1天？4小时？30分钟都有可能，而且不是频繁的释放和销毁），因此这种情况下，内存就比较及时的回收了，所以大家可以根据自己的项目逻辑来选择ReleaseFreeMemory的时机，最好不要频繁的申请和释放，这对tcmalloc来说也是难受。

![img](https://pic1.zhimg.com/80/v2-99350e63cdd4b97cf6758449d50e5cbc_720w.webp)

所以你不仅仅要关注tcmalloc申请大小内存块，还要关注内存块的在合适的时间及时回收，否则造成内存占用过高。

原文地址：https://zhuanlan.zhihu.com/p/501249551

作者：linux