# 【NO.484】Linux服务器开发,手写内存检测组件

## 1.前言

导致内存泄漏的根本原因是因为C++没有内存回收机制gc，内存分配和内存释放次数不对等，是一个很常见的问题，常用的工具有valgrind、mtrace等等，不过这些工具的问题点是我们已经发现了内存泄漏。危害是堆上内存被耗尽，当我们想使用的时候就分配不出来，代码运行不下去被系统强行回收。

- 如何预防内存泄漏
- 如何解决内存泄漏
- 如何发现内存泄漏

分配时候计数器加1，释放时候计数器减1。当我们看到计数器不为0，正常退出的话计数器不等于0，可以大胆的推测出可能存在内存泄漏。
如何定位哪一行内存泄漏

- _FILE,*FUNCTION*,_LINE宏定位代码执行情况
- built_return_address() 返回函数被哪里调用的_libc_malloc()。
  破除递归的方法的代码需要注意。



![在这里插入图片描述](https://img-blog.csdnimg.cn/97bee5b5ada6427cae59f313420b606d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

**通过addr2line贡酒对指定地址进行代码行数定位，这个做法也是我第一次见，感觉开启了新世界的大门。**



## 2.第一种方法 文件做法

malloc时创建文件，free时删除文件，看程序最后剩下什么文件就可以直观的看到内存泄漏情况。

![在这里插入图片描述](https://img-blog.csdnimg.cn/23065953cd9641d29359f3277257635a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


可以通过cat命令，将内存泄漏的地址进行查看，再通过addr2line工具对泄漏位置进行定位。



## 3.第二种方法 宏定义

**别人的库有内存泄漏怎么办？**

只要它底层调用的malloc也是可行的。

## 4.第三种方法 指针方案（好用，不推荐）

在/usr/local/malloc.h
__malloc_hook是一个函数指针，当调用malloc时会调用这个函数指针。相对应的也有__free_hook这个函数指针。我们重新将指针赋一个新的地址，这样就有点类似于钩子的感觉，调用malloc/free时将被我们截获。

## 5.第四种方法mtrace工具 版本 纯操作

导入库 export MALLOC_TRACE=./test.log 否则日志文件无法生成。需要把分配释放的地址进行对应分析，然后找出落单的，再用add2line工具计算出具体的泄漏行数，和上面三种方法有异曲同工之妙。

![在这里插入图片描述](https://img-blog.csdnimg.cn/01b642f5577c45dc94f92714bf44b9ae.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)



## 6.总结

首先，不要认为内存泄漏这个问题复杂，核心主要两个问题。如果发现内存泄漏，发现泄漏找到具体出错位置，对错误进行排查。虽然认真学了一上午，感觉还是有些收获，但是在代码层面上还需要进一步加强。从另一个角度来说，尽然C++11封装了智能指针，还是应该充分的使用智能指针，要不然内存泄漏真的是让人头痛的一个事啊！

## 7.补充

**共享内存mmap shmget**

用户空间内专门有一个地方所谓的共享内存，在task_struct结构体中的mm_struct结构体中，堆栈，代码段数据段，参数环境变量等等都整洁的排布着。进程初始化时将各个数值进行赋，mmap_base是共享内存的初始值。值得注意的是，共享内存不是在堆栈上，是有一块虚拟的空间，不同的进程映射在相同的内存空间。有点匿名管道的意思。

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604975478