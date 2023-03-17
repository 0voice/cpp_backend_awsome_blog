# 【NO.593】一道腾讯面试题目：没有listen，能否建立TCP连接

TCP与UDP最大的不同，就是有连接的概念，而连接的建立是由内核完成的。系统调用listen，就是为了告诉内核，它要处理发给这个TCP端口的连接请求。所以对于这个题目，最直接的想法就是由应用层自己负责TCP的连接。为了能够收到TCP的握手数据包，可以尝试使用原始套接字来接收IP报文，这样就可以在应用层替代内核做TCP的三次握手了。这个想法不错，可惜现实比较残酷。

当没有对于TCP 套接字处于listen状态时，使用raw socket处理握手报文时，即使收到了syn报文并给对端发送了syn+ack报文，也无法完成连接。因为内核一般会提前发送RST中断该连接。七年前，我当时只知道这个结果。现在呢，可以明确的知道为什么内核会发送RST报文，中断连接。内核在ip_local_deliver_finish先将报文复制一份给原始套接字，然后会继续后面的处理，进入tcp的接收函数tcp_v4_rcv。在这个函数中，要进行套接字的查找。

![img](https://pic2.zhimg.com/80/v2-8144d1faa8cbf55b27ef593dfcfad341_720w.webp)

因为没有监听的tcp套接字，自然无法找到对应的套接字。于是跳转到no_tcp_socket。

![img](https://pic4.zhimg.com/80/v2-cf9f42c760a1d4d22772de4d82b7f917_720w.webp)

在这个错误处理中，只要数据包skb的校验和没错，内核就会调用tcp_v4_send_reset发送RST中止这个连接。因此，这个单独使用raw socket的方案是行不通的。我们需要找到一种方法，禁止内核处理该TCP报文。这时，可以使用iptables的NFQUEUE，在网络层将数据包取到应用层，并回复syn+ack，同时在reinject时通知内核该数据包被“偷”了，即NF_STOLEN。这样内核就不会继续处理该数据包skb了。虽然当时我没有继续实验尝试，但理论上，通过这个IPtables的NFQUEUE+NFSTOLEN的方案，是可以实现“没有listen，建立TCP连接”的。

可惜，在与那位同学的讨论中，腾讯面试题目的本意不是这个意思，而是对于普通的TCP套接字来说，如果没有listen调用，是否可以创建连接。即使限定了条件，答案依然是肯定的。只不过限定了条件之后，我们需要确定2个事情：

1. 与前面类似，如何避免内核发送RST。在不能使用iptable的前提下，这意味着在tcp_v4_rcv中，要能够找到对应的套接字。
2. 没有listen状态的套接字，内核是否能够完成TCP的三次握手呢？

确定第一个问题，比较简单。只需要对三次握手深入的思考一下，就可以得到答案。在正常的三次握手中，当服务端回复syn+ack时，客户端实际上也没有处于listen状态的套接字，但却可以完成三次握手。这意味着，客户端进行connect调用后，该套接字一定被加入到某个表中，并可以被匹配到。跟踪内核源码tcp_v4_connect->inet_hash_connect->__inet_check_established，可以看到当调用connect时，对应的套接字就被加入了全局的tcp已连接的表中，即tcp_hashinfo.ehash中。对应的匹配TCP套接字过程，如下__inet_lookup_skb->__inet_lookup

![img](https://pic1.zhimg.com/80/v2-67c79f1cf00f61fa7dd93e1c86448f4c_720w.webp)

内核是先在已经连接的表中查找，再进行listen表的查找。对于客户端来说，syn+ack报文必然可以在已连接表中匹配上对应的套接字。那么，对于本题目来说，要想两端都可以找到套接字，就要求在报文到达前，两端都调用了connect。也就是说，当两端同时调用connect时，两端的syn包就都可以匹配上本地的套接字。

接下来只需要确定对于客户端套接字来说，收到syn报文，是否可以正常处理。tcp_rcv_synsent_state_process函数是TCP套接字处于synsent状态（即调用了connect）的处理函数。下面是其一部分实现代码：

![img](https://pic2.zhimg.com/80/v2-84b12d0dfd1238480d8643fd70d74c01_720w.webp)

![img](https://pic2.zhimg.com/80/v2-eeb5afdef2e1acb5724ea54eb7d6a889_720w.webp)

从上，可以得出，对于处于synsent状态的套接字来说，如果收到了syn报文，则会正常回复synack，完成TCP三次握手。

对于腾讯的这道面试题目来说，其答案就是当两端同时发起connect调用时，即使没有listen调用，也可以成功创建TCP连接。

如果去掉“两端”的限制，还有一个答案就是，TCP套接字可以connect它本身bind的地址和端口，也可以达成要求。下面是测试代码，实现了一个TCP套接字成功连接自己，并发送消息。

```text
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

#define LOCAL_IP_ADDR		(0x7F000001)
#define LOCAL_TCP_PORT		(34567)

int main(void)
{
	struct sockaddr_in local, peer;
	int ret;
	char buf[128];
	int sock = socket(AF_INET, SOCK_STREAM, 0);

	memset(&local, 0, sizeof(local));
	memset(&peer, 0, sizeof(peer));

	local.sin_family = AF_INET;
	local.sin_port = htons(LOCAL_TCP_PORT);
	local.sin_addr.s_addr = htonl(LOCAL_IP_ADDR);

	peer = local;	

    int flag = 1;
    ret = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
    if (ret == -1) {
        printf("Fail to setsocket SO_REUSEADDR: %s\n", strerror(errno));
        exit(1);
    }

	ret = bind(sock, (const struct sockaddr *)&local, sizeof(local));
	ret = connect(sock, (const struct sockaddr *)&peer, sizeof(peer));

	if (ret) {
		printf("Fail to connect myself: %s\n", strerror(errno));
		exit(1);
	}
	printf("Connect to myself successfully\n");

	strcpy(buf, "Hello, myself~");
	send(sock, buf, strlen(buf), 0);

	memset(buf, 0, sizeof(buf));
	recv(sock, buf, sizeof(buf), 0);

	printf("Recv the msg: %s\n", buf);
	close(sock);
	return 0;
}
```

其连接截图如下：

![img](https://pic4.zhimg.com/80/v2-0e6f0cb40fc5878c214a6397169aa5b3_720w.webp)

从截图中，可以看到TCP套接字成功的“连接”了自己，并发送和接收了数据包围。netstat的输出更证明了TCP的两端地址和端口是完全相同的。

原文地址：https://zhuanlan.zhihu.com/p/506042622

作者：CPP后端技术