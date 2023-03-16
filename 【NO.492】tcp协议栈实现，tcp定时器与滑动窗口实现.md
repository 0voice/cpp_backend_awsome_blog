# 【NO.492】tcp协议栈实现，tcp定时器与滑动窗口实现

要实现用户态协议栈，必须要搞懂TCP，TCP 11个状态、滑动窗口、拥塞控制、定时器等等。

要使用用户态协议栈，内核提供的epoll就不起作用了，我们需要自己实现用户态的epoll。epoll内部涉及到一个回调的时机，回调的作用是将红黑树中的节点添加进就绪队列，具体在epoll原理里面会具体讲解。搞清楚TCP的11个状态，我们就明白应该在什么时机进行回调了。

### 1.TCP状态转换图

在前面的[posix与网络协议栈](Build software better, together api和网络协议栈.md)中，已经介绍了tcp的状态转换。可以结合tcp状态转换图一起看。

TCP状态保存在哪里？保存在TCB中，即TCP PCB，协议控制块。里面包含了socket信息，以及sendbuffer，recvbuffer。TCB保存了从listen到time_wait的所有状态。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216163719_81035.jpg)

### 2.用户态TCP协议栈实现

前面实现了UDP协议栈，TCP协议栈实现也是类似的，但是比UDP要复杂很多。

### 3.TCP头定义

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216163720_17869.jpg)

seq num初始值是多少，到达最大值(2^32 - 1)后怎么样, 会越界吗？

seq num初始值是一个随机值，之后累加。到达最大值后又从0开始计算，不会越界。

seq num指的是包的数量，还是字节数量？

计算的时候，使用的都是字节数。

TCP的包是什么意思？TCP头为什么没有包长？

TCP前后两个包都有序号，就可以计算出包的长度。

ack num = seq num + 包长。

header length是4bit，最大值是15，单位是4个字节，所以TCP头最大时15*4 = 60字节。没有option的话，TCP头是20字节，header length值就是5。

window size，能够接收数据的最大容量。

urgent pointer, 如果URG位置1，就是告诉对端从这个位置开始的数据，要马上处理。

```
struct tcphdr {
    unsigned short sport;
    unsigned short dport;
    unsigned int seqnum;
    unsigned int acknum;
    unsigned char hdrlen_resv;
    unsigned char flag; 
    unsigned short window;
    unsigned short checksum;
    unsigned short urgent_pointer;
    unsigned int options[0];
};
```

### 4.定义TCP flag

```
#define TCP_CWR_FLAG        0x80
#define TCP_ECE_FLAG        0x40
#define TCP_URG_FLAG        0x20
#define TCP_ACK_FLAG        0x10
#define TCP_PSH_FLAG        0x08
#define TCP_RST_FLAG        0x04
#define TCP_SYN_FLAG        0x02
#define TCP_FIN_FLAG        0x01
```

后面5个flag比较重要

ACK 是用来确认的

PSH 告诉对端赶紧通知应用程序把数据包处理了，在数据传输过程都可以设置成PSH。

RST，收到的ack num，或者seq num、widow size非法，或者数据不对了，就给对端回一个RST。三次握手发送第一次后，超时没收到对端的第二次握手，也会发送一个RST。

SYN只是在连接开始的时候，用于告诉对端seq num，也就是发送的第一个包的序号。

FIN，终止。

### 5.定义TCP包

```
struct tcppkt {
    struct ethhdr eh; // 14
    struct iphdr ip;  // 20 
    struct tcphdr tcp; // 8
    unsigned char data[0];
};
```

### 6.定义TCP状态

```
typedef enum _tcp_status {    TCP_STATUS_CLOSED,    TCP_STATUS_LISTEN,    TCP_STATUS_SYN_REVD,    TCP_STATUS_SYN_SENT,    TCP_STATUS_ESTABLISHED,    TCP_STATUS_FIN_WAIT_1,    TCP_STATUS_FIN_WAIT_2,    TCP_STATUS_CLOSING,    TCP_STATUS_TIME_WAIT,    TCP_STATUS_CLOSE_WAIT,    TCP_STATUS_LAST_ACK,};
```

### 7.定义TCB

```
typedef enum _tcp_status {
    TCP_STATUS_CLOSED,
    TCP_STATUS_LISTEN,
    TCP_STATUS_SYN_REVD,
    TCP_STATUS_SYN_SENT,
    TCP_STATUS_ESTABLISHED,
    TCP_STATUS_FIN_WAIT_1,
    TCP_STATUS_FIN_WAIT_2,
    TCP_STATUS_CLOSING,
    TCP_STATUS_TIME_WAIT,
    TCP_STATUS_CLOSE_WAIT,
    TCP_STATUS_LAST_ACK,
};
```

### 8.实现TCP三次握手

服务端处理好三次握手的状态转换，客户端就能与服务器建立连接。

```
int main() {
    struct nm_pkthdr h;
    struct nm_desc *nmr = nm_open("netmap:eth0", NULL, 0, NULL);
    if (nmr == NULL) return -1;
    struct pollfd pfd = {0};
    pfd.fd = nmr->fd;
    pfd.events = POLLIN;
    struct ntcb tcb;
    while (1) {
        int ret = poll(&pfd, 1, -1);
        if (ret < 0) continue;
        if (pfd.revents & POLLIN) {
            unsigned char *stream = nm_nextpkt(nmr, &h);
            struct ethhdr *eh = (struct ethhdr *)stream;
            if (ntohs(eh->h_proto) ==  PROTO_IP) {
                struct udppkt *tcp = (struct udppkt *)stream;
                if (tcp->ip.type == PROTO_TCP) {
                    struct tcppkt *tcp = (struct tcppkt *)stream;
                    unsigned int sip = tcp->ip.sip;
                    unsigned int dip = tcp->ip.dip;
                    unsigned short sport = tcp->tcp.sport;
                    unsigned short dport = tcp->tcp.dport;
                    tcb = search_tcb();
                    if (tcb->status == TCP_STATUS_LISTEN) { //
                        if (tcp->tcp.flag & TCP_SYN_FLAG) {
                            client_tcb = create_tcb()；
                            client_tcb->status = TCP_STATUS_SYN_REVD;
                            // 将sip，sport，smac与dip，dport，dmac互换
                            // send syn, ack pkt
                            // seqnum, ack 
                        } 
                    } else if (tcb->status == TCP_STATUS_SYN_REVD) {
                        if (tcp->tcp.flag & TCP_ACK_FLAG) {
                            client_tcb->status = TCP_STATUS_ESTABLISHED;
                        }
                    }
                }
            }
        }
    }
}
```

### 9.数据发送过程

MSS（Maximum Segment Size，最大报文长度），是TCP协议定义的一个选项，MSS选项用于在TCP连接建立时，收发双方协商通信时每一个报文段所能承载的最大数据长度。

MTU，是对数据链路层的限制。

客户端到服务器

1. 发送1M的文件
2. sendbuffer = 2k
3. mss = 512
4. mtu = 1500

```
while (1) {
    poll(fd)
    send(fd, buffer, 1k, 0);
}
```

如果客户端sendbuff = 2k, mss = 512。要分4个包发送

客户端能否发出去这4个包呢？

不一定，取决于服务器的接收窗口window size大小。如果window size是1024。客户端如果发送两个包，每个包大小512，则如果服务器的应用程序没有取，那么回的ack包里面的window size就会是0，那么客户端的sendbuffer里面就会剩下1k数据发送不了。就会等服务器数据处理完了再发送。

如果每发送一个包，就等待ack，这种效率太慢。我们需要能够同时发送多个包，就是慢启动的过程。

### 10.慢启动的过程

第一次发送 1 * mss

第二次发送 2 * mss

第三次发送 4 * mss

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216163721_51370.jpg)

慢启动的过程，发送1mss，2mss，4*mss，…

如何判断数据包超出网络负载？

通过判断超时。超时时间又怎么算呢？

拥塞避免，从客户端往服务器发送数据包，网络上的数据越来越多，造成网络拥塞，致使服务器没有办法正确接收数据。

如何判断数据包超出网络负载？

rtt， round trip time， 数据包往返一次的时间。

进入电梯这种弱网的环境下，rtt突然变大，叫做抖动。

当前rtt计算方法

rtt = 0.1 * rtt(new) + 0.9 * rtt(old), 是一个消抖的过程。

用于判断当前这一次有没有超时，一旦出现超时，判断在发送包的数量上是否需要减一减，超出网络负载。

如果服务器的window size是0，没有接收的空间了，客户端就不能再发送了。如果服务器处理完数据，有空间了，客户端怎么能知道服务器有空间了呢？

服务器端window是0，等到服务端将数据处理完，window不为0的时候，客户端怎么能知道服务器已经有接收空间了呢？

1. 服务器主动告诉客户端。– 不好的地方，如果通知包在网络中丢失了怎么办？
2. 客户端定时查询 – TCP是这种做法，当收到对端window为0，定时发送探测包, 就是探测定时器。

客户端定时查询更好。

### 11.滑动窗口

滑动窗口也是以mss作为单位的。

滑动窗口。

在接收的过程中间，准备好指针。一根指针对应已经发送确认的，另一根指针对应允许接收的最大位置。两根指针之间的长度表示window size。

回ack的表示前面的数据都已经收到，都可以调用recv进行处理；未发送ack先不用管，表示数据还没有组织好，还不能调用recv处理。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216163722_93980.jpg)

window大小和recvbuff的关系？

window size和recvbuff是有关系的，但是是两个概念。

看起来window size = recvbuff / 2， 这个没有找到具体的说明。

### 12.定时器

重传定时器、探测定时器(坚持定时器)、keepalive、TIME_WAIT定时器、延迟ack定时器

重传定时器，发送端发送一个包后，启动重传定时器，RTT超时重传，如果在规定时间内收到ack包，则撤销定时器；

探测定时器，如果对端window size 是0，则启动探测定时器；

TCP已经有keepalive，应用层为什么还要提供心跳包？

TCP keepalive也是心跳包，超时主动回收TCB，应用层感知不到。应用层心跳包可控制性更强。

TIME_WAIT定时器，time_wait时间是2msl，防止4次挥手的最后一次ack丢失。

延迟ack定时器，接收端收到TCP包，启动200ms定时器，后面再次收到数据，则重置定时器，超时后发送ack。

原文链接：https://zhuanlan.zhihu.com/p/503077683

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)