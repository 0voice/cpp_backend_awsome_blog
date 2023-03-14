# 【NO.221】实例分析Linux内存泄漏检测方法

## **1.mtrace分析内存泄露**

mtrace（memory trace），是 GNU Glibc 自带的内存问题检测工具，它可以用来协助定位内存泄露问题。它的实现源码在glibc源码的malloc目录下，其基本设计原理为设计一个函数 void mtrace ()，函数对 libc 库中的 malloc/free 等函数的调用进行追踪，由此来检测内存是否存在泄漏的情况。mtrace是一个C函數，在<mcheck.h>里声明及定义，函数原型为：

```text
void mtrace(void);
```

### **1.1 mtrace原理**

`mtrace() `函数会为那些和动态内存分配有关的函数（譬如 malloc()、realloc()、memalign() 以及 free()）安装 “钩子（hook）” 函数，这些 hook 函数会为我们记录所有有关内存分配和释放的跟踪信息，而 muntrace() 则会卸载相应的 hook 函数。基于这些 hook 函数生成的调试跟踪信息，我们就可以分析是否存在 “内存泄露” 这类问题了。

### **1.2 设置日志生成路径**

mtrace 机制需要我们实际运行一下程序，然后才能生成跟踪的日志，但在实际运行程序之前还有一件要做的事情是需要告诉 mtrace （即前文提到的 hook 函数）生成日志文件的路径。设置日志生成路径有两种，一种是设置环境变量：`export MALLOC_TRACE=./test.log // 当前目录下` 另一种是在代码层面设置：`setenv("MALLOC_TRACE", "output_file_name", 1);``output_file_name`就是储存检测结果的文件的名称。

### **1.3 测试实例**

```text
#include <mcheck.h>
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char **argv)
{
    mtrace();  // 开始跟踪

    char *p = (char *)malloc(100);
    free(p);
    p = NULL;
    p = (char *)malloc(100);

    muntrace();   // 结束跟踪，并生成日志信息
    return 0;
}
```

从上述代码中，我们希望能够在程序开始到结束检查内存是否泄漏的问题，例子简单，一眼就能看出存在内存泄漏的问题，所以我们需要验证 mtrace 是否能够检查出来内存泄漏问题，且检查的结果如何分析定位。 `gcc -g test.c -o test`生成可执行文件。

### **1.4 日志**

程序运行结束，会在当前目录生成 test.log 文件，打开可以看到一下内容：

```text
= Start
@ ./test:[0x400624] + 0x21ed450 0x64
@ ./test:[0x400634] - 0x21ed450
@ ./test:[0x400646] + 0x21ed450 0x64
= End
```

从这个文件中可以看出中间三行分别对应源码中的 malloc -> free -> malloc 操作；解读：./test 指的是我们执行的程序名字，[0x400624] 是第一次调用 malloc 函数机器码中的地址信息，+ 表示申请内存（ - 表示释放），0x21ed450 是 malloc 函数申请到的地址信息，0x64 表示的是申请的内存大小。由此分析第一次申请已经释放，第二次申请没有释放，存在内存泄漏的问题。

### **1.5 泄露分析**

**使用addr2line工具定位源码位置**

通过使用 "addr2line" 命令工具，得到源文件的行数（通过这个可以根据机器码地址定位到具体源码位置）

```text
# addr2line -e test 0x400624
/home/test.c:9
```

**使用mtrace工具分析日志信息**

mtrace + 可执行文件路径 + 日志文件路径 `mtrace test ./test.log`执行，输出如下信息：

```text
Memory not freed:
-----------------
           Address     Size     Caller
0x00000000021ed450     0x64  at /home/test.c:14
```



## **2.Valgrind分析内存泄露**

### 2.1 **Valgrind工具介绍**

Valgrind是一套Linux下，开放源代码（GPL V2）的仿真调试工具的集合。Valgrind由内核（core）以及基于内核的其他调试工具组成。内核类似于一个框架（framework），它模拟了一个CPU环境，并提供服务给其他工具；而其他工具则类似于插件 (plug-in)，利用内核提供的服务完成各种特定的内存调试任务。Valgrind的体系结构如下图所示

![img](https://pic3.zhimg.com/80/v2-b9cfeec617d871c4623ffdb3c52a3352_720w.webp)

**1、Memcheck**

最常用的工具，用来检测程序中出现的内存问题，所有对内存的读写都会被检测到，一切对malloc() / free() / new / delete 的调用都会被捕获。所以，它能检测以下问题：对未初始化内存的使用；读/写释放后的内存块；读/写超出malloc分配的内存块；读/写不适当的栈中内存块；内存泄漏，指向一块内存的指针永远丢失；不正确的malloc/free或new/delete匹配；memcpy()相关函数中的dst和src指针重叠。

**2、Callgrind**

和 gprof 类似的分析工具，但它对程序的运行观察更是入微，能给我们提供更多的信息。和 gprof 不同，它不需要在编译源代码时附加特殊选项，但加上调试选项是推荐的。Callgrind 收集程序运行时的一些数据，建立函数调用关系图，还可以有选择地进行 cache 模拟。在运行结束时，它会把分析数据写入一个文件。callgrind_annotate 可以把这个文件的内容转化成可读的形式。

**3、Cachegrind**

Cache 分析器，它模拟 CPU 中的一级缓存 I1，Dl 和二级缓存，能够精确地指出程序中 cache 的丢失和命中。如果需要，它还能够为我们提供 cache 丢失次数，内存引用次数，以及每行代码，每个函数，每个模块，整个程序产生的指令数。这对优化程序有很大的帮助。

**4、Helgrind**

它主要用来检查多线程程序中出现的竞争问题。Helgrind 寻找内存中被多个线程访问，而又没有一贯加锁的区域，这些区域往往是线程之间失去同步的地方，而且会导致难以发掘的错误。Helgrind 实现了名为“Eraser”的竞争检测算法，并做了进一步改进，减少了报告错误的次数。不过，Helgrind 仍然处于实验阶段。

**5、Massif**

堆栈分析器，它能测量程序在堆栈中使用了多少内存，告诉我们堆块，堆管理块和栈的大小。Massif 能帮助我们减少内存的使用，在带有虚拟内存的现代系统中，它还能够加速我们程序的运行，减少程序停留在交换区中的几率。此外，lackey 和 nulgrind 也会提供。Lackey 是小型工具，很少用到；Nulgrind 只是为开发者展示如何创建一个工具。

### **2.1Memcheck原理**

本文的重点是在检测内存泄露，所以对于valgrind的其他工具不做过多的说明，主要说明下Memcheck的工作。Memcheck检测内存问题的原理如下图所示：

![img](https://pic3.zhimg.com/80/v2-21fb6d668ed503d540c206ab3fdf79f2_720w.webp)

Memcheck 能够检测出内存问题，关键在于其建立了两个全局表。

- Valid-Value 表 对于进程整个地址空间中的每一个字节(byte)，都有与之对应的 8个bits；对于 CPU 的每个寄存器，也有一个与之对应的 bit 向量。这些 bits 负责记录该字节或者寄存器值是否具有有效的、已初始化的值。
- Valid-Address 表 对于进程整个地址空间中的每一个字节(byte)，还有与之对应的1个 bit，负责记录该地址是否能够被读写。
- 检测原理：当要读写内存中某个字节时，首先检查这个字节对应的Valid-Address 表中的 A bit。如果该 A bit显示该位置是无效位置，memcheck 则报告读写错误。内核（core）类似于一个虚拟的 CPU 环境，这样当内存中的某个字节被加载到真实的 CPU 中时，该字节对应的Valid-Value 表中的 V bit 也被加载到虚拟的 CPU 环境中。一旦寄存器中的值，被用来产生内存地址，或者该值能够影响程序输出，则 memcheck 会检查对应的V bits，如果该值尚未初始化，则会报告使用未初始化内存错误。

### **2.2 内存泄露类型**

valgrind 将内存泄漏分成 4 类：

- 确立泄露（definitely lost）：运行内存还没有释放出来，但早已沒有表针偏向运行内存，运行内存早已不能浏览。确立泄露的运行内存是强烈要求修补的。
- 间接性泄露（indirectly lost）：泄露的运行内存表针储存在确立泄露的运行内存中，伴随着确立泄露的运行内存不能浏览，造成间接性泄露的运行内存也不能浏览。例如：

```text
struct list {
 struct list *next;
};

int main(int argc, char **argv)
{
 struct list *root;
 root = (struct list *)malloc(sizeof(struct list));
 root->next = (struct list *)malloc(sizeof(struct list));
 printf("root %p roop->next %p\n", root, root->next);
 root = NULL;
 return 0;
}
```

这里遗失的是 root 表针（是确立泄露类型），造成 root 储存的 next 表针变成了间接性泄露。间接性泄露的运行内存毫无疑问也要修补的，但是一般会伴随着 确立泄露 的修补而修补。

- 很有可能泄露（possibly lost）：表针并不偏向运行内存头详细地址，只是偏向运行内存內部的部位。valgrind 往往会猜疑很有可能泄露，是由于表针早已偏位，并沒有偏向运行内存头，只是有运行内存偏位，偏向运行内存內部的部位。有一些情况下，这并并不是泄露，由于这种程序流程便是那么设计方案的，比如为了更好地完成内存对齐，附加申请办理运行内存，回到两端对齐后的内存地址。
- 仍可访达（still reachable）：表针一直存有且偏向运行内存头顶部，直到程序流程撤出时运行内存还没有释放出来。

### **2.3 Valgrind参数设置**

- --leak-check=<no|summary|yes|full> 如果设为 yes 或 full，在被调程序结束后，valgrind 会详细叙述每一个内存泄露情况 默认是summary，只报道发生了几次内存泄露
- --log-file=
- --log-fd= [default: 2, stderr] valgrind 打印日志转存到指定文件或者文件描述符。如果没有这个参数，valgrind 的日志会连同用户程序的日志一起输出，会显得非常乱。
- --trace-children=<yes | no> [default: no] 是否跟踪子进程，若是多进程的程序，则建议使用这个功能。不过单进程使能了也不会有多大影响。
- --keep-debuginfo=<yes | no> [default: no] 如果程序有使用 动态加载库（dlopen），在动态库卸载时（dlclose），debug信息都会被清除。使能这个选项后，即使动态库被卸载，也会保留调用栈信息。
- --keep-stacktraces=<alloc | free | alloc-and-free | alloc-then-free | none> [default: alloc-and-free] 内存泄漏不外乎申请和释放不配对，函数调用栈是只在申请时记录，还是在申请释放时都记录 如果我们只关注内存泄漏，其实完全没必要申请释放都记录，因为这会占用非常多的额外内存和更多的 CPU 损耗，让本来就执行慢的程序雪上加霜。
- --freelist-vol= 当客户程序用 free 或 delete 释放一个内存块时，这个内存块不会立即可用于再分配，它只会被放在一个freed blocks的队列中（freelist）并被标记为不可访问，这样有利于探测到在一段很重要的时间后，客户程序又对被释放的块进行访问的错误。这个选项规定了队列所占的字节块大小，默认是20MB。增大这个选项的会增大memcheck的内存开销，但查这类错的能力也会提升。
- --freelist-big-blocks= 当从 freelist 队列中取可用内存块用于再分配时，memcheck 将会从那些比 number 大的内存块中按优先级取出一个块出来用。这个选项就防止了 freelist 中那些小的内存块的频繁调用，这个选项提高了 查到针对小内存块的野指针错误的几率。若这个选项设为0，则所有的块将按先进先出的原则用于再分配。默认是1M。参考：valgrind 简介(内存检查工具)

### **2.4 编译参数推荐**

为了更好地在出难题时要详尽打印出出去栈信息内容，实际上大家最好是在编译程序时加上 -g 选择项。如果有动态性载入的库，必须再加上 `--keep-debuginfo=yes `，不然假如发觉是动态性载入的库发生泄露，因为动态库被卸载掉了，造成找不到符号表。编码编译程序提升，不建议应用 -O2既之上。-O0很有可能会造成运作变慢，建议使用-O1。

### **2.5 检测实例说明**

**申请不释放内存**

```text
#include <stdlib.h>
#include <stdio.h>
void func()
{
  //只申请内存而不释放
    void *p=malloc(sizeof(int));
}
int main()
{
    func();
    return 0;
}
```

使用valgrind命令来执行程序同时输出日志到文件

```text
valgrind --log-file=valReport --leak-check=full --show-reachable=yes --leak-resolution=low ./a.out
```

参数说明：

- –log-file=valReport 是指定生成分析日志文件到当前执行目录中，文件名为valReport
- –leak-check=full 显示每个泄露的详细信息
- –show-reachable=yes 是否检测控制范围之外的泄漏，比如全局指针、static指针等，显示所有的内存泄露类型
- –leak-resolution=low 内存泄漏报告合并等级
- –track-origins=yes表示开启“使用未初始化的内存”的检测功能，并打开详细结果。如果没有这句话，默认也会做这方面的检测，但不会打印详细结果。执行输出后，报告解读，其中54017是指进程号，如果程序使用了多进程的方式来执行，那么就会显示多个进程的内容。

```text
==54017== Memcheck, a memory error detector
==54017== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==54017== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==54017== Command: ./a.out
==54017== Parent PID: 52130
```

第二段是对堆内存分配的总结信息，其中提到程序一共申请了1次内存，其中0次释放了，4 bytes被分配(`1 allocs, 0 frees, 4 bytes allocated`)。在head summary中，有该程序使用的总heap内存量，分配内存次数和释放内存次数，如果分配内存次数和释放内存次数不一致则说明有内存泄漏。

```text
==54017== HEAP SUMMARY:
==54017==   in use at exit: 4 bytes in 1 blocks
==54017==   total heap usage: 1 allocs, 0 frees, 4 bytes allocated
```

第三段的内容描述了内存泄露的具体信息，其中有一块内存占用4字节（`4 bytes in 1 blocks`），在调用malloc分配，调用栈中可以看到是func函数最后调用了malloc，所以这一个信息是比较准确的定位了我们泄露的内存是在哪里申请的。

```text
==54017== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==54017==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==54017==    by 0x40057E: func() (in /home/oceanstar/CLionProjects/Share/src/a.out)
==54017==    by 0x40058D: main (in /home/oceanstar/CLionProjects/Share/src/a.out)
```

最后这一段是总结，4字节为一块的内存泄露

```text
==54017== LEAK SUMMARY:
==54017==    definitely lost: 4 bytes in 1 blocks  // 确立泄露
==54017==    indirectly lost: 0 bytes in 0 blocks  // 间接性泄露
==54017==    possibly lost: 0 bytes in 0 blocks   // 很有可能泄露
==54017==    still reachable: 0 bytes in 0 blocks // 仍可访达
==54017==    suppressed: 0 bytes in 0 blocks
```

**读写越界**

```text
#include <stdio.h>
#include <iostream>
int main()
{
    int len = 5;
    int *pt = (int*)malloc(len*sizeof(int)); //problem1: not freed
    int *p = pt;
    for (int i = 0; i < len; i++){
        p++;
    }
    *p = 5; //problem2: heap block overrun
    printf("%d\n", *p); //problem3: heap block overrun
    // free(pt);
    return 0;
}
```

problem1: 指针pt申请了空间，但是没有释放; problem2: pt申请了5个int的空间，p经过5次循环已达到p[5]的位置, `*p = 5`时，访问越界（写越界）。(下面valgrind报告中 Invalid write of size 4)

```text
==58261== Invalid write of size 4
==58261==    at 0x400707: main (main.cpp:12)
==58261==  Address 0x5a23054 is 0 bytes after a block of size 20 alloc'd
==58261==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==58261==    by 0x4006DC: main (main.cpp:7)
```

problem1: 读越界 (下面valgrind报告中 Invalid read of size 4 )

```text
==58261== Invalid read of size 4
==58261==    at 0x400711: main (main.cpp:13)
==58261==  Address 0x5a23054 is 0 bytes after a block of size 20 alloc'd
==58261==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==58261==    by 0x4006DC: main (main.cpp:7)
```

**重复释放**

```text
#include <stdio.h>
#include <iostream>
int main()
{
    int *x;
    x = static_cast<int *>(malloc(8 * sizeof(int)));
    x = static_cast<int *>(malloc(8 * sizeof(int)));
    free(x);
    free(x);
    return 0;
}
```

报告如下，`Invalid free() / delete / delete[] / realloc()`

```text
==59602== Invalid free() / delete / delete[] / realloc()
==59602==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==59602==    by 0x4006FE: main (main.cpp:10)
==59602==  Address 0x5a230a0 is 0 bytes inside a block of size 32 free'd
==59602==    at 0x4C2B06D: free (vg_replace_malloc.c:540)
==59602==    by 0x4006F2: main (main.cpp:9)
==59602==  Block was alloc'd at
==59602==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==59602==    by 0x4006E2: main (main.cpp:8)
```

**申请释放接口不匹配**

申请释放接口不匹配的报告如下，用malloc申请空间的指针用free释放；用new申请的空间用delete释放(`Mismatched free() / delete / delete []`)：

```text
==61950== Mismatched free() / delete / delete []
==61950==    at 0x4C2BB8F: operator delete[](void*) (vg_replace_malloc.c:651)
==61950==    by 0x4006E8: main (main.cpp:8)
==61950==  Address 0x5a23040 is 0 bytes inside a block of size 5 alloc'd
==61950==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==61950==    by 0x4006D1: main (main.cpp:7)
```

**内存覆盖**

```text
int main()
{
    char str[11];
    for (int i = 0; i < 11; i++){
        str[i] = i;
    }
    memcpy(str + 1, str, 5);
    char x[5] = "abcd";
    strncpy(x + 2, x, 3);
}
```

问题出在memcpy上， 将str指针位置开始copy 5个char到str+1所指空间，会造成内存覆盖。strncpy也是同理。报告如下，`Source and destination overlap`：

```text
==61609== Source and destination overlap in memcpy(0x1ffefffe31, 0x1ffefffe30, 5)
==61609==    at 0x4C2E81D: memcpy@@GLIBC_2.14 (vg_replace_strmem.c:1035)
==61609==    by 0x400721: main (main.cpp:11)
==61609== 
==61609== Source and destination overlap in strncpy(0x1ffefffe25, 0x1ffefffe23, 3)
==61609==    at 0x4C2D453: strncpy (vg_replace_strmem.c:552)
==61609==    by 0x400748: main (main.cpp:14)
```

## **3.总结**

内存检测方式无非分为两种：

1、维护一个内存操作链表，当有内存申请操作时，将其加入此链表中，当有释放操作时，从申请操作从链表中移除。如果到程序结束后此链表中还有内容，说明有内存泄露了；如果要释放的内存操作没有在链表中找到对应操作，则说明是释放了多次。使用此方法的有内置的调试工具，Visual Leak Detecter，mtrace, memwatch, debug_new。

2、模拟进程的地址空间。仿照操作系统对进程内存操作的处理，在用户态下维护一个地址空间映射，此方法要求对进程地址空间的处理有较深的理解。因为Windows的进程地址空间分布不是开源的，所以模拟起来很困难，因此只支持Linux。采用此方法的是valgrind。

原文地址：https://zhuanlan.zhihu.com/p/597168559

作者：linux