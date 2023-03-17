# 【NO.551】Linux服务器百万并发实现与问题排查

## 0.前言

  实现一台服务器的百万并发，服务器支撑百万连接会出现哪些问题，如何排查与解决这些问题 是本文的重点

- 服务器能够同时建立连接的数量 不是 并发量，它只是并发量一个基础。

- 服务器的并发量：一个服务器能够同时承载客户端的数量；
- 承载：服务器能够稳定的维持这些连接，能够响应请求，在200ms内返回响应就认为是ok的，其中这200ms包括数据库的操作，网络带宽，内存操作，日志等时间。



## 1.测试介绍

  服务器 采用 1台 centos7 12G 1核虚拟机

  客户端 采用 2台 centos7 3G 1核虚拟机

  服务器代码：单reactor单线程，IO多路复用使用epoll

  客户端代码：IO多路复用使用epoll，每个客户端发51w个连接，每个连接发送一次数据，读取一次数据之后不再发送数据

## 2.服务器代码

  由于fd的数量未知，这里设计ntyreactor 里面包含 eventblock ，eventblock 包含1024个fd。每个fd通过 fd/1024定位到在第几个eventblock，通过fd%1024定位到在eventblock第几个位置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/cd1bc4dc86e84de8ae69958fd5676d49.png)

```
struct ntyevent {
    int fd;
    int events;
    void *arg;

    NCALLBACK callback;
    
    int status;
    char buffer[BUFFER_LENGTH];
    int length;

};
struct eventblock {
    struct eventblock *next;
    struct ntyevent *events;
};

struct ntyreactor {
    int epfd;
    int blkcnt;
    struct eventblock *evblk;
};



```

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

#define BUFFER_LENGTH           4096
#define MAX_EPOLL_EVENTS        1024
#define SERVER_PORT             8081
#define PORT_COUNT              100

typedef int (*NCALLBACK)(int, int, void *);

struct ntyevent {
    int fd;
    int events;
    void *arg;

    NCALLBACK callback;
    
    int status;
    char buffer[BUFFER_LENGTH];
    int length;

};
struct eventblock {
    struct eventblock *next;
    struct ntyevent *events;
};

struct ntyreactor {
    int epfd;
    int blkcnt;
    struct eventblock *evblk;
};


int recv_cb(int fd, int events, void *arg);

int send_cb(int fd, int events, void *arg);

struct ntyevent *ntyreactor_find_event_idx(struct ntyreactor *reactor, int sockfd);

void nty_event_set(struct ntyevent *ev, int fd, NCALLBACK *callback, void *arg) {
    ev->fd = fd;
    ev->callback = callback;
    ev->events = 0;
    ev->arg = arg;
}

int nty_event_add(int epfd, int events, struct ntyevent *ev) {
    struct epoll_event ep_ev = {0, {0}};
    ep_ev.data.ptr = ev;
    ep_ev.events = ev->events = events;
    int op;
    if (ev->status == 1) {
        op = EPOLL_CTL_MOD;
    }
    else {
        op = EPOLL_CTL_ADD;
        ev->status = 1;
    }
    if (epoll_ctl(epfd, op, ev->fd, &ep_ev) < 0) {
        printf("event add failed [fd=%d], events[%d]\n", ev->fd, events);
        return -1;
    }
    return 0;
}

int nty_event_del(int epfd, struct ntyevent *ev) {
    struct epoll_event ep_ev = {0, {0}};
    if (ev->status != 1) {
        return -1;
    }
    ep_ev.data.ptr = ev;
    ev->status = 0;
    epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, &ep_ev);
    return 0;
}

int recv_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    struct ntyevent *ev = ntyreactor_find_event_idx(reactor, fd);
    int len = recv(fd, ev->buffer, BUFFER_LENGTH, 0); //
    nty_event_del(reactor->epfd, ev);

    if (len > 0) {
        ev->length = len;
        ev->buffer[len] = '\0';

//        printf("recv[%d]:%s\n", fd, ev->buffer);
        printf("recv fd=[%d\n", fd);

        nty_event_set(ev, fd, send_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLOUT, ev);
    }
    else if (len == 0) {
        close(ev->fd);
        //printf("[fd=%d] pos[%ld], closed\n", fd, ev-reactor->events);
    }
    else {
        close(ev->fd);

//        printf("recv[fd=%d] error[%d]:%s\n", fd, errno, strerror(errno));
    }
    return len;
}


int send_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    struct ntyevent *ev = ntyreactor_find_event_idx(reactor, fd);

    int len = send(fd, ev->buffer, ev->length, 0);
    if (len > 0) {

//        printf("send[fd=%d], [%d]%s\n", fd, len, ev->buffer);
        printf("send fd=[%d\n]", fd);

        nty_event_del(reactor->epfd, ev);
        nty_event_set(ev, fd, recv_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLIN, ev);
    }
    else {
        nty_event_del(reactor->epfd, ev);
        close(ev->fd);
        printf("send[fd=%d] error %s\n", fd, strerror(errno));
    }
    return len;

}

int accept_cb(int fd, int events, void *arg) {//非阻塞
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    if (reactor == NULL) return -1;

    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);
    
    int clientfd;
    if ((clientfd = accept(fd, (struct sockaddr *) &client_addr, &len)) == -1) {
        printf("accept: %s\n", strerror(errno));
        return -1;
    }
    if ((fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0) {
        printf("%s: fcntl nonblocking failed, %d\n", __func__, MAX_EPOLL_EVENTS);
        return -1;
    }
    struct ntyevent *event = ntyreactor_find_event_idx(reactor, clientfd);
    
    nty_event_set(event, clientfd, recv_cb, reactor);
    nty_event_add(reactor->epfd, EPOLLIN, event);
    
    printf("new connect [%s:%d], pos[%d]\n",
           inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), clientfd);
    return 0;

}

int init_sock(short port) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(fd, F_SETFL, O_NONBLOCK);
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(port);

    bind(fd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    
    if (listen(fd, 20) < 0) {
        printf("listen failed : %s\n", strerror(errno));
    }
    return fd;

}


int ntyreactor_alloc(struct ntyreactor *reactor) {
    if (reactor == NULL) return -1;
    if (reactor->evblk == NULL) return -1;

    struct eventblock *blk = reactor->evblk;
    while (blk->next != NULL) {
        blk = blk->next;
    }
    
    struct ntyevent *evs = (struct ntyevent *) malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    if (evs == NULL) {
        printf("ntyreactor_alloc ntyevents failed\n");
        return -2;
    }
    memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    
    struct eventblock *block = (struct eventblock *) malloc(sizeof(struct eventblock));
    if (block == NULL) {
        printf("ntyreactor_alloc eventblock failed\n");
        return -2;
    }
    memset(block, 0, sizeof(struct eventblock));
    
    block->events = evs;
    block->next = NULL;
    
    blk->next = block;
    reactor->blkcnt++; //
    return 0;

}

struct ntyevent *ntyreactor_find_event_idx(struct ntyreactor *reactor, int sockfd) {
    int blkidx = sockfd / MAX_EPOLL_EVENTS;

    while (blkidx >= reactor->blkcnt) {
        ntyreactor_alloc(reactor);
    }
    int i = 0;
    struct eventblock *blk = reactor->evblk;
    while (i++ < blkidx && blk != NULL) {
        blk = blk->next;
    }
    return &blk->events[sockfd % MAX_EPOLL_EVENTS];

}


int ntyreactor_init(struct ntyreactor *reactor) {
    if (reactor == NULL) return -1;
    memset(reactor, 0, sizeof(struct ntyreactor));

    reactor->epfd = epoll_create(1);
    if (reactor->epfd <= 0) {
        printf("create epfd in %s err %s\n", __func__, strerror(errno));
        return -2;
    }
    
    struct ntyevent *evs = (struct ntyevent *) malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    if (evs == NULL) {
        printf("ntyreactor_alloc ntyevents failed\n");
        return -2;
    }
    memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    
    struct eventblock *block = (struct eventblock *) malloc(sizeof(struct eventblock));
    if (block == NULL) {
        printf("ntyreactor_alloc eventblock failed\n");
        return -2;
    }
    memset(block, 0, sizeof(struct eventblock));
    
    block->events = evs;
    block->next = NULL;
    
    reactor->evblk = block;
    reactor->blkcnt = 1;
    return 0;

}

int ntyreactor_destory(struct ntyreactor *reactor) {
    close(reactor->epfd);
    //free(reactor->events);

    struct eventblock *blk = reactor->evblk;
    struct eventblock *blk_next = NULL;
    
    while (blk != NULL) {
        blk_next = blk->next;
        free(blk->events);
        free(blk);
        blk = blk_next;
    }
    return 0;

}


int ntyreactor_addlistener(struct ntyreactor *reactor, int sockfd, NCALLBACK *acceptor) {
    if (reactor == NULL) return -1;
    if (reactor->evblk == NULL) return -1;

    struct ntyevent *event = ntyreactor_find_event_idx(reactor, sockfd);
    
    nty_event_set(event, sockfd, acceptor, reactor);
    nty_event_add(reactor->epfd, EPOLLIN, event);
    return 0;

}


_Noreturn int ntyreactor_run(struct ntyreactor *reactor) {
    if (reactor == NULL) return -1;
    if (reactor->epfd < 0) return -1;
    if (reactor->evblk == NULL) return -1;

    struct epoll_event events[MAX_EPOLL_EVENTS + 1];
    
    int i;
    
    while (1) {
        int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);
        if (nready < 0) {
            printf("epoll_wait error, exit\n");
            continue;
        }
        for (i = 0; i < nready; i++) {
            struct ntyevent *ev = (struct ntyevent *) events[i].data.ptr;
            if ((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
            if ((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }

}


// <remoteip, remoteport, localip, localport,protocol>
int main(int argc, char *argv[]) {
    unsigned short port = SERVER_PORT; // listen 8081
    if (argc == 2) {
        port = atoi(argv[1]);
    }
    struct ntyreactor *reactor = (struct ntyreactor *) malloc(sizeof(struct ntyreactor));
    ntyreactor_init(reactor);
    int i = 0;
    int sockfds[PORT_COUNT] = {0};
    for (i = 0; i < PORT_COUNT; i++) {
        sockfds[i] = init_sock(port + i);
        ntyreactor_addlistener(reactor, sockfds[i], accept_cb);
    }
    ntyreactor_run(reactor);
    ntyreactor_destory(reactor);
    for (i = 0; i < PORT_COUNT; i++) {
        close(sockfds[i]);
    }
    free(reactor);
    return 0;
}
```

## 3.客户端代码

```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <errno.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <fcntl.h>
#include <sys/time.h>
#include <unistd.h>

#define MAX_BUFFER		128
#define MAX_EPOLLSIZE	(384*1024)
#define MAX_PORT		100
#define TIME_SUB_MS(tv1, tv2)  ((tv1.tv_sec - tv2.tv_sec) * 1000 + (tv1.tv_usec - tv2.tv_usec) / 1000)

int isContinue = 0;

static int ntySetNonblock(int fd) {
	int flags;

	flags = fcntl(fd, F_GETFL, 0);
	if (flags < 0) return flags;
	flags |= O_NONBLOCK;
	if (fcntl(fd, F_SETFL, flags) < 0) return -1;
	return 0;

}

static int ntySetReUseAddr(int fd) {
	int reuse = 1;
	return setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char *)&reuse, sizeof(reuse));
}



int main(int argc, char **argv) {
	if (argc <= 2) {
		printf("Usage: %s ip port\n", argv[0]);
		exit(0);
	}
	const char *ip = argv[1];
	int port = atoi(argv[2]);
	int connections = 0;
	char buffer[128] = {0};
	int i = 0, index = 0;
	struct epoll_event events[MAX_EPOLLSIZE];
	int epoll_fd = epoll_create(MAX_EPOLLSIZE);
	strcpy(buffer, " Data From MulClient\n");
	struct sockaddr_in addr;
	memset(&addr, 0, sizeof(struct sockaddr_in));
	addr.sin_family = AF_INET;
	addr.sin_addr.s_addr = inet_addr(ip);
	struct timeval tv_begin;
	gettimeofday(&tv_begin, NULL);

	while (1) {
		if (++index >= MAX_PORT) index = 0;
		struct epoll_event ev;
		int sockfd = 0;
		if (connections < 340000 && !isContinue) {
			sockfd = socket(AF_INET, SOCK_STREAM, 0);
			if (sockfd == -1) {
				perror("socket");
				goto err;
			}
			//ntySetReUseAddr(sockfd);
			addr.sin_port = htons(port+index);
			if (connect(sockfd, (struct sockaddr*)&addr, sizeof(struct sockaddr_in)) < 0) {
				perror("connect");
				goto err;
			}
			ntySetNonblock(sockfd);
			ntySetReUseAddr(sockfd);
			sprintf(buffer, "Hello Server: client --> %d\n", connections);
			send(sockfd, buffer, strlen(buffer), 0);
			ev.data.fd = sockfd;
			ev.events = EPOLLIN | EPOLLOUT;
			epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sockfd, &ev);
			connections ++;
		}
		//connections ++;
		if (connections % 1000 == 999 || connections >= 340000) {
			struct timeval tv_cur;
			memcpy(&tv_cur, &tv_begin, sizeof(struct timeval));
			gettimeofday(&tv_begin, NULL);
			int time_used = TIME_SUB_MS(tv_begin, tv_cur);
			printf("connections: %d, sockfd:%d, time_used:%d\n", connections, sockfd, time_used);
			int nfds = epoll_wait(epoll_fd, events, connections, 100);
			for (i = 0;i < nfds;i ++) {
				int clientfd = events[i].data.fd;
				if (events[i].events & EPOLLOUT) {
					sprintf(buffer, "data from %d\n", clientfd);
					send(sockfd, buffer, strlen(buffer), 0);
				} else if (events[i].events & EPOLLIN) {
					char rBuffer[MAX_BUFFER] = {0};				
					ssize_t length = recv(sockfd, rBuffer, MAX_BUFFER, 0);
					if (length > 0) {
						printf(" RecvBuffer:%s\n", rBuffer);
						if (!strcmp(rBuffer, "quit")) {
							isContinue = 0;
						}
					} else if (length == 0) {
						printf(" Disconnect clientfd:%d\n", clientfd);
						connections --;
						close(clientfd);
					} else {
						if (errno == EINTR) continue;
						printf(" Error clientfd:%d, errno:%d\n", clientfd, errno);
						close(clientfd);
					}
				} else {
					printf(" clientfd:%d, errno:%d\n", clientfd, errno);
					close(clientfd);
				}
			}
		}
		usleep(1 * 1000);
	}
	return 0;

err:
	printf("error : %s\n", strerror(errno));
	return 0;
}
```



## 4.**error : Too many open files**

### 4.1 确定问题

  程序执行到一半，创建了1023个连接后，报错Too many open files

```
//服务端
new connect [192.168.109.101:36994], pos[1019]
new connect [192.168.109.101:55832], pos[1020]
new connect [192.168.109.101:43460], pos[1021]
new connect [192.168.109.101:59938], pos[1022]
new connect [192.168.109.101:46098], pos[1023]
accept: Too many open files
accept: Too many open files

//客户端
connect: Connection refused
error : Connection refused
```

  怀疑是文件系统默认允许打开文件描述符数量个数（默认1024）的限制，使用ulimit -a查看open files的数量

open files：一个进程能够打开文件描述符的数量

```
[root@master temp]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 47748
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 47748
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```


  那么我们把open files调大一点点，看是否会停在2047，如果是，则说明问题就是open files太小的问题，实验发现就是这个原因。

```
[root@master temp]# ulimit -n 2048
[root@master temp]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 47748
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 2048
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 47748
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited


new connect [192.168.109.101:53996], pos[2046]
new connect [192.168.109.101:60742], pos[2047]
accept: Too many open files
```

### 4.2 解决问题

1. 临时修改，只在当前这个会话有效：ulimit -n 1048576
2. 永久修改，对所有会话有效：添加下面两行代码


注意这里修改的是：一个进程能够打开文件描述符的数量

```
[root@master temp]# vim /etc/security/limits.conf

# 修改

[root@master temp]# reboot

# 重启生效
```


* ```
  *               soft    nofile          1048576
  *               hard    nofile          1048576
  ```

  软限制：超出软限制会发出警告
  硬限制：绝对限制,在任何情况下都不允许用户超过这个限制

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/f7b87a1dd9914d28a8cbc785f987cb01.png)

  这里还需要注意一点：file-max : 系统一共可以打开的最大文件数（所有进程加起来）

```
[root@master temp]# cat /proc/sys/fs/file-max
1202172
```



```
# 编辑内核参数配置文件

vim /etc/sysctl.conf

# 修改fs.file-max参数

fs.file-max = 1048576

# 重新加载配置文件

sysctl -p
```


  另外这里建议ulimit -n 和limits.conf里nofile 设定最好不要超过/proc/sys/fs/file-max的值（虽然我测试了超过也没关系），这个小问题仁者见仁智者见智了，网上找到比较好的文章是这篇linux最大文件句柄数量之（file-max ulimit -n limit.conf）

## 5.error : Cannot assign requested address

### 5.1 确定问题

  现在的环境背景：服务器只开放一个端口，客户端不断的去请求去连接。然后客户端error : Cannot assign requested address

  Cannot assign requested address这代表着客户端端口耗尽，我们先来看看如何确定一个fd，反过来说一个fd代表着什么

  socket fd --- < 源IP地址 , 源端口 , 目的IP地址 , 目的端口 , 协议 > 一个fd就是一个五元组，在现在的环境中，五元组里面确定了四个，所以最多创建 1 * 源端口 * 1 * 1 * 1个fd

```
# 服务端

new connect [192.168.109.101:57921], pos[28234]
new connect [192.168.109.101:57923], pos[28235]
send[fd=21003] error Connection reset by peer
send[fd=22003] error Connection reset by peer

# 客户端

connections: 26999, sockfd:27002, time_used:2399
connections: 27999, sockfd:28002, time_used:2404
connect: Cannot assign requested address
error : Cannot assign requested address
```


  我们看到大概创建了2.8w的fd ， 可是我们知道端口一个有6w多个，也就是说有6w个端口，为什么我们只使用了2.8w个？

  我们看到大概创建了2.8w的fd ， 可是我们知道端口一个有6w多个，也就是说有6w个端口，为什么我们只使用了2.8w个？

  Linux中有限定端口的使用范围:60999 - 32768 = 2.8w ,与我们上面实验结果相符。

```
The /proc/sys/net/ipv4/ip_local_port_range defines the local port range that is used by TCP and UDP traffic to choose the local port. You will see in the parameters of this file two numbers: The first number is the first local port allowed for TCP and UDP traffic on the server, the second is the last local port number. For high-usage systems you may change its default parameters to 32768-61000 -first-last.
```

```
proc/sys/net/ipv4/ip_local_port_range范围定义TCP和UDP通信用于选择本地端口的本地端口范围。您将在该文件的参数中看到两个数字：第一个数字是服务器上允许TCP和UDP通信的第一个本地端口，第二个是最后一个本地端口号。对于高使用率的系统，您可以将其默认参数更改为32768-61000(first-last)。
```



```
[root@master temp]# sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range = 32768	60999
```



### 5.2 解决问题

  

修改net.ipv4.ip_local_port_range的范围，一般不这样做，我们这里研究的是服务器，怎么会去对客户端进行修改呢
之前已经说了这个问题的背景，就是只开放了一个端口，并且socket fd --- < 源IP地址 , 源端口, 目的IP地址 , 目的端口 , 运输层协议 >，在这个背景下才产生的这个问题，所以我们可以开放更多的端口，比如说100个，那么一个客户端就能连到280w了
error : Connection timed out
确定问题
  我们将服务器端口开100个，按理说客户端可以连280w，但是现在只连接到13w就error : Connection timed out，与我们的预期不符

```
//服务端
new connect [192.168.109.101:54585], pos[131165]
new connect [192.168.109.101:48265], pos[131166]
new connect [192.168.109.101:51997], pos[131167]
new connect [192.168.109.101:43239], pos[131168]
send[fd=20102] error Connection reset by peer
send[fd=21102] error Connection reset by peer
send[fd=22102] error Connection reset by peer

//客户端
connections: 127999, sockfd:128002, time_used:7576
connections: 128999, sockfd:129002, time_used:2683
connections: 129999, sockfd:130002, time_used:2669
connections: 130999, sockfd:131002, time_used:4610

connect: Connection timed out
error : Connection timed out
```


  网卡接收的数据，会发送到协议栈里面，通过sk_buff将数据传到协议栈，协议栈处理完再交给应用程序。由于操作系统在使用的时候，为防止被攻击，在数据发送给协议栈之前进行一个过滤，在协议栈前面加了一个小组件：过滤器，叫做netfilter。
  netfilter主要是对网络数据包进行一个过滤，在netfilter的基础上我们就可以实现防火墙，在linux里面有一个就叫做iptables，iptables是基于netfilter做的，iptables分为两部分，一部分是内核实现的netfilter接口，一部分是应用程序提供给用户使用的。iptables真正实现的是netfilter提供的接口。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d61f3f57bcb6484a83437fc952e68a6a.png)

  Connection timed out译为连接超时，也就是说，client发送的请求超时了，那么这个超时有两种情况，第一种：三次握手第一次的SYN没发出去，第二种：三次握手第二次ACK没收到。

![在这里插入图片描述](https://img-blog.csdnimg.cn/ff4ef956709d408b810464af46e3380e.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/b388247e018b45a5ba21ff565fe7987b.png)  

netfilter不管对发送的数据，还是对接收的数据，都是可以过滤的。当连接数量达到一定数量的时候，netfilter就会不允许再对外发连接了。所以现在推测是情况1造成的，发送的SYN被netfilter拦截了。

  事实是这样吗，我们来查看一下netfilter允许对外最大连接数量是多少。13w，与我们上面建立成功的数量一致，所以现在就可以确定是netfilter允许对外开放的最大连接数造成的了

```
[root@node1 temp]# cat /proc/sys/net/netfilter/nf_conntrack_max
131072
```


解决问题
  我们可以通过设置netfilter允许对外最大连接数量，来解决这个问题

```
# 查看允许对外最大连接数量

[root@node1 temp]# cat /proc/sys/net/netfilter/nf_conntrack_max
131072

# 进行配置

vim /etc/sysctl.conf

# 在配置文件中把net.nf_conntrack_max参数修改为1048576（如果配置就自己添加一行）

net.nf_conntrack_max = 1048576

# 重新加载配置文件

sysctl -p

# 再次查看，发现生效了

[root@node1 temp]# cat /proc/sys/net/netfilter/nf_conntrack_max
1048576
```



## 6.killed（已杀死）

### 6.1 确定问题

  这里我们先给客户端虚拟机2G的内存，然后发现到24w的时候，客户端进程被杀死了

```
connections: 239999, sockfd:240002, time_used:9837
connections: 240999, sockfd:241002, time_used:10608
connections: 241999, sockfd:242002, time_used:13109
connections: 242999, sockfd:243002, time_used:15112
connections: 243999, sockfd:244002, time_used:12606
已杀死
```


  我们来看一下kill记录，发现是内存不足。

```
[root@node1 ~]# dmesg | egrep -i -B100 'killed process'
[ 2310.265218] Out of memory: Kill process 7266 (C1000Kclient) score 1 or sacrifice child
[ 2310.265962] Killed process 7266 (C1000Kclient) total-vm:8708kB, anon-rss:2960kB, file-rss:0kB, shmem-rss:0kB
```


  这里直接说原因吧，是因为程序每个fd都有一个tcp接收缓冲区和tcp发送缓冲区。而默认的太大了，导致Linux内存不足，进程被杀死，所有我们需要适当的缩小。进程空间，代码段，堆栈都是要占用内存的。

### 6.2 解决问题

  我们只需要对net.ipv4.tcp_mem，net.ipv4.tcp_wmem，net.ipv4.tcp_rmem进行适合的修改即可

```
# 编辑内核参数配置文件

vim /etc/sysctl.conf

# 添加以下内容

# 					最小值   默认值   最大值

net.ipv4.tcp_mem = 252144 524288 786432	# tcp协议栈的大小，单位为内存页（4K），分别是 1G 2G 3G，如果大于2G，tcp协议栈会进行一定的优化
net.ipv4.tcp_wmem = 1024 1024 2048 # tcp接收缓存区(用于tcp接受滑动窗口)的最小值，默认值和最大值（单位byte）1k 1k 2k,每一个连接fd都有一个接收缓存区
net.ipv4.tcp_rmem = 1024 1024 2048 # tcp发送缓存区(用于tcp发送滑动窗口)的最小值，默认值和最大值（单位byte）1k 1k 2k,每一个连接fd都有一个发送缓存区

# 总缓存 = （每个fd发送缓存区 + 每个fd接收缓存区） * fd数量

# （1024byte + 1024byte ） * 100w 约等于 2G
```

  如果服务器是用来接收大文件，传输量很大的时候，就要把send buffer和read buffer调大。
  如果服务器只是接收小数据字符的时候。把buffer调小是为了把fd的数量做到更多，并发数量能做到更大。如果buffer调大的话，内存会不够。

![在这里插入图片描述](https://img-blog.csdnimg.cn/a1489051ed9147a396c1be49dc2253b5.png)

## 7.百万并发测试结果

## 8.出现的问题总结

  想要实现服务器百万并发：

一个进程能够打开文件描述符的数量open files 和 file-max 改成100w以上
在不同的环境下要看开放的端口够不够socket fd --- < 源IP地址 , 源端口 , 目的IP地址 , 目的端口 , 协议 >
设置netfilter允许对外最大连接数量100w以上
根据内存和场景，适当调整net.ipv4.tcp_mem，net.ipv4.tcp_wmem，net.ipv4.tcp_rmem
————————————————
版权声明：本文为CSDN博主「cheems~」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_42956653/article/details/125653754