# 【NO.447】网络IO管理-简单一问一答、多线程方式

## 1.思考：

1. 那网络中进程之间如何通信，浏览器的进程怎么与web服务器通信的？

2. 什么时候用一请求一线程的方式？

3. 什么时候用select/poll？

4. 什么时候用epoll?

## 2.准备工作

下面展示socket几个常用的函数`listenfd, bind, listen, accept具体作用`。

```
// 聘请迎宾的小姐姐
if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
    printf("create socket error: %s(errno: %d)\n", strerror(errno), errno);
    return 0;
}
// 迎宾的小姐姐在哪个门口工作
if (bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) == -1) {
        printf("bind socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
}
// 小姐姐正式走到门口开始迎宾
if (listen(listenfd, 10) == -1) {
       printf("listen socket error: %s(errno: %d)\n", strerror(errno), errno);
       return 0;
}
// 把客户带到大厅介绍一个服务员,coonfd就是服务员
if ((connfd = accept(listenfd, (struct sockaddr *)&client, &len)) == -1) {
    printf("accept socket error: %s(errno: %d)\n", strerror(errno), errno);
    return 0;
}
```

**网络连接的简单过程**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163030_58013.jpg)

## 3.网络IO模型

### 3.1.阻塞IO

#### 3.1.1. 简单一问一答服务器/客户机模型

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163030_75124.jpg)

当用户进程调用了 read 这个系统调用，kernel 就开始了 IO 的第一个阶段：准备数据。对于network io 来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的数据包），这个时候 kernel 就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞。当 kernel一直等到数据准备好了它就会将数据从 kernel 中拷贝到用户内存，然后 kernel 返回结果，用户进程才解除 block 的状态，重新运行起来。所以，blocking IO 的特点就是在 IO 执行的两个阶段（等待数据和拷贝数据两个阶段）都被block 了。

几乎所有的程序员第一次接触到的网络编程都是从 listen()、send()、recv() 等接口开始的，这些接口都是阻塞型的。使用这些接口可以很方便的构建服务器/客户机的模型。

**代码展示**

```
#include <errno.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#define MAXLNE  4096
int main(int argc, char **argv) 
{
    int listenfd, connfd, n;
    struct sockaddr_in servaddr;
    char buff[MAXLNE];
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        printf("create socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(9999);
    if (bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) == -1) {
        printf("bind socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    if (listen(listenfd, 10) == -1) {
        printf("listen socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    struct sockaddr_in client;
    socklen_t len = sizeof(client);
    if ((connfd = accept(listenfd, (struct sockaddr *)&client, &len)) == -1) {
        printf("accept socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    printf("========waiting for client's request========\n");
    while (1) {
        n = recv(connfd, buff, MAXLNE, 0);
        if (n > 0) {
            buff[n] = '\0';
            printf("recv msg from client: %s\n", buff);
        send(connfd, buff, n, 0);
        } else if (n == 0) {
            close(connfd);
        }
        //close(connfd);
    }
    close(listenfd);
    return 0;
}
```

#### 3.2.2. 多线程方式

**提出问题**

实际上，除非特别指定，几乎所有的 IO 接口 ( 包括 socket 接口 ) 都是阻塞型的。这给网络编程带来了一个很大的问题，如在调用 send()的同时，线程将被阻塞，在此期间，线程将无法执行任何运算或响应任何的网络请求。

**解决方案**

改进方案是在服务器端使用多线程（或多进程）。多线程（或多进程）的目的是让每个连接都拥有独立的线程（或进程），这样任何一个连接的阻塞都不会影响其他的连接。具体使用多进程还是多线程并没有一个特定的模式。

**适用场景与注意的问题**

户端不多的情况下，在一个局域网内，做几个人的会议这类的，理论上客户端很难突破C10k，使用 pthread_create ()创建新线程。

多线程的服务器模型似乎完美的解决了为多个客户机提供问答服务的要求，但其实并不尽然。如果要同时响应成百上千路的连接请求，则无论多线程还是多进程都会严重占据系统资源，降低系统对外界响应效率，而线程与进程本身也更容易进入假死状态。

这时候很多人会考虑使用“线程池”或“连接池”。这里就需要深刻理解一下其概念，“线程池”旨在减少创建和销毁线程的频率，其维持一定合理数量的线程，并让空闲的线程重新承担新的执行任务。“连接池”维持连接的缓存池，尽量重用已有的连接、减少创建和关闭连接的频率。所谓“池”始终有其上限，当请求大大超过上限时，“池”构成的系统对外界的响应并不比没有池的时候效果好多少。所以使用“池”必须考虑其面临的响应规模，并根据响应规模调整“池”的大小。

**代码展示**

```
#include <errno.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <pthread.h>
#define MAXLNE  4096
//8m * 4G = 128 , 512
//C10k
void *client_routine(void *arg) { //
    int connfd = *(int *)arg;
    char buff[MAXLNE];
    while (1) {
        int n = recv(connfd, buff, MAXLNE, 0);
        if (n > 0) {
            buff[n] = '\0';
            printf("recv msg from client: %s\n", buff);
            send(connfd, buff, n, 0);
        } else if (n == 0) {
            close(connfd);
            break;
        }
    }
    return NULL;
}
int main(int argc, char **argv) 
{
    int listenfd, connfd, n;
    struct sockaddr_in servaddr;
    char buff[MAXLNE];
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        printf("create socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(9999);
    if (bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) == -1) {
        printf("bind socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    if (listen(listenfd, 10) == -1) {
        printf("listen socket error: %s(errno: %d)\n", strerror(errno), errno);
        return 0;
    }
    printf("========waiting for client's request========\n");
    while (1) {
        struct sockaddr_in client;
        socklen_t len = sizeof(client);
        if ((connfd = accept(listenfd, (struct sockaddr *)&client, &len)) == -1) {
            printf("accept socket error: %s(errno: %d)\n", strerror(errno), errno);
            return 0;
        }
        pthread_t threadid;
        pthread_create(&threadid, NULL, client_routine, (void*)&connfd);
    }
    close(listenfd);
    return 0;
}
```

**理解多线程方式不足**

假设有A, B, C三个客户端连接服务器，其中A, B发送数据，C连接进来。A, B服务员正在服务，C迎宾的小姐姐迎接。这时候服务器没办法准确的预测哪个fd会有数据来临。大量的客户端链接进来，需处理多个客户端，我们不知道哪个客户端需要我们处理。这时，如果有一个组件把所有的fd放在一起，准确的知道哪个客户端需要我们处理。

原文链接：https://zhuanlan.zhihu.com/p/492069172

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)