# 【NO.621】深度剖析linux socket的epollin/epollout是何时触发的

本文的问题是，在 EPOLLET 模式下，socket的 EPOLLIN 和 EPOLLOUT 是何时触发的？

由于epollin比较简单，我们先来看这个。

根据epoll相关的man文档我们可以知道，epollin表示有数据可读，所以它发生的时间必然是有新的tcp数据到来。

我们来写段代码验证下：

```text
#include <arpa/inet.h>
#include <assert.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define PORT 9999
#define MAX_EVENTS 10

static int tcp_listen() {
  int lfd, opt, err;
  struct sockaddr_in addr;

  lfd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  assert(lfd != -1);

  opt = 1;
  err = setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
  assert(!err);

  bzero(&addr, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_addr.s_addr = INADDR_ANY;
  addr.sin_port = htons(PORT);

  err = bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
  assert(!err);

  err = listen(lfd, 8);
  assert(!err);

  return lfd;
}

static void epoll_ctl_add(int epfd, int fd, int evts) {
  struct epoll_event ev;
  ev.events = evts;
  ev.data.fd = fd;
  int err = epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
  assert(!err);
}

static void handle_events(struct epoll_event *e, int epfd) {
  printf("events %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN ");
    e->events &= ~EPOLLIN;
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
  }

  assert(e->events == 0);
  printf("\n");
}

int main(int argc, char *argv[]) {
  int epfd, lfd, cfd, err, n;
  struct epoll_event events[MAX_EVENTS];

  epfd = epoll_create1(0);
  assert(epfd != -1);

  lfd = tcp_listen();
  epoll_ctl_add(epfd, lfd, EPOLLIN);

  for (;;) {
    n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    assert(n != -1);

    for (int i = 0; i < n; i++) {
      if (events[i].data.fd != lfd) {
        handle_events(&events[i], epfd);
        continue;
      }

      cfd = accept(lfd, NULL, NULL);
      assert(cfd != -1);

      err = fcntl(cfd, F_SETFL, O_NONBLOCK);
      assert(!err);

      epoll_ctl_add(epfd, cfd, EPOLLIN | EPOLLOUT | EPOLLET);
    }
  }

  return 0;
}
```

这段代码中我们主要关注的就是handle_events方法，该方法会输出socket的epollin或epollout事件。

该程序作为我们的服务端，而客户端我们用ncat来模拟，下面我们来执行看下。

下面是服务端输出：

```text
$ gcc server.c && ./a.out
events 5: EPOLLOUT
events 5: EPOLLIN EPOLLOUT
events 5: EPOLLIN EPOLLOUT
events 5: EPOLLIN EPOLLOUT
```

下面是客户端输出：

```text
$ ncat localhost 9999
1
2
^C
```

客户端及服务端的执行步骤如下：

\1. 编译并执行服务端程序，此时服务端在等待客户端连接，终端里没有任何输出。

\2. 执行ncat命令，建立从客户端到服务端的tcp连接，此时，服务端的终端会输出第一个epollout事件，原因我们后边讲epollout时会说到。

\3. 在客户端终端输入1，此时服务端终端会输出epollin和epollout，epollin产生的原因是因为客户端发来数据，此时服务端的socket可读，epollout产生的原因是因为服务端的socket可写。

\4. 在客户端终端输入2，此时服务端终端还是会输出epollin和epollout，原因如3。

\5. 用ctrl-c关闭ncat模拟的客户端，此时服务端还是会输出epollin和epollout，epollout产生的原因不变，epollin产生的原因多了个RCV_SHUTDOWN，这个我们后边会讲到。

总之，epollin事件产生的原因就是因为有新数据到来，对应到内核的源码为：

```text
// net/ipv4/tcp_input.c
void tcp_data_ready(struct sock *sk)
{
        ...
        sk->sk_data_ready(sk);
}
```

该方法会调用sk->sk_data_ready指向的方法，作用是通知epoll，该socket有epollin事件发生。

sk->sk_data_ready对应的方法为sock_def_readable，有兴趣的同学可以沿着这个方法继续看下。

```text
// net/core/sock.c
static void sock_def_readable(struct sock *sk)
{
        struct socket_wq *wq;
        ...
        wq = rcu_dereference(sk->sk_wq);
        if (skwq_has_sleeper(wq))
                wake_up_interruptible_sync_poll(&wq->wait, EPOLLIN | EPOLLPRI |
                                                EPOLLRDNORM | EPOLLRDBAND);
        ...
}
```

所以，当我们在客户端终端输入1、2时，服务端的socket就会收到epollin事件。

那为什么我们关闭客户端，服务端还是会收到epollin呢？这就和下面这个方法有关系了。

```text
// net/ipv4/tcp.c
__poll_t tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
        __poll_t mask;
        struct sock *sk = sock->sk;
        const struct tcp_sock *tp = tcp_sk(sk);
        int state;
        ...
        state = inet_sk_state_load(sk);
        ...
        mask = 0;
        ...
        // 该socket的既是RCV_SHUTDOWN，又是SEND_SHUTDOWN，或者状态是TCP_CLOSE
        // 对应的epoll事件都是EPOLLHUP
        if (sk->sk_shutdown == SHUTDOWN_MASK || state == TCP_CLOSE)
                mask |= EPOLLHUP;
        
        // 该socket是RCV_SHUTDOWN，比如对方用shutdown(sockfd, SHUT_WR)方法
        // 关闭它的SEND_SHUTDOWN，也就是关闭了我们的RCV_SHUTDOWN
        // 又比如，我们用shutdown(sockfd, SHUT_RD)方法，关闭我们自己的RCV_SHUTDOWN
        // 在此模式下，epoll事件为EPOLLIN
        if (sk->sk_shutdown & RCV_SHUTDOWN)
                mask |= EPOLLIN | EPOLLRDNORM | EPOLLRDHUP;

        // 当我们的socket处于TCP_ESTABLISHED等状态时
        if (state != TCP_SYN_SENT &&
            (state != TCP_SYN_RECV || tp->fastopen_rsk)) {
                ...
                // 如果我们的socket里有可读字节，epoll对应的事件就是EPOLLIN
                if (tcp_stream_is_readable(tp, target, sk))
                        mask |= EPOLLIN | EPOLLRDNORM;

                if (!(sk->sk_shutdown & SEND_SHUTDOWN)) {
                        // 如果我们的socket有可写空间，epoll事件就是EPOLLOUT
                        if (sk_stream_is_writeable(sk)) {
                                mask |= EPOLLOUT | EPOLLWRNORM;
                        } else {
                                ...
                        }
                } else
                        // 如果我们的socket关闭了SEND_SHUTDOWN，epoll事件就是EPOLLOUT
                        mask |= EPOLLOUT | EPOLLWRNORM;
                ...
        } else if (state == TCP_SYN_SENT && inet_sk(sk)->defer_connect) {
                ...
        }
        ...
        // 如果我们的socket发生错误了，epoll事件就是EPOLLERR
        if (sk->sk_err || !skb_queue_empty(&sk->sk_error_queue))
                mask |= EPOLLERR;

        return mask;
}
EXPORT_SYMBOL(tcp_poll);
```

当内核通知epoll，某个socket有事件发生时，epoll会调用上面的tcp_poll方法，检查该socket到底有什么事件发生，所以该方法是tcp/epoll体系中的非常重要的一个方法，它最终决定了用户能看到socket发生的哪些事件。

而且，该方法返回给用户的是，该socket此时所有满足条件的事件，例如上面有新数据到达后，内核会调用tcp_data_ready，通知epoll该socket有epollin事件发生，但是，epoll会再次调用tcp_poll方法，检查到原来不止是有epollin事件，还有epollout事件。

所以，我们上面的服务端终端会同时输出epollin和epollout。

看到这个方法，我们也就理解了，为什么上面操作流程5中说到，epollin产生的原因多了个RCV_SHUTDOWN，因为当我们关闭客户端时，服务端的socket会收到tcp的fin包，它的shutdown状态会设置为RCV_SHUTDOWN。

**由上面的所有分析可知，epollin事件产生的原因是：**

**1. 有新数据到达，socket可读。**

**2. 对方关闭了连接或只关闭了SEND_SHUTDOWN，导致我们关闭了RCV_SHUTDOWN。**

下面我们来看下epollout产生的原因。

由上面我们可以知道，epollin产生的原因是因为有新数据到达，那epollout产生的原因是不是因为，在我们往socket中写数据后，该数据有部分或全部被发送成功呢？

看上去好像是这样，不过我们要用实例验证看下。

我们先将上面代码的handle_events部分改成下面这样：

```text
static void handle_events(struct epoll_event *e, int epfd) {
  int n = write(e->data.fd, "hi\n", 3);
  assert(n == 3);

  printf("events %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN ");
    e->events &= ~EPOLLIN;
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
  }

  assert(e->events == 0);
  printf("\n");
}
```

该方法里，就是在最开始的地方加了个write方法。

那是不是说，在此种模式下，程序会陷入死循环呢？

因为tcp一连接上，就会有epollout事件发生，然后我们就往socket中写了些数据，该数据发送完毕之后又会触发epollout，然后又发数据，这样就进入了死循环。

是这样吗？我们来执行看下。

下面是服务端输出：

```text
$ gcc server.c && ./a.out
events 5: EPOLLOUT
```

下面是客户端输出：

```text
$ ncat localhost 9999
hi
```

执行流程为：

\1. 开启服务端，等待客户端连接，此时服务端终端没有任何输出。

\2. 用ncat模拟客户端连服务端，在连接上之后，服务端会输出epollout，客户端会输出hi，说明服务端的数据确实发到了客户端。

\3. 没有任何其他的反应了。

可以看到，这个输出和我们预想的并不一样，服务端的tcp在发送完数据后，并没有通知给我们epollout事件，所以没有我们上文猜测的死循环出现。

这是为什么呢？如果此时不通知，什么时候才会通知呢？

如果细心读过epoll文档的朋友可能会注意到下面这段话：

```text
The suggested way to use epoll as an edge-triggered (EPOLLET) interface is as follows:

  i   with nonblocking file descriptors; and
  
  ii  by waiting for an event only after read(2) or write(2) return EAGAIN.
```

条件1我们是满足的，条件2我们不满足，难道是因为这个原因？

看下源码：

```text
// net/ipv4/tcp.c
int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
{
        ...
        struct sk_buff *skb;
        ...
        int copied = 0;
        ...
        long timeo;
        ...
        // 因为我们设置了nonblock，所以该方法会设置timeo为0
        timeo = sock_sndtimeo(sk, flags & MSG_DONTWAIT);
        ...
restart:
        ...
        while (msg_data_left(msg)) {
                int copy = 0;

                skb = tcp_write_queue_tail(sk);
                if (skb)
                        copy = size_goal - skb->len;
                // 如果skb中没空间了，或者该skb不能在尾部添加数据，我们就需要新创建一个skb
                if (copy <= 0 || !tcp_skb_can_collapse_to(skb)) {
                        ...
                        // 我们该socket的内存使用量超过了系统要求的最大使用量
                        // 就等待sndbuf中的数据发送出去，这样可以有额外空间，
                        // 将我们的数据写到sndbuf中
                        if (!sk_stream_memory_free(sk))
                                goto wait_for_sndbuf;
                        ...
                        skb = sk_stream_alloc_skb(sk, 0, sk->sk_allocation,
                                                  first_skb);
                        ...                        
                }
                ...
                if (skb_availroom(skb) > 0 && !zc) {
                        // 拷贝数据到socket的sndbuf
                        copy = min_t(int, copy, skb_availroom(skb));
                        err = skb_add_data_nocache(sk, skb, &msg->msg_iter, copy);
                        ...
                }
                ...
                copied += copy;
                ...
                continue;

wait_for_sndbuf:
                set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
                ...
                // 由于我们是nonblock模式，该方法会返回错误码EAGAIN，之后该错误码又会返回给用户
                err = sk_stream_wait_memory(sk, &timeo);
                if (err != 0)
                        goto do_error;
                ...
        }
        ...
        return copied + copied_syn;
        ...
}
EXPORT_SYMBOL_GPL(tcp_sendmsg_locked);
```

由该方法我们可以看到，一直write直到返回EAGAIN，和只是write一次的区别是，有EAGAIN的设置了SOCK_NOSPACE，没EAGAIN的没设置，是这个原因？

我们来看下tcp中，收到ack包对应的内核代码，因为收到ack，才能证明对方收到数据，我们才可以丢掉sndbuf里的数据。

```text
// include/net/sock.h
static inline void sk_wmem_free_skb(struct sock *sk, struct sk_buff *skb)
{
        sock_set_flag(sk, SOCK_QUEUE_SHRUNK);
        ...
        __kfree_skb(skb);
}
```

在收到对方的ack后，tcp的sndbuf里的sk_buff会被释放掉，上面的方法就是对应的释放方法。

由上可见，该方法在释放skb之前，还设置了sk的flag为SOCK_QUEUE_SHRUNK。

在释放完ack对应的sk_buff之后，又会调用下面的方法：

```text
// net/ipv4/tcp_input.c
static void tcp_check_space(struct sock *sk)
{
        if (sock_flag(sk, SOCK_QUEUE_SHRUNK)) {
                sock_reset_flag(sk, SOCK_QUEUE_SHRUNK);
                ...
                if (sk->sk_socket &&
                    test_bit(SOCK_NOSPACE, &sk->sk_socket->flags)) {
                        tcp_new_space(sk);
                        ...
                }
        }
}
```

该方法经过各种判断之后，最终会调用方法：

```text
// net/ipv4/tcp_input.c
static void tcp_new_space(struct sock *sk)
{
        ...
        sk->sk_write_space(sk);
}
```

而这个方法，会调用sk->sk_write_space指向的方法，通知epoll，该socket有epollout事件发生。

所以说，只要满足tcp_check_space方法中的各种条件，epoll就会被通知，我们的socket有epollout事件，那我们的代码里也就会输出epollout，并陷入死循环。

让我们来看下tcp_check_space方法中的各种条件我们是否都满足。

首先是检测sock的flag中是否有SOCK_QUEUE_SHRUNK，在上面的sk_wmem_free_skb方法中，我们可以看到这个flag是设置了的，所以这个条件满足。

其次它会检测是否有SOCK_NOSPACE这个flag，由tcp_sendmsg_locked方法可以看到，如果一直write，直到返回EAGAIN，这个flag是设置了的，如果write没返回EAGAIN，则没有这个flag。

综上可知，由write导致的epollout事件，是要满足下面的各种条件才会发生。

首先，要一直write，直到返回EAGAIN，此时socket的send buffer是被占满的。

其次，当send buffer里的数据被发送并释放到一定程度时，tcp才会告知epoll，该socket有epollout事件发生。

我们用代码来实际验证下。

我们先将handle_events方法改成如下，再执行服务端程序，客户端还是用ncat模拟：

```text
static void handle_events(struct epoll_event *e, int epfd) {
  int n;
  char buf[8192];
  while (1) {
    n = write(e->data.fd, buf, 8192);
    if (n == -1) {
      assert(errno == EAGAIN);
      break;
    }
  }

  printf("events %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN ");
    e->events &= ~EPOLLIN;
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
  }

  assert(e->events == 0);
  printf("\n");
}
```

当tcp建立连接之后，我们会发现服务端终端一直输出epollout，进入了死循环，和我们分析的结果是一样的。

**下面我们来总结下epollout产生的原因：**

**1. 建立tcp连接，这部分我们没有源码分析，不过由于比较简单，有兴趣的同学可以自己看下。**

**2. 一直write，直到返回EAGAIN，然后等到write的数据发送完到一定程度后。**

还有些其他的不是很重要的原因，我们这里就不说了。

原文地址：https://zhuanlan.zhihu.com/p/335721118

作者：Linux