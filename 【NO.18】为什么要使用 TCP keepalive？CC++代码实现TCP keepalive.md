# 【NO.18】为什么要使用 TCP keepalive？C/C++代码实现TCP keepalive

为了理解 TCP keepalive的作用。我们需要清楚，当TCP的Peer A ，Peer B 两端建立了连接之后，如果一端突然拔掉网线或拔掉电源时，怎么检测到拔掉网线或者拔掉电源、链路不通？原因是在需要长连接的网络通信程序中，经常需要心跳检测机制，来实现检测对方是否在线或者维持网络连接的需要。

## **1.什么是 TCP 保活？**

当你建立一个 TCP 连接时，你关联了一组定时器。其中一些计时器处理保活过程。当保活计时器达到零时，向对等方发送一个保活探测数据包，其中没有数据并且 ACK 标志打开。

由于 TCP/IP 规范，可以这样做，作为一种重复的 ACK，并且远程端点将没有参数，因为 TCP 是面向流的协议。另一方面，将收到来自远程主机的回复，没有数据和ACK 集。

如果收到对 keepalive 探测的回复，则可以断言连接仍在运行。事实上，TCP 允许处理流，而不是数据包，因此零长度数据包对用户程序没有危险。

此过程很有用，因为如果其他对等方失去连接（例如通过重新启动），即使没有流量，也会注意到连接已断开。如果对等方未回复 keepalive 探测，可以断言连接不能被视为有效，然后采取正确的操作。

## **2.为什么要使用 TCP keepalive？**

> 1、检查死节点 2、 防止因网络不活动而断开连接

### **2.1检查死节点**

想一想 Peer A 和 Peer B 之间的简单 TCP 连接：初始的三次握手，从 A 到 B 的一个 SYN 段，从 B 到 A 的 SYN/ACK，以及从 A 到 B 的最终 ACK。

![img](https://pic1.zhimg.com/80/v2-867faba3a42f1bcdb407c4053cc63394_720w.webp)

此时，我们处于稳定状态：连接已建立，现在我们通常会等待有人通过通道发送数据。

那么问题来了：从 B 上拔下电源，它会立即断电，而不会通过网络发送任何信息来通知 A 连接将断开。

从它的角度来看，A 已准备好接收数据，并且不知道 B 已经崩溃。现在恢复B的电源，等待系统重启。A 和 B 现在又回来了，但是当 A 知道与 B 仍然处于活动状态的连接时，B 不知道。当 A 尝试通过死连接向 B 发送数据时，情况自行解决，B 回复 RST 数据包，导致 A 最终关闭连接。

```
    _____                                                     _____
   |     |                                                   |     |
   |  A  |                                                   |  B  |
   |_____|                                                   |_____|
      ^                                                         ^
      |--->--->--->-------------- SYN -------------->--->--->---|
      |---<---<---<------------ SYN/ACK ------------<---<---<---|
      |--->--->--->-------------- ACK -------------->--->--->---|
      |                                                         |
      |                                       system crash ---> X
      |
      |                                     system restart ---> ^
      |                                                         |
      |--->--->--->-------------- PSH -------------->--->--->---|
      |---<---<---<-------------- RST --------------<---<---<---|
      |                                                         |
```

Keepalive 可以告诉您何时无法访问另一个对等点，而不会出现误报的风险。

### **2.2防止因网络不活动而断开连接**

keepalive 的另一个有用目标是防止不活动断开通道。当你在 NAT 代理或防火墙后面时，无缘无故断开连接是一个非常常见的问题。这种行为是由代理和防火墙中实现的连接跟踪过程引起的，它们跟踪通过它们的所有连接。

它们跟踪通过它们的所有连接。由于这些机器的物理限制，它们只能在内存中保留有限数量的连接。最常见和合乎逻辑的策略是保持最新的连接并首先丢弃旧的和不活动的连接。

```
    _____           _____                                     _____
   |     |         |     |                                   |     |
   |  A  |         | NAT |                                   |  B  |
   |_____|         |_____|                                   |_____|
      ^               ^                                         ^
      |--->--->--->---|----------- SYN ------------->--->--->---|
      |---<---<---<---|--------- SYN/ACK -----------<---<---<---|
      |--->--->--->---|----------- ACK ------------->--->--->---|
      |               |                                         |
      |               | <--- connection deleted from table      |
      |               |                                         |
      |--->- PSH ->---| <--- invalid connection                 |
      |               |                                         |
```



## **3.Linux下使用TCP keepalive**

Linux 内置了对 keepalive 的支持。涉及 keepalive 的过程使用三个用户驱动的变量，可以使用 cat 查看参数值。

![img](https://pic1.zhimg.com/80/v2-228c38791ed25f830fb6c79418d8386c_720w.webp)

前两个参数以秒表示，最后一个是纯数字。这意味着keepalive 例程在发送第一个keepalive 探测之前等待两个小时（7200 秒），然后每75 秒重新发送一次。如果连续9次没有收到 ACK 响应，则连接被标记为断开。

修改这个值很简单，可以这样修改：

> echo 7000 > /proc/sys/net/ipv4/tcp_keepalive_time echo 40 > /proc/sys/net/ipv4/tcp_keepalive_intvl echo 10 > /proc/sys/net/ipv4/tcp_keepalive_probes

还有另一种访问内核变量的方法，使用 sysctl 命令

![img](https://pic3.zhimg.com/80/v2-87436a797e3363cf882e93c8773a56a2_720w.webp)

## **4.setsockopt 、getsockopt 函数调用**

在 Linux 操作系统中，我们可以通过代码启用一个 socket 的心跳检测，为特定套接字启用 keepalive 所需要做的就是在套接字本身上设置特定的套接字选项。函数原型如下：

```
int getsockopt(int sockfd, int level, int optname,
                      void *optval, socklen_t *optlen);

int setsockopt(int sockfd, int level, int optname,
                      const void *optval, socklen_t optlen);
                      
```

![img](https://pic1.zhimg.com/80/v2-2fd4c07d5c568c4f939dc66772e17f9c_720w.webp)

第一个参数是socket；第二个必须是 SOL_SOCKET，第三个必须是 SO_KEEPALIVE。第四个参数必须是布尔整数值，表示我们要启用该选项，而最后一个是之前传递的值的大小。

在编写应用程序时，还可以为 keepalive 设置其他三个套接字选项。它们都使用 SOL_TCP 级别而不是 SOL_SOCKET，并且它们仅针对当前套接字覆盖系统范围的变量。如果不先写入就读取，将返回当前系统范围的参数。

```
TCP_KEEPCNT：覆盖 tcp_keepalive_probes
TCP_KEEPIDLE：覆盖 tcp_keepalive_time
TCP_KEEPINTVL：覆盖 tcp_keepalive_intvl   
```

## **5.TCP keepalive 代码实现**

在写TCP keepalive 服务程序时，除了要处理SIGPIPE外，还要有客户端连接检测机制，用于及时发现崩溃的客户端连接。我们使用TCP的 keepalive 机制方式。

tcp_keepalive_client：

```
int main(int argc, char *argv[])
{
 kat_arg0 = basename(argv[0]);
 bzero(&cp, sizeof (cp));
 cp.cp_keepalive = 1;
 cp.cp_keepidle = -1;
 cp.cp_keepcnt = -1;
 cp.cp_keepintvl = -1;

 while ((c = getopt(argc, argv, ":c:d:i:")) != -1) {
  switch (c) {
  case 'c':
   cp.cp_keepcnt = parse_positive_int_option(
       optopt, optarg);
   break;

  case 'd':
   cp.cp_keepidle = parse_positive_int_option(
       optopt, optarg);
   break;

  case 'i':
   cp.cp_keepintvl = parse_positive_int_option(
       optopt, optarg);
   break;
  
  case ':':
   warnx("option requires an argument: -%c", optopt);
   usage();
   break;

  case '?':
   warnx("unrecognized option: -%c", optopt);
   usage();
   break;
  }
 }

 if (optind > argc - 1) {
  warnx("missing required arguments");
  usage();
 }

 ipport = argv[optind++];
 if (parse_ip4port(ipport, &cp.cp_ip) == -1) {
  warnx("invalid IP/port: \"%s\"", ipport);
  usage();
 }

 (void) fprintf(stderr, "going connect to: %s port %d\n",
     inet_ntoa(cp.cp_ip.sin_addr), ntohs(cp.cp_ip.sin_port));
 (void) fprintf(stderr, "set SO_KEEPALIVE  = %d\n", cp.cp_keepalive);
 (void) fprintf(stderr, "set TCP_KEEPIDLE  = %d\n", cp.cp_keepidle);
 (void) fprintf(stderr, "set TCP_KEEPCNT   = %d\n", cp.cp_keepcnt);
 (void) fprintf(stderr, "set TCP_KEEPINTVL = %d\n", cp.cp_keepintvl);
 rv = connectandwait(&cp);
 return (rv == 0 ? EXIT_SUCCESS : EXIT_FAILURE);
}
```

tcp_keepalive_server:

```
int main(int argc, char *argv[] )
{

   /* 创建套接字 */
   if((listen_sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0) {
      perror("socket()");
      exit(EXIT_FAILURE);
   }

   /* 检查 keepalive 选项的状态  */
   if(getsockopt(listen_sock, SOL_SOCKET, SO_KEEPALIVE, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(listen_sock);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE default is %s\n", (optval ? "ON" : "OFF"));

   /* 将选项设置为活动  */
   optval = 1;
   optlen = sizeof(optval);
   
   if(setsockopt(listen_sock, SOL_SOCKET, SO_KEEPALIVE, &optval, optlen) < 0) {
      perror("setsockopt()");
      close(listen_sock);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE set on socket\n");

   /* 再次检查状态  */
   if(getsockopt(listen_sock, IPPROTO_TCP, TCP_KEEPIDLE, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(listen_sock);
      exit(EXIT_FAILURE);
   }
   printf("TCP_KEEPIDLE is %d\n", optval );
      /* 再次检查状态  */
   if(getsockopt(listen_sock, IPPROTO_TCP, TCP_KEEPCNT, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(listen_sock);
      exit(EXIT_FAILURE);
   }
   printf("TCP_KEEPCNT is %d\n", optval);
      /* 再次检查状态  */
   if(getsockopt(listen_sock, IPPROTO_TCP, TCP_KEEPINTVL, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(listen_sock);
      exit(EXIT_FAILURE);
   }
   printf("TCP_KEEPINTVL is %d\n", optval );
  /* 初始化套接字结构 */
   bzero((char *) &serv_addr, sizeof(serv_addr));
   int portno = atoi(argv[1]);
   serv_addr.sin_family = AF_INET;
   serv_addr.sin_addr.s_addr = INADDR_ANY;
   serv_addr.sin_port = htons(portno);
 
  ...
 }  
```

![img](https://pic3.zhimg.com/80/v2-191350c3a091692dd7639a99c4f95a7e_720w.webp)

![img](https://pic4.zhimg.com/80/v2-7249343ab61572da61fba081b8a25c5f_720w.webp)

程序创建一个 TCP 套接字并将 SO_KEEPALIVE 套接字选项设置为 1。如果指定了“-c”、“-d”和“-i”选项中的任何一个，则设置 TCP_KEEPCNT、TCP_KEEPIDLE 和 TCP_KEEPINTVL 套接字选项 在相应选项参数的套接字上。

通过测试程序，我们可以使用tcpdump、或者tshark是命令行抓包工具，来分析KeepAlive。

> tshark -nn -i lo port 5050 tcpdump -nn -i lo port 5050

![img](https://pic4.zhimg.com/80/v2-1d3f9ab15ddbdb19ebd5cb17252265c3_720w.webp)

tcpdump -nn -i lo port 5050

![img](https://pic3.zhimg.com/80/v2-11cfeb52d1c4ee979f0128bbfa174bae_720w.webp)

整个keepalive过程很简单，就是client给server发送一个包，server返回给用户一个包。注意包内没有数据，只有ACK标识 被打开。

ps -aux | grep tcp_keepalive

![img](https://pic4.zhimg.com/80/v2-7ed343859db5b4becd9e1b6eaf6248bf_720w.webp)

## **6.总结**

keepalive 是一个设备向另一个设备发送的消息，用于检查两者之间的链路是否正在运行，或防止链路中断。

原文链接：https://zhuanlan.zhihu.com/p/582867947