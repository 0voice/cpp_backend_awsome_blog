# 【NO.587】linux异步IO编程实例分析

![动图封面](https://pic2.zhimg.com/v2-0855a1508418e6ca6cb4b933f2dc3e61_b.jpg)



异步阻塞IO模型的流程图

在Direct IO模式下，异步是非常有必要的（因为绕过了pagecache，直接和磁盘交互）。linux Native AIO正是基于这种场景设计的，具体的介绍见：KernelAsynchronousI/O (AIO)SupportforLinux。下面我们就来分析一下AIO编程的相关知识。

阻塞模式下的IO过程如下：

```text
int fd = open(const char *pathname, int flags, mode_t mode);
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
int close(int fd);
```

因为整个过程会等待read/write的返回，所以不需要任何额外的数据结构。但异步IO的思想是：应用程序不能阻塞在昂贵的系统调用上让CPU睡大觉，而是将IO操作抽象成一个个的任务单元提交给内核，内核完成IO任务后将结果放在应用程序可以取到的地方。这样在底层做I/O的这段时间内，CPU可以去干其他的计算任务。但异步的IO任务批量的提交和完成，必须有自身可描述的结构，最重要的两个就是iocb和io_event。

## 1.libaio中的structs

```text
struct iocb {
        void     *data;  /* Return in the io completion event */
        unsigned key;   /*r use in identifying io requests */
        short           aio_lio_opcode;
        short           aio_reqprio;
        int             aio_fildes;
        union {
                struct io_iocb_common           c;
                struct io_iocb_vector           v;
                struct io_iocb_poll             poll;
                struct io_iocb_sockaddr saddr;
        } u;
};
struct io_iocb_common {
        void            *buf;
        unsigned long   nbytes;
        long long       offset;
        unsigned        flags;
        unsigned        resfd;
};
```

iocb是提交IO任务时用到的，可以完整地描述一个IO请求：

data是留给用来自定义的指针：可以设置为IO完成后的callback函数；

aio_lio_opcode表示操作的类型：IO_CMD_PWRITE | IO_CMD_PREAD；

aio_fildes是要操作的文件：fd；

io_iocb_common中的buf, nbytes, offset分别记录的IO请求的mem buffer，大小和偏移。

```text
struct io_event {
        void *data;
        struct iocb *obj;
        unsigned long res;
        unsigned long res2;
};
```

io_event是用来描述返回结果的：

obj就是之前提交IO任务时的iocb；

res和res2来表示IO任务完成的状态。

## 2.libaio提供的API和完成IO的过程

libaio提供的API有：io_setup, io_submit, io_getevents, io_destroy。

\1. 建立IO任务

```text
int io_setup (int maxevents, io_context_t *ctxp);
```

io_context_t对应内核中一个结构，为异步IO请求提供上下文环境。注意在setup前必须将io_context_t初始化为0。

当然，这里也需要open需要操作的文件，注意设置O_DIRECT标志。

2.提交IO任务

```text
long io_submit (aio_context_t ctx_id, long nr, struct iocb **iocbpp);
```

提交任务之前必须先填充iocb结构体，libaio提供的包装函数说明了需要完成的工作：

```text
void io_prep_pread(struct iocb *iocb, int fd, void *buf, size_t count, long long offset)
{
        memset(iocb, 0, sizeof(*iocb));
        iocb->aio_fildes = fd;
        iocb->aio_lio_opcode = IO_CMD_PREAD;
        iocb->aio_reqprio = 0;
        iocb->u.c.buf = buf;
        iocb->u.c.nbytes = count;
        iocb->u.c.offset = offset;
}
void io_prep_pwrite(struct iocb *iocb, int fd, void *buf, size_t count, long long offset)
{
        memset(iocb, 0, sizeof(*iocb));
        iocb->aio_fildes = fd;
        iocb->aio_lio_opcode = IO_CMD_PWRITE;
        iocb->aio_reqprio = 0;
        iocb->u.c.buf = buf;
        iocb->u.c.nbytes = count;
        iocb->u.c.offset = offset;
}
```

这里注意读写的buf都必须是按扇区对齐的，可以用posix_memalign来分配。

3.获取完成的IO

```text
long io_getevents (aio_context_t ctx_id, long min_nr, long nr, struct io_event *events, struct timespec *timeout);
```

这里最重要的就是提供一个io_event数组给内核来copy完成的IO请求到这里，数组的大小是io_setup时指定的maxevents。

timeout是指等待IO完成的超时时间，设置为NULL表示一直等待所有到IO的完成。

4.销毁IO任务

```text
int io_destroy (io_context_t ctx);
```

## 3.libaio和epoll的结合

在异步编程中，任何一个环节的阻塞都会导致整个程序的阻塞，所以一定要避免在io_getevents调用时阻塞式的等待。还记得io_iocb_common中的flags和resfd吗？看看libaio是如何提供io_getevents和事件循环的结合：

```text
void io_set_eventfd(struct iocb *iocb, int eventfd)
{
        iocb->u.c.flags |= (1 << 0) /* IOCB_FLAG_RESFD */;
        iocb->u.c.resfd = eventfd;
}
```

这里的resfd是通过系统调用eventfd生成的。

```text
int eventfd(unsigned int initval, int flags);
```

eventfd是linux 2.6.22内核之后加进来的syscall，作用是内核用来通知应用程序发生的事件的数量，从而使应用程序不用频繁地去轮询内核是否有时间发生，而是由内核将发生事件的数量写入到该fd，应用程序发现fd**可读**后，从fd读取该数值，并马上去内核读取。

有了eventfd，就可以很好地将libaio和epoll事件循环结合起来：

\1. 创建一个eventfd

```text
efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
```

\2. 将eventfd设置到iocb中

```text
io_set_eventfd(iocb, efd);
```

\3. 交接AIO请求

```text
io_submit(ctx, NUM_EVENTS, iocb);
```

\4. 创建一个epollfd，并将eventfd加到epoll中

```text
epfd = epoll_create(1);
epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &epevent);
epoll_wait(epfd, &epevent, 1, -1);
```

\5. 当eventfd可读时，从eventfd读出完成IO请求的数量，并调用io_getevents获取这些IO

```text
read(efd, &finished_aio, sizeof(finished_aio);
r = io_getevents(ctx, 1, NUM_EVENTS, events, &tms);
```



![动图封面](https://pic4.zhimg.com/v2-1d5d9ee387fab4bc680d85f9489ca1b7_b.jpg)



异步非阻塞IO模型的流程图

## 4.一个完整的编程实例

```text
#define _GNU_SOURCE
#define __STDC_FORMAT_MACROS


#include <stdio.h>
#include <errno.h>
#include <libaio.h>
#include <sys/eventfd.h>
#include <sys/epoll.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdint.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <inttypes.h>


#define TEST_FILE   "aio_test_file"
#define TEST_FILE_SIZE  (127 * 1024)
#define NUM_EVENTS  128
#define ALIGN_SIZE  512
#define RD_WR_SIZE  1024


struct custom_iocb
{
    struct iocb iocb;
    int nth_request;
};


void aio_callback(io_context_t ctx, struct iocb *iocb, long res, long res2)
{
    struct custom_iocb *iocbp = (struct custom_iocb *)iocb;
    printf("nth_request: %d, request_type: %s, offset: %lld, length: %lu, res: %ld, res2: %ld\n", 
            iocbp->nth_request, (iocb->aio_lio_opcode == IO_CMD_PREAD) ? "READ" : "WRITE",
            iocb->u.c.offset, iocb->u.c.nbytes, res, res2);
}


int main(int argc, char *argv[])
{
    int efd, fd, epfd;
    io_context_t ctx;
    struct timespec tms;
    struct io_event events[NUM_EVENTS];
    struct custom_iocb iocbs[NUM_EVENTS];
    struct iocb *iocbps[NUM_EVENTS];
    struct custom_iocb *iocbp;
    int i, j, r;
    void *buf;
    struct epoll_event epevent;


    efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (efd == -1) {
        perror("eventfd");
        return 2;
    }


    fd = open(TEST_FILE, O_RDWR | O_CREAT | O_DIRECT, 0644);
    if (fd == -1) {
        perror("open");
        return 3;
    }
    ftruncate(fd, TEST_FILE_SIZE);
    
    ctx = 0;
    if (io_setup(8192, &ctx)) {
        perror("io_setup");
        return 4;
    }


    if (posix_memalign(&buf, ALIGN_SIZE, RD_WR_SIZE)) {
        perror("posix_memalign");
        return 5;
    }
    printf("buf: %p\n", buf);


    for (i = 0, iocbp = iocbs; i < NUM_EVENTS; ++i, ++iocbp) {
        iocbps[i] = &iocbp->iocb;
        io_prep_pread(&iocbp->iocb, fd, buf, RD_WR_SIZE, i * RD_WR_SIZE);
        io_set_eventfd(&iocbp->iocb, efd);
        io_set_callback(&iocbp->iocb, aio_callback);
        iocbp->nth_request = i + 1;
    }


    if (io_submit(ctx, NUM_EVENTS, iocbps) != NUM_EVENTS) {
        perror("io_submit");
        return 6;
    }


    epfd = epoll_create(1);
    if (epfd == -1) {
        perror("epoll_create");
        return 7;
    }


    epevent.events = EPOLLIN | EPOLLET;
    epevent.data.ptr = NULL;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &epevent)) {
        perror("epoll_ctl");
        return 8;
    }


    i = 0;
    while (i < NUM_EVENTS) {
        uint64_t finished_aio;


        if (epoll_wait(epfd, &epevent, 1, -1) != 1) {
            perror("epoll_wait");
            return 9;
        }


        if (read(efd, &finished_aio, sizeof(finished_aio)) != sizeof(finished_aio)) {
            perror("read");
            return 10;
        }


        printf("finished io number: %"PRIu64"\n", finished_aio);
    
        while (finished_aio > 0) {
            tms.tv_sec = 0;
            tms.tv_nsec = 0;
            r = io_getevents(ctx, 1, NUM_EVENTS, events, &tms);
            if (r > 0) {
                for (j = 0; j < r; ++j) {
                    ((io_callback_t)(events[j].data))(ctx, events[j].obj, events[j].res, events[j].res2);
                }
                i += r;
                finished_aio -= r;
            }
        }
    }
    
    close(epfd);
    free(buf);
    io_destroy(ctx);
    close(fd);
    close(efd);
    remove(TEST_FILE);


    return 0;
}
```


说明：

\1. 在centos 6.2 (libaio-devel 0.3.107-10) 上运行通过

\2. struct io_event中的res字段表示读到的字节数或者一个负数错误码。在后一种情况下，-res表示对应的

errno。res2字段为0表示成功，否则失败

\3. iocb在aio请求执行过程中必须是valid的

\4. 在上面的程序中，通过扩展iocb结构来保存额外的信息(nth_request)，并使用iocb.data

来保存回调函数的地址。如果回调函数是固定的，那么也可以使用iocb.data来保存额外信息。

原文地址：https://zhuanlan.zhihu.com/p/610147885

作者：linux