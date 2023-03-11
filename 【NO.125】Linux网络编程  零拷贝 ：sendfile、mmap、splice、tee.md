# 【NO.125】Linux网络编程 | 零拷贝 ：sendfile、mmap、splice、tee

## 1.传统文件传输的问题

在网络编程中，如果我们想要提供文件传输的功能，最简单的方法就是用read将数据从磁盘上的文件中读取出来，再将其用write写入到socket中，通过网络协议发送给客户端。

```text
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

但是就是这两个简单的操作，却带来了大量的性能丢失

例如我们的服务器需要为客户端提供一个下载操作，此时的操作如下

![img](https://pic2.zhimg.com/80/v2-d5bcafefde2ebf48d0b0c3efbb3c6f51_720w.webp)

从上图可以看出，虽然仅仅只有这两行代码，但是却在发生了**四次用户态和内核态的上下文切换**，以及**四次数据拷贝**，也就是在这个地方产生了大量不必要的损耗。

那么为什么会发生这些操作呢？

**上下文切换**

由于read和recv是系统调用，所以每次调用该函数我们都需要从用户态切换至内核态，等待内核完成任务后再从内核态切换回用户态。

**数据拷贝**

上面也说了，由于数据的读取与写入都是由系统进行的，那么我们就得将数据从用户的缓冲区中拷贝到内核，

- 第一次拷贝：将磁盘中的数据拷贝到内核的缓冲区中
- 第二次拷贝：内核将数据处理完，接着拷贝到用户缓冲区中
- 第三次拷贝：此时需要通过socket将数据发送出去，将用户缓冲区中的数据拷贝至内核中socket的缓冲区中
- 第四次拷贝：把内核中socket缓冲区的数据拷贝到网卡的缓冲区中，通过网卡将数据发送出去。

所以要想优化传输性能，就要从**减少数据拷贝和用户态内核态的上下文切换**下手，这也就是**零拷贝**技术的由来。

## 2.**什么是零拷贝呢？**

**零拷贝的主要任务就是避免CPU将数据从一块存储中拷贝到另一块存储**，主要就是利用各种技术，避免让CPU做大量的数据拷贝任务，以此减少不必要的拷贝。或者借助其他的一些组件来完成简单的数据传输任务，让CPU解脱出来专注别的任务，使得系统资源的利用更加有效

Linux中实现零拷贝的方法主要有以下几种，下面一一对其进行介绍

1. sendfile
2. mmap
3. splice
4. tee

## 3.sendfile

sendfile函数的作用是直接在两个文件描述符之间传递数据。由于整个操作完全在内核中（直接从内核缓冲区拷贝到socket缓冲区），从而避免了内核缓冲区和用户缓冲区之间的数据拷贝。

需要注意的是，in_fd必须是一个支持类似mmap函数的文件描述符，不能是socket或者管道，而out_fd必须是一个socket，由此可见sendfile是专门为了在网络上传输文件而实现的函数。

![img](https://pic3.zhimg.com/80/v2-0e9a420edd62f49ddcc0241a984a04f6_720w.webp)

```text
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

> 参数：
> out_fd : 待写入内容的文件描述符
> in_fd : 待读出内容的文件描述符
> offset : 文件的偏移量
> count : 需要传输的字节数
> 返回值：
> 成功：返回传输的字节数
> 失败：返回-1并设置errno

## 4.**mmap**

mmap用于申请一段内存空间，也就是我们在进程间通信中提到过的**共享内存**，通过将内核缓冲区的数据映射到用户空间中，两者通过共享缓冲区直接访问统一资源，此时内核与用户空间就不需要再进行任何的数据拷贝操作了

![img](https://pic4.zhimg.com/80/v2-13655c6ca2a0154f7d75fd348251e287_720w.webp)

其中mmap用于申请空间，额munmap用于释放这段空间。

```text
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
int munmap(void *addr, size_t length);
```

参数：

addr : 内存的起始地址，如果设置为空则系统会自动分配

length : 指定内存段的长度

prot : 内存段的访问权限，通过按位与或可以取以下几种值

![img](https://pic3.zhimg.com/80/v2-40c76a8d03204163345541e5b166640a_720w.webp)

flag : 选项

![img](https://pic3.zhimg.com/80/v2-92610c23cd71caf6cee15fdceea1a356_720w.webp)

fd : 被映射文件对应的文件描述符

offset : 文件的偏移量

返回值：

成功：成功时返回指向内存区域的指针

失败：返回MAP_FAILED并设置errno

## 5.**splice**

splice函数用于**在两个文件描述符之间移动数据**，而不需要数据在内核空间和用户空间中来回拷贝

需要注意的是，使用splice函数时fd_in和fd_out**至少有一个是管道文件描述符，即**

```text
#include <fcntl.h>
ssize_t splice(int fd_in, loff_t *off_in, int fd_out,
               loff_t *off_out, size_t len, unsigned int flags);
```

参数：

out_fd : 待写入内容的文件描述符

off_out : 待写入文件描述符的偏移量，如果文件描述符为管道则必须为空

in_fd : 待读出内容的文件描述符

off_in : 待读出文件描述符的偏移量，如果文件描述符为管道则必须为空

len : 需要复制的字节数

flags : 选项

![img](https://pic1.zhimg.com/80/v2-27e32c19c51d98b2392b064a5821c450_720w.webp)

返回值：

成功：返回在两个文件描述符之间复制的字节数

没有数据：返回0

失败：返回-1并设置errno

可能产生的errno

![img](https://pic1.zhimg.com/80/v2-f31d8051fb9e4ce9374caf76b9e64f64_720w.webp)

## 6.**tee**

tee函数用于**在两个管道文件描述符之间复制数据**，并且它是直接复制，不会将数据读出，所以源文件上的数据仍可以用于后面的读操作

```text
#include <fcntl.h>
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
```

参数：

out_fd : 待写入内容的文件描述符

in_fd : 待读出内容的文件描述符

len : 需要复制的字节数

flags : 选项

返回值：

成功：返回在两个文件描述符之间复制的字节数

没有数据：返回0

原文地址：https://zhuanlan.zhihu.com/p/592397046

作者：linux