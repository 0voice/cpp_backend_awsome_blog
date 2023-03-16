# 【NO.433】高并发高吞吐IO秘密武器——epoll池化技术

## 1.epoll函数详解

epoll是Linux特有的IO复用函数，使用一组函数来完成任务，而不是单个函数。

epoll把用户关心的文件描述符上的事件放在内核的一个事件表中，不需要像select、poll那样每次调用都要重复传入文件描述符集或事件集。

epoll需要使用一个额外的文件描述符，来唯一标识内核中的时间表，由epoll_create创建。

函数原型

```text
#include <sys/epoll.h>
 
    int epoll_create(int size);
    int epoll_create1(int flags);
 
    int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
 
    int epoll_wait(int epfd, struct epoll_event *events,
                int maxevents, int timeout);
    int epoll_pwait(int epfd, struct epoll_event *events,
                int maxevents, int timeout,
                const sigset_t *sigmask);
```

- epoll_create：创建一个epoll实例，size参数给内核一个提示，标识事件表的大小。函数返回的文件描述符将作用其他所有epoll系统调用的第一个参数，以指定要访问的内核事件表。
- epoll_ctl：操作文件描述符。fd表示要操作的文件描述符，op指定操作类型，event指定事件。
- epoll_wait：在一段超时时间内等待一组文件描述符上的事件。如果监测到事件，就将所有就绪的事件从内核事件表(epfd参数指定)中复制到第二个参数events指向的数组中。因为events数组只用于输出epoll_wait监测到的就绪事件，而不像select、poll那样就用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这样极大提高了应用程序索引就绪文件描述符的效率。

**函数返回**

特别注意epoll_wait函数成功时返回就绪的文件描述符总数。select和poll返回文件描述符总数。

以寻找已经就绪的文件描述符，举个例子如下：

epoll_wait只需要遍历返回的文件描述符，但是poll和select需要遍历所有文件描述符

```text
//  poll
int ret = poll(fds, MAX_EVENT_NUMBER, -1);
// 必须遍历所有已注册的文件描述符
for (int i = 0; i < MAX_EVENT_NUMBER; i++) {
    if (fds[i].revents & POLLIN) {
        int sockfd = fds[i].fd;
    }
}
 
// epoll_wait
int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
// 仅需要遍历就绪的ret个文件描述符
for (int i = 0; i < ret; i++) {
    int sockfd = events[i].data.fd;
}
```

**LT水平触发模式和ET边沿触发模式**

epoll监控多个文件描述符的I/O事件。epoll支持边缘触发(edge trigger，ET)或水平触发（level trigger，LT)，通过epoll_wait等待I/O事件，如果当前没有可用的事件则阻塞调用线程。

select和poll只支持LT工作模式，epoll的默认的工作模式是LT模式。

水平触发：

- 当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序可以不立即处理此事件。这样应用程序下一次调用epoll_wait的时候，epoll_wait还会再次向应用程序通告此事件，直到事件被处理。

边沿触发：

- 当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理此事件，后续的epoll_wait调用将不再向应用程序通知这一事件。

所以，边沿触发模式很大程度上降低了同一个epoll事件被重复触发的次数，所以效率更高。

## 2.三组IO复用函数对比

\1. 用户态将文件描述符传入内核的方式

- select：创建3个文件描述符集并拷贝到内核中，分别监听读、写、异常动作。这里受到单个进程可以打开的fd数量限制，默认是1024。
- poll：将传入的struct pollfd结构体数组拷贝到内核中进行监听。
- epoll：执行epoll_create会在内核的高速cache区中建立一颗红黑树以及就绪链表(该链表存储已经就绪的文件描述符)。接着用户执行的epoll_ctl函数添加文件描述符会在红黑树上增加相应的结点。

\2. 内核态检测文件描述符读写状态的方式

- select：采用轮询方式，遍历所有fd，最后返回一个描述符读写操作是否就绪的mask掩码，根据这个掩码给fd_set赋值。
- poll：同样采用轮询方式，查询每个fd的状态，如果就绪则在等待队列中加入一项并继续遍历。
- epoll：采用回调机制。在执行epoll_ctl的add操作时，不仅将文件描述符放到红黑树上，而且也注册了回调函数，内核在检测到某文件描述符可读/可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。

\3. 找到就绪的文件描述符并传递给用户态的方式

- select：将之前传入的fd_set拷贝传出到用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。
- poll：将之前传入的fd数组拷贝传出用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。
- epoll：epoll_wait只用观察就绪链表中有无数据即可，最后将链表的数据返回给数组并返回就绪的数量。内核将就绪的文件描述符放在传入的数组中，所以只用遍历依次处理即可。

\4. 重复监听的处理方式

- select：将新的监听文件描述符集合拷贝传入内核中，继续以上步骤。
- poll：将新的struct pollfd结构体数组拷贝传入内核中，继续以上步骤。
- epoll：无需重新构建红黑树，直接沿用已存在的即可。

## 3.epoll池化技术使用步骤

（1）epoll_create创建epoll池

```text
int epoll_create(int size);
```

size标识内核事件表的大小，返回的文件描述符将作用于其他所有epoll系统调用的第一个参数。

加上异常处理，一般的写法：

```text
epollfd = epoll_create(1024);
if (epollfd == -1) {
    perror("epoll_create");
    exit(EXIT_FAILURE);
}
```

这里epollfd来唯一标识epoll池，等于-1时表示出现了异常。

（2）在epoll池中添加fd

```text
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

epfd就是刚才创建epoll池的fd，op代表操作类型，常见三种操作类型如下：

1. EPOLL_CTL_ADD：在事件表中注册fd上的事件
2. EPOLL_CTL_MOD：修改fd上的注册事件
3. EPOLL_CTL_DEL：删除fd上的注册事件

所以我们这里使用EPOLL_CTL_ADD在epoll池中添加注册事件：

```text
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, 11, &ev) == -1) {
    perror("epoll_ctl: listen_sock");
    exit(EXIT_FAILURE);
}
```

（3）返回就绪文件描述符

主要通过epoll_wait来实现：

```text
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

在一段超时时间内等待一组文件描述符上的事件。如果监测到事件，就将所有就绪的事件从内核事件表(epfd参数指定)中复制到第二个参数events指向的数组中。因为events数组只用于输出epoll_wait监测到的就绪事件，而不像select、poll那样就用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这样极大提高了应用程序索引就绪文件描述符的效率。

举个例子：

```text
int ret = epoll_wait(epollfd, events, MAX_EVENT_NUM, -1);
for (int i = 0; i < ret; i++) {
    int sockfd = events[i].data.fd;
    // sockfd肯定已经就绪，直接处理
}
```

## 4.隐藏在epoll高效背后的秘密：红黑树和双向链表

- 为什么poll、select需要遍历所有已注册的文件描述符才能找到就绪fd，但是epoll不需要？
- 为什么在epoll池中fd频繁增删查改的过程中保持高效？
- 为什么epoll能通过事件通知的形式，做到最高效的运行？

要弄懂这些问题，需要深入了解epoll背后的数据结构。

**红黑树**

Linux 内核对于 epoll 池的内部实现就是用红黑树的结构体来管理这些注册进程来的句柄 fd。红黑树是一种平衡二叉树，时间复杂度为 O(log n)，就算这个池子就算不断的增删改，也能保持非常稳定的查找性能。

关于红黑树为什么可以实现高效的增删查改，就是另一个故事了，可以简单地概括如下：

![img](https://pic1.zhimg.com/80/v2-0f17dbba501e31df7bc62fbfd7697070_720w.webp)

（1）每个节点或者是黑色，或者是红色。

（2）根节点是黑色。

（3）每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]

（4）如果一个节点是红色的，则它的子节点必须是黑色的。

（5）从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。

**双向链表**

epoll_ctl内部实现中，通过poll回调，poll 机制让上层能直接告诉底层，如果这个 fd 一旦读写就绪了，请底层硬件（比如网卡）回调的时候自动把这个 fd 相关的结构体放到指定队列中，并且唤醒操作系统。

poll内部挂了个钩子，设置了唤醒的回调路径。这个路径存放在哪里？放到一个特定的队列（就绪队列，ready list），这个就绪队列其实是一个双向链表。放到就绪队列中，就可以唤醒epoll啦！

放到就绪队列中的epoll结构体的名字叫做epitem，每个注册到 epoll 池的 fd 都会对应一个。

![img](https://pic1.zhimg.com/80/v2-84d05a1effc8ee139a25a0641d1618b8_720w.webp)

## 5.epoll天生为网络编程而生

为什么说为网络编程而生呢？因为文件系统不能使用epoll

1. ext2，ext4，xfs 等这种真正的文件系统的 fd ，无法使用 epoll 管理；
2. socket fd，eventfd，timerfd 这些实现了 poll 调用的可以放到 epoll 池进行管理；

## 6.代码实例

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <errno.h>
 
#define MAXEVENTS 64
 
static int make_socket_non_blocking (int sfd)
{
  int flags, s;
 
  flags = fcntl (sfd, F_GETFL, 0);
  if (flags == -1)
    {
      perror ("fcntl");
      return -1;
    }
 
  flags |= O_NONBLOCK;
  s = fcntl (sfd, F_SETFL, flags);
  if (s == -1)
    {
      perror ("fcntl");
      return -1;
    }
 
  return 0;
}
 
static int create_and_bind (char *port)
{
  struct addrinfo hints;
  struct addrinfo *result, *rp;
  int s, sfd;
 
  memset (&hints, 0, sizeof (struct addrinfo));
  hints.ai_family = AF_UNSPEC;     /* Return IPv4 and IPv6 choices */
  hints.ai_socktype = SOCK_STREAM; /* We want a TCP socket */
  hints.ai_flags = AI_PASSIVE;     /* All interfaces */
 
  s = getaddrinfo (NULL, port, &hints, &result);
  if (s != 0)
    {
      fprintf (stderr, "getaddrinfo: %s\n", gai_strerror (s));
      return -1;
    }
 
  for (rp = result; rp != NULL; rp = rp->ai_next)
    {
      sfd = socket (rp->ai_family, rp->ai_socktype, rp->ai_protocol);
      if (sfd == -1)
        continue;
 
      s = bind (sfd, rp->ai_addr, rp->ai_addrlen);
      if (s == 0)
        {
          /* We managed to bind successfully! */
          break;
        }
 
      close (sfd);
    }
 
  if (rp == NULL)
    {
      fprintf (stderr, "Could not bind\n");
      return -1;
    }
 
  freeaddrinfo (result);
 
  return sfd;
}
 
int main (int argc, char *argv[])
{
  int sfd, s;
  int efd;
  struct epoll_event event;
  struct epoll_event *events;
 
  if (argc != 2)
    {
      fprintf (stderr, "Usage: %s [port]\n", argv[0]);
      exit (EXIT_FAILURE);
    }
 
  sfd = create_and_bind (argv[1]);
  if (sfd == -1)
    abort ();
 
  s = make_socket_non_blocking (sfd);
  if (s == -1)
    abort ();
 
  s = listen (sfd, SOMAXCONN);
  if (s == -1)
    {
      perror ("listen");
      abort ();
    }
 
  efd = epoll_create1 (0);
  if (efd == -1)
    {
      perror ("epoll_create");
      abort ();
    }
 
  event.data.fd = sfd;
  event.events = EPOLLIN | EPOLLET;
  s = epoll_ctl (efd, EPOLL_CTL_ADD, sfd, &event);
  if (s == -1)
    {
      perror ("epoll_ctl");
      abort ();
    }
 
  /* Buffer where events are returned */
  events = calloc (MAXEVENTS, sizeof event);
 
  /* The event loop */
  while (1)
    {
      int n, i;
 
      n = epoll_wait (efd, events, MAXEVENTS, -1);
      for (i = 0; i < n; i++)
	{
	  if ((events[i].events & EPOLLERR) ||
              (events[i].events & EPOLLHUP) ||
              (!(events[i].events & EPOLLIN)))
	    {
              /* An error has occured on this fd, or the socket is not
                 ready for reading (why were we notified then?) */
	      fprintf (stderr, "epoll error\n");
	      close (events[i].data.fd);
	      continue;
	    }
 
	  else if (sfd == events[i].data.fd)
	    {
              /* We have a notification on the listening socket, which
                 means one or more incoming connections. */
              while (1)
                {
                  struct sockaddr in_addr;
                  socklen_t in_len;
                  int infd;
                  char hbuf[NI_MAXHOST], sbuf[NI_MAXSERV];
 
                  in_len = sizeof in_addr;
                  infd = accept (sfd, &in_addr, &in_len);
                  if (infd == -1)
                    {
                      if ((errno == EAGAIN) ||
                          (errno == EWOULDBLOCK))
                        {
                          /* We have processed all incoming
                             connections. */
                          break;
                        }
                      else
                        {
                          perror ("accept");
                          break;
                        }
                    }
 
                  s = getnameinfo (&in_addr, in_len,
                                   hbuf, sizeof hbuf,
                                   sbuf, sizeof sbuf,
                                   NI_NUMERICHOST | NI_NUMERICSERV);
                  if (s == 0)
                    {
                      printf("Accepted connection on descriptor %d "
                             "(host=%s, port=%s)\n", infd, hbuf, sbuf);
                    }
 
                  /* Make the incoming socket non-blocking and add it to the
                     list of fds to monitor. */
                  s = make_socket_non_blocking (infd);
                  if (s == -1)
                    abort ();
 
                  event.data.fd = infd;
                  event.events = EPOLLIN | EPOLLET;
                  s = epoll_ctl (efd, EPOLL_CTL_ADD, infd, &event);
                  if (s == -1)
                    {
                      perror ("epoll_ctl");
                      abort ();
                    }
                }
              continue;
            }
          else
            {
              /* We have data on the fd waiting to be read. Read and
                 display it. We must read whatever data is available
                 completely, as we are running in edge-triggered mode
                 and won't get a notification again for the same
                 data. */
              int done = 0;
 
              while (1)
                {
                  ssize_t count;
                  char buf[512];
 
                  count = read (events[i].data.fd, buf, sizeof buf);
                  if (count == -1)
                    {
                      /* If errno == EAGAIN, that means we have read all
                         data. So go back to the main loop. */
                      if (errno != EAGAIN)
                        {
                          perror ("read");
                          done = 1;
                        }
                      break;
                    }
                  else if (count == 0)
                    {
                      /* End of file. The remote has closed the
                         connection. */
                      done = 1;
                      break;
                    }
 
                  /* Write the buffer to standard output */
                  s = write (1, buf, count);
                  if (s == -1)
                    {
                      perror ("write");
                      abort ();
                    }
                }
 
              if (done)
                {
                  printf ("Closed connection on descriptor %d\n",
                          events[i].data.fd);
 
                  /* Closing the descriptor will make epoll remove it
                     from the set of descriptors which are monitored. */
                  close (events[i].data.fd);
                }
            }
        }
    }
 
  free (events);
 
  close (sfd);
 
  return EXIT_SUCCESS;
}
```

原文地址：https://zhuanlan.zhihu.com/p/411809420

作者：linux