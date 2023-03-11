# 【NO.178】操作系统与存储：解析Linux内核全新异步IO引擎io_uring设计与实现

## **0.引言**

存储场景中，我们对性能的要求非常高。在存储引擎底层的IO技术选型时，可能会有如下讨论关于IO的讨论。

> http://davmac.org/davpage/linux/async-io.html
> So from the above documentation, it seems that Linux doesn't have a true async file I/O that is not blocking (AIO, Epoll or POSIX AIO are all broken in some ways). I wonder if tlinux has any remedy. We should reach out to tlinux experts to get their opinions.

看完这段话，读者可能会有如下的问题。

1. 这是在讨论什么，为何会有此番讨论？
2. 有没有更好的解决方案？
3. 更好的解决方案是通过怎样的设计和实现解决问题？
4. ...

2019年，Linux Kernel正式进入5.x时代，众多新特性中，与存储领域相关度最高的便是最新的IO引擎——io_uring。从一些性能测试的结论来看，io_uring性能远高于native AIO方式，带来了巨大的性能提升，这对当前异步IO领域也是一个big news。

1. 对于问题1，本文简述了Linux过往的的IO发展历程，同步IO接口、原生异步IO接口AIO的缺陷，为何原有方式存在缺陷。
2. 对于问题2，本文从设计的角度出发，介绍了最新的IO引擎io_uring的相关内容。
3. 对于问题3，本文深入最新版内核linux-5.10中解析了io_uring的大体实现（关键数据结构、流程、特性实现等）。
4. ...

## 1.**一切过往，皆为序章**

以史为镜，可以知兴替。我们先看看现存过往IO接口的缺陷。

### 1.1 过往同步IO接口

当今Linux对文件的操作有很多种方式，过往同步IO接口从功能上划分，大体分为如下几种。

- 原始版本
- offset版本
- 向量版本
- offset+向量版本

#### 1.1.1 **read，write**

最原始的文件IO系统调用就是read，write

read系统调用从文件描述符所指代的打开文件中读取数据。

read简单介绍：

```
NAME
    read - read from a file descriptor
SYNOPSIS
    #include <unistd.h>

    ssize_t read(int fd, void *buf, size_t count);
DESCRIPTION
    read() attempts to read up to count bytes from file descriptor fd
    into the buffer starting at buf.
    
    On files that support seeking, the read operation commences at the
    file offset, and the file offset is incremented by the number of
    bytes read.  If the file offset is at or past the end of file, no
    bytes are read, and read() returns zero.
    
    If count is zero, read() may detect the errors described below.  In
    the absence of any errors, or if read() does not check for errors, a
    read() with a count of 0 returns zero and has no other effects.
    
    According to POSIX.1, if count is greater than SSIZE_MAX, the result
    is implementation-defined; see NOTES for the upper limit on Linux.
```

write系统调用将数据写入一个已打开的文件中。

write简单介绍：

```
NAME
    write - write to a file descriptor
SYNOPSIS
    #include <unistd.h>
    
    ssize_t write(int fd, const void *buf, size_t count);
DESCRIPTION
    write() writes up to count bytes from the buffer starting at buf to
    the file referred to by the file descriptor fd.
    
    The number of bytes written may be less than count if, for example,
    there is insufficient space on the underlying physical medium, or the
    RLIMIT_FSIZE resource limit is encountered (see setrlimit(2)), or the
    call was interrupted by a signal handler after having written less
    than count bytes.  (See also pipe(7).)
    
    For a seekable file (i.e., one to which lseek(2) may be applied, for
    example, a regular file) writing takes place at the file offset, and
    the file offset is incremented by the number of bytes actually
    written.  If the file was open(2)ed with O_APPEND, the file offset is
    first set to the end of the file before writing.  The adjustment of
    the file offset and the write operation are performed as an atomic
    step.
    
    POSIX requires that a read(2) that can be proved to occur after a
    write() has returned will return the new data.  Note that not all
    filesystems are POSIX conforming.
    
    According to POSIX.1, if count is greater than SSIZE_MAX, the result
    is implementation-defined; see NOTES for the upper limit on Linux.
```

#### 1.1.2 **在文件特定偏移处的IO：pread，pwrite**

在多线程环境下，为了保证线程安全，需要保证下列操作的原子性。

```
    off_t orig;
    orig = lseek(fd, 0, SEEK_CUR); // Save current offset
    lseek(fd, offset, SEEK_SET);
    s = read(fd, buf, len);
    lseek(fd, orig, SEEK_SET); // Restore original file offset
```

让使用者来保证原子性较繁，从接口上就有保证是一个好的选择，后来出现的pread便实现了这一点。

与read, write类似，pread, pwrite调用时可以指定位置进行文件IO操作，而非始于文件的当前偏移处，且他们不会改变文件的当前偏移量。这种方式，减少了编码，并提高了代码的健壮性。

pread、pwrite简单介绍：

```
NAME
       pread,  pwrite  -  read from or write to a file descriptor at a given
       offset
SYNOPSIS
       #include <unistd.h>

       ssize_t pread(int fd, void *buf, size_t count, off_t offset);

       ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
       
DESCRIPTION
       pread() reads up to count bytes from file descriptor fd at offset
       offset (from the start of the file) into the buffer starting at buf.
       The file offset is not changed.

       pwrite() writes up to count bytes from the buffer starting at buf to
       the file descriptor fd at offset offset.  The file offset is not
       changed.

       The file referenced by fd must be capable of seeking.
```

当然，往read，write接口参数的标志位集合中加入新标志，用以表征新逻辑，可能达到相同的效果，但是这可能不够优雅——如果某个参数有多种可能的值，而函数内又以条件表达式检查这些参数值，并根据不同参数值做出不同的行为，那么以明确函数取代参数（Replace Parameter with Explicit Methods）也是一种合适的重构手法。

如果需要反复执行lseek，并伴之以文件IO，那么pread和pwrite系统调用在某些情况下是具有性能优势的。这是因为执行单个pread或pwrite系统调用的成本要低于执行lseek和read/write两个系统调用（当然，相对地，执行实际IO的开销通常要远大于执行系统调用，系统调用的性能优势作用有限）。历史上，一些数据库，通过使用kernel的这一新接口，获得了不菲的收益。如PostgreSQL：[*[PATCH\] Using pread instead of lseek (with analysis)*](https://www.postgresql-archive.org/PATCH-Using-pread-instead-of-lseek-with-analysis-td2215257.html)

#### 1.1.3 **分散输入和集中输出（Scatter-Gather IO）：readv, writev**

“物质的组成与结构决定物质的性质，性质决定用途，用途体现性质。”是自然科学的重要思想，在计算机科学中也是如此。现有计算机体系结构下，数据存储由一个或多个基本单元组成，物理、逻辑上的结构，决定了数据存储的性质——可能是连续的，也可能是不连续的。

对于不连续的数据的处理相对较繁，例如，使用read将数据读到不连续的内存，使用write将不连续的内存发送出去。更具体地看，如果要从文件中读一片连续的数据至进程的不同区域，有两种方案：

1. 使用read一次将它们读至一个较大的缓冲区中，然后将它们分成若干部分复制到不同的区域。
2. 调用read若干次分批将它们读至不同区域。

同样地，如果想将程序中不同区域的数据块连续地写至文件，也必须进行类似的处理。而且这种方案需要多次调用read、write系统调用，有损性能。

那么如何简化编程，如何解决这种开销呢？一种有效的解法就是使用特定的数据结构对非连续的数据进行管理，批量传输数据。从接口上就有此保证是一个好的选择，后来出现的readv，writev便实现了这一点。

这种基于向量的，分散输入和集中输出的系统调用并非只对单个缓冲区进行读写操作，而是一次即可传输多个缓冲区的数据，免除了多次系统调用的开销。该机制使用一个数组iov定义了一组用来传输数据的缓冲区，一个整形数iovcnt指定iov的成员个数，其中，iov中的每个成员都是如下形式的数据结构。

```
struct iovec {
   void  *iov_base;    /* Starting address */
   size_t iov_len;     /* Number of bytes to transfer */
};
```

**功能交集：preadv，pwritev**

上述两种功能都是一种进步，不过似乎格格不入，那么是否能合二为一，进两步呢？

数学上，集合是指具有某种特定性质的具体的或抽象的对象汇总而成的集体。其中，构成集合的这些对象则称为该集合的元素。我这里将接口定义成一种集合，一种特定功能就是其中的一个元素。根据已知有限集构造一个子集，该子集对于每一个元素要么包含要么不包含，那么根据乘法原理，这个子集共有2^N 种构造方式，即有2^N个子集。这么多可能的集合，显然较繁。基于场景对于功能子集的需求、元素之间的容斥、集合中元素是否需要有序（接口层面对功能的表现）、简约性等因素，我们会确立一些优雅的接口，这也是函数接口设计的一个哲学话题。

后来出现的preadv，pwritev，便是偏移和向量的交集，也是一种在排列组合的巨大可能性下确立的少部分简约的接口。

**带标志位集合的IO：preadv2，pwritev2**

再后来，还出现了变种函数preadv2和pwritev2，相比较preadv，pwritev，v2版本还能设置本次IO的标志，比如RWF_DSYNC、RWF_HIPRI、RWF_SYNC、RWF_NOWAIT、RWF_APPEND。

readv、preadv、preadv2系列简单介绍：

```
NAME
    readv,  writev,  preadv,  pwritev,  preadv2, pwritev2 - read or write
       data into multiple buffers

SYNOPSIS
    #include <sys/uio.h>

   ssize_t readv(int fd, const struct iovec *iov, int iovcnt);

   ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

   ssize_t preadv(int fd, const struct iovec *iov, int iovcnt,
                  off_t offset);

   ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt,
                   off_t offset);

   ssize_t preadv2(int fd, const struct iovec *iov, int iovcnt,
                   off_t offset, int flags);

   ssize_t pwritev2(int fd, const struct iovec *iov, int iovcnt,
                    off_t offset, int flags);

DESCRIPTION
   The readv() system call reads iovcnt buffers from the file associated
       with the file descriptor fd into the buffers described by iov
       ("scatter input").

       The writev() system call writes iovcnt buffers of data described by
       iov to the file associated with the file descriptor fd ("gather
       output").

       The pointer iov points to an array of iovec structures, defined in
       <sys/uio.h> as:

           struct iovec {
               void  *iov_base;    /* Starting address */
               size_t iov_len;     /* Number of bytes to transfer */
           };

       The readv() system call works just like read(2) except that multiple
       buffers are filled.

       The writev() system call works just like write(2) except that multi‐
       ple buffers are written out.

       Buffers are processed in array order.  This means that readv() com‐
       pletely fills iov[0] before proceeding to iov[1], and so on.  (If
       there is insufficient data, then not all buffers pointed to by iov
       may be filled.)  Similarly, writev() writes out the entire contents
       of iov[0] before proceeding to iov[1], and so on.

       The data transfers performed by readv() and writev() are atomic: the
       data written by writev() is written as a single block that is not in‐
       termingled with output from writes in other processes (but see
       pipe(7) for an exception); analogously, readv() is guaranteed to read
       a contiguous block of data from the file, regardless of read opera‐
       tions performed in other threads or processes that have file descrip‐
       tors referring to the same open file description (see open(2)).

   preadv() and pwritev()
       The preadv() system call combines the functionality of readv() and
       pread(2).  It performs the same task as readv(), but adds a fourth
       argument, offset, which specifies the file offset at which the input
       operation is to be performed.

       The pwritev() system call combines the functionality of writev() and
       pwrite(2).  It performs the same task as writev(), but adds a fourth
       argument, offset, which specifies the file offset at which the output
       operation is to be performed.

       The file offset is not changed by these system calls.  The file re‐
       ferred to by fd must be capable of seeking.

   preadv2() and pwritev2()
       These system calls are similar to preadv() and pwritev() calls, but
       add a fifth argument, flags, which modifies the behavior on a per-
       call basis.

       Unlike preadv() and pwritev(), if the offset argument is -1, then the
       current file offset is used and updated.

       The flags argument contains a bitwise OR of zero or more of the fol‐
       lowing flags:

       RWF_DSYNC (since Linux 4.7)
              Provide a per-write equivalent of the O_DSYNC open(2) flag.
              This flag is meaningful only for pwritev2(), and its effect
              applies only to the data range written by the system call.

       RWF_HIPRI (since Linux 4.6)
              High priority read/write.  Allows block-based filesystems to
              use polling of the device, which provides lower latency, but
              may use additional resources.  (Currently, this feature is us‐
              able only on a file descriptor opened using the O_DIRECT
              flag.)

       RWF_SYNC (since Linux 4.7)
              Provide a per-write equivalent of the O_SYNC open(2) flag.
              This flag is meaningful only for pwritev2(), and its effect
              applies only to the data range written by the system call.

       RWF_NOWAIT (since Linux 4.14)
              Do not wait for data which is not immediately available.  If
              this flag is specified, the preadv2() system call will return
              instantly if it would have to read data from the backing stor‐
              age or wait for a lock.  If some data was successfully read,
              it will return the number of bytes read.  If no bytes were
              read, it will return -1 and set errno to EAGAIN.  Currently,
              this flag is meaningful only for preadv2().

       RWF_APPEND (since Linux 4.16)
              Provide a per-write equivalent of the O_APPEND open(2) flag.
              This flag is meaningful only for pwritev2(), and its effect
              applies only to the data range written by the system call.
              The offset argument does not affect the write operation; the
              data is always appended to the end of the file.  However, if
              the offset argument is -1, the current file offset is updated.
```

### 1.2 同步IO接口的缺陷

上述接口，尽管形式多样，但它们都有一个共同的特征，就是同步，即在读写IO时，系统调用会阻塞住等待，在数据读取或写入后才返回结果。

对于传统、普通的编程模型，这类同步接口编程简单，结果可预测，倒也无妨，但是在要求高效的场景下，同步导致的后果就是caller在阻塞的同时无法继续执行其他的操作，只能等待IO结果返回，其实caller本可以利用这段时间继续往后执行。例如，一个FTP server，接收到客户机上传的文件，然后将文件写入到本机的过程中，若FTP服务程序忙于等待文件读写结果的返回，则会拒绝其他此刻正需要连接的客户机请求。在这种场景下，更好的方式是采用异步编程模型，就上述例子而言，当服务器接收到某个客户机上传文件后，直接、无阻塞地将写入IO的buffer提交给内核，然后caller继续接受下一个客户请求，内核处理完IO之后，主动调用某种通知机制，告诉caller该IO已完成，完成状态保存在某位置。

存储场景中，我们对性能的要求非常高，所以我们需要异步IO。

### 1.3 AIO

后来，应这类诉求，产生了异步IO接口，即Linux Native异步IO——AIO。

AIO接口简单介绍（表格引用自*Understanding Nginx Modules Development and Architecture Resolving(Second Edition)*）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvasa9lQclibbU8QD0Gq8Jib16GKicX3CuSeI3AKVeqH4VnILf1PJWuHIQMhsovNDRib0bbED8cBDQwFYrg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

类似地，如同前文所提PostgreSQL——历史上，也有一些项目通过使用kernel的新接口，获得了不菲的收益。

例如，高性能服务器nginx就使用了这样的机制，nginx把读取文件的操作异步地提交给内核后，内核会通知IO设备独立地执行操作，这样，nginx进程可以继续充分地占用CPU，而且，当大量读事件堆积到IO设备的队列中时，将会发挥出内核中“电梯算法”的优势，从而降低随机读取磁盘扇区的成本。

### 1.4 AIO的缺陷

但是，AIO仍然不够完美，同样存在很多缺陷，同样以nginx为例，目前，nginx仅支持在读取文件时使用AIO，因为正常写入文件往往是写入内存就立刻返回，效率很高，而如果替换成AIO写入速度会明显下降。

这是因为AIO不支持缓存操作，即使需要操作的文件块在linux文件缓存中存在，也不会通过操作缓存中的文件块来代替实际对磁盘的操作，这可能降低实际处理的性能。需要看具体的使用场景，如果大部分用户请求对文件操作都会落到文件缓存中，那么使用AIO可能不是一个好的选择。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvasa9lQclibbU8QD0Gq8Jib16GkJPRD77tIhdrIRp6DNEqU3uyCv1Dbw9V8dVMBlCfBBJJYoOYA2fcxw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

以上是AIO的不足之一，分析AIO缘何不足，需要较大的篇幅，这里按下不表直接总结一下其他不足之处。

- **仅支持direct IO**。在采用AIO的时候，只能使用O_DIRECT，不能借助文件系统缓存来缓存当前的IO请求，还存在size对齐（直接操作磁盘，所有写入内存块数量必须是文件系统块大小的倍数，而且要与内存页大小对齐。）等限制，这直接影响了aio在很多场景的使用。
- **仍然可能被阻塞。语义不完备**。即使应用层主观上，希望系统层采用异步IO，但是客观上，有时候还是可能会被阻塞。io_getevents(2)调用read_events读取AIO的完成events，read_events中的wait_event_interruptible_hrtimeout等待aio_read_events，如果条件不成立（events未完成）则调用__wait_event_hrtimeout进入睡眠（当然，支持用户态设置最大等待时间）。
- **拷贝开销大**。每个IO提交需要拷贝64+8字节，每个IO完成需要拷贝32字节，总共104字节的拷贝。这个拷贝开销是否可以承受，和单次IO大小有关：如果需要发送的IO本身就很大，相较之下，这点消耗可以忽略，而在大量小IO的场景下，这样的拷贝影响比较大。
- **API不友好**。每一个IO至少需要两次系统调用才能完成（submit和wait-for-completion)，需要非常小心地使用完成事件以避免丢事件。
- **系统调用开销大**。也正是因为上一条，io_submit/io_getevents造成了较大的系统调用开销，在存在spectre/meltdown（CPU熔断幽灵漏洞，CVE-2017-5754）的机器上，若要避免漏洞问题，则系统调用性能会大幅下降。所以在存储场景下，高频系统调用的性能影响较大。

在过去的数年间，针对上述限制的很多改进努力都未尽如人意。

终于，全新的异步IO引擎io_uring就在这样的环境下诞生了。

## 2.**设计——应该是什么样子**

既然是全新实现，我们是否可以不囿于现状，思考它应该是什么样子？

关于“应该是什么样子”，我曾听智超兄说过这样的一句话：“Linux应该是什么样子，它现在就是什么样子。”，这并不是类似于“存在即合理”这样的谬传，而是对Linux系统优雅哲学的高度概括，同时也是对开源自由软件精神的肯定——因为自始至终都是自由的，所以大家觉得应该是什么样子（哪里有缺陷，哪里不够优雅），大家就会自由地去修改它，所以，经过时代的发展，它的面貌与大家所期望的最相符，即众人拾柴，众望所归。

以后世上可能会有无数文章讲述io_uring是什么样子，我们先看看它应该是什么样子。

### 2.1 设计原则

如上所述，历史实现在一定场景下，会有一定问题，新实现理应反思问题、解决问题。与此同时，需要遵循一定设计原则，如下是若干原则。

- 简单：接口需要足够简单，这一点不言自明。
- 易用：同时需要足够克制，保持易于理解，就不容易误用，对于使用者来说，这是一种有效的助推（之所以如上没有采用“简单易用”这样的惯用语，是因为简单并不一定意味着易用。我们尽量避免这种不合逻辑的隐喻）。
- 可扩展：接口要有足够的扩展性，尽管某个接口是为了某种场景（如存储）而建立，但是我们需要面向未来，若有朝一日需要支持非阻塞设备（非块存储）以及网络I/O时，这里不应是桎梏。
- 特性丰富：当然，接口需要支持足够丰富的功能。
- 高效：在存储场景下，高效率始终是关键目标。
- 可伸缩性：满足峰值场景的性能需要（高效和低延迟很重要，但是峰值速率对于存储设备来讲也很重要）底层软件是基于硬件建构的，为了适应新硬件的要求，接口还需要考虑到伸缩性。

另外，因为我们的部分目标之间，本质上往往是存在一定互斥性的（如可伸缩与足够简单互斥、特性丰富与高效互斥）很难同时满足，所以，我们设计时也需要权衡。其中，io_uring始终需要围绕高效进行设计。

### 2.2 实现思路

#### 2.2.1 **解决“系统调用开销大”的问题**

针对这个问题，考虑是否每次都需要系统调用。如果能将多次系统调用中的逻辑放到有限次数中来，就能将消耗降为常数时间复杂度。

#### 2.2.2 **解决“拷贝开销大”的问题**

之所以在提交和完成事件中存在大量的内存拷贝，是因为应用程序和内核之间的通信需要拷贝数据，所以为了避免这个问题，需要重新考量应用与内核间的通信方式。我们发现，两者通信，不是必须要拷贝，通过现有技术，可以让应用与内核共享内存，用于彼此通信，需要生产者-消费者模型。

要实现核外与内核的零拷贝，最佳方式就是实现一块内存映射区域，两者共享一段内存，核外往这段内存写数据，然后通知内核使用这段内存数据，或者内核填写这段数据，核外使用这部分数据。因此，我们需要一对共享的ring buffer用于应用程序和内核之间的通信。

共享ring buffer的设计主要带来以下几个好处：

- 提交、完成请求时节省应用和内核之间的内存拷贝
- 使用SQPOLL高级特性时，应用程序无需调用系统调用
- 无锁操作，用memory ordering实现同步，通过几个简单的头尾指针的移动就可以实现快速交互。

一块用于核外传递数据给内核，一块是内核传递数据给核外，一方只读，一方只写。

- 提交队列SQ(submission queue)中，应用是IO提交的生产者，内核是消费者。
- 完成队列CQ(completion queue)中，内核是IO完成的生产者，应用是消费者。

内核控制SQ ring的head和CQ ring的tail，应用程序控制SQ ring的tail和CQ ring的head

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvasa9lQclibbU8QD0Gq8Jib16GAAbF0n5q3joFXXS6dUonrrXyZVqqEeVfcHwYIZdCia80NLzbMeUia2RQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

那么他们分别需要保存的是什么数据呢？

假设A缓存区为核外写，内核读，就是将IO数据写到这个缓存区，然后通知内核来读；再假设B缓存区为内核写，核外读，他所承担的责任就是返回完成状态，标记A缓存区的其中一个entry的完成状态为成功或者失败等信息。

#### 2.2.3 **解决“API不友好”的问题**

问题在于需要多个系统调用才能完成，考虑是否可以把多个系统调用合而为一。

你可能会想到，这与上文所说的重构手法相悖，即以明确函数取代参数（Replace Parameter with Explicit Methods）——如果某个参数有多种可能的值，而函数内又以条件表达式检查这些参数值，并根据不同参数值做出不同的行为。

然而，手法只是手法，选择具体的重构手法需要遵循重构原则。在不同场景下，可能事实恰恰相反——令函数携带参数（Parameterize Method）可能是一个好的选择。

话说天下大势，分久必合，合久必分。你可能会发现这样的两个函数，它们做着类似的工作，但因少数几个值致使行为略有不同。在这种情况下，你可以将这些各自分离的函数统一起来，并通过参数来处理那些变化情况，用以简化问题。这样的修改可以去除重复代码，并提高灵活性，因为你可以用这个参数处理更多的变化情况。

也许你会发现，你无法用这种办法处理整个函数，但可以处理函数中的一部分代码。这种情况下，你应该首先将这部分代码提炼到一个独立函数中，然后再对那个提炼所得的函数使用令函数携带参数（Parameterize Method）。

2.3 实现——现在是什么样子

推导完了应该是什么样子，解析一下现在是什么样子。

#### 2.2.4 **关键数据结构**

程序等于数据结构加算法，这里先解析io_uring有哪些关键数据结构。

**io_uring、io_rings结构**

结构前面是一些标志位集合和掩码，尾部是一个柔性数组。这两个数据在前面使用mmap分配内存的时候，对应到了不同的offset，即前面IORING_OFF_SQ_RING、IORING_OFF_CQ_RING和IORING_OFF_SQES的预定于的值。

其中io_rings结构中sq, cq成员，分别代表了提交的请求的ring和已经完成的请求返回结构的ring。io_uring结构中是head和tail，用于控制队列中的头尾索引。即前文提到的，内核控制SQ ring的head和CQ ring的tail，应用程序控制SQ ring的tail和CQ ring的head。

```
struct io_uring {
 u32 head ____cacheline_aligned_in_smp;
 u32 tail ____cacheline_aligned_in_smp;
};

/*
 * This data is shared with the application through the mmap at offsets
 * IORING_OFF_SQ_RING and IORING_OFF_CQ_RING.
 *
 * The offsets to the member fields are published through struct
 * io_sqring_offsets when calling io_uring_setup.
 */
struct io_rings {
 /*
  * Head and tail offsets into the ring; the offsets need to be
  * masked to get valid indices.
  *
  * The kernel controls head of the sq ring and the tail of the cq ring,
  * and the application controls tail of the sq ring and the head of the
  * cq ring.
  */
 struct io_uring  sq, cq;
 /*
  * Bitmasks to apply to head and tail offsets (constant, equals
  * ring_entries - 1)
  */
 u32   sq_ring_mask, cq_ring_mask;
 /* Ring sizes (constant, power of 2) */
 u32   sq_ring_entries, cq_ring_entries;
 /*
  * Number of invalid entries dropped by the kernel due to
  * invalid index stored in array
  *
  * Written by the kernel, shouldn't be modified by the
  * application (i.e. get number of "new events" by comparing to
  * cached value).
  *
  * After a new SQ head value was read by the application this
  * counter includes all submissions that were dropped reaching
  * the new SQ head (and possibly more).
  */
 u32   sq_dropped;
 /*
  * Runtime SQ flags
  *
  * Written by the kernel, shouldn't be modified by the
  * application.
  *
  * The application needs a full memory barrier before checking
  * for IORING_SQ_NEED_WAKEUP after updating the sq tail.
  */
 u32   sq_flags;
 /*
  * Runtime CQ flags
  *
  * Written by the application, shouldn't be modified by the
  * kernel.
  */
 u32                     cq_flags;
 /*
  * Number of completion events lost because the queue was full;
  * this should be avoided by the application by making sure
  * there are not more requests pending than there is space in
  * the completion queue.
  *
  * Written by the kernel, shouldn't be modified by the
  * application (i.e. get number of "new events" by comparing to
  * cached value).
  *
  * As completion events come in out of order this counter is not
  * ordered with any other data.
  */
 u32   cq_overflow;
 /*
  * Ring buffer of completion events.
  *
  * The kernel writes completion events fresh every time they are
  * produced, so the application is allowed to modify pending
  * entries.
  */
 struct io_uring_cqe cqes[] ____cacheline_aligned_in_smp;
};
```

**Submission Queue Entry单元数据结构**

Submission Queue（下称SQ）是提交队列，核外写内核读的地方。Submission Queue Entry（下称SQE），即提交队列中的条目，队列由一个个条目组成。

描述一个SQE会复杂很多，不仅是因为要描述更多的信息，也是因为可扩展性这一设计原则。

我们需要操作码、标志集合、关联文件描述符、地址、偏移量，另外地，可能还需要表示优先级。

```
/*
 * IO submission data structure (Submission Queue Entry)
 */
struct io_uring_sqe {
 __u8 opcode;  /* type of operation for this sqe */
 __u8 flags;  /* IOSQE_ flags */
 __u16 ioprio;  /* ioprio for the request */
 __s32 fd;  /* file descriptor to do IO on */
 union {
  __u64 off; /* offset into file */
  __u64 addr2;
 };
 union {
  __u64 addr; /* pointer to buffer or iovecs */
  __u64 splice_off_in;
 };
 __u32 len;  /* buffer size or number of iovecs */
 union {
  __kernel_rwf_t rw_flags;
  __u32  fsync_flags;
  __u16  poll_events; /* compatibility */
  __u32  poll32_events; /* word-reversed for BE */
  __u32  sync_range_flags;
  __u32  msg_flags;
  __u32  timeout_flags;
  __u32  accept_flags;
  __u32  cancel_flags;
  __u32  open_flags;
  __u32  statx_flags;
  __u32  fadvise_advice;
  __u32  splice_flags;
 };
 __u64 user_data; /* data to be passed back at completion time */
 union {
  struct {
   /* pack this to avoid bogus arm OABI complaints */
   union {
    /* index into fixed buffers, if used */
    __u16 buf_index;
    /* for grouped buffer selection */
    __u16 buf_group;
   } __attribute__((packed));
   /* personality to use, if used */
   __u16 personality;
   __s32 splice_fd_in;
  };
  __u64 __pad2[3];
 };
};
```

- opcode是操作码，例如IORING_OP_READV，代表向量读。
- flags是标志位集合。
- ioprio是请求的优先级，对于普通的读写，具体定义可以参照ioprio_set(2)，
- fd是这个请求相关的文件描述符
- off是操作的偏移量
- addr表示这次IO操作执行的地址，如果操作码opcode描述了一个传输数据的操作，这个操作是基于向量的，addr就指向struct iovec的数组首地址，这和前文所说的preadv系统调用是一样的用法；如果不是基于向量的，那么addr必须直接包含一个地址，len这里（非向量场景）就表示这段buffer的长度，而向量场景就表示iovec的数量。
- 接下来的是一个union，表示一系列针对特定操作码opcode的一些flag。例如，对于上文所提的IORING_OP_READV，随后的flags就遵循preadv2系统调用。
- user_data是各操作码opcode通用的，内核并未染指，仅仅只是拷贝给完成事件completion event
- 结构的最后用于内存对齐，对齐到64字节，为了更丰富的特性，未来这个请求结构应该会包含更多的内容。

这就是核外往内核填写的Submission Queue Entry的数据结构，准备好这样的一个数据结构，将它写到对应的sqes所在的内存位置，然后再通知内核去对应的位置取数据，这样就完成了一次数据交接。

**Completion Queue Entry单元数据结构**

Completion Queue（下称CQ）是完成队列，内核写核外读的地方。Completion Queue Entry（下称CQE），即完成队列中的条目，队列由一个个条目组成。

描述一个CQE就简单得多。

```
/*
 * IO completion data structure (Completion Queue Entry)
 */
struct io_uring_cqe {
 __u64 user_data; /* sqe->data submission passed back */
 __s32 res;  /* result code for this event */
 __u32 flags;
};
```

- user_data就是sqe发送时核外填写的，只不过在完成时回传而已，一个常见的用例就是作为一个指针，指向原始请求。从submission queue到completion queue，内核不会动这里面的数据。
- res用来保存最终的这个sqe的执行结果，就是这个event的返回码，可以认为是系统调用的返回值，表示成功或失败等。如果接口成功的话返回传输的字节数，如果失败的话，就是错误码。如果错误发生，res就等于-EIO。
- flags是标志位集合。如果flags设置为IORING_CQE_F_BUFFER，则前16位是buffer ID（调用链：io_uring_enter -> io_iopoll_check -> io_iopoll_getevents -> io_do_iopoll -> io_iopoll_complete -> io_put_rw_kbuf -> io_put_kbuf，最终会调用io_put_kbuf，如代码所示）。

```
/*
 * cqe->flags
 *
 * IORING_CQE_F_BUFFER If set, the upper 16 bits are the buffer ID
 */
#define IORING_CQE_F_BUFFER  (1U << 0)

enum {
 IORING_CQE_BUFFER_SHIFT  = 16,
};
static unsigned int io_put_kbuf(struct io_kiocb *req, struct io_buffer *kbuf)
{
 unsigned int cflags;

 cflags = kbuf->bid << IORING_CQE_BUFFER_SHIFT;
 cflags |= IORING_CQE_F_BUFFER;
 req->flags &= ~REQ_F_BUFFER_SELECTED;
 kfree(kbuf);
 return cflags;
}
```

**上下文结构io_ring_ctx**

前面介绍了SQE/CQE等关键的数据结构，他们是用来承载数据流的关键部分，有了数据流的关键数据结构我们还需要一个上下文数据结构，用于整个io_uring控制流。这就是io_ring_ctx，贯穿整个io_uring所有过程的数据结构，基本上在任何位置只需要你能持有该结构就可以找到任何数据所在的位置，例如，sq_sqes就是指向io_uring_sqe结构的指针，指向SQEs的首地址。

```
struct io_ring_ctx {
 struct {
  struct percpu_ref refs;
 } ____cacheline_aligned_in_smp;

 struct {
  unsigned int  flags;
  unsigned int  compat: 1;
  unsigned int  limit_mem: 1;
  unsigned int  cq_overflow_flushed: 1;
  unsigned int  drain_next: 1;
  unsigned int  eventfd_async: 1;
  unsigned int  restricted: 1;

  /*
   * Ring buffer of indices into array of io_uring_sqe, which is
   * mmapped by the application using the IORING_OFF_SQES offset.
   *
   * This indirection could e.g. be used to assign fixed
   * io_uring_sqe entries to operations and only submit them to
   * the queue when needed.
   *
   * The kernel modifies neither the indices array nor the entries
   * array.
   */
  u32   *sq_array;
  unsigned  cached_sq_head;
  unsigned  sq_entries;
  unsigned  sq_mask;
  unsigned  sq_thread_idle;
  unsigned  cached_sq_dropped;
  unsigned  cached_cq_overflow;
  unsigned long  sq_check_overflow;

  struct list_head defer_list;
  struct list_head timeout_list;
  struct list_head cq_overflow_list;

  wait_queue_head_t inflight_wait;
  struct io_uring_sqe *sq_sqes;
 } ____cacheline_aligned_in_smp;

 struct io_rings *rings;

 /* IO offload */
 struct io_wq  *io_wq;

 /*
  * For SQPOLL usage - we hold a reference to the parent task, so we
  * have access to the ->files
  */
 struct task_struct *sqo_task;

 /* Only used for accounting purposes */
 struct mm_struct *mm_account;

#ifdef CONFIG_BLK_CGROUP
 struct cgroup_subsys_state *sqo_blkcg_css;
#endif

 struct io_sq_data *sq_data; /* if using sq thread polling */

 struct wait_queue_head sqo_sq_wait;
 struct wait_queue_entry sqo_wait_entry;
 struct list_head sqd_list;

 /*
  * If used, fixed file set. Writers must ensure that ->refs is dead,
  * readers must ensure that ->refs is alive as long as the file* is
  * used. Only updated through io_uring_register(2).
  */
 struct fixed_file_data *file_data;
 unsigned  nr_user_files;

 /* if used, fixed mapped user buffers */
 unsigned  nr_user_bufs;
 struct io_mapped_ubuf *user_bufs;

 struct user_struct *user;

 const struct cred *creds;

#ifdef CONFIG_AUDIT
 kuid_t   loginuid;
 unsigned int  sessionid;
#endif

 struct completion ref_comp;
 struct completion sq_thread_comp;

 /* if all else fails... */
 struct io_kiocb  *fallback_req;

#if defined(CONFIG_UNIX)
 struct socket  *ring_sock;
#endif

 struct idr  io_buffer_idr;

 struct idr  personality_idr;

 struct {
  unsigned  cached_cq_tail;
  unsigned  cq_entries;
  unsigned  cq_mask;
  atomic_t  cq_timeouts;
  unsigned long  cq_check_overflow;
  struct wait_queue_head cq_wait;
  struct fasync_struct *cq_fasync;
  struct eventfd_ctx *cq_ev_fd;
 } ____cacheline_aligned_in_smp;

 struct {
  struct mutex  uring_lock;
  wait_queue_head_t wait;
 } ____cacheline_aligned_in_smp;

 struct {
  spinlock_t  completion_lock;

  /*
   * ->iopoll_list is protected by the ctx->uring_lock for
   * io_uring instances that don't use IORING_SETUP_SQPOLL.
   * For SQPOLL, only the single threaded io_sq_thread() will
   * manipulate the list, hence no extra locking is needed there.
   */
  struct list_head iopoll_list;
  struct hlist_head *cancel_hash;
  unsigned  cancel_hash_bits;
  bool   poll_multi_file;

  spinlock_t  inflight_lock;
  struct list_head inflight_list;
 } ____cacheline_aligned_in_smp;

 struct delayed_work  file_put_work;
 struct llist_head  file_put_llist;

 struct work_struct  exit_work;
 struct io_restriction  restrictions;
};
```

#### **2.2.5  关键流程**

数据结构定义好了，逻辑实现具体是如何驱动这些数据结构的呢？使用上，大体分为准备、提交、收割过程。

有几个io_uring相关的系统调用：

```
#include <linux/io_uring.h>

int io_uring_setup(u32 entries, struct io_uring_params *p);

int io_uring_enter(unsigned int fd, unsigned int to_submit,
                   unsigned int min_complete, unsigned int flags,
                   sigset_t *sig);
                   
int io_uring_register(unsigned int fd, unsigned int opcode,
                      void *arg, unsigned int nr_args);
```

下面分析关键流程。

**io_uring准备阶段**

io_uring通过io_uring_setup完成准备阶段。

```
int io_uring_setup(u32 entries, struct io_uring_params *p);
```

io_uring_setup系统调用的过程就是初始化相关数据结构，建立好对应的缓存区，然后通过系统调用的参数io_uring_params结构传递回去，告诉核外环内存地址在哪，起始指针的地址在哪等关键的信息。

需要初始化内存的内存分为三个区域，分别是SQ，CQ，SQEs。内核初始化SQ和CQ，此外，提交请求在SQ，CQ之间有一个间接数组，即内核提供了一个Submission Queue Entries（SQEs）数组。之所以额外采用了一个数组保存SQEs，是为了方便通过环形缓冲区提交内存上不连续的请求。SQ和CQ中每个节点保存的都是SQEs数组的索引，而不是实际的请求，实际的请求只保存在SQEs数组中。这样在提交请求时，就可以批量提交一组SQEs上不连续的请求。

通常，SQE被独立地使用，意味着它的执行不影响在ring中的连续SQE条目。它允许全面、灵活的操作，并且使它们最高性能地并行执行完成。一个顺序的使用案例就是数据的整体写入。它的一个通常的例子就是一系列写，随之的是fsync/fdatasync，应用通常转变成程序同步-等待操作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvasa9lQclibbU8QD0Gq8Jib16GCgEc4Twct3f5E0OaWI2gCwjc00chtKHu8JrsW8YYfkIvclSNUKvQeA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

先从参数上来解析

- 核外需要告诉io_uring_setup提交的整个缓存区数组的大小。（代表 queue depth？），这里就是entries参数。

- params这个参数从IO的角度看有两种，一种是输入参数，一种是输出参数。

- - sq_entries是输出参数，由内核填充，让应用程序知道这个ring支持多少SQE。
  - 类似地，cq_entries告诉应用程序，CQ ring有多大。
  - sq_off和cq_off分别是io_sqring_offsets和io_cqring_offsets结构，是内核与核外的约定，分别描述了SQ和CQ的指针在mmap中的offset
  - 其他的结构成员涉及到高级用法，暂时按下不表。
  - 比如params->flags，这个成员变量是用来设置当前整个io_uring 的标志的，它将决定是否启动sq_thread，是否采用iopoll模式等等
  - sq_thread_cpu、sq_thread_idle也由用户设置，用来指定io_sq_thread内核线程CPU、idle时间。
  - 一部分属于输入参数，是用户设置、核外传递给核外的，用于定义io_uring在内核中的行为，这些都是在创建阶段就决定了的。
  - 还有一部分属于输出参数，由内核设置（io_uring_create）、传递数据到核外的，核外根据这些数据来使用mmap分配内存，初始化一些数据结构。

```
/*
 * Passed in for io_uring_setup(2). Copied back with updated info on success
 */
struct io_uring_params {
 __u32 sq_entries;
 __u32 cq_entries;
 __u32 flags;
 __u32 sq_thread_cpu;
 __u32 sq_thread_idle;
 __u32 features;
 __u32 wq_fd;
 __u32 resv[3];
 struct io_sqring_offsets sq_off;
 struct io_cqring_offsets cq_off;
};

/*
 * io_uring_params->features flags
 */
#define IORING_FEAT_SINGLE_MMAP  (1U << 0)
#define IORING_FEAT_NODROP  (1U << 1)
#define IORING_FEAT_SUBMIT_STABLE (1U << 2)
#define IORING_FEAT_RW_CUR_POS  (1U << 3)
#define IORING_FEAT_CUR_PERSONALITY (1U << 4)
#define IORING_FEAT_FAST_POLL  (1U << 5)
#define IORING_FEAT_POLL_32BITS  (1U << 6)
```

再从实现上来解析，如下为io_uring_setup代码。

```
/*
 * Sets up an aio uring context, and returns the fd. Applications asks for a
 * ring size, we return the actual sq/cq ring sizes (among other things) in the
 * params structure passed in.
 */
static long io_uring_setup(u32 entries, struct io_uring_params __user *params)
{
 struct io_uring_params p;
 int i;

 if (copy_from_user(&p, params, sizeof(p)))
  return -EFAULT;
 for (i = 0; i < ARRAY_SIZE(p.resv); i++) {
  if (p.resv[i])
   return -EINVAL;
 }

 if (p.flags & ~(IORING_SETUP_IOPOLL | IORING_SETUP_SQPOLL |
   IORING_SETUP_SQ_AFF | IORING_SETUP_CQSIZE |
   IORING_SETUP_CLAMP | IORING_SETUP_ATTACH_WQ |
   IORING_SETUP_R_DISABLED))
  return -EINVAL;

 return  io_uring_create(entries, &p, params);
}
```

经过标志位非法检查之后，关键是调用内部函数io_uring_create实现实例创建过程。

- 首先需要创建一个上下文结构io_ring_ctx用来管理整个会话。
- 随后实现SQ和CQ内存区的映射，使用IORING_OFF_CQ_RING偏移量，使用io_cqring_offsets结构的实例，即io_uring_params中cq_off这个成员，SQ使用IORING_OFF_SQES这个偏移量。
- 其余的是一些错误检查、权限检查、资源配额检查等检查逻辑。

```
static int io_uring_create(unsigned entriesstatic int io_uring_create(unsigned entries, struct io_uring_params *p,
      struct io_uring_params __user *params)
{
 struct user_struct *user = NULL;
 struct io_ring_ctx *ctx;
 bool limit_mem;
 int ret;

 if (!entries)
  return -EINVAL;
 if (entries > IORING_MAX_ENTRIES) {
  if (!(p->flags & IORING_SETUP_CLAMP))
   return -EINVAL;
  entries = IORING_MAX_ENTRIES;
 }

 /*
  * Use twice as many entries for the CQ ring. It's possible for the
  * application to drive a higher depth than the size of the SQ ring,
  * since the sqes are only used at submission time. This allows for
  * some flexibility in overcommitting a bit. If the application has
  * set IORING_SETUP_CQSIZE, it will have passed in the desired number
  * of CQ ring entries manually.
  */
 p->sq_entries = roundup_pow_of_two(entries);
 if (p->flags & IORING_SETUP_CQSIZE) {
  /*
   * If IORING_SETUP_CQSIZE is set, we do the same roundup
   * to a power-of-two, if it isn't already. We do NOT impose
   * any cq vs sq ring sizing.
   */
  if (!p->cq_entries)
   return -EINVAL;
  if (p->cq_entries > IORING_MAX_CQ_ENTRIES) {
   if (!(p->flags & IORING_SETUP_CLAMP))
    return -EINVAL;
   p->cq_entries = IORING_MAX_CQ_ENTRIES;
  }
  p->cq_entries = roundup_pow_of_two(p->cq_entries);
  if (p->cq_entries < p->sq_entries)
   return -EINVAL;
 } else {
  p->cq_entries = 2 * p->sq_entries;
 }

 user = get_uid(current_user());
 limit_mem = !capable(CAP_IPC_LOCK);

 if (limit_mem) {
  ret = __io_account_mem(user,
    ring_pages(p->sq_entries, p->cq_entries));
  if (ret) {
   free_uid(user);
   return ret;
  }
 }

 ctx = io_ring_ctx_alloc(p);
 if (!ctx) {
  if (limit_mem)
   __io_unaccount_mem(user, ring_pages(p->sq_entries,
        p->cq_entries));
  free_uid(user);
  return -ENOMEM;
 }
 ctx->compat = in_compat_syscall();
 ctx->user = user;
 ctx->creds = get_current_cred();
#ifdef CONFIG_AUDIT
 ctx->loginuid = current->loginuid;
 ctx->sessionid = current->sessionid;
#endif
 ctx->sqo_task = get_task_struct(current);

 /*
  * This is just grabbed for accounting purposes. When a process exits,
  * the mm is exited and dropped before the files, hence we need to hang
  * on to this mm purely for the purposes of being able to unaccount
  * memory (locked/pinned vm). It's not used for anything else.
  */
 mmgrab(current->mm);
 ctx->mm_account = current->mm;

#ifdef CONFIG_BLK_CGROUP
 /*
  * The sq thread will belong to the original cgroup it was inited in.
  * If the cgroup goes offline (e.g. disabling the io controller), then
  * issued bios will be associated with the closest cgroup later in the
  * block layer.
  */
 rcu_read_lock();
 ctx->sqo_blkcg_css = blkcg_css();
 ret = css_tryget_online(ctx->sqo_blkcg_css);
 rcu_read_unlock();
 if (!ret) {
  /* don't init against a dying cgroup, have the user try again */
  ctx->sqo_blkcg_css = NULL;
  ret = -ENODEV;
  goto err;
 }
#endif

 /*
  * Account memory _before_ installing the file descriptor. Once
  * the descriptor is installed, it can get closed at any time. Also
  * do this before hitting the general error path, as ring freeing
  * will un-account as well.
  */
 io_account_mem(ctx, ring_pages(p->sq_entries, p->cq_entries),
         ACCT_LOCKED);
 ctx->limit_mem = limit_mem;

 ret = io_allocate_scq_urings(ctx, p);
 if (ret)
  goto err;

 ret = io_sq_offload_create(ctx, p);
 if (ret)
  goto err;

 if (!(p->flags & IORING_SETUP_R_DISABLED))
  io_sq_offload_start(ctx);

 memset(&p->sq_off, 0, sizeof(p->sq_off));
 p->sq_off.head = offsetof(struct io_rings, sq.head);
 p->sq_off.tail = offsetof(struct io_rings, sq.tail);
 p->sq_off.ring_mask = offsetof(struct io_rings, sq_ring_mask);
 p->sq_off.ring_entries = offsetof(struct io_rings, sq_ring_entries);
 p->sq_off.flags = offsetof(struct io_rings, sq_flags);
 p->sq_off.dropped = offsetof(struct io_rings, sq_dropped);
 p->sq_off.array = (char *)ctx->sq_array - (char *)ctx->rings;

 memset(&p->cq_off, 0, sizeof(p->cq_off));
 p->cq_off.head = offsetof(struct io_rings, cq.head);
 p->cq_off.tail = offsetof(struct io_rings, cq.tail);
 p->cq_off.ring_mask = offsetof(struct io_rings, cq_ring_mask);
 p->cq_off.ring_entries = offsetof(struct io_rings, cq_ring_entries);
 p->cq_off.overflow = offsetof(struct io_rings, cq_overflow);
 p->cq_off.cqes = offsetof(struct io_rings, cqes);
 p->cq_off.flags = offsetof(struct io_rings, cq_flags);

 p->features = IORING_FEAT_SINGLE_MMAP | IORING_FEAT_NODROP |
   IORING_FEAT_SUBMIT_STABLE | IORING_FEAT_RW_CUR_POS |
   IORING_FEAT_CUR_PERSONALITY | IORING_FEAT_FAST_POLL |
   IORING_FEAT_POLL_32BITS;

 if (copy_to_user(params, p, sizeof(*p))) {
  ret = -EFAULT;
  goto err;
 }

 /*
  * Install ring fd as the very last thing, so we don't risk someone
  * having closed it before we finish setup
  */
 ret = io_uring_get_fd(ctx);
 if (ret < 0)
  goto err;

 trace_io_uring_create(ret, ctx, p->sq_entries, p->cq_entries, p->flags);
 return ret;
err:
 io_ring_ctx_wait_and_kill(ctx);
 return ret;
}
```

io_sqring_offsets、io_cqring_offsets等相关结构、标志位集合。

预定义offset
如果要表征分配的是io uring相关的一些内存，就需要预定义一些offset，如IORING_OFF_SQ_RING、IORING_OFF_SQES和IORING_OFF_CQ_RING，这些offset值定义了保存到这个三个结构保存到位置。这里mmap的时候，就使用了这些offset。

```
/*
 * Magic offsets for the application to mmap the data it needs
 */
#define IORING_OFF_SQ_RING  0ULL
#define IORING_OFF_CQ_RING  0x8000000ULL
#define IORING_OFF_SQES   0x10000000ULL

/*
 * Filled with the offset for mmap(2)
 */
struct io_sqring_offsets {
 __u32 head;
 __u32 tail;
 __u32 ring_mask;
 __u32 ring_entries;
 __u32 flags;
 __u32 dropped;
 __u32 array;
 __u32 resv1;
 __u64 resv2;
};

/*
 * sq_ring->flags
 */
#define IORING_SQ_NEED_WAKEUP (1U << 0) /* needs io_uring_enter wakeup */
#define IORING_SQ_CQ_OVERFLOW (1U << 1) /* CQ ring is overflown */

struct io_cqring_offsets {
 __u32 head;
 __u32 tail;
 __u32 ring_mask;
 __u32 ring_entries;
 __u32 overflow;
 __u32 cqes;
 __u32 flags;
 __u32 resv1;
 __u64 resv2;
};

/*
 * cq_ring->flags
 */

/* disable eventfd notifications */
#define IORING_CQ_EVENTFD_DISABLED (1U << 0)

/*
 * io_uring_enter(2) flags
 */
#define IORING_ENTER_GETEVENTS (1U << 0)
#define IORING_ENTER_SQ_WAKEUP (1U << 1)
#define IORING_ENTER_SQ_WAIT (1U << 2)

/*
 * io_uring_register(2) opcodes and arguments
 */
enum {
 IORING_REGISTER_BUFFERS   = 0,
 IORING_UNREGISTER_BUFFERS  = 1,
 IORING_REGISTER_FILES   = 2,
 IORING_UNREGISTER_FILES   = 3,
 IORING_REGISTER_EVENTFD   = 4,
 IORING_UNREGISTER_EVENTFD  = 5,
 IORING_REGISTER_FILES_UPDATE  = 6,
 IORING_REGISTER_EVENTFD_ASYNC  = 7,
 IORING_REGISTER_PROBE   = 8,
 IORING_REGISTER_PERSONALITY  = 9,
 IORING_UNREGISTER_PERSONALITY  = 10,
 IORING_REGISTER_RESTRICTIONS  = 11,
 IORING_REGISTER_ENABLE_RINGS  = 12,

 /* this goes last */
 IORING_REGISTER_LAST
};
```

具体的实践，可以参考如下liburing中的初始化函数io_uring_queue_init中对io_uring_setup的使用（http://git.kernel.dk/cgit/liburing/tree/src/setup.c）。

liburing中使用io_uring_setup的部分代码

```
/*
 * Returns -1 on error, or zero on success. On success, 'ring'
 * contains the necessary information to read/write to the rings.
 */
int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags)
{
 struct io_uring_params p;
 int fd, ret;

 memset(&p, 0, sizeof(p));
 p.flags = flags;

 fd = io_uring_setup(entries, &p);
 if (fd < 0)
  return fd;

 ret = io_uring_queue_mmap(fd, &p, ring);
 if (ret)
  close(fd);

 return ret;
}
```

注意mmap的时候需要传入MAP_POPULATE参数，为文件映射通过预读的方式准备好页表，随后对映射区的访问不会被page fault。

**IO提交**

在初始化完成之后，应用程序就可以使用这些队列来添加IO请求，即填充SQE。当请求都加入SQ后，应用程序还需要某种方式告诉内核，生产的请求待消费，这就是提交IO请求，可以通过io_uring_enter系统调用。

```
int io_uring_enter(unsigned int fd, unsigned int to_submit,
                   unsigned int min_complete, unsigned int flags,
                   sigset_t *sig);
```

内核将SQ中的请求提交给Block层。这个系统调用既能提交，也能等待。

具体的实现是找到一个空闲的SQE，根据请求设置SQE，并将这个SQE的索引放到SQ中。SQ是一个典型的ring buffer，有head，tail两个成员，如果head == tail，意味着队列为空。SQE设置完成后，需要修改SQ的tail，以表示向ring buffer中插入了一个请求。

先从参数上来解析

- fd即由io_uring_setup(2)返回的文件描述符，

- to_submit告诉内核待消费和提交的SQE的数量，表示一次提交多少个 IO，

- min_complete请求完成请求的个数。

- flags是修饰接口行为的标志集合，这里主要例举两个flags

- - 如果在io_uring_setup的时候flag设置了IORING_SETUP_SQPOLL，内核会额外启动一个特定的内核线程来执行轮询的操作，称作SQ线程，这里使用的轮询结构会最终对应到struct file_operations中的iopoll操作，这个操作作为一个新的接口在最近才添加到这里，Linux native aio的新功能也使用了这个iopoll。这里io _uring实际上只有vfs层的改动，其它的都是使用已经存在的东西，而且几个核心的东西和aio使用的相同/类似。直接通过访问相关的队列就可以获取到执行完的任务，不需要经过系统调用。关于这个线程，通过io_uring_params结构中的sq_thread_cpu配置，这个内核线程可以运行在某个指定的 CPU核心 上。这个内核线程会不停的 Poll SQ，直到在通过sq_thread_idle配置的时间内没有Poll到任何请求为止。
  - 如果flags中设置了IORING_ENTER_GETEVENTS，并且min_complete > 0，这个系统调用会一直 block，直到 min_complete 个 IO 已经完成才返回。这个系统调用会同时处理 IO 收割。
  - 另外的，IORING_SQ_NEED_WAKEUP可以表示在一些时候唤醒休眠中的轮询线程。

static int io_sq_thread(void *data)即内核轮询线程。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvasa9lQclibbU8QD0Gq8Jib16GgXCtyc2gibXmpBJAjCiaRfDI0XHanoP7RK9yGsk72TF2kWlzfm3Dr3lg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

同样地，可以用这个系统调用等待完成。除非应用程序，内核会直接修改CQ，因此调用io_uring_enter系统调用时不必使用IORING_ENTER_GETEVENTS，完成就可以被应用程序消费。

io_uring提供了submission offload模式，使得提交过程完全不需要进行系统调用。当程序在用户态设置完SQE，并通过修改SQ的tail完成一次插入时，如果此时SQ线程处于唤醒状态，那么可以立刻捕获到这次提交，这样就避免了用户程序调用io_uring_enter。如上所说，如果SQ线程处于休眠状态，则需要通过使用IORING_SQ_NEED_WAKEUP标志位调用io_uring_enter来唤醒SQ线程。

以io_iopoll_check为例，正常情况下执行路线是io_iopoll_check -> io_iopoll_getevents -> io_do_iopoll -> (kiocb->ki_filp->f_op->iopoll). 在完成请求的操作之后，会调用下面这个函数提交结果到cqe数组中，这样应用就能看到结果了。这里的io_cqring_fill_event就是获取一个目前可以写入到cqe，写入数据。这里最终调用的会是io_get_cqring，可以见就是返回目前tail的后面的一个。

更详细的内容可以直接参考io_uring_enter(2)的man page。

内核中io_uring_enter的相关代码如下。

```
SYSCALL_DEFINE6(io_uring_enter, unsigned int, fd, u32, to_submit,
  u32, min_complete, u32, flags, const sigset_t __user *, sig,
  size_t, sigsz)
{
 struct io_ring_ctx *ctx;
 long ret = -EBADF;
 int submitted = 0;
 struct fd f;

 io_run_task_work();

 if (flags & ~(IORING_ENTER_GETEVENTS | IORING_ENTER_SQ_WAKEUP |
   IORING_ENTER_SQ_WAIT))
  return -EINVAL;

 f = fdget(fd);
 if (!f.file)
  return -EBADF;

 ret = -EOPNOTSUPP;
 if (f.file->f_op != &io_uring_fops)
  goto out_fput;

 ret = -ENXIO;
 ctx = f.file->private_data;
 if (!percpu_ref_tryget(&ctx->refs))
  goto out_fput;

 ret = -EBADFD;
 if (ctx->flags & IORING_SETUP_R_DISABLED)
  goto out;

 /*
  * For SQ polling, the thread will do all submissions and completions.
  * Just return the requested submit count, and wake the thread if
  * we were asked to.
  */
 ret = 0;
 if (ctx->flags & IORING_SETUP_SQPOLL) {
  if (!list_empty_careful(&ctx->cq_overflow_list))
   io_cqring_overflow_flush(ctx, false, NULL, NULL);
  if (flags & IORING_ENTER_SQ_WAKEUP)
   wake_up(&ctx->sq_data->wait);
  if (flags & IORING_ENTER_SQ_WAIT)
   io_sqpoll_wait_sq(ctx);
  submitted = to_submit;
 } else if (to_submit) {
  ret = io_uring_add_task_file(ctx, f.file);
  if (unlikely(ret))
   goto out;
  mutex_lock(&ctx->uring_lock);
  submitted = io_submit_sqes(ctx, to_submit);
  mutex_unlock(&ctx->uring_lock);

  if (submitted != to_submit)
   goto out;
 }
 if (flags & IORING_ENTER_GETEVENTS) {
  min_complete = min(min_complete, ctx->cq_entries);

  /*
   * When SETUP_IOPOLL and SETUP_SQPOLL are both enabled, user
   * space applications don't need to do io completion events
   * polling again, they can rely on io_sq_thread to do polling
   * work, which can reduce cpu usage and uring_lock contention.
   */
  if (ctx->flags & IORING_SETUP_IOPOLL &&
      !(ctx->flags & IORING_SETUP_SQPOLL)) {
   ret = io_iopoll_check(ctx, min_complete);
  } else {
   ret = io_cqring_wait(ctx, min_complete, sig, sigsz);
  }
 }

out:
 percpu_ref_put(&ctx->refs);
out_fput:
 fdput(f);
 return submitted ? submitted : ret;
}
```

io_iopoll_complete实现

```
/*
 * Find and free completed poll iocbs
 */
static void io_iopoll_complete(struct io_ring_ctx *ctx, unsigned int *nr_events,
          struct list_head *done)
{
 struct req_batch rb;
 struct io_kiocb *req;
 LIST_HEAD(again);

 /* order with ->result store in io_complete_rw_iopoll() */
 smp_rmb();

 io_init_req_batch(&rb);
 while (!list_empty(done)) {
  int cflags = 0;

  req = list_first_entry(done, struct io_kiocb, inflight_entry);
  if (READ_ONCE(req->result) == -EAGAIN) {
   req->result = 0;
   req->iopoll_completed = 0;
   list_move_tail(&req->inflight_entry, &again);
   continue;
  }
  list_del(&req->inflight_entry);

  if (req->flags & REQ_F_BUFFER_SELECTED)
   cflags = io_put_rw_kbuf(req);

  __io_cqring_fill_event(req, req->result, cflags);
  (*nr_events)++;

  if (refcount_dec_and_test(&req->refs))
   io_req_free_batch(&rb, req);
 }

 io_commit_cqring(ctx);
 if (ctx->flags & IORING_SETUP_SQPOLL)
  io_cqring_ev_posted(ctx);
 io_req_free_batch_finish(ctx, &rb);

 if (!list_empty(&again))
  io_iopoll_queue(&again);
}
```

io_get_cqring实现

```
static struct io_uring_cqe *io_get_cqring(struct io_ring_ctx *ctx)
{
 struct io_rings *rings = ctx->rings;
 unsigned tail;

 tail = ctx->cached_cq_tail;
 /*
  * writes to the cq entry need to come after reading head; the
  * control dependency is enough as we're using WRITE_ONCE to
  * fill the cq entry
  */
 if (tail - READ_ONCE(rings->cq.head) == rings->cq_ring_entries)
  return NULL;

 ctx->cached_cq_tail++;
 return &rings->cqes[tail & ctx->cq_mask];
}
```

**IO收割**

来都来了，搞点事情吧，在我们提交IO的同时，使用同一个io_uring_enter系统调用就可以回收完成状态，这样的好处就是一次系统调用接口就完成了原本需要两次系统调用的工作，大大的减少了系统调用的次数，也就是减少了内核核外的切换，这是一个很明显的优化，内核与核外的切换极其耗时。

当IO完成时，内核负责将完成IO在SQEs中的index放到CQ中。由于IO在提交的时候可以顺便返回完成的IO，所以收割IO不需要额外系统调用。

如果使用了IORING_SETUP_SQPOLL参数，IO收割也不需要系统调用的参与。由于内核和用户态共享内存，所以收割的时候，用户态遍历[cring->head, cring->tail)区间，即已经完成的IO队列，然后找到相应的CQE并进行处理，最后移动head指针到tail，IO收割至此而终。

所以，在最理想的情况下，IO提交和收割都不需要使用系统调用。

### 2.3 高级特性

此外，我们可以使用一些优化思想，进行更进一步的优化，这些优化，以一种可选的方式成为io_uring的其它一些高级特性。

## 3.**Fixed Files模式**

### 3.1 优化思想

非关键逻辑上提至循环外，简化关键路径。

### 3.2 优化实现

可以调用io_uring_register系统调用，使用IORING_REGISTER_FILES操作码，将一组file注册到内核，最终调用io_sqe_files_register，这样内核在注册阶段就批量完成文件的一些基本操作（对于这组文件填充相应的数据结构fixed_file_data，其中fixed_file_table是维护的file表。内核态下，如何获得文件描述符获取相关的信息呢，就需要通过fget，根据fd号获得指向文件的struct file），之后的再次批量IO时就不需要重复地进行此类基本信息设置（更具体地，例如对文件进行fget/fput操作）。如果需要进行IO操作的文件相对固定（比如数据库日志），这会节省一定量的IO时间。

### 3.3 fixed_file_data结构

```
struct fixed_file_data {
 struct fixed_file_table  *table;
 struct io_ring_ctx  *ctx;

 struct fixed_file_ref_node *node;
 struct percpu_ref  refs;
 struct completion  done;
 struct list_head  ref_list;
 spinlock_t   lock;
};
```

### 3.4 io_sqe_files_register实现Fixed Files操作

```
static int io_sqe_files_register(struct io_ring_ctx *ctx, void __user *arg,
     unsigned nr_args)
{
 __s32 __user *fds = (__s32 __user *) arg;
 unsigned nr_tables, i;
 struct file *file;
 int fd, ret = -ENOMEM;
 struct fixed_file_ref_node *ref_node;
 struct fixed_file_data *file_data;

 if (ctx->file_data)
  return -EBUSY;
 if (!nr_args)
  return -EINVAL;
 if (nr_args > IORING_MAX_FIXED_FILES)
  return -EMFILE;

 file_data = kzalloc(sizeof(*ctx->file_data), GFP_KERNEL);
 if (!file_data)
  return -ENOMEM;
 file_data->ctx = ctx;
 init_completion(&file_data->done);
 INIT_LIST_HEAD(&file_data->ref_list);
 spin_lock_init(&file_data->lock);

 nr_tables = DIV_ROUND_UP(nr_args, IORING_MAX_FILES_TABLE);
 file_data->table = kcalloc(nr_tables, sizeof(*file_data->table),
       GFP_KERNEL);
 if (!file_data->table)
  goto out_free;

 if (percpu_ref_init(&file_data->refs, io_file_ref_kill,
    PERCPU_REF_ALLOW_REINIT, GFP_KERNEL))
  goto out_free;

 if (io_sqe_alloc_file_tables(file_data, nr_tables, nr_args))
  goto out_ref;
 ctx->file_data = file_data;

 for (i = 0; i < nr_args; i++, ctx->nr_user_files++) {
  struct fixed_file_table *table;
  unsigned index;

  if (copy_from_user(&fd, &fds[i], sizeof(fd))) {
   ret = -EFAULT;
   goto out_fput;
  }
  /* allow sparse sets */
  if (fd == -1)
   continue;

  file = fget(fd);
  ret = -EBADF;
  if (!file)
   goto out_fput;

  /*
   * Don't allow io_uring instances to be registered. If UNIX
   * isn't enabled, then this causes a reference cycle and this
   * instance can never get freed. If UNIX is enabled we'll
   * handle it just fine, but there's still no point in allowing
   * a ring fd as it doesn't support regular read/write anyway.
   */
  if (file->f_op == &io_uring_fops) {
   fput(file);
   goto out_fput;
  }
  table = &file_data->table[i >> IORING_FILE_TABLE_SHIFT];
  index = i & IORING_FILE_TABLE_MASK;
  table->files[index] = file;
 }

 ret = io_sqe_files_scm(ctx);
 if (ret) {
  io_sqe_files_unregister(ctx);
  return ret;
 }

 ref_node = alloc_fixed_file_ref_node(ctx);
 if (IS_ERR(ref_node)) {
  io_sqe_files_unregister(ctx);
  return PTR_ERR(ref_node);
 }

 file_data->node = ref_node;
 spin_lock(&file_data->lock);
 list_add_tail(&ref_node->node, &file_data->ref_list);
 spin_unlock(&file_data->lock);
 percpu_ref_get(&file_data->refs);
 return ret;
out_fput:
 for (i = 0; i < ctx->nr_user_files; i++) {
  file = io_file_from_index(ctx, i);
  if (file)
   fput(file);
 }
 for (i = 0; i < nr_tables; i++)
  kfree(file_data->table[i].files);
 ctx->nr_user_files = 0;
out_ref:
 percpu_ref_exit(&file_data->refs);
out_free:
 kfree(file_data->table);
 kfree(file_data);
 ctx->file_data = NULL;
 return ret;
}
```

## **4.Fixed Buffers模式**

### 4.1 优化思想

优化思想也是将非关键逻辑上提至循环外，简化关键路径。

### 4.2 优化实现

如果应用提交到内核的虚拟内存地址是固定的，那么可以提前完成虚拟地址到物理pages的映射，将这个并不是每次都要做的非关键路径从关键的IO 路径中剥离，避免每次I/O都进行转换，从而优化性能。可以在io_uring_setup之后，调用io_uring_register，使用IORING_REGISTER_BUFFERS 操作码，将一组buffer注册到内核（参数是一个指向iovec的数组，表示这些地址需要map到内核），最终调用io_sqe_buffer_register，这样内核在注册阶段就批量完成buffer的一些基本操作（减小get_user_pages、put_page开销，提前使用get_user_pages来获得userspace虚拟地址对应的物理pages，初始化在io_ring_ctx上下文中用于管理用户态buffer的io_mapped_ubuf数据结构，map/unmap，传递IOV的地址和长度等），之后的再次批量IO时就不需要重复地进行此类内存拷贝和基础信息检测。

在操作IO的时，如果需要进行IO操作的buffer相对固定，提交的虚拟地址曾经被注册过，那么可以使用带FIXED系列的opcode（IORING_OP_READ_FIXED/IORING_OP_WRITE_FIXED）IO，可以看到底层调用链：io_issue_sqe->io_read->io_import_iovec->__io_import_iovec->io_import_fixed，会直接使用已经完成的“成果”，如此就免去了虚拟地址到pages的转换，这会节省一定量的IO时间。

#### 4.2.1 **io_mapped_ubuf结构**

```
struct io_mapped_ubuf {
 u64  ubuf;
 size_t  len;
 struct  bio_vec *bvec;
 unsigned int nr_bvecs;
 unsigned long acct_pages;
};
```

#### **4.2.2 io_sqe_buffer_register实现Fixed Buffers操作**

```
static int io_sqe_buffer_register(struct io_ring_ctx *ctx, void __user *arg,
      unsigned nr_args)
{
 struct vm_area_struct **vmas = NULL;
 struct page **pages = NULL;
 struct page *last_hpage = NULL;
 int i, j, got_pages = 0;
 int ret = -EINVAL;

 if (ctx->user_bufs)
  return -EBUSY;
 if (!nr_args || nr_args > UIO_MAXIOV)
  return -EINVAL;

 ctx->user_bufs = kcalloc(nr_args, sizeof(struct io_mapped_ubuf),
     GFP_KERNEL);
 if (!ctx->user_bufs)
  return -ENOMEM;

 for (i = 0; i < nr_args; i++) {
  struct io_mapped_ubuf *imu = &ctx->user_bufs[i];
  unsigned long off, start, end, ubuf;
  int pret, nr_pages;
  struct iovec iov;
  size_t size;

  ret = io_copy_iov(ctx, &iov, arg, i);
  if (ret)
   goto err;

  /*
   * Don't impose further limits on the size and buffer
   * constraints here, we'll -EINVAL later when IO is
   * submitted if they are wrong.
   */
  ret = -EFAULT;
  if (!iov.iov_base || !iov.iov_len)
   goto err;

  /* arbitrary limit, but we need something */
  if (iov.iov_len > SZ_1G)
   goto err;

  ubuf = (unsigned long) iov.iov_base;
  end = (ubuf + iov.iov_len + PAGE_SIZE - 1) >> PAGE_SHIFT;
  start = ubuf >> PAGE_SHIFT;
  nr_pages = end - start;

  ret = 0;
  if (!pages || nr_pages > got_pages) {
   kvfree(vmas);
   kvfree(pages);
   pages = kvmalloc_array(nr_pages, sizeof(struct page *),
      GFP_KERNEL);
   vmas = kvmalloc_array(nr_pages,
     sizeof(struct vm_area_struct *),
     GFP_KERNEL);
   if (!pages || !vmas) {
    ret = -ENOMEM;
    goto err;
   }
   got_pages = nr_pages;
  }

  imu->bvec = kvmalloc_array(nr_pages, sizeof(struct bio_vec),
      GFP_KERNEL);
  ret = -ENOMEM;
  if (!imu->bvec)
   goto err;

  ret = 0;
  mmap_read_lock(current->mm);
  pret = pin_user_pages(ubuf, nr_pages,
          FOLL_WRITE | FOLL_LONGTERM,
          pages, vmas);
  if (pret == nr_pages) {
   /* don't support file backed memory */
   for (j = 0; j < nr_pages; j++) {
    struct vm_area_struct *vma = vmas[j];

    if (vma->vm_file &&
        !is_file_hugepages(vma->vm_file)) {
     ret = -EOPNOTSUPP;
     break;
    }
   }
  } else {
   ret = pret < 0 ? pret : -EFAULT;
  }
  mmap_read_unlock(current->mm);
  if (ret) {
   /*
    * if we did partial map, or found file backed vmas,
    * release any pages we did get
    */
   if (pret > 0)
    unpin_user_pages(pages, pret);
   kvfree(imu->bvec);
   goto err;
  }

  ret = io_buffer_account_pin(ctx, pages, pret, imu, &last_hpage);
  if (ret) {
   unpin_user_pages(pages, pret);
   kvfree(imu->bvec);
   goto err;
  }

  off = ubuf & ~PAGE_MASK;
  size = iov.iov_len;
  for (j = 0; j < nr_pages; j++) {
   size_t vec_len;

   vec_len = min_t(size_t, size, PAGE_SIZE - off);
   imu->bvec[j].bv_page = pages[j];
   imu->bvec[j].bv_len = vec_len;
   imu->bvec[j].bv_offset = off;
   off = 0;
   size -= vec_len;
  }
  /* store original address for later verification */
  imu->ubuf = ubuf;
  imu->len = iov.iov_len;
  imu->nr_bvecs = nr_pages;

  ctx->nr_user_bufs++;
 }
 kvfree(pages);
 kvfree(vmas);
 return 0;
err:
 kvfree(pages);
 kvfree(vmas);
 io_sqe_buffer_unregister(ctx);
 return ret;
}
```

## 5.**Polled IO模式**

### 5.1 优化思想

将较多的CPU时间放到重要的事情上，全速完成关键路径。

状态从未完成变成已完成，就需要对完成状态进行探测，很多时候，可以使用中断模型，也就是等待后端数据处理完毕之后，内核会发起一个SIGIO或eventfd的EPOLLIN状态提醒核外有数据已经完成了，可以开始处理。但是，中断其实是比较耗时的，如果是高IOPS的场景，就会不停地中断，中断开销就得不偿失。

我们可以更激进一些，让内核采用Polled IO模式收割块设备层请求。这在一定的程度上加速了IO，这在追求低延时和高IOPS的应用场景非常有用。

### 5.2 优化实现

io_uring_enter通过正确设置IORING_ENTER_GETEVENTS，IORING_SETUP_IOPOLL等flag（如下代码设置IORING_SETUP_IOPOLL并且不设置IORING_SETUP_SQPOLL，即没有使用SQ线程）调用io_iopoll_check。

```
SYSCALL_DEFINE6(io_uring_enter, unsigned int, fd, u32, to_submit,
  u32, min_complete, u32, flags, const sigset_t __user *, sig,
  size_t, sigsz)
{
 struct io_ring_ctx *ctx;
 long ret = -EBADF;
 int submitted = 0;
 struct fd f;

 io_run_task_work();

 if (flags & ~(IORING_ENTER_GETEVENTS | IORING_ENTER_SQ_WAKEUP |
   IORING_ENTER_SQ_WAIT))
  return -EINVAL;

 f = fdget(fd);
 if (!f.file)
  return -EBADF;

 ret = -EOPNOTSUPP;
 if (f.file->f_op != &io_uring_fops)
  goto out_fput;

 ret = -ENXIO;
 ctx = f.file->private_data;
 if (!percpu_ref_tryget(&ctx->refs))
  goto out_fput;

 ret = -EBADFD;
 if (ctx->flags & IORING_SETUP_R_DISABLED)
  goto out;

 /*
  * For SQ polling, the thread will do all submissions and completions.
  * Just return the requested submit count, and wake the thread if
  * we were asked to.
  */
 ret = 0;
 if (ctx->flags & IORING_SETUP_SQPOLL) {
  if (!list_empty_careful(&ctx->cq_overflow_list))
   io_cqring_overflow_flush(ctx, false, NULL, NULL);
  if (flags & IORING_ENTER_SQ_WAKEUP)
   wake_up(&ctx->sq_data->wait);
  if (flags & IORING_ENTER_SQ_WAIT)
   io_sqpoll_wait_sq(ctx);
  submitted = to_submit;
 } else if (to_submit) {
  ret = io_uring_add_task_file(ctx, f.file);
  if (unlikely(ret))
   goto out;
  mutex_lock(&ctx->uring_lock);
  submitted = io_submit_sqes(ctx, to_submit);
  mutex_unlock(&ctx->uring_lock);

  if (submitted != to_submit)
   goto out;
 }
 if (flags & IORING_ENTER_GETEVENTS) {
  min_complete = min(min_complete, ctx->cq_entries);

  /*
   * When SETUP_IOPOLL and SETUP_SQPOLL are both enabled, user
   * space applications don't need to do io completion events
   * polling again, they can rely on io_sq_thread to do polling
   * work, which can reduce cpu usage and uring_lock contention.
   */
  if (ctx->flags & IORING_SETUP_IOPOLL &&
      !(ctx->flags & IORING_SETUP_SQPOLL)) {
   ret = io_iopoll_check(ctx, min_complete);
  } else {
   ret = io_cqring_wait(ctx, min_complete, sig, sigsz);
  }
 }

out:
 percpu_ref_put(&ctx->refs);
out_fput:
 fdput(f);
 return submitted ? submitted : ret;
}
```

io_iopoll_check开始poll核外程序可以不停的轮询需要的完成事件数量min_complete，循环内主要调用io_iopoll_getevents。

```
static int io_iopoll_check(struct io_ring_ctx *ctx, long min)
{
 unsigned int nr_events = 0;
 int iters = 0, ret = 0;

 /*
  * We disallow the app entering submit/complete with polling, but we
  * still need to lock the ring to prevent racing with polled issue
  * that got punted to a workqueue.
  */
 mutex_lock(&ctx->uring_lock);
 do {
  /*
   * Don't enter poll loop if we already have events pending.
   * If we do, we can potentially be spinning for commands that
   * already triggered a CQE (eg in error).
   */
  if (io_cqring_events(ctx, false))
   break;

  /*
   * If a submit got punted to a workqueue, we can have the
   * application entering polling for a command before it gets
   * issued. That app will hold the uring_lock for the duration
   * of the poll right here, so we need to take a breather every
   * now and then to ensure that the issue has a chance to add
   * the poll to the issued list. Otherwise we can spin here
   * forever, while the workqueue is stuck trying to acquire the
   * very same mutex.
   */
  if (!(++iters & 7)) {
   mutex_unlock(&ctx->uring_lock);
   io_run_task_work();
   mutex_lock(&ctx->uring_lock);
  }

  ret = io_iopoll_getevents(ctx, &nr_events, min);
  if (ret <= 0)
   break;
  ret = 0;
 } while (min && !nr_events && !need_resched());

 mutex_unlock(&ctx->uring_lock);
 return ret;
}
```

io_iopoll_getevents调用io_do_iopoll。

```
/*
 * Poll for a minimum of 'min' events. Note that if min == 0 we consider that a
 * non-spinning poll check - we'll still enter the driver poll loop, but only
 * as a non-spinning completion check.
 */
static int io_iopoll_getevents(struct io_ring_ctx *ctx, unsigned int *nr_events,
    long min)
{
 while (!list_empty(&ctx->iopoll_list) && !need_resched()) {
  int ret;

  ret = io_do_iopoll(ctx, nr_events, min);
  if (ret < 0)
   return ret;
  if (*nr_events >= min)
   return 0;
 }

 return 1;
}
```

io_do_iopoll中的kiocb->ki_filp->f_op->iopoll，即blkdev_iopoll，不断地轮询探测确认提交给Block层的请求的完成状态，直到足够数量的IO完成。

```
static int io_do_iopoll(struct io_ring_ctx *ctx, unsigned int *nr_events,
   long min)
{
 struct io_kiocb *req, *tmp;
 LIST_HEAD(done);
 bool spin;
 int ret;

 /*
  * Only spin for completions if we don't have multiple devices hanging
  * off our complete list, and we're under the requested amount.
  */
 spin = !ctx->poll_multi_file && *nr_events < min;

 ret = 0;
 list_for_each_entry_safe(req, tmp, &ctx->iopoll_list, inflight_entry) {
  struct kiocb *kiocb = &req->rw.kiocb;

  /*
   * Move completed and retryable entries to our local lists.
   * If we find a request that requires polling, break out
   * and complete those lists first, if we have entries there.
   */
  if (READ_ONCE(req->iopoll_completed)) {
   list_move_tail(&req->inflight_entry, &done);
   continue;
  }
  if (!list_empty(&done))
   break;

  ret = kiocb->ki_filp->f_op->iopoll(kiocb, spin);
  if (ret < 0)
   break;

  /* iopoll may have completed current req */
  if (READ_ONCE(req->iopoll_completed))
   list_move_tail(&req->inflight_entry, &done);

  if (ret && spin)
   spin = false;
  ret = 0;
 }

 if (!list_empty(&done))
  io_iopoll_complete(ctx, nr_events, &done);

 return ret;
}
```

块设备层相关file_operations。

```
const struct file_operations def_blk_fops = {
 .open  = blkdev_open,
 .release = blkdev_close,
 .llseek  = block_llseek,
 .read_iter = blkdev_read_iter,
 .write_iter = blkdev_write_iter,
 .iopoll  = blkdev_iopoll,
 .mmap  = generic_file_mmap,
 .fsync  = blkdev_fsync,
 .unlocked_ioctl = block_ioctl,
#ifdef CONFIG_COMPAT
 .compat_ioctl = compat_blkdev_ioctl,
#endif
 .splice_read = generic_file_splice_read,
 .splice_write = iter_file_splice_write,
 .fallocate = blkdev_fallocate,
};
```

当使用POLL IO时，大多数CPU时间花费在blkdev_iopoll上。即全速完成关键路径。

```
static int blkdev_iopoll(struct kiocb *kiocb, bool wait)
{
 struct block_device *bdev = I_BDEV(kiocb->ki_filp->f_mapping->host);
 struct request_queue *q = bdev_get_queue(bdev);

 return blk_poll(q, READ_ONCE(kiocb->ki_cookie), wait);
}
```

### 5.3 Kernel Side Polling

IORING_SETUP_SQPOLL，当前应用更新SQ并填充一个新的SQE，内核线程sq_thread会自动完成提交，这样应用无需每次调用io_uring_enter系统调用来提交IO。应用可通过IORING_SETUP_SQ_AFF和sq_thread_cpu绑定特定的CPU。

实际机器上，不仅有高IOPS场景，还有些场景的IOPS有些时间段会非常低。为了节省无IO场景的CPU开销，一段时间空闲，该内核线程可以自动睡眠。核外在下发新的IO时，通过IORING_ENTER_SQ_WAKEUP唤醒该内核线程。

### 5.4 小结

如上可见，内核提供了足够多的选择，不同的方案有着不同角度的优化方向，这些优化方案可以自行组合。通过合理地使用，可以使io_uring 全速运转。

## 6.**io_uring用户态库liburing**

正如前文所说，简单并不一定意味着易用——io_uring的接口足够简单，但是相对于这种简单，操作上需要手动mmap来映射内存，稍显复杂。为了更方便地使用io_uring，原作者Jens Axboe还开发了一套liburing库。liburing库提供了一组辅助函数实现设置和内存映射，应用不必了解诸多io_uring的细节就可以简单地使用起来。例如，无需担心memory barrier，或者是ring buffer管理之类等。上文所提的一些高级特性，在liburing中也有封装。

### 6.1 核心数据结构

liburing中，核心的结构有io_uring、io_uring_sq、io_uring_cq

```
/*
 * Library interface to io_uring
 */
struct io_uring_sq {
 unsigned *khead;
 unsigned *ktail;
 unsigned *kring_mask;
 unsigned *kring_entries;
 unsigned *kflags;
 unsigned *kdropped;
 unsigned *array;
 struct io_uring_sqe *sqes;

 unsigned sqe_head;
 unsigned sqe_tail;

 size_t ring_sz;
};

struct io_uring_cq {
 unsigned *khead;
 unsigned *ktail;
 unsigned *kring_mask;
 unsigned *kring_entries;
 unsigned *koverflow;
 struct io_uring_cqe *cqes;

 size_t ring_sz;
};

struct io_uring {
 struct io_uring_sq sq;
 struct io_uring_cq cq;
 int ring_fd;
};
```

### 6.2 核心接口

相关接口在头文件linux/tools/io_uring/liburing.h，如果是通过标准方式安装的liburing，则在/usr/include/下。

```
/*
 * System calls
 */
extern int io_uring_setup(unsigned entries, struct io_uring_params *p);
extern int io_uring_enter(int fd, unsigned to_submit,
 unsigned min_complete, unsigned flags, sigset_t *sig);
extern int io_uring_register(int fd, unsigned int opcode, void *arg,
 unsigned int nr_args);

/*
 * Library interface
 */
extern int io_uring_queue_init(unsigned entries, struct io_uring *ring,
 unsigned flags);
extern int io_uring_queue_mmap(int fd, struct io_uring_params *p,
 struct io_uring *ring);
extern void io_uring_queue_exit(struct io_uring *ring);
extern int io_uring_peek_cqe(struct io_uring *ring,
 struct io_uring_cqe **cqe_ptr);
extern int io_uring_wait_cqe(struct io_uring *ring,
 struct io_uring_cqe **cqe_ptr);
extern int io_uring_submit(struct io_uring *ring);
extern struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);
```

### 6.3 主要流程

- 使用io_uring_queue_init，完成io_uring相关结构的初始化。在这个函数的实现中，会调用多个mmap来初始化一些内存。

- 初始化完成之后，为了提交IO请求，需要获取里面queue的一个项，使用io_uring_get_sqe。

- - 获取到了空闲项之后，使用io_uring_prep_readv、io_uring_prep_writev初始化读、写请求。和前文所提preadv、pwritev的思想差不多，这里直接以不同的操作码委托io_uring_prep_rw，io_uring_prep_rw只是简单地初始化io_uring_sqe。

- 准备完成之后，使用io_uring_submit提交请求。

- 提交了IO请求时，可以通过非阻塞式函数io_uring_peek_cqe、阻塞式函数io_uring_wait_cqe获取请求完成的情况。默认情况下，完成的IO请求还会存在内部的队列中，需要通过io_uring_cqe_seen表标记完成操作。

- 使用完成之后要通过io_uring_queue_exit来完成资源清理的工作。

### 6.4 核心实现

io_uring_queue_init的实现，前文已略有提及。其中的操作主要就是io_uring_setup和io_uring_queue_mmap，io_uring_setup前文已解析过，这里主要看io_uring_queue_mmap。

```
/*
 * Returns -1 on error, or zero on success. On success, 'ring'
 * contains the necessary information to read/write to the rings.
 */
int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags)
{
 struct io_uring_params p;
 int fd, ret;

 memset(&p, 0, sizeof(p));
 p.flags = flags;

 fd = io_uring_setup(entries, &p);
 if (fd < 0)
  return fd;

 ret = io_uring_queue_mmap(fd, &p, ring);
 if (ret)
  close(fd);

 return ret;
}
```

io_uring_queue_mmap初始化io_uring结构，然后主要调用io_uring_mmap。

```
/*
 * For users that want to specify sq_thread_cpu or sq_thread_idle, this
 * interface is a convenient helper for mmap()ing the rings.
 * Returns -1 on error, or zero on success.  On success, 'ring'
 * contains the necessary information to read/write to the rings.
 */
int io_uring_queue_mmap(int fd, struct io_uring_params *p, struct io_uring *ring)
{
 int ret;

 memset(ring, 0, sizeof(*ring));
 ret = io_uring_mmap(fd, p, &ring->sq, &ring->cq);
 if (!ret)
  ring->ring_fd = fd;
 return ret;
}
```

io_uring_mmap初始化io_uring_sq结构和io_uring_cq结构的内存，另外还会分配一个io_uring_sqe结构的数组。

```
static int io_uring_mmap(int fd, struct io_uring_params *p,
    struct io_uring_sq *sq, struct io_uring_cq *cq)
{
 size_t size;
 void *ptr;
 int ret;

 sq->ring_sz = p->sq_off.array + p->sq_entries * sizeof(unsigned);
 ptr = mmap(0, sq->ring_sz, PROT_READ | PROT_WRITE,
   MAP_SHARED | MAP_POPULATE, fd, IORING_OFF_SQ_RING);
 if (ptr == MAP_FAILED)
  return -errno;
 sq->khead = ptr + p->sq_off.head;
 sq->ktail = ptr + p->sq_off.tail;
 sq->kring_mask = ptr + p->sq_off.ring_mask;
 sq->kring_entries = ptr + p->sq_off.ring_entries;
 sq->kflags = ptr + p->sq_off.flags;
 sq->kdropped = ptr + p->sq_off.dropped;
 sq->array = ptr + p->sq_off.array;

 size = p->sq_entries * sizeof(struct io_uring_sqe);
 sq->sqes = mmap(0, size, PROT_READ | PROT_WRITE,
    MAP_SHARED | MAP_POPULATE, fd,
    IORING_OFF_SQES);
 if (sq->sqes == MAP_FAILED) {
  ret = -errno;
err:
  munmap(sq->khead, sq->ring_sz);
  return ret;
 }

 cq->ring_sz = p->cq_off.cqes + p->cq_entries * sizeof(struct io_uring_cqe);
 ptr = mmap(0, cq->ring_sz, PROT_READ | PROT_WRITE,
   MAP_SHARED | MAP_POPULATE, fd, IORING_OFF_CQ_RING);
 if (ptr == MAP_FAILED) {
  ret = -errno;
  munmap(sq->sqes, p->sq_entries * sizeof(struct io_uring_sqe));
  goto err;
 }
 cq->khead = ptr + p->cq_off.head;
 cq->ktail = ptr + p->cq_off.tail;
 cq->kring_mask = ptr + p->cq_off.ring_mask;
 cq->kring_entries = ptr + p->cq_off.ring_entries;
 cq->koverflow = ptr + p->cq_off.overflow;
 cq->cqes = ptr + p->cq_off.cqes;
 return 0;
}
```

### 6.5 具体例程

如下是一个基于liburing的helloworld示例。

```
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <stdio.h>
#include <liburing.h>

#define ENTRIES 4

int main(int argc, char *argv[])
{
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    struct iovec iov = {
        .iov_base = "Hello World",
        .iov_len = strlen("Hello World"),
    };
    int fd, ret;
    if (argc != 2) {
        printf("%s: <testfile>\n", argv[0]);
        return 1;
    }
    /* setup io_uring and do mmap */
    ret = io_uring_queue_init(ENTRIES, &ring, 0);
    if (ret < 0) {
        printf("io_uring_queue_init: %s\n", strerror(-ret));
        return 1;
    }
    fd = open("testfile", O_WRONLY | O_CREAT);
    if (fd < 0) {
        printf("open failed\n");
        ret = 1;
        goto exit;
    }
    /* get an sqe and fill in a WRITEV operation */
    sqe = io_uring_get_sqe(&ring);
    if (!sqe) {
        printf("io_uring_get_sqe failed\n");
        ret = 1;
        goto out;
    }
    io_uring_prep_writev(sqe, fd, &iov, 1, 0);
    /* tell the kernel we have an sqe ready for consumption */
    ret = io_uring_submit(&ring);
    if (ret < 0) {
        printf("io_uring_submit: %s\n", strerror(-ret));
        goto out;
    }
    /* wait for the sqe to complete */
    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
        printf("io_uring_wait_cqe: %s\n", strerror(-ret));
        goto out;
    }
    /* read and process cqe event */
    io_uring_cqe_seen(&ring, cqe);
out:
    close(fd);
exit:
    /* tear down */
    io_uring_queue_exit(&ring);
    return ret;
}
```

更多的示例可参考：
http://git.kernel.dk/cgit/liburing/tree/examples
https://git.kernel.dk/cgit/liburing/tree/test

## 7.**性能**

如上，推演过了设计与实现，回归到存储的需求上来，io_uring子系统是否能满足我们对高性能的极致需求呢？这一切还是需要profile。

### 7.1 测试方法

io_uring原作者Jens Axboe在fio中提供了ioengine=io_uring的支持，可以使用fio进行测试，使用ioengine选项指定异步IO引擎。

可以基于不同的IO栈：

- libaio
- kernel+io_uring
- kernel+io_uring polling mode

可以基于一些硬件之上：

- NVMe SSD
- ...

测试过程中主要4k数据的顺序读、顺序写、随机读、随机写，对比几种IO引擎的性能及QoS等指标

io_uring polling mode测试实例：

```
fio -name=testname -filename=/mnt/vdd/testfilename -iodepth=64 -thread -rw=randread -ioengine=io_uring -sqthread_poll=1 -direct=1 -bs=4k -size=10G -numjobs=1 -runtime=600 -group_reporting
```

### 7.2 测试结果

网上可以找到一些关于io uring的性能测试，这里列出部分供参考：

- [*Im**proved Flash Performance Using the New Linux Kernel I/O Interface*](https://www.flashmemorysummit.com/Proceedings2019/08-07-Wednesday/20190807_SOFT-202-1_Verma.pdf)
- [*io_uring echo server benchs*](https://github.com/frevib/io_uring-echo-server/blob/io-uring-feat-fast-poll/benchs/benchs.md)
- [*[PATCHSET v5\] io_uring IO interface*](https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/)
- ...

主要有以下几个测试结果

- io_uring在非polling模式下，相比libaio，性能提升不是非常显著。
- io_uring在polling模式下，性能提升显著，与spdk接近，在队列深度较高时性能更好。
- 在meltdown和spectre漏洞没有修复的场景下，io_uring的提升并不太高。虽然减少了大量的用户态到内核态的上下文切换，在meldown和spectre漏洞没有修复的场景下，用户态到内核态的切换开销本比较小，所以提升不太高。
- 在某些场景下使用io_uring + Kernel NVMe的驱动，效果甚至要比使用SPDK 用户态NVMe 驱动更好

从测试中，我们可以得出结论，在存储中使用io_uring，相比使用libaio，应用的性能会有显著的提升。

在同样的硬件平台上，仅仅更换IO引擎，就可以带来较大的提升，是很难得的，对于存储这种延时敏感的应用而言十分宝贵。

### 7.3 io_uring的优势

综合前文和测试，io_uring有如此出众的性能，主要来源于以下几个方面：

- 用户态和内核态共享提交队列SQ和完成队列CQ实现零拷贝。
- IO提交和收割可以offload给Kernel，不需要经过系统调用。
- 支持块设备层的Polling模式。
- 可以提前注册用户态内存地址，从而减少地址映射的开销。
- 相比libaio，支持buffered IO

## 8.**发展方向**

事物的发展是一个哲学话题。前文阐述了io_uring作为一个新事物，发展的根本动力、内因和外因，谨此简述一些可预见的未来的发展方向。

### 8.1 普及

应用层多使用。目前主要应用在存储的场景中，这是一个不仅需要高性能，也需要稳定的场景，而一般来说，新事物并不具备“稳定”的属性。但是io_uring同样也是稳定的，因为虽然io_uring使用到了若干新概念，但是这些新的东西已经有了实践的检验，如eventfd通知机制，SIGIO信号机制，与AIO基本相似。它是一个质变的新事物。

就我们腾讯而言，内核使用tlinux，tlinux3基于4.14.99主线；tlinux4基于5.4.23主线。

所以，tlinux3可以用native aio，tlinux4之后已经可以用native io_uring。

相信通过大家的努力，正如前文所说的PostgreSQL使用彼时新接口pread，Nginx使用彼时的新接口AIO一样，通过使用新街口，我们的工程也能获得巨大收益。

### 8.2 优化方向

#### 8.2.1 **降低本身的工作负载**

持续降低系统调用开销、拷贝开销、框架本身的负载。

#### 8.2.2 **重构**

> "Politics are for the moment. An equation is for eternity.
> 　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　——Albert Einstein

追求真理的人不可避免地追求永恒。“政治只是一时，方程却是永恒。”——爱因斯坦如是说，时值以色列的第一任总统魏兹曼于1952年逝世，继任首相古理安建议邀请爱因斯坦担任第二任总统。

我们说折衷权衡、精益求精，字里行间都是永恒，然而软件应该持续重构，这实际上并不只是io_uring需要做的，有机会我会写一篇关于重构的文章。

## 9.**总结**

首先，本文简述了Linux过往的的IO发展历程，同步IO接口、原生异步IO接口AIO的缺陷，为何原有方式存在缺陷。其次，再从设计的角度出发，介绍了最新的IO引擎io_uring的相关内容。最后，深入最新版内核linux-5.10中解析了io_uring的大体实现（关键数据结构、流程、特性实现等）。

## 10.**关于**

难免纰漏，欢迎交流，可以通过以下网址找到本文。

- 知乎：https://www.zhihu.com/people/linkerist-61
- Github: https://github.com/Linkerist/blog/issues

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvasa9lQclibbU8QD0Gq8Jib16Gl18nIYG3o715BRWUw0uqkv90xxBM3F7xYrBjjvmXGFa4AnsV0EHIMg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

内容会更新，可以关注我的公众号，欢迎交流。

## 11.**参考**

[*PATCH 12/19\]io_uring: add support for pre-mapped user IO buffers*](https://lore.kernel.org/linux-block/20190211190049.7888-14-axboe@kernel.dk/)

[*Add pread/pwrite support bits to match the lseek bit*](https://lwn.net/Articles/97178/)

[*Toward non-blocking asynchronous I/O*](https://lwn.net/Articles/724198/)

[*A new kernel polling interface*](https://lwn.net/Articles/743714/)

[*The rapid growth of io_uring*](https://lwn.net/Articles/810414/)

[*Ringing in a new asynchronous I/O API*](https://lwn.net/Articles/776703/)

[*Efficient IO with io_uring*](http://kernel.dk/io_uring.pdf)

[*The current state of kernel page-table isolation*](https://lwn.net/Articles/741878)

[The Linux man-pages project](https://www.kernel.org/doc/man-pages/)

https://zhuanlan.zhihu.com/p/62682475

[*why we need io_uring? by byteisland*](https://www.byteisland.com/io_uring（1）-我们为什么会需要-io_uring/)

*Computer Systems: A Programmer's Perspective, Third Edition*

*Advanced Programming in the UNIX Environment, Third Edition*

*The Linux Programming Interface: A Linux and UNIX System Programming Handbook*

[*Introduction to io_uring*](http://kernel.taobao.org/2020/08/Introduction_to_IO_uring/)

*Understanding Nginx Modules Development and Architecture Resolving(Second Edition)*

原文作者：子晨

原文链接：https://mp.weixin.qq.com/s/QshDG-nbmBcF1OBZbBFwjg