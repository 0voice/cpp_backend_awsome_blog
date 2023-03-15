# 【NO.353】内存泄漏的原因，内存泄漏如何避免？内存泄漏如何定位？

## 1. 内存溢出

内存溢出 OOM （out of memory），是指程序在申请内存时，没有足够的内存空间供其使用，出现out of memory；比如申请了一个int,但给它存了long才能存下的数，那就是内存溢出。

## 2. 内存泄漏

内存泄露 memory leak，是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光。最终的结果就是导致OOM。

内存泄漏是指你向系统申请分配内存进行使用(new)，可是使用完了以后却不归还(delete)，结果你申请到的那块内存你自己也不能再访问（也许你把它的地址给弄丢了），而系统也不能再次将它分配给需要的程序。

## 3. 造成内存泄露常见的三种情况

1，指针重新赋值

2，错误的内存释放

3，返回值的不正确处理

**3.1 指针重新赋值**

如下代码：

```text
char * p = (char *)malloc(10);
char * np = (char *)malloc(10);
```

其中，指针变量 p 和 np 分别被分配了 10 个字节的内存。

如果程序需要执行如下赋值语句：

```text
p=np;
```

这时候，指针变量 p 被 np 指针重新赋值，其结果是 p 以前所指向的内存位置变成了孤立的内存。它无法释放，因为没有指向该位置的引用，从而导致 10 字节的内存泄漏。

因此，在对指针赋值前，一定确保内存位置不会变为孤立的。

类似的情况，连续重复new的情况也是类似：

```text
 int *p = new int; 
 p = new int...;//错误
```

**3.2 错误的内存释放**

假设有一个指针变量 p，它指向一个 10 字节的内存位置。该内存位置的第三个字节又指向某个动态分配的 10 字节的内存位置。

如果程序需要执行如下赋值语句时：

```text
free(p);
```

很显然，如果通过调用 free 来释放指针 p，则 np 指针也会因此而变得无效。np 以前所指向的内存位置也无法释放，因为已经没有指向该位置的指针。换句话说，np 所指向的内存位置变为孤立的，从而导致内存泄漏。

因此，每当释放结构化的元素，而该元素又包含指向动态分配的内存位置的指针时，应首先遍历子内存位置（如本示例中的 np），并从那里开始释放，然后再遍历回父节点，如下面的代码所示：

```text
free(p->np);
free(p);
```

**3.3 返回值的不正确处理**

有时候，某些函数会返回对动态分配的内存的引用，如下面的示例代码所示：

```text
char *f(){
	return (char *)malloc(10);
}
void f1(){
	f();
}
```

函数 f1 中对 f 函数的调用并未处理该内存位置的返回地址，其结果将导致 f 函数所分配的 10 个字节的块丢失，并导致内存泄漏。

**4 在内存分配后忘记使用 free 进行释放**

## 4. 如何避免内存泄露？

- 确保没有在访问空指针。
- 每个内存分配函数都应该有一个 free 函数与之对应，alloca 函数除外。
- 每次分配内存之后都应该及时进行初始化，可以结合 memset 函数进行初始化，calloc 函数除外。
- 每当向指针写入值时，都要确保对可用字节数和所写入的字节数进行交叉核对。
- 在对指针赋值前，一定要确保没有内存位置会变为孤立的。
- 每当释放结构化的元素（而该元素又包含指向动态分配的内存位置的指针）时，都应先遍历子内存位置并从那里开始释放，然后再遍历回父节点。
- 始终正确处理返回动态分配的内存引用的函数返回值。

## 5.定位内存泄漏（valgrind）（重点）

**5.1、基本概念**

Valgrind是一个GPL的软件，用于Linux（For x86, amd64 and ppc32）程序的内存调试和代码剖析。你可以在它的环境中运行你的程序来监视内存的使用情况，比如C 语言中的malloc和free或者 C++中的new和 delete。使用Valgrind的工具包，你可以自动的检测许多内存管理和线程的bug，避免花费太多的时间在bug寻找上，使得你的程序更加稳固。

安装Valgrind

```text
//valgrind下载：
http://valgrind.org/downloads/valgrind-3.12.0.tar.bz2

valgrind安装：
1. tar -jxvf valgrind-3.12.0.tar.bz2
2. cd valgrind-3.12.0
3. ./configure
4. make
5. sudo make install
```

应用环境：Linux

编程语言：C/C++

使用方法： 编译时加上-g选项，如 gcc -g filename.c -o filename,使用如下命令检测内存使用情况：

```text
最常用的命令格式：
valgrind --tool=memcheck --leak-check=full ./test

valgrind --tool=memcheck --leak-check=full --show-reachable=yes --trace-children=yes  ./filename

其中--leak-check=full指的是完全检查内存泄漏，--show-reachable=yes是显示内存泄漏的地点，--trace-children=yes是跟入子进程。
```

如果您的程序是会正常退出的程序，那么当程序退出的时候valgrind自然会输出内存泄漏的信息。如果您的程序是个守护进程，那么也不要紧，我们 只要在别的终端下杀死memcheck进程（因为valgrind默认使用memcheck工具，就是默认参数--tools=memcheck）

参数选择

```text
 -tool=<name> 最常用的选项。运行 valgrind中名为toolname的工具。默认memcheck。
        memcheck ------> 这是valgrind应用最广泛的工具，一个重量级的内存检查器，能够发现开发中绝大多数内存错误使用情况，比如：使用未初始化的内存，使用已经释放了的内存，内存访问越界等。
        callgrind ------> 它主要用来检查程序中函数调用过程中出现的问题。
        cachegrind ------> 它主要用来检查程序中缓存使用出现的问题。
        helgrind ------> 它主要用来检查多线程程序中出现的竞争问题。
        massif ------> 它主要用来检查程序中堆栈使用中出现的问题。
        extension ------> 可以利用core提供的功能，自己编写特定的内存调试工具
    -h –help 显示帮助信息。
    -version 显示valgrind内核的版本，每个工具都有各自的版本。
    -q –quiet 安静地运行，只打印错误信息。
    -v –verbose 更详细的信息, 增加错误数统计。
    -trace-children=no|yes 跟踪子线程? [default: no]
    -track-fds=no|yes 跟踪打开的文件描述？[default: no]
    -time-stamp=no|yes 增加时间戳到LOG信息? [default: no]
    -log-fd=<number> 输出LOG到描述符文件 [2=stderr]
    -log-file=<file> 将输出的信息写入到filename.PID的文件里，PID是运行程序的进行ID
    -log-file-exactly=<file> 输出LOG信息到 file
    -log-file-qualifier=<VAR> 取得环境变量的值来做为输出信息的文件名。 [none]
    -log-socket=ipaddr:port 输出LOG到socket ，ipaddr:port

LOG信息输出

    -xml=yes 将信息以xml格式输出，只有memcheck可用
    -num-callers=<number> show <number> callers in stack traces [12]
    -error-limit=no|yes 如果太多错误，则停止显示新错误? [yes]
    -error-exitcode=<number> 如果发现错误则返回错误代码 [0=disable]
    -db-attach=no|yes 当出现错误，valgrind会自动启动调试器gdb。[no]
    -db-command=<command> 启动调试器的命令行选项[gdb -nw %f %p]
```

设计思路：根据软件的内存操作维护一个有效地址空间表和无效地址空间表（进程的地址空间）

**5.2、多个工具**

1、Memcheck

最常用的工具，用来检测程序中出现的内存问题，所有对内存的读写都会被检测到，一切对malloc()/free()/new/delete的调用都会被捕获。所以，Memcheck 工具主要检查下面的程序错误

能够检测：

- 使用未初始化的内存 (Use of uninitialised memory)
- 使用已经释放了的内存 (Reading/writing memory after it has been free’d)
- 使用超过 malloc分配的内存空间(Reading/writing off the end of malloc’d blocks)
- 对堆栈的非法访问 (Reading/writing inappropriate areas on the stack)
- 申请的空间是否有释放 (Memory leaks – where pointers to malloc’d blocks are lost forever)
- malloc/free/new/delete申请和释放内存的匹配(Mismatched use of malloc/new/new [] vs free/delete/delete [])
- src和dst的重叠(Overlapping src and dst pointers in memcpy() and related functions)
- 重复free

![img](https://pic3.zhimg.com/80/v2-b698870a15e09c53a8a023c668e8d38a_720w.webp)

Callgrind

和gprof类似的分析工具，但它对程序的运行观察更是入微，能给我们提供更多的信息。和gprof不同，它不需要在编译源代码时附加特殊选项，但加上调试选项是推荐的。Callgrind收集程序运行时的一些数据，建立函数调用关系图，还可以有选择地进行cache模拟。在运行结束时，它会把分析数据写入一个文件。callgrind_annotate可以把这个文件的内容转化成可读的形式。

Cachegrind

Cache分析器，它模拟CPU中的一级缓存I1，Dl和二级缓存，能够精确地指出程序中cache的丢失和命中。如果需要，它还能够为我们提供cache丢失次数，内存引用次数，以及每行代码，每个函数，每个模块，整个程序产生的指令数。这对优化程序有很大的帮助。

Helgrind

它主要用来检查多线程程序中出现的竞争问题。Helgrind寻找内存中被多个线程访问，而又没有一贯加锁的区域，这些区域往往是线程之间失去同步的地方，而且会导致难以发掘的错误。Helgrind实现了名为“Eraser”的竞争检测算法，并做了进一步改进，减少了报告错误的次数。不过，Helgrind仍然处于实验阶段。

Massif

堆栈分析器，它能测量程序在堆栈中使用了多少内存，告诉我们堆块，堆管理块和栈的大小。Massif能帮助我们减少内存的使用，在带有虚拟内存的现代系统中，它还能够加速我们程序的运行，减少程序停留在交换区中的几率。

**5.3、使用原理**

![img](https://pic3.zhimg.com/80/v2-7d0f81e4c1a96f95d1b9a6a69763dd7a_720w.webp)

![img](https://pic2.zhimg.com/80/v2-abb0316ea56fd7260878196540f5e559_720w.webp)

![img](https://pic1.zhimg.com/80/v2-210cf3c1a5ad0f62f74bc3a95217f8a8_720w.webp)

Memcheck 能够检测出内存问题，关键在于其建立了两个全局表。

1、Valid-Value 表：

对于进程的整个地址空间中的每一个字节(byte)，都有与之对应的 8 个 bits；对于 CPU 的每个寄存器，也有一个与之对应的 bit 向量。这些 bits 负责记录该字节或者寄存器值是否具有有效的、已初始化的值。

2、Valid-Address 表

对于进程整个地址空间中的每一个字节(byte)，还有与之对应的 1 个 bit，负责记录该地址是否能够被读写。

检测原理：

- 当要读写内存中某个字节时，首先检查这个字节对应的 A bit。如果该A bit显示该位置是无效位置，memcheck 则报告读写错误。
- 内核（core）类似于一个虚拟的 CPU 环境，这样当内存中的某个字节被加载到真实的 CPU 中时，该字节对应的 V bit也被加载到虚拟的 CPU 环境中。一旦寄存器中的值，被用来产生内存地址，或者该值能够影响程序输出，则 memcheck 会检查对应的V bits，如果该值尚未初始化，则会报告使用未初始化内存错误。

**5.4、具体使用**

\1. 使用未初始化的内存（使用野指针）

这里我们定义了一个指针p，但并未给他开辟空间，即他是一个野指针，但我们却使用它了

![img](https://pic1.zhimg.com/80/v2-b042352183808f63bc5e94f347dc1078_720w.webp)

Valgrind检测出我们程序使用了未初始化的变量，但并未检测出内存泄漏。

![img](https://pic3.zhimg.com/80/v2-06dc0e4201d006c17f789e41eac15452_720w.webp)

2.在内存被释放后进行读/写（使用野指针）

p所指向的内存被释放了，p变成了野指针，但是我们却继续使用这片内存。

![img](https://pic3.zhimg.com/80/v2-acd367b68bf765932b3c20589b4e2932_720w.webp)

Valgrind检测出我们使用了已经free掉的内存，并给出这片内存是哪里分配哪里释放的。

![img](https://pic2.zhimg.com/80/v2-5e889c52245362966222a1456586821d_720w.webp)

3.从已分配内存块的尾部进行读/写（动态内存越界）

我们动态地分配了一段数组，但我们在访问个数组时发生了越界读写，程序crash掉。

![img](https://pic2.zhimg.com/80/v2-03928d457a18abb5bfe973db54d85121_720w.webp)

Valgrind检测出越界的位置。

![img](https://pic3.zhimg.com/80/v2-8ac0a30fd7438532de6a2afc1212cef6_720w.webp)

注意：Valgrind不检查静态分配数组的使用情况！所以对静态分配的数组，Valgrind表示无能为力！比如下面的例子，程序crash掉，我们却不知道为什么。

![img](https://pic1.zhimg.com/80/v2-b26b13508ce182b2e59d471b2c0b4b58_720w.webp)

![img](https://pic2.zhimg.com/80/v2-2d3c16429c39cc063777e58db9b4fc35_720w.webp)

4.内存泄漏

内存泄漏的原因在于没有成对地使用malloc/free和new/delete，比如下面的例子。

![img](https://pic2.zhimg.com/80/v2-de361db64325f9c93d61c2bbd5450741_720w.webp)

Valgrind会给出程序中malloc和free的出现次数以判断是否发生内存泄漏，比如对上面的程序运行memcheck，Valgrind的记录显示上面的程序用了1次malloc，却调用了0次free，明显发生了内存泄漏！

![img](https://pic4.zhimg.com/80/v2-9d32d6c4faf6ebb0943e11d33b3baed3_720w.webp)

上面提示了我们可以使用–leak-check=full进一步获取内存泄漏的信息，比如malloc和free的具体行号。

![img](https://pic3.zhimg.com/80/v2-01d09eaf22538c4af7b091c6646f9d0a_720w.webp)

\5. 不匹配地使用malloc/new/new[] 和 free/delete/delete[]

正常使用new/delete和malloc/free是这样子的：

![img](https://pic4.zhimg.com/80/v2-b411a6ffbd053018a2ec34015ce9bb23_720w.webp)

![img](https://pic3.zhimg.com/80/v2-d35526f6668e7b0498d89c756403ecb6_720w.webp)

而不匹配地使用malloc/new/new[] 和 free/delete/delete[]则会被提示mismacth：

![img](https://pic2.zhimg.com/80/v2-0aedd0d926b7c7a35de1493975c38601_720w.webp)

![img](https://pic2.zhimg.com/80/v2-bf9c45b31ba38a218916c43cbc73aeb9_720w.webp)

6.两次释放内存

double free的情况同样是根据malloc/free的匹配对数来体现的，比如free多了一次，Valgrind也会提示。

![img](https://pic4.zhimg.com/80/v2-0baa71eabaa30d0552b4730a0d1d0e6f_720w.webp)

![img](https://pic1.zhimg.com/80/v2-f2cf40cc91eb282f972c2bce6e9777a4_720w.webp)

原文地址：https://zhuanlan.zhihu.com/p/458541056

作者：linux