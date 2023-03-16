# 【NO.471】进程原理及系统调用

# 1.进程概念

操作系统作为硬件的使用层，提供使用硬件资源的能力，进程作为操作系统使用层， 提供使用操作系统抽象出的资源层的能力。
进程：是指计算机中已运行的程序。进程本身不是基本的运行单位，而是线程的容器。 程序本身只是指令、数据及其组织形式的描述，进程才是程序（那些指令和数据）的真正运行实例。

- 进程是处于执行时期的程序。
- 内核调度的对象是线程，不是进程。

## 1.1 进程四要素与线程区别

1. 有一段程序代其执行;
2. 有进程专用的系统堆栈空间;
3. 在内核有task_struct数据结构;
4. **进程有独立的存储空间,拥有专有的用户空间;**
   如果具备前面三条而缺少第四条,那就称为"线程"。如果完全没有用户空间,那就称为"内核线程";如果共享用户空间则就称为"用户线程"。

# 2.进程生命周期

Linux操作系统属于多任务操作系统，系统中的每个进程能够分时复用CPU时间片，通过有效的进程调度策略实现多任务并行执行。而进程在被CPU调度运行，等待CPU资源分配以及等待外部事件时会属于不同的状态。进程之间的状态关系：
运行 ：该进程此刻正在执行。
等待：进程能够运行，但没有得到许可，因为CPU分配给另一个进程。调度器可以在 下一次任务切换时选择该进程。
睡眠 ：进程正在睡眠无法运行，因为它在等待一个外部事件。调度器无法在下一次任 务切换时选择该进程。

【进程状态之间转换】如下:

![在这里插入图片描述](https://img-blog.csdnimg.cn/a643215a5be440bd9e09c201745b2d93.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_19,color_FFFFFF,t_70,g_se,x_16)



# 3.task_struct数据结构

Linux内核涉及进程和程序的所有算法都围绕一个名为task_struct的数据结构建立，该结构定义在include/linux/sched.h中。这是系统中主要的一个结构。在阐述调度器的实现之前，了解一下Linux管理进程的方式是很有必要的。 task_struct包含很多成员，将进程与各个内核子系统联系，task_struct定义如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b7337c7fd3f942d3b61bfdc705501e0e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)





![请添加图片描述](https://img-blog.csdnimg.cn/33bb87ab5efa43d09170ccbbdaf2ea2c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



# 4.进程优先级

并非所有进程都具有相同的重要性。除了大多数我们所熟悉的进程优先级之外，进程 还有不同的关键度类别，以满足不同需求。首先进行比较粗糙的划分，进程可以分为实时进程 和非实时进程（普通进程）。 实时进程优先级（0-99）都比普通 进程的优先级（100-139）高。当系统中有实时进程 运行时，普通进程几乎无法分到赶时间片（只能分到5%的CPU时间）。

# 5.进程系统调用

讨论fork和exec系列系统调用的实现。通常这些调用不是由应用程序直接发出的，而是通过一个中间层调用，即负责与内核通信的C标准库。从用户状态切换到核心态的方法，依不同的体系结构而各有不同。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d007403518c0471387834decddb72619.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



## 5.1 进程复制

传统的UNIX中用于复制进程的系统调用是fork。但它并不是Linux为此实现的唯一调用， 实际上Linux实现了3个
(1) fork是重量级调用，因为它建立了父进程的一个完整副本，然后作为子进程执行。 为减少与该调用相关的工作量，Linux使用了写时复制（copy-on-write）技术。
(2) vfork类似于fork，但并不创建父进程数据的副本。相反，父子进程之间共享数据。 这节省了大量CPU时间（如果一个进程操纵共享数据，则另一个会自动注意到）。
(3) clone产生线程，可以对父子进程之间的共享、复制进行精确控制。

![在这里插入图片描述](https://img-blog.csdnimg.cn/87c83d42042847ce9293862a0656ee08.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


只有在不得不复制数据内容时才去复制数据内容,这就是写时复制核心思想,可以看到因为修改页Z导致子进程不得不去复制原页Z来保证父子进程互不干扰
**内核只为新生成的子进程创建虚拟空间结构,它们来复制于父进程的虚拟结构,但是不为这些分配物理内存,它们共享父进程的物理空间,当父进程中有更改相应段的行为发生时,再为子进程相应段分配物理空间**



**【写时复制】**
内核使用了写时复制（Copy-On-Write，COW）技术，以防止在fork执行时将父进程的所有数据 复制到子进程。在调用fork时，内核通常对父进程的每个内存页，都为子进程创建一个相同的副本。
**【执行系统调用】**
fork、vfork和clone系统调用的入口点分别是sys_fork、sys_vfork和sys_clone函数。其定义依赖于 具体的体系结构，因为在用户空间和内核空间之间传递参数的方法因体系结构而异。
**【do_fork实现】**
所有3个fork机制最终都调用kernel/fork.c中的do_fork（一个体系结构无关的函数），其代码流程 如图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4a513dbefc3847c4ae089f6c0ba0f15f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)





![在这里插入图片描述](https://img-blog.csdnimg.cn/f4bbd2d0977b4182be7bcb4bf42f5d03.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



## 5.2 内核线程

内核线程是直接由内核本身启动的进程。内核线程实际上是将内核函数委托给独立的进程，与系统中其他进程“并行”执行（实际上，也并行于内核自身的执行）。内核线程经常称之为（内核）守护进程。它们用于执行下列任务。
• 周期性地将修改的内存页与页来源块设备同步（例如，使用mmap的文件映射）。
• 如果内存页很少使用，则写入交换区。
• 管理延时动作（deferred action）。
• 实现文件系统的事务日志。

## 5.3 退出进程

进程必须用exit系统调用终止。这使得内核有机会将该进程使用的资源释放回系统。见kernel/exit.c------>do_exit。简而言之， 该函数的实现就是将各个引用计数器减1，如果引用计数器归0而 没有进程再使用对应的结构，那么将相应的内存区域返还给内存管理模块。

![在这里插入图片描述](https://img-blog.csdnimg.cn/252e58f205ac4a2cbf87bca9ca078a9f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)

原文作者：[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605746202