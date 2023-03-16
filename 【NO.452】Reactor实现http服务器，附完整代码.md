# 【NO.452】Reactor实现http服务器，附完整代码

如何在reactor的基础上实现业务？就是怎么利用reactor做服务器，并实现服务器的业务。

本文基于reactor，实现简单的http协议封装。只是为了说明reactor如何做业务，真正的http服务器业务逻辑是很复杂的。

服务器网络这一层，如nginx、redis，核心是epoll，实现使用的是reactor。

## 1.http协议封装与reactor的关系

**reactor**包含内容

- reactor_run，event loop，通过epoll进行io检测；
- accept_cb，处理网络连接；
- recv_cb 接收客户端http请求
- send_cb 发送http响应

**应用层什么场景使用accept_cb?**

1. 对特定ip的访问进行限制；
2. 负载均衡的功能，就是将请求转发到哪台服务器进行处理。

如果http服务器没有特殊的需求，是不需要修改reactor中的accept_cb的。

## 2.如何在reactor基础上实现http

http使用的是reqeust-reply的模型进行通讯，客户端发送http请求，服务器处理请求并返回应答。http的传输层是基于TCP的。

http的tcp链接的生命周期：

1. accept_cb
2. recv_cb
3. send_cb

对于服务器，首先需要使用recv_cb接收http数据，通过拆包粘包得到一帧完整的http数据；之后对http数据进行解析处理；最后通过send_cb向客户端发送response。

## 3.实现简单的response

先实现一个最简单的response，对于所有http请求，直接回复一个response，内容是一段html。返回给客户端后，浏览器可以直接显示。只在send_cb中调用发送数据就可以了。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170719_80342.jpg)

```
int http_response(struct ntyevent *ev) {
    if (ev == NULL) return -1;
    memset(ev->buffer, 0, BUFFER_LENGTH);
    const char *html = "<html><head><title>hello http</title></head><body><H1>Cong</H1></body></html>\r\n\r\n";
    ev->length = sprintf(ev->buffer, 
        "HTTP/1.1 200 OK\r\n\
         Date: Sun, 30 Jan 2022 05:55:32 GMT\r\n\
         Content-Type: text/html;charset=UTF-8\r\n\
         Content-Length: 81\r\n\r\n%s", 
         html);
    return ev->length;
}
int send_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor*)arg;
    struct ntyevent *ev = ntyreactor_idx(reactor, fd);
    http_response(ev);
    //
    int len = send(fd, ev->buffer, ev->length, 0);
    if (len > 0) {
        printf("send[fd=%d], [%d]%s\n", fd, len, ev->buffer);
        nty_event_del(reactor->epfd, ev);
        nty_event_set(ev, fd, recv_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLIN, ev);
    } else {
        close(ev->fd);
        nty_event_del(reactor->epfd, ev);
        printf("send[fd=%d] error %s\n", fd, strerror(errno));
    }
    return len;
}
```

HTTP响应由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170720_41247.jpg)

## 4.http get请求实现

实现http get请求获取静态资源。

客户端发送一个HTTP 请求到服务器的请求消息包括以下格式：请求行（ request line ）、请求头部（ header ）、空行和请求数据四个部分组成，下图给出了请求报文的一般格式。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170720_79818.jpg)

## 5.读请求行

需要对http请求进行解析，判断请求类型(GET、POST等)，获取资源(uri)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170721_85399.jpg)

```
int http_request(struct ntyevent *ev) {
    // GET, POST
    char linebuf[1024] = {0};
    int idx = readline(ev->buffer, 0, linebuf);
    if (strstr(linebuf, "GET")) {
        ev->method = HTTP_METHOD_GET;
        //uri
        int i = 0;
        while (linebuf[sizeof("GET ") + i] != ' ') i++;
        linebuf[sizeof("GET ")+i] = '\0';
        sprintf(ev->resource, "./%s/%s", HTTP_WEBSERVER_HTML_ROOT, linebuf+sizeof("GET "));
    } else if (strstr(linebuf, "POST")) {
    }
}
```

在读取http请求头的时候，需要注意，http是以\r\n对每一行进行分隔的，第一行就是请求行，里面有请求类型(GET、POST)、uri、协议版本等。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170721_54382.jpg)

自定义协议，如何界定包的完整性：

1. 使用分隔符，比如http使用\r\n；
2. 协议头中加包的长度。

### 6.http response

根据上面的uri找到静态资源，并准备好response状态行和消息头。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170722_66982.jpg)

发送不同类型的静态资源的时候，可以使用相应的Content-Type。

**发送response**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215170724_56050.jpg)

### 7.mmap

发送静态资源文件的时候，需要先将文件读入内存，再将内存中的数据send到相应的网络fd。通过使用sendfile完成文件的发送，不再需要两步操作。

**sendfile使用mmap**

零拷贝，使用的是mmap方式，本质是DMA的方式，不需要CPU参与。普通copy，从磁盘copy数据到内存，需要CPU的move指令。

在进程中有一块区域叫内存分配区，当调用mmap的时候，会把文件映射到对应的区域，操作文件就跟操作内存一样。

### 8.完整代码

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
#include <time.h>
#include <sys/stat.h>
#include <sys/sendfile.h>
#define BUFFER_LENGTH        4096
#define MAX_EPOLL_EVENTS    1024
#define SERVER_PORT            9105
#define PORT_COUNT            1
#define HTTP_WEBSERVER_HTML_ROOT    "html"
#define HTTP_METHOD_GET        0
#define HTTP_METHOD_POST    1
typedef int NCALLBACK(int ,int, void*);
struct ntyevent {
    int fd;
    int events;
    void *arg;
    int (*callback)(int fd, int events, void *arg);
    int status;
    char buffer[BUFFER_LENGTH];
    int length;
    long last_active;
    // http param
    int method; //
    char resource[BUFFER_LENGTH];
    int ret_code;
};
struct eventblock {
    struct eventblock *next;
    struct ntyevent *events;
};
struct ntyreactor {
    int epfd;
    int blkcnt;
    struct eventblock *evblk; //fd --> 100w
};
int recv_cb(int fd, int events, void *arg);
int send_cb(int fd, int events, void *arg);
struct ntyevent *ntyreactor_idx(struct ntyreactor *reactor, int sockfd);
void nty_event_set(struct ntyevent *ev, int fd, NCALLBACK callback, void *arg) {
    ev->fd = fd;
    ev->callback = callback;
    ev->events = 0;
    ev->arg = arg;
    ev->last_active = time(NULL);
    return ;
}
int nty_event_add(int epfd, int events, struct ntyevent *ev) {
    struct epoll_event ep_ev = {0, {0}};
    ep_ev.data.ptr = ev;
    ep_ev.events = ev->events = events;
    int op;
    if (ev->status == 1) {
        op = EPOLL_CTL_MOD;
    } else {
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
int readline(char *allbuf, int idx, char *linebuf) {
    int len = strlen(allbuf);
    for(;idx < len;idx ++) {
        if (allbuf[idx] == '\r' && allbuf[idx+1] == '\n') {
            return idx+2;
        } else {
            *(linebuf++) = allbuf[idx];
        }
    }
    return -1;
}
int http_request(struct ntyevent *ev) {
    // GET, POST
    char linebuf[1024] = {0};
    int idx = readline(ev->buffer, 0, linebuf);
    if (strstr(linebuf, "GET")) {
        ev->method = HTTP_METHOD_GET;
        //uri
        int i = 0;
        while (linebuf[sizeof("GET ") + i] != ' ') i++;
        linebuf[sizeof("GET ")+i] = '\0';
        sprintf(ev->resource, "./%s/%s", HTTP_WEBSERVER_HTML_ROOT, linebuf+sizeof("GET "));
    } else if (strstr(linebuf, "POST")) {
    }
}
int recv_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor*)arg;
    struct ntyevent *ev = ntyreactor_idx(reactor, fd);
    int len = recv(fd, ev->buffer, BUFFER_LENGTH, 0); // 
    if (len > 0) {
        ev->length = len;
        ev->buffer[len] = '\0';
        printf("C[%d]:%s\n", fd, ev->buffer); //http
        http_request(ev);
        //send();
        nty_event_del(reactor->epfd, ev);
        nty_event_set(ev, fd, send_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLOUT, ev);
    } else if (len == 0) {
        nty_event_del(reactor->epfd, ev);
        close(ev->fd);
        //printf("[fd=%d] pos[%ld], closed\n", fd, ev-reactor->events);
    } else {
        nty_event_del(reactor->epfd, ev);
        close(ev->fd);
        printf("recv[fd=%d] error[%d]:%s\n", fd, errno, strerror(errno));
    }
    return len;
}
int http_response(struct ntyevent *ev) {
    if (ev == NULL) return -1;
    memset(ev->buffer, 0, BUFFER_LENGTH);
#if 0
    const char *html = "<html><head><title>hello http</title></head><body><H1>Cong</H1></body></html>\r\n\r\n";
    ev->length = sprintf(ev->buffer, 
        "HTTP/1.1 200 OK\r\n\
         Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n\
         Content-Type: text/html;charset=ISO-8859-1\r\n\
         Content-Length: 83\r\n\r\n%s", 
         html);
#else
    printf("resource: %s\n", ev->resource);
    int filefd = open(ev->resource, O_RDONLY);
    if (filefd == -1) { // return 404
        ev->ret_code = 404;
        ev->length = sprintf(ev->buffer, 
            "HTTP/1.1 404 Not Found\r\n"
             "Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
             "Content-Type: text/html;charset=ISO-8859-1\r\n"
            "Content-Length: 85\r\n\r\n"
             "<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n" );
    } else {
        struct stat stat_buf;
        fstat(filefd, &stat_buf);
        close(filefd);
        if (S_ISDIR(stat_buf.st_mode)) {
            ev->ret_code = 404;
            ev->length = sprintf(ev->buffer, 
                "HTTP/1.1 404 Not Found\r\n"
                "Date: Sun, 30 Jan 2022 05:55:32 GMT\r\n"
                "Content-Type: text/html;charset=ISO-8859-1\r\n"
                "Content-Length: 85\r\n\r\n"
                "<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n" );
        } else if (S_ISREG(stat_buf.st_mode)) {
            ev->ret_code = 200;
            ev->length = sprintf(ev->buffer, 
                "HTTP/1.1 200 OK\r\n"
                 "Date: Sun, 30 Jan 2022 05:55:32 GMT\r\n"
                 // "Content-Type: text/html;charset=ISO-8859-1\r\n"
                // "Content-Type: application/pdf\r\n"
                "Content-Type: image/jpeg\r\n"
                "Content-Length: %ld\r\n\r\n", 
                     stat_buf.st_size );
        }
    }
#endif
    return ev->length;
}
int send_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor*)arg;
    struct ntyevent *ev = ntyreactor_idx(reactor, fd);
    http_response(ev);
    //
    int len = send(fd, ev->buffer, ev->length, 0);
    if (len > 0) { // http header和body分两次发送
        printf("send[fd=%d], [%d]%s\n", fd, len, ev->buffer);
        if (ev->ret_code == 200) {
            int filefd = open(ev->resource, O_RDONLY);
            struct stat stat_buf;
            fstat(filefd, &stat_buf);
            // sendfile使用mmap
            sendfile(fd, filefd, NULL, stat_buf.st_size);
            close(filefd);
        }
        nty_event_del(reactor->epfd, ev);
        nty_event_set(ev, fd, recv_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLIN, ev);
    } else {
        close(ev->fd);
        nty_event_del(reactor->epfd, ev);
        printf("send[fd=%d] error %s\n", fd, strerror(errno));
    }
    return len;
}
int accept_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor*)arg;
    if (reactor == NULL) return -1;
    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);
    int clientfd;
    if ((clientfd = accept(fd, (struct sockaddr*)&client_addr, &len)) == -1) {
        if (errno != EAGAIN && errno != EINTR) {
        }
        printf("accept: %s\n", strerror(errno));
        return -1;
    }
    int flag = 0;
    if ((flag = fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0) {
        printf("%s: fcntl nonblocking failed, %d\n", __func__, MAX_EPOLL_EVENTS);
        return -1;
    }
    struct ntyevent *event = ntyreactor_idx(reactor, clientfd);
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
    bind(fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
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
    struct ntyevent *evs = (struct ntyevent*)malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    if (evs == NULL) {
        printf("ntyreactor_alloc ntyevents failed\n");
        return -2;
    }
    memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    struct eventblock *block = (struct eventblock *)malloc(sizeof(struct eventblock));
    if (block == NULL) {
        printf("ntyreactor_alloc eventblock failed\n");
        return -2;
    }
    memset(block, 0, sizeof(struct eventblock));
    block->events = evs;
    block->next = NULL;
    blk->next = block;
    reactor->blkcnt ++; //
    return 0;
}
struct ntyevent *ntyreactor_idx(struct ntyreactor *reactor, int sockfd) {
    int blkidx = sockfd / MAX_EPOLL_EVENTS;
    while (blkidx >= reactor->blkcnt) {
        ntyreactor_alloc(reactor);
    }
    int i = 0;
    struct eventblock *blk = reactor->evblk;
    while(i ++ < blkidx && blk != NULL) {
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
    struct ntyevent *evs = (struct ntyevent*)malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    if (evs == NULL) {
        printf("ntyreactor_alloc ntyevents failed\n");
        return -2;
    }
    memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    struct eventblock *block = (struct eventblock *)malloc(sizeof(struct eventblock));
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
    //reactor->evblk->events[sockfd];
    struct ntyevent *event = ntyreactor_idx(reactor, sockfd);
    nty_event_set(event, sockfd, acceptor, reactor);
    nty_event_add(reactor->epfd, EPOLLIN, event);
    return 0;
}
int ntyreactor_run(struct ntyreactor *reactor) {
    if (reactor == NULL) return -1;
    if (reactor->epfd < 0) return -1;
    if (reactor->evblk == NULL) return -1;
    struct epoll_event events[MAX_EPOLL_EVENTS+1];
    int checkpos = 0, i;
    while (1) {
/*
        long now = time(NULL);
        for (i = 0;i < 100;i ++, checkpos ++) {
            if (checkpos == MAX_EPOLL_EVENTS) {
                checkpos = 0;
            }
            if (reactor->events[checkpos].status != 1) {
                continue;
            }
            long duration = now - reactor->events[checkpos].last_active;
            if (duration >= 60) {
                close(reactor->events[checkpos].fd);
                printf("[fd=%d] timeout\n", reactor->events[checkpos].fd);
                nty_event_del(reactor->epfd, &reactor->events[checkpos]);
            }
        }
*/
        int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);
        if (nready < 0) {
            printf("epoll_wait error, exit\n");
            continue;
        }
        for (i = 0;i < nready;i ++) {
            struct ntyevent *ev = (struct ntyevent*)events[i].data.ptr;
            if ((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
            if ((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }
}
// 3, 6w, 1, 100 == 
// <remoteip, remoteport, localip, localport>
int main(int argc, char *argv[]) {
    unsigned short port = SERVER_PORT; // listen 8888
    if (argc == 2) {
        port = atoi(argv[1]);
    }
    struct ntyreactor *reactor = (struct ntyreactor*)malloc(sizeof(struct ntyreactor));
    ntyreactor_init(reactor);
    int i = 0;
    int sockfds[PORT_COUNT] = {0};
    for (i = 0;i < PORT_COUNT;i ++) {
        sockfds[i] = init_sock(port+i);
        ntyreactor_addlistener(reactor, sockfds[i], accept_cb);
    }
    //dup2(sockfd, STDIN);
    ntyreactor_run(reactor);
    ntyreactor_destory(reactor);
    for (i = 0;i < PORT_COUNT;i ++) {
        close(sockfds[i]);
    }
    free(reactor);
    return 0;
}
nts[i].events, ev->arg);
            }
        }
    }
}
// 3, 6w, 1, 100 == 
// <remoteip, remoteport, localip, localport>
int main(int argc, char *argv[]) {
    unsigned short port = SERVER_PORT; // listen 8888
    if (argc == 2) {
        port = atoi(argv[1]);
    }
    struct ntyreactor *reactor = (struct ntyreactor*)malloc(sizeof(struct ntyreactor));
    ntyreactor_init(reactor);
    int i = 0;
    int sockfds[PORT_COUNT] = {0};
    for (i = 0;i < PORT_COUNT;i ++) {
        sockfds[i] = init_sock(port+i);
        ntyreactor_addlistener(reactor, sockfds[i], accept_cb);
    }
    //dup2(sockfd, STDIN);
    ntyreactor_run(reactor);
    ntyreactor_destory(reactor);
    for (i = 0;i < PORT_COUNT;i ++) {
        close(sockfds[i]);
    }
    free(reactor);
    return 0;
}
```

原文链接：https://zhuanlan.zhihu.com/p/496676483

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)