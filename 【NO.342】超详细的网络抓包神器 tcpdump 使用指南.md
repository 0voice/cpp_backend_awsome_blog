# 【NO.342】超详细的网络抓包神器 tcpdump 使用指南

tcpdump 是一款强大的网络抓包工具，它使用 libpcap 库来抓取网络数据包，这个库在几乎在所有的 Linux/Unix 中都有。熟悉 tcpdump 的使用能够帮助你分析调试网络数据，本文将通过一个个具体的示例来介绍它在不同场景下的使用方法。不管你是系统管理员，程序员，云原生工程师还是 yaml 工程师，掌握 tcpdump 的使用都能让你如虎添翼，升职加薪。

## 1. 基本语法和使用方法

tcpdump 的常用参数如下：

```text
$ tcpdump -i eth0 -nn -s0 -v port 80
```

- **-i** : 选择要捕获的接口，通常是以太网卡或无线网卡，也可以是 vlan 或其他特殊接口。如果该系统上只有一个网络接口，则无需指定。
- **-nn** : 单个 n 表示不解析域名，直接显示 IP；两个 n 表示不解析域名和端口。这样不仅方便查看 IP 和端口号，而且在抓取大量数据时非常高效，因为域名解析会降低抓取速度。
- **-s0** : tcpdump 默认只会截取前 96 字节的内容，要想截取所有的报文内容，可以使用 -s number， number 就是你要截取的报文字节数，如果是 0 的话，表示截取报文全部内容。
- **-v** : 使用 -v，-vv 和 -vvv 来显示更多的详细信息，通常会显示更多与特定协议相关的信息。
- port 80 : 这是一个常见的端口过滤器，表示仅抓取 80 端口上的流量，通常是 HTTP。

额外再介绍几个常用参数：

- **-p** : 不让网络接口进入混杂模式。默认情况下使用 tcpdump 抓包时，会让网络接口进入混杂模式。一般计算机网卡都工作在非混杂模式下，此时网卡只接受来自网络端口的目的地址指向自己的数据。当网卡工作在混杂模式下时，网卡将来自接口的所有数据都捕获并交给相应的驱动程序。如果设备接入的交换机开启了混杂模式，使用 -p 选项可以有效地过滤噪声。
- **-e** : 显示数据链路层信息。默认情况下 tcpdump 不会显示数据链路层信息，使用 -e 选项可以显示源和目的 MAC 地址，以及 VLAN tag 信息。例如：

```text
$ tcpdump -n -e -c 5 not ip6

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br-lan, link-type EN10MB (Ethernet), capture size 262144 bytes
18:27:53.619865 24:5e:be:0c:17:af > 00:e2:69:23:d3:3b, ethertype IPv4 (0x0800), length 1162: 192.168.100.20.51410 > 180.176.26.193.58695: Flags [.], seq 2045333376:2045334484, ack 3398690514, win 751, length 1108
18:27:53.626490 00:e2:69:23:d3:3b > 24:5e:be:0c:17:af, ethertype IPv4 (0x0800), length 68: 220.173.179.66.36017 > 192.168.100.20.51410: UDP, length 26
18:27:53.626893 24:5e:be:0c:17:af > 00:e2:69:23:d3:3b, ethertype IPv4 (0x0800), length 1444: 192.168.100.20.51410 > 220.173.179.66.36017: UDP, length 1402
18:27:53.628837 00:e2:69:23:d3:3b > 24:5e:be:0c:17:af, ethertype IPv4 (0x0800), length 1324: 46.97.169.182.6881 > 192.168.100.20.59145: Flags [P.], seq 3058450381:3058451651, ack 14349180, win 502, length 1270
18:27:53.629096 24:5e:be:0c:17:af > 00:e2:69:23:d3:3b, ethertype IPv4 (0x0800), length 54: 192.168.100.20.59145 > 192.168.100.1.12345: Flags [.], ack 3058451651, win 6350, length 0
5 packets captured
```

### 1.1 显示 ASCII 字符串

-A 表示使用 ASCII 字符串打印报文的全部数据，这样可以使读取更加简单，方便使用 grep 等工具解析输出内容。-X 表示同时使用十六进制和 ASCII 字符串打印报文的全部数据。这两个参数不能一起使用。例如：

```text
$ tcpdump -A -s0 port 80
```

### 1.2 抓取特定协议的数据

后面可以跟上协议名称来过滤特定协议的流量，以 UDP 为例，可以加上参数 udp 或 protocol 17，这两个命令意思相同。

```text
$ tcpdump -i eth0 udp
$ tcpdump -i eth0 proto 17
```

同理，tcp 与 protocol 6 意思相同。

### 1.3 抓取特定主机的数据

使用过滤器 host 可以抓取特定目的地和源 IP 地址的流量。

```text
$ tcpdump -i eth0 host 10.10.1.1
```

也可以使用 src 或 dst 只抓取源或目的地：

```text
$ tcpdump -i eth0 dst 10.10.1.20
```

### 1.4 将抓取的数据写入文件

使用 tcpdump 截取数据报文的时候，默认会打印到屏幕的默认输出，你会看到按照顺序和格式，很多的数据一行行快速闪过，根本来不及看清楚所有的内容。不过，tcpdump 提供了把截取的数据保存到文件的功能，以便后面使用其他图形工具（比如 wireshark，Snort）来分析。

-w 选项用来把数据报文输出到文件：

```text
$ tcpdump -i eth0 -s0 -w test.pcap
```

### 1.5 行缓冲模式

如果想实时将抓取到的数据通过管道传递给其他工具来处理，需要使用 -l 选项来开启行缓冲模式（或使用 -c 选项来开启数据包缓冲模式）。使用 -l 选项可以将输出通过立即发送给其他命令，其他命令会立即响应。

```text
$ tcpdump -i eth0 -s0 -l port 80 | grep 'Server:'
```

### 1.6 组合过滤器

过滤的真正强大之处在于你可以随意组合它们，而连接它们的逻辑就是常用的 与/AND/&& 、 或/OR/|| 和 非/not/!。

```text
and or &&
or or ||
not or !
```

## 2. 过滤器

关于 tcpdump 的过滤器，这里有必要单独介绍一下。

机器上的网络报文数量异常的多，很多时候我们只关系和具体问题有关的数据报（比如访问某个网站的数据，或者 icmp 超时的报文等等），而这些数据只占到很小的一部分。把所有的数据截取下来，从里面找到想要的信息无疑是一件很费时费力的工作。而 tcpdump 提供了灵活的语法可以精确地截取关心的数据报，简化分析的工作量。这些选择数据包的语句就是过滤器（filter）！

### 2.1 Host 过滤器

Host 过滤器用来过滤某个主机的数据报文。例如：

```text
$ tcpdump host 1.2.3.4复制代码
```

该命令会抓取所有发往主机 1.2.3.4 或者从主机 1.2.3.4 发出的流量。如果想只抓取从该主机发出的流量，可以使用下面的命令：

```text
$ tcpdump src host 1.2.3.4复制代码
```

### 2.2 Network 过滤器

Network 过滤器用来过滤某个网段的数据，使用的是 CIDR 模式。可以使用四元组（x.x.x.x）、三元组（x.x.x）、二元组（x.x）和一元组（x）。四元组就是指定某个主机，三元组表示子网掩码为 255.255.255.0，二元组表示子网掩码为 255.255.0.0，一元组表示子网掩码为 255.0.0.0。例如，

抓取所有发往网段 192.168.1.x 或从网段 192.168.1.x 发出的流量：

```text
$ tcpdump net 192.168.1
```

抓取所有发往网段 10.x.x.x 或从网段 10.x.x.x 发出的流量：

```text
$ tcpdump net 10
```

和 Host 过滤器一样，这里也可以指定源和目的：

```text
$ tcpdump src net 10
```

也可以使用 CIDR 格式：

```text
$ tcpdump src net 172.16.0.0/12
```

### 2.3 Proto 过滤器

Proto 过滤器用来过滤某个协议的数据，关键字为 proto，可省略。proto 后面可以跟上协议号或协议名称，支持 icmp, igmp, igrp, pim, ah, esp, carp, vrrp, udp和 tcp。因为通常的协议名称是保留字段，所以在于 proto 指令一起使用时，必须根据 shell 类型使用一个或两个反斜杠（/）来转义。Linux 中的 shell 需要使用两个反斜杠来转义，MacOS 只需要一个。

例如，抓取 icmp 协议的报文：

```text
$ tcpdump -n proto \\icmp
# 或者
$ tcpdump -n icmp
```

### 2.4 Port 过滤器

Port 过滤器用来过滤通过某个端口的数据报文，关键字为 port。例如：

```text
$ tcpdump port 389
```

## 3. 理解 tcpdump 的输出

截取数据只是第一步，第二步就是理解这些数据，下面就解释一下 tcpdump 命令输出各部分的意义。

```text
21:27:06.995846 IP (tos 0x0, ttl 64, id 45646, offset 0, flags [DF], proto TCP (6), length 64)
    192.168.1.106.56166 > 124.192.132.54.80: Flags [S], cksum 0xa730 (correct), seq 992042666, win 65535, options [mss 1460,nop,wscale 4,nop,nop,TS val 663433143 ecr 0,sackOK,eol], length 0

21:27:07.030487 IP (tos 0x0, ttl 51, id 0, offset 0, flags [DF], proto TCP (6), length 44)
    124.192.132.54.80 > 192.168.1.106.56166: Flags [S.], cksum 0xedc0 (correct), seq 2147006684, ack 992042667, win 14600, options [mss 1440], length 0

21:27:07.030527 IP (tos 0x0, ttl 64, id 59119, offset 0, flags [DF], proto TCP (6), length 40)
    192.168.1.106.56166 > 124.192.132.54.80: Flags [.], cksum 0x3e72 (correct), ack 2147006685, win 65535, length 0
```

最基本也是最重要的信息就是数据报的源地址/端口和目的地址/端口，上面的例子第一条数据报中，源地址 ip 是 192.168.1.106，源端口是 56166，目的地址是 124.192.132.54，目的端口是 80。 > 符号代表数据的方向。

此外，上面的三条数据还是 tcp 协议的三次握手过程，第一条就是 SYN 报文，这个可以通过 Flags [S] 看出。下面是常见的 TCP 报文的 Flags:

- [S] : SYN（开始连接）
- [.] : 没有 Flag
- [P] : PSH（推送数据）
- [F] : FIN （结束连接）
- [R] : RST（重置连接）

而第二条数据的 [S.] 表示 SYN-ACK，就是 SYN 报文的应答报文。

## 4. 例子

下面给出一些具体的例子，每个例子都可以使用多种方法来获得相同的输出，你使用的方法取决于所需的输出和网络上的流量。我们在排障时，通常只想获取自己想要的内容，可以通过过滤器和 ASCII 输出并结合管道与 grep、cut、awk 等工具来实现此目的。

例如，在抓取 HTTP 请求和响应数据包时，可以通过删除标志 SYN/ACK/FIN 来过滤噪声，但还有更简单的方法，那就是通过管道传递给 grep。在达到目的的同时，我们要选择最简单最高效的方法。下面来看例子。

### 4.1 提取 HTTP 用户代理

从 HTTP 请求头中提取 HTTP 用户代理：

```text
$ tcpdump -nn -A -s1500 -l | grep "User-Agent:"
```

通过 egrep 可以同时提取用户代理和主机名（或其他头文件）：

```text
$ tcpdump -nn -A -s1500 -l | egrep -i 'User-Agent:|Host:'
```

### 4.2 只抓取 HTTP GET 和 POST 流量

抓取 HTTP GET 流量：

```text
$ tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

也可以抓取 HTTP POST 请求流量：

```text
$ tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```

注意：该方法不能保证抓取到 HTTP POST 有效数据流量，因为一个 POST 请求会被分割为多个 TCP 数据包。

上述两个表达式中的十六进制将会与 GET 和 POST 请求的 ASCII 字符串匹配。例如，tcp[((tcp[12:1] & 0xf0) >> 2):4] 首先会确定我们感兴趣的字节的位置（在 TCP header 之后），然后选择我们希望匹配的 4 个字节。

### 4.3 提取 HTTP 请求的 URL

提取 HTTP 请求的主机名和路径：

```text
$ tcpdump -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"

tcpdump: listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
    POST /wp-login.php HTTP/1.1
    Host: dev.example.com
    GET /wp-login.php HTTP/1.1
    Host: dev.example.com
    GET /favicon.ico HTTP/1.1
    Host: dev.example.com
    GET / HTTP/1.1
    Host: dev.example.com
```

### 4.4 提取 HTTP POST 请求中的密码

从 HTTP POST 请求中提取密码和主机名：

```text
$ tcpdump -s 0 -A -n -l | egrep -i "POST /|pwd=|passwd=|password=|Host:"

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:25:54.799014 IP 10.10.1.30.39224 > 10.10.1.125.80: Flags [P.], seq 1458768667:1458770008, ack 2440130792, win 704, options [nop,nop,TS val 461552632 ecr 208900561], length 1341: HTTP: POST /wp-login.php HTTP/1.1
.....s..POST /wp-login.php HTTP/1.1
Host: dev.example.com
.....s..log=admin&pwd=notmypassword&wp-submit=Log+In&redirect_to=http%3A%2F%2Fdev.example.com%2Fwp-admin%2F&testcookie=1
```

### 4.5 提取 Cookies

提取 Set-Cookie（服务端的 Cookie）和 Cookie（客户端的 Cookie）：

```text
$ tcpdump -nn -A -s0 -l | egrep -i 'Set-Cookie|Host:|Cookie:'

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wlp58s0, link-type EN10MB (Ethernet), capture size 262144 bytes
Host: dev.example.com
Cookie: wordpress_86be02xxxxxxxxxxxxxxxxxxxc43=admin%7C152xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxfb3e15c744fdd6; _ga=GA1.2.21343434343421934; _gid=GA1.2.927343434349426; wordpress_test_cookie=WP+Cookie+check; wordpress_logged_in_86be654654645645645654645653fc43=admin%7C15275102testtesttesttestab7a61e; wp-settings-time-1=1527337439
```

### 4.6 抓取 ICMP 数据包

查看网络上的所有 ICMP 数据包：

```text
$ tcpdump -n icmp

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:34:21.590380 IP 10.10.1.217 > 10.10.1.30: ICMP echo request, id 27948, seq 1, length 64
11:34:21.590434 IP 10.10.1.30 > 10.10.1.217: ICMP echo reply, id 27948, seq 1, length 64
11:34:27.680307 IP 10.10.1.159 > 10.10.1.1: ICMP 10.10.1.189 udp port 59619 unreachable, length 115
```

### 4.7 抓取非 ECHO/REPLY 类型的 ICMP 数据包

通过排除 echo 和 reply 类型的数据包使抓取到的数据包不包括标准的 ping 包：

```text
$ tcpdump 'icmp[icmptype] != icmp-echo and icmp[icmptype] != icmp-echoreply'

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:37:04.041037 IP 10.10.1.189 > 10.10.1.20: ICMP 10.10.1.189 udp port 36078 unreachable, length 156
```

### 4.8 抓取 SMTP/POP3 协议的邮件

可以提取电子邮件的正文和其他数据。例如，只提取电子邮件的收件人：

```text
$ tcpdump -nn -l port 25 | grep -i 'MAIL FROM\|RCPT TO'
```

### 4.9 抓取 NTP 服务的查询和响应

```text
$ tcpdump dst port 123

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
21:02:19.112502 IP test33.ntp > 199.30.140.74.ntp: NTPv4, Client, length 48
21:02:19.113888 IP 216.239.35.0.ntp > test33.ntp: NTPv4, Server, length 48
21:02:20.150347 IP test33.ntp > 216.239.35.0.ntp: NTPv4, Client, length 48
21:02:20.150991 IP 216.239.35.0.ntp > test33.ntp: NTPv4, Server, length 48
```

### 4.10 抓取 SNMP 服务的查询和响应

通过 SNMP 服务，渗透测试人员可以获取大量的设备和系统信息。在这些信息中，系统信息最为关键，如操作系统版本、内核版本等。使用 SNMP 协议快速扫描程序 onesixtyone，可以看到目标系统的信息：

```text
$ onesixtyone 10.10.1.10 public

Scanning 1 hosts, 1 communities
10.10.1.10 [public] Linux test33 4.15.0-20-generic #21-Ubuntu SMP Tue Apr 24 06:16:15 UTC 2018 x86_64
```

可以通过 tcpdump 抓取 GetRequest 和 GetResponse：

```text
$ tcpdump -n -s0  port 161 and udp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wlp58s0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:39:13.725522 IP 10.10.1.159.36826 > 10.10.1.20.161:  GetRequest(28)  .1.3.6.1.2.1.1.1.0
23:39:13.728789 IP 10.10.1.20.161 > 10.10.1.159.36826:  GetResponse(109)  .1.3.6.1.2.1.1.1.0="Linux testmachine 4.15.0-20-generic #21-Ubuntu SMP Tue Apr 24 06:16:15 UTC 2018 x86_64"
```

### 4.11 切割 pcap 文件

当抓取大量数据并写入文件时，可以自动切割为多个大小相同的文件。例如，下面的命令表示每 3600 秒创建一个新文件 capture-(hour).pcap，每个文件大小不超过 200*1000000 字节：

```text
$ tcpdump  -w /tmp/capture-%H.pcap -G 3600 -C 200
```

这些文件的命名为 capture-{1-24}.pcap，24 小时之后，之前的文件就会被覆盖。

### 4.12 抓取 IPv6 流量

可以通过过滤器 ip6 来抓取 IPv6 流量，同时可以指定协议如 TCP：

```text
$ tcpdump -nn ip6 proto 6
```

从之前保存的文件中读取 IPv6 UDP 数据报文：

```text
$ tcpdump -nr ipv6-test.pcap ip6 proto 17
```

### 4.13 检测端口扫描

在下面的例子中，你会发现抓取到的报文的源和目的一直不变，且带有标志位 [S] 和 [R]，它们与一系列看似随机的目标端口进行匹配。当发送 SYN 之后，如果目标主机的端口没有打开，就会返回一个 RESET。这是 Nmap 等端口扫描工具的标准做法。

```text
$ tcpdump -nn

21:46:19.693601 IP 10.10.1.10.60460 > 10.10.1.199.5432: Flags [S], seq 116466344, win 29200, options [mss 1460,sackOK,TS val 3547090332 ecr 0,nop,wscale 7], length 0
21:46:19.693626 IP 10.10.1.10.35470 > 10.10.1.199.513: Flags [S], seq 3400074709, win 29200, options [mss 1460,sackOK,TS val 3547090332 ecr 0,nop,wscale 7], length 0
21:46:19.693762 IP 10.10.1.10.44244 > 10.10.1.199.389: Flags [S], seq 2214070267, win 29200, options [mss 1460,sackOK,TS val 3547090333 ecr 0,nop,wscale 7], length 0
21:46:19.693772 IP 10.10.1.199.389 > 10.10.1.10.44244: Flags [R.], seq 0, ack 2214070268, win 0, length 0
21:46:19.693783 IP 10.10.1.10.35172 > 10.10.1.199.1433: Flags [S], seq 2358257571, win 29200, options [mss 1460,sackOK,TS val 3547090333 ecr 0,nop,wscale 7], length 0
21:46:19.693826 IP 10.10.1.10.33022 > 10.10.1.199.49153: Flags [S], seq 2406028551, win 29200, options [mss 1460,sackOK,TS val 3547090333 ecr 0,nop,wscale 7], length 0
21:46:19.695567 IP 10.10.1.10.55130 > 10.10.1.199.49154: Flags [S], seq 3230403372, win 29200, options [mss 1460,sackOK,TS val 3547090334 ecr 0,nop,wscale 7], length 0
21:46:19.695590 IP 10.10.1.199.49154 > 10.10.1.10.55130: Flags [R.], seq 0, ack 3230403373, win 0, length 0
21:46:19.695608 IP 10.10.1.10.33460 > 10.10.1.199.49152: Flags [S], seq 3289070068, win 29200, options [mss 1460,sackOK,TS val 3547090335 ecr 0,nop,wscale 7], length 0
21:46:19.695622 IP 10.10.1.199.49152 > 10.10.1.10.33460: Flags [R.], seq 0, ack 3289070069, win 0, length 0
21:46:19.695637 IP 10.10.1.10.34940 > 10.10.1.199.1029: Flags [S], seq 140319147, win 29200, options [mss 1460,sackOK,TS val 3547090335 ecr 0,nop,wscale 7], length 0
21:46:19.695650 IP 10.10.1.199.1029 > 10.10.1.10.34940: Flags [R.], seq 0, ack 140319148, win 0, length 0
21:46:19.695664 IP 10.10.1.10.45648 > 10.10.1.199.5060: Flags [S], seq 2203629201, win 29200, options [mss 1460,sackOK,TS val 3547090335 ecr 0,nop,wscale 7], length 0
21:46:19.695775 IP 10.10.1.10.49028 > 10.10.1.199.2000: Flags [S], seq 635990431, win 29200, options [mss 1460,sackOK,TS val 3547090335 ecr 0,nop,wscale 7], length 0
21:46:19.695790 IP 10.10.1.199.2000 > 10.10.1.10.49028: Flags [R.], seq 0, ack 635990432, win 0, length 0
```

### 4.14 过滤 Nmap NSE 脚本测试结果

本例中 Nmap NSE 测试脚本 http-enum.nse 用来检测 HTTP 服务的合法 URL。

在执行脚本测试的主机上：

```text
$ nmap -p 80 --script=http-enum.nse targetip
```

在目标主机上：

```text
$ tcpdump -nn port 80 | grep "GET /"

GET /w3perl/ HTTP/1.1
GET /w-agora/ HTTP/1.1
GET /way-board/ HTTP/1.1
GET /web800fo/ HTTP/1.1
GET /webaccess/ HTTP/1.1
GET /webadmin/ HTTP/1.1
GET /webAdmin/ HTTP/1.1
```

### 4.15 抓取 DNS 请求和响应

向 Google 公共 DNS 发起的出站 DNS 请求和 A 记录响应可以通过 tcpdump 抓取到：

```text
$ tcpdump -i wlp58s0 -s0 port 53

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on wlp58s0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:19:06.879799 IP test.53852 > google-public-dns-a.google.com.domain: 26977+ [1au] A? play.google.com. (44)
14:19:07.022618 IP google-public-dns-a.google.com.domain > test.53852: 26977 1/0/1 A 216.58.203.110 (60)
```

### 4.16 抓取 HTTP 有效数据包

抓取 80 端口的 HTTP 有效数据包，排除 TCP 连接建立过程的数据包（SYN / FIN / ACK）：

```text
$ tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

### 4.17 将输出内容重定向到 Wireshark

通常 Wireshark（或 tshark）比 tcpdump 更容易分析应用层协议。一般的做法是在远程服务器上先使用 tcpdump 抓取数据并写入文件，然后再将文件拷贝到本地工作站上用 Wireshark 分析。

还有一种更高效的方法，可以通过 ssh 连接将抓取到的数据实时发送给 Wireshark 进行分析。以 MacOS 系统为例，可以通过 brew cask install wireshark 来安装，然后通过下面的命令来分析：

```text
$ ssh root@remotesystem 'tcpdump -s0 -c 1000 -nn -w - not port 22' | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i - 
```

例如，如果想分析 DNS 协议，可以使用下面的命令：

```text
$ ssh root@remotesystem 'tcpdump -s0 -c 1000 -nn -w - port 53' | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -
```

抓取到的数据：

![img](https://pic3.zhimg.com/80/v2-c045644c1a9a88acc047182474609da2_720w.webp)

-c 选项用来限制抓取数据的大小。如果不限制大小，就只能通过 ctrl-c 来停止抓取，这样一来不仅关闭了 tcpdump，也关闭了 wireshark。

### 4.18 找出发包最多的 IP

找出一段时间内发包最多的 IP，或者从一堆报文中找出发包最多的 IP，可以使用下面的命令：

```text
$ tcpdump -nnn -t -c 200 | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
200 packets captured
261 packets received by filter
0 packets dropped by kernel
    108 IP 10.10.211.181
     91 IP 10.10.1.30
      1 IP 10.10.1.50
```

- **cut -f 1,2,3,4 -d '.'** : 以 . 为分隔符，打印出每行的前四列。即 IP 地址。
- **sort | uniq -c** : 排序并计数
- **sort -nr** : 按照数值大小逆向排序

### 4.19 抓取用户名和密码

本例将重点放在标准纯文本协议上，过滤出于用户名和密码相关的报文：

```text
$ tcpdump port http or port ftp or port smtp or port imap or port pop3 or port telnet -l -A | egrep -i -B5 'pass=|pwd=|log=|login=|user=|username=|pw=|passw=|passwd=|password=|pass:|user:|username:|password:|login:|pass |user '
```

### 4.20 抓取 DHCP 报文

最后一个例子，抓取 DHCP 服务的请求和响应报文，67 为 DHCP 端口，68 为客户机端口。

```text
$ tcpdump -v -n port 67 or 68

tcpdump: listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:37:50.059662 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 00:0c:xx:xx:xx:d5, length 300, xid 0xc9779c2a, Flags [none]
      Client-Ethernet-Address 00:0c:xx:xx:xx:d5
      Vendor-rfc1048 Extensions
        Magic Cookie 0x63825363
        DHCP-Message Option 53, length 1: Request
        Requested-IP Option 50, length 4: 10.10.1.163
        Hostname Option 12, length 14: "test-ubuntu"
        Parameter-Request Option 55, length 16: 
          Subnet-Mask, BR, Time-Zone, Default-Gateway
          Domain-Name, Domain-Name-Server, Option 119, Hostname
          Netbios-Name-Server, Netbios-Scope, MTU, Classless-Static-Route
          NTP, Classless-Static-Route-Microsoft, Static-Route, Option 252
14:37:50.059667 IP (tos 0x10, ttl 128, id 0, offset 0, flags [none], proto UDP (17), length 328)
    0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from 00:0c:xx:xx:xx:d5, length 300, xid 0xc9779c2a, Flags [none]
      Client-Ethernet-Address 00:0c:xx:xx:xx:d5
      Vendor-rfc1048 Extensions
        Magic Cookie 0x63825363
        DHCP-Message Option 53, length 1: Request
        Requested-IP Option 50, length 4: 10.10.1.163
        Hostname Option 12, length 14: "test-ubuntu"
        Parameter-Request Option 55, length 16: 
          Subnet-Mask, BR, Time-Zone, Default-Gateway
          Domain-Name, Domain-Name-Server, Option 119, Hostname
          Netbios-Name-Server, Netbios-Scope, MTU, Classless-Static-Route
          NTP, Classless-Static-Route-Microsoft, Static-Route, Option 252
14:37:50.060780 IP (tos 0x0, ttl 64, id 53564, offset 0, flags [none], proto UDP (17), length 339)
    10.10.1.1.67 > 10.10.1.163.68: BOOTP/DHCP, Reply, length 311, xid 0xc9779c2a, Flags [none]
      Your-IP 10.10.1.163
      Server-IP 10.10.1.1
      Client-Ethernet-Address 00:0c:xx:xx:xx:d5
      Vendor-rfc1048 Extensions
        Magic Cookie 0x63825363
        DHCP-Message Option 53, length 1: ACK
        Server-ID Option 54, length 4: 10.10.1.1
        Lease-Time Option 51, length 4: 86400
        RN Option 58, length 4: 43200
        RB Option 59, length 4: 75600
        Subnet-Mask Option 1, length 4: 255.255.255.0
        BR Option 28, length 4: 10.10.1.255
        Domain-Name-Server Option 6, length 4: 10.10.1.1
        Hostname Option 12, length 14: "test-ubuntu"
        T252 Option 252, length 1: 10
        Default-Gateway Option 3, length 4: 10.10.1.1
```

## 5. 总结

本文主要介绍了 tcpdump 的基本语法和使用方法，并通过一些示例来展示它强大的过滤功能。将 tcpdump 与 wireshark 进行组合可以发挥更强大的功效，本文也展示了如何优雅顺滑地结合 tcpdump 和 wireshark。如果你想了解更多的细节，可以查看 tcpdump 的 man 手册。

原文地址：https://zhuanlan.zhihu.com/p/482617730

作者：linux