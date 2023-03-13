# 【NO.202】HTTP—TCP/IP—SOCKET理解及浅析

## 1.一个完整的HTTP请求的过程

此举例为抛砖引玉，引导大家进入思考状态。

当你按输入[http://www.baidu.com](https://link.zhihu.com/?target=http%3A//www.baidu.com) ，浏览器接收到这个消息之后，浏览器根据自己的算法识别出你要访问的URL,为您展示出来搜索页面和**广告**，那么这些经历了哪些过程呢？

### **1.1.大致过程如下：**

- （1）浏览器查询 DNS，获取域名对应的IP地址； 具体过程包括浏览器搜索自身的DNS缓存、搜索操作系统的DNS缓存、读取本地的Host文件和向本地DNS服 务器进行查询等。
- （2）浏览器获得域名对应的IP地址以后，浏览器向服务器请求建立链接，发起三次握手；
- （3）TCP/IP链接建立起来后，浏览器向服务器发送HTTP请求；
- （4）服务器接收到这个请求，并根据路径参数映射到特定的请求处理器进行处理，并将处理结果及相应的视图返回给浏览器；
- （5）浏览器解析并渲染视图，若遇到对js文件、css文件及图片等静态资源的引用，则重复上述步骤并向服务器请求这些资源；
- （6）浏览器根据其请求到的资源、数据渲染页面，最终向用户呈现一个完整的页面。

**下面，我们从底到上来一层层理解这个问题。**

## 2.网络参考模型

**开放式系统互联通信参考模型**（英语：**O**pen **S**ystem **I**nterconnection Reference Model，缩写：OSI；简称为**OSI模型**）是一种概念模型，由国际标准化组织提出，一个试图使各种计算机在世界范围内互连为网络的标准框架。定义于ISO/IEC 7498-1。（摘自维基百科）

|  7   |  应用层 application layer  | 例如HTTP、SMTP、SNMP、FTP、Telnet、SIP、SSH、NFS、RTSP、XMPP、Whois、ENRP、TLS |
| :--: | :------------------------: | :----------------------------------------------------------: |
|  6   | 表示层 presentation layer  |                例如XDR、ASN.1、SMB、AFP、NCP                 |
|  5   |    会话层 session layer    | 例如ASAP、ISO 8327 / CCITT X.225、RPC、NetBIOS、ASP、IGMP、Winsock、BSD sockets |
|  4   |   传输层 transport layer   |            例如TCP、UDP、RTP、SCTP、SPX、ATP、IL             |
|  3   |    网络层 network layer    | 例如IP、ICMP、IPX、BGP、OSPF、RIP、IGRP、EIGRP、ARP、RARP、X.25 |
|  2   | 数据链路层 data link layer | 例如以太网、令牌环、HDLC、帧中继、ISDN、ATM、IEEE 802.11、FDDI、PPP |
|  1   |   物理层 physical layer    |                    例如线路、无线电、光纤                    |

### **2.1.通常人们认为OSI模型的最上面三层（应用层、表示层和会话层）在TCP/IP组中是一个应用层。**

由于TCP/IP有一个相对较弱的会话层，由TCP和RTP下的打开和关闭连接组成，并且在TCP和UDP下的各种应用提供不同的端口号，这些功能能够被单个的应用程序（或者那些应用程序所使用的库）增加。与此相似的是，IP是按照将它下面的网络当作一个黑盒子的思想设计的，这样在讨论TCP/IP的时候就可以把它当作一个独立的层。

### **2.2.TCP/IP 参考模型**

|  4   |          应用层 application layer           | 例如HTTP、FTP、DNS （如BGP和RIP这样的路由协议，尽管由于各种各样的原因它们分别运行在TCP和UDP上，仍然可以将它们看作网络层的一部分） |
| :--: | :-----------------------------------------: | :----------------------------------------------------------: |
|  3   |           传输层 transport layer            | 例如TCP、UDP、RTP、SCTP （如OSPF这样的路由协议，尽管运行在IP上也可以看作是网络层的一部分） |
|  2   |          网络互连层 internet layer          | 对于TCP/IP来说这是因特网协议（IP） （如ICMP和IGMP这样的必须协议尽管运行在IP上，也仍然可以看作是网络互连层的一部分；ARP不运行在IP上） |
|  1   | 网络访问(链接)层 Network Access(link) layer |                 例如以太网、Wi-Fi、MPLS等。                  |

**下面一张图更有助于你的理解**

![img](https://pic4.zhimg.com/80/v2-34632647cb55abaf0dc9fedd7e75cb2f_720w.webp)

## 3.HTTP 协议与 TCP/IP 协议

**HTTP 是 TCP/IP 参考模型中应用层的其中一种实现。**HTTP 协议的网络层基于 IP 协议，传输层基于 TCP 协议：HTTP 协议是基于 TCP/IP 协议的应用层协议。

TCP/IP 协议需要向程序员提供可编程的 API，该 API 就是 Socket，它是对 TCP/IP 协议的一个重要的实现，几乎所有的计算机系统都提供了对 TCP/IP 协议族的 Socket 实现。

**Socket是进程通讯的一种方式，即调用这个网络库的一些API函数实现分布在不同主机的相关进程之间的数据交换。**

- 流格式套接字（SOCK_STREAM） 流格式套接字（Stream Sockets）也叫“面向连接的套接字”，它基于 TCP 协议，在代码中使用 SOCK_STREAM 表示。
- 数据报格式套接字（SOCK_DGRAM） 数据报格式套接字（Datagram Sockets）也叫“无连接的套接字”，基于 UDP 协议，在代码中使用 SOCK_DGRAM 表示。 TCP与UDP 协议区别与优劣势 TCP 是面向连接的传输协议，建立连接时要经过三次握手，断开连接时要经过四次握手，中间传输数据时也要回复 ACK 包确认，多种机制保证了数据能够正确到达，不会丢失或出错。 UDP 是非连接的传输协议，没有建立连接和断开连接的过程，它只是简单地把数据丢到网络中，也不需要 ACK 包确认。 如果只考虑可靠性，TCP 的确比 UDP 好。但 UDP 在结构上比 TCP 更加简洁，不会发送 ACK 的应答消息， 也不 会给数据包分配 Seq 序号，所以 UDP 的传输效率有时会比 TCP 高出很多，编程中实现 UDP 也比 TCP 简单。 与 UDP 相比，TCP 的生命在于流控制，这保证了数据传输的正确性。

最后需要说明的是：TCP 的速度无法超越 UDP，但在收发某些类型的数据时有可能接近 UDP。例如，每次交换的数据量越大，TCP 的传输速率就越接近于 UDP。

## 4.TCP/IP 协议、HTTP 协议和 Socket 有什么区别？

**从包含范围来看，它们的继承关系是这样的：**

![img](https://pic1.zhimg.com/80/v2-d159a33b45ea76b184581c8b102f2a5c_720w.webp)

**从横向来看，它们的继承关系是这样的：**

![img](https://pic1.zhimg.com/80/v2-400a8532c1150d5162128c050e90d0b4_720w.webp)

关于TCP/IP和HTTP协议的关系，有一段比较容易理解的介绍：

**我们在传输数据时，可以只使用（传输层）TCP/IP协议，但是那样的话，如果没有应用层，便无法识别数据内容，如果想要使传输的数据有意义，则必须使用到应用层协议，应用层协议有很多，比如HTTP、FTP、TELNET等，也可以自己定义应用层协议。WEB使用HTTP协议作应用层协议，以封装HTTP文本信息，然后使用TCP/IP做传输层协议将它发到网络上。**

Socket是什么呢，实际上S**ocket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议。**

TCP/IP只是一个协议栈，就像操作系统的运行机制一样，必须要具体实现，同时还要提供对外的操作接口。这个就像操作系统会提供标准的编程接口，比如win32编程接口一样，TCP/IP也要提供可供程序员做网络开发所用的接口，这就是Socket编程接口。”

## 5.TCP/IP 和 HTTP 的数据结构

HTTP 作为 TCP/IP 参考模型的应用层，把 HTTP 放到 TCP/IP 参考模型中，它们的继承结构是这样的：

![img](https://pic3.zhimg.com/80/v2-874e1a93f2a12ef9f29f01bfd7150472_720w.webp)

在 TCP/IP 参考模型中它们的整体的数据结构是：IP 作为以太网的直接底层，IP 的头部和数据合起来作为以太网的数据，同样的 TCP/UDP 的头部和数据合起来作为 IP 的数据，HTTP 的头部和数据合起来作为 TCP/UDP 的数据。

![img](https://pic2.zhimg.com/80/v2-f8f09846c3cb3a85110aa8c476f2ebad_720w.webp)

**IP 的数据结构和交互流程**

我们都知道在一个成功的 HTTP 请求中，服务端可以在一个请求中获取到客户端 IP 地址，也可以获取到客户端请求的主机的 IP 地址。然而这是怎么做到的呢？这就有赖于 IP 协议了，在 IP 协议中规定了，IP 的头部必须包含源 IP 地址和目的 IP 地址，这也是为什么在 TCP/IP 参考模型中IP 处在网络互联层，其中一个原因就是可以定位服务端地址和客户端地址，我们来看一下 IP 的数据结构：

![img](https://pic2.zhimg.com/80/v2-543fe5bc79e1464ccaafb795822c4b31_720w.webp)

可以很清晰的看到源 IP 地址和目的 IP 地址，在 IP 的头部各占 32 位，而 IPV4 的 IP 地址是用点式十进制表示的，例如：192.168.1.1，在 IP 头部用二进制表示的话，刚好是 4 个字节 32 位。

32 位可以表示的 IP 地址是有限的，使用了 IP 地址转换技术 NAT。例如 ABC 三个小区的所有设备可能公用了一个**公网 IP**，通过 **NAT 技术分给每一户一个私有 IP** 地址，大家在小区内交流时可能使用的是私有 IP 地址，但是向外交流时就用公网 IP。

## 6.TCP 的数据结构和交互流程

我们通常说的 HTTP 的 3 次握手和 4 次挥手都是由 TCP 来完成的，其实这都没 HTTP 什么事，但是有不少人喜欢这么说，严格来说我们应该说 TCP 的 3 次握手 4 次挥手。要搞清楚 TCP 的交互流程，首先要清楚 TCP 的数据结构，接下来我们来看一下 TCP 的数据结构：

![img](https://pic3.zhimg.com/80/v2-8f58bf59758005dc89d9d61c385a09b6_720w.webp)

上述 TCP 的数据结构图对于后面理解 HTTP 的交互流程非常重要，我们要记住 5 个关键的位置：

> SYN：建立连接标识 ACK：响应标识 FIN：断开连接标识 seq：seq number，发送序号 ack：ack number，响应序号

服务端应用启动后，会在指定端口监听客户端的连接请求，当客户端尝试创建一个到服务端指定端口的 TCP 连接，服务端收到请求后接受数据并处理完业务后，会向客户端作出响应，客户端收到响应后接受响应数据，然后断开连接，一个完整的请求流程就完成了。这样的一个完整的 TCP 的**生命周期会经历以下 4 个步骤**：

> 1,建立 TCP 连接，3 次握手
>
> 客户端发送SYN, seq=x，进入 SYN_SEND 状态
>
> 服务端回应SYN, ACK, seq=y, ack=x+1，进入 SYN_RCVD 状态
>
> 客户端回应ACK, seq=x+1, ack=y+1，进入 ESTABLISHED 状态，服务端收到后进入 ESTABLISHED 状态 2,进行数据传输
>
> 客户端发送ACK, seq=x+1, ack=y+1, len=m
>
> 服务端回应ACK, seq=y+1, ack=x+m+1, len=n
>
> 客户端回应ACK, seq=x+m+1, ack=y+n+1
>
> 3,断开 TCP 连接， 4 次挥手
>
> 主机 A 发送FIN, ACK, seq=x+m+1, ack=y+n+1，进入 FNI_WAIT_1 状态
>
> 主机 B 回应ACK, seq=y+n+1, ack=x+m+1，进入 CLOSE_WAIT 状态，主机 A 收到后 进入 FIN_WAIT_2 状态
>
> 主机 B 发送FIN, ACK, seq=y+n+1, ack=x+m+1，进入 LAST_ACK 状态
>
> 主机 A 回应ACk, seq=x+m+1, ack=y+n+1，进入 TIME_WAIT 状态，等待主机 B 可能要求重传 ACK 包，主机 B 收到后关闭连接，进入 CLOSED 状态或者要求主机 A 重传 ACK，客户端在一定的时间内没收到主机 B 重传 ACK 包的要求后，断开连接进入 CLOSED 状态
>
> 为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？
>
> 虽然按道理，四个报文都发送完毕，我们可以直接进入CLOSE状态了，但是我们必须假设网络是不可靠的，一切都可能发生，比如有可能最后一个ACK丢失。所以TIME_WAIT状态是用来重发可能丢失的ACK报文。

![img](https://pic2.zhimg.com/80/v2-246cd0f85cb90edb426e1d3647deeba5_720w.webp)

客户端与服务端建立连接、传输数据和断开连接等全靠这几个标识，比如 SYN 也可以被用来作为 DOS 攻击的一个手段，FIN 可以用来扫描服务端指定端口。

## 7.HTTP 的数据结构

Socket 是 TCP/IP 的可编程 API，HTTP 的可编程 API 的实现要依赖 Socket。HTTP 是超文本传输协议，HTTP 的头和数据看起来更加直观，在大多数情况下，它们都是字符或者字符串，所以对于大多数人来说理解 HTTP 的头和数据格式显得很简单。确实，HTTP 的数据格式理解起来非常容易，上部分是头，下部分是身体。

HTTP 的请求时的数据结构和响应时的数据结构整体上是一样的，但是有一些细微的区别，我们先来看一下 HTTP 请求时的数据结构：

![img](https://pic2.zhimg.com/80/v2-eabaf2ef62bebebd61b8b4eff18eeff5_720w.webp)

HTTP 响应时的数据结构：

![img](https://pic2.zhimg.com/80/v2-8273bb87cca06d1bf9fe745897991109_720w.webp)

现在我们使用谷歌浏览器请求某度，按下Ｆ１２，来对比理解上述结构图，下面是请求某度

![img](https://pic2.zhimg.com/80/v2-173efc2744ae1de476fe58fa93dcc989_720w.webp)

我们就可以简单的理解 HTTP 的数据结构了。

## 8.Linux下的socket演示程序

下面用最基础的Socket来进行服务端与客户端的交互，让你理解的更为清晰。

**接口详解**：

|    方法名    |                     用途                      |
| :----------: | :-------------------------------------------: |
|  socket()：  |                  创建socket                   |
|   bind()：   | 绑定socket到本地地址和端口，通常由服务端调用  |
|  listen()：  |             TCP专用，开启监听模式             |
|  accept()：  |  TCP专用，服务器等待客户端连接，一般是阻塞态  |
| connect()：  |         TCP专用，客户端主动连接服务器         |
|   send()：   |               TCP专用，发送数据               |
|   recv()：   |               TCP专用，接收数据               |
|  sendto()：  |     UDP专用，发送数据到指定的IP地址和端口     |
| recvfrom()： | UDP专用，接收数据，返回数据远端的IP地址和端口 |
|  close()：   |                  关闭socket                   |

## 9.基于TCP协议实现CS端

使用Socket进行网络通信的过程

① 服务器程序将一个套接字绑定到一个特定的端口，并通过此套接字等待和监听客户的连接请求。

② 客户程序根据服务器程序所在的主机和端口号发出连接请求。

③ 如果一切正常，服务器接受连接请求。并获得一个新的绑定到不同端口地址的套接字。

④ 客户和服务器通过读、写套接字进行通讯。

![img](https://pic4.zhimg.com/80/v2-34899140a3b1f807c9a95d22d7e318d3_720w.webp)

**客户机/服务器模式**

在TCP/IP网络应用中，通信的两个进程间相互作用的主要模式是客户机/服务器模式*(client/server)，即客户像服务其提出请求，服务器接受到请求后，提供相应的服务。

服务器：

（1）首先服务器方要先启动，打开一个通信通道并告知本机，它愿意在某一地址和端口上接收客户请求

（2）等待客户请求到达该端口

（3）接收服务请求，处理该客户请求，服务完成后，关闭此新进程与客户的通信链路，并终止

（4）返回第二步，等待另一个客户请求

（5）关闭服务器

**客户方：**

（1）打开一个通信通道，并连接到服务器所在的主机特定的端口

（2）向服务器发送请求，等待并接收应答，继续提出请求

（3）请求结束后关闭通信信道并终止

具体实现，新建服务端socket_server_tcp.c

具体代码如下： socket_server_tcp.c

```
//
// Created by android on 19-8-9.
//
#include <stdio.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#define PORT 3040        //端口号
#define BACKLOG 5    //最大监听数
int main() {
    int iSocketFD = 0;  //socket句柄
    int iRecvLen = 0;   //接收成功后的返回值
    int new_fd = 0;    //建立连接后的句柄
    char buf[4096] = {0}; //
    struct sockaddr_in stLocalAddr = {0}; //本地地址信息结构图，下面有具体的属性赋值
    struct sockaddr_in stRemoteAddr = {0}; //对方地址信息
    socklen_t socklen = 0;
    iSocketFD = socket(AF_INET, SOCK_STREAM, 0); //建立socket SOCK_STREAM代表以tcp方式进行连接
    if (0 > iSocketFD) {
        printf("创建socket失败！\n");
        return 0;
    }
    stLocalAddr.sin_family = AF_INET;  /*该属性表示接收本机或其他机器传输*/
    stLocalAddr.sin_port = htons(PORT); /*端口号*/
    stLocalAddr.sin_addr.s_addr = htonl(INADDR_ANY); /*IP，括号内容表示本机IP*/
    //绑定地址结构体和socket
    if (0 > bind(iSocketFD, (void *) &stLocalAddr, sizeof(stLocalAddr))) {
        printf("绑定失败！\n");
        return 0;
    }
    //开启监听 ，第二个参数是最大监听数
    if (0 > listen(iSocketFD, BACKLOG)) {
        printf("监听失败！\n");
        return 0;
    }
    printf("iSocketFD: %d\n", iSocketFD);
    //在这里阻塞知道接收到消息，参数分别是socket句柄，接收到的地址信息以及大小 
    new_fd = accept(iSocketFD, (void *) &stRemoteAddr, &socklen);
    if (0 > new_fd) {
        printf("接收失败！\n");
        return 0;
    } else {
        printf("接收成功！\n");
        //发送内容，参数分别是连接句柄，内容，大小，其他信息（设为0即可） 
        send(new_fd, "这是服务器接收成功后发回的信息!", sizeof("这是服务器接收成功后发回的信息!"), 0);
    }
    printf("new_fd: %d\n", new_fd);
    iRecvLen = recv(new_fd, buf, sizeof(buf), 0);
    if (0 >= iRecvLen)    //对端关闭连接 返回0
    {
        printf("对端关闭连接或者接收失败！\n");
    } else {
        printf("buf: %s\n", buf);
    }
    close(new_fd);
    close(iSocketFD);
    return 0;
}
```

新建客户端端socket_client_tcp.c socket_client_tcp.c

```
//
// Created by android on 19-8-9.
//
#include <stdio.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#define PORT 3040            //目标地址端口号
#define ADDR "10.6.191.177" //目标地址IP
int main() {
    int iSocketFD = 0; //socket句柄
    unsigned int iRemoteAddr = 0;
    struct sockaddr_in stRemoteAddr = {0}; //对端，即目标地址信息
    socklen_t socklen = 0;
    char buf[4096] = {0}; //存储接收到的数据
    iSocketFD = socket(AF_INET, SOCK_STREAM, 0); //建立socket
    if (0 > iSocketFD) {
        printf("创建socket失败！\n");
        return 0;
    }
    stRemoteAddr.sin_family = AF_INET;
    stRemoteAddr.sin_port = htons(PORT);
    inet_pton(AF_INET, ADDR, &iRemoteAddr);
    stRemoteAddr.sin_addr.s_addr = iRemoteAddr;
    //连接方法： 传入句柄，目标地址，和大小
    if (0 > connect(iSocketFD, (void *) &stRemoteAddr, sizeof(stRemoteAddr))) {
        printf("连接失败！\n");
        //printf("connect failed:%d",errno);//失败时也可打印errno
    } else {
        printf("连接成功！\n");
        recv(iSocketFD, buf, sizeof(buf), 0); ////将接收数据打入buf，参数分别是句柄，储存处，最大长度，其他信息（设为0即可）。 
        printf("Received:%s\n", buf);
    }
    close(iSocketFD);//关闭socket
    return 0;
}
```

下面是我的编译及运行效果：

![img](https://pic2.zhimg.com/80/v2-921c7544e7a415e18022a60e0f631f9d_720w.webp)

编译命令如下：

```
gcc -o server socket_server_tcp.c
 gcc -o client socket_client_tcp.c
 #运行命令
  ./server  #首先启动
  ./client #次之启动
```

## 10.基于UDP协议实现CS端

**基于UDP（面向无连接）的socket编程——**数据报式套接字（SOCK_DGRAM) 网络间通信AF_INET，典型的TCP/IP四型模型的通信过程

服务器：（多线程的【每10秒会打印一行#号】 与 循环监听） socket_server_udp.c

```
//
// Created by android on 19-8-9.
//
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <pthread.h>
void * test(void *pvData)
{
    while(1)
    {
        sleep(５);
        printf("################################\n");
    }
    return NULL;
}
int main(void)
{
    pthread_t stPid = 0;
    int iRecvLen = 0;
    int iSocketFD = 0;
    char acBuf[4096] = {0};
    struct sockaddr_in stLocalAddr = {0};
    struct sockaddr_in stRemoteAddr = {0};
    socklen_t iRemoteAddrLen = 0;
    /* 创建socket */
    iSocketFD = socket(AF_INET, SOCK_DGRAM, 0);
    if(iSocketFD < 0)
    {
        printf("创建socket失败!\n");
        return 0;
    }
    /* 填写地址 */
    stLocalAddr.sin_family = AF_INET;
    stLocalAddr.sin_port   = htons(12345);
    stLocalAddr.sin_addr.s_addr = 0;
    /* 绑定地址 */
    if(0 > bind(iSocketFD, (void *)&stLocalAddr, sizeof(stLocalAddr)))
    {
        printf("绑定地址失败!\n");
        close(iSocketFD);
        return 0;
    }
    pthread_create(&stPid, NULL, test, NULL);   //实现了多线程
    while(1)     //实现了循环监听
    {
        iRecvLen = recvfrom(iSocketFD, acBuf, sizeof(acBuf), 0, (void *)&stRemoteAddr, &iRemoteAddrLen);
        printf("iRecvLen: %d\n", iRecvLen);
        printf("acBuf:%s\n", acBuf);
    }
    close(iSocketFD);
    return 0;
}
```

客户端： socket_client_udp.c

```
//
// Created by android on 19-8-9.
//
#include <stdio.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
int main(void)
{
    int iRecvLen = 0;
    int iSocketFD = 0;
    int iRemotAddr = 0;
    char acBuf[4096] = {0};
    struct sockaddr_in stLocalAddr = {0};
    struct sockaddr_in stRemoteAddr = {0};
    socklen_t iRemoteAddrLen = 0;
    /* 创建socket */
    iSocketFD = socket(AF_INET, SOCK_DGRAM, 0);
    if(iSocketFD < 0)
    {
        printf("创建socket失败!\n");
        return 0;
    }
    /* 填写服务端地址 */
    stLocalAddr.sin_family = AF_INET;
    stLocalAddr.sin_port   = htons(12345);
    inet_pton(AF_INET, "10.6.191.177", (void *)&iRemotAddr);
    stLocalAddr.sin_addr.s_addr = iRemotAddr;
    iRecvLen = sendto(iSocketFD, "这是一个测试字符串", strlen("这是一个测试字符串"), 0, (void *)&stLocalAddr, sizeof(stLocalAddr));
    close(iSocketFD);
    return 0;
}
```

测试：

1、编译服务器：因为有多线程，所以服务器端进程要进行pthread编译

```
gcc socket_server_udp.c -pthread -g -o server_udp #客户端和上方相同
```

执行结果如下：

右下为客户端重复执行

![img](https://pic4.zhimg.com/80/v2-13f09e9418e8595ce925753af38b78c3_720w.webp)

服务器端有主线程和辅线程，主线程，打印客户端发送的请求；辅线程每隔５秒钟打印一排#号。

原文链接：https://zhuanlan.zhihu.com/p/354316892

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)