# 【NO.73】linux性能优化之网络篇

网络通讯的链路为：机器A-----》路由器----〉机器B

如果我们站在本机机器作为参考物的话，应该拆分成下面三个阶段：

1.消息入口流量部分的处理流程

2.流量在本机上面的处理流程

3.流量出口流量部分的处理流程

在上面的这三个阶段，分别涉及到了不同的基础知识、工具和命令来进行诊断，笔者本篇文章按照下面的内容来整理：

1.基础知识

2.常用工具和命令

3.测试工具

## **1.基础知识**

**衡量网络性能的主要指标为：**

**带宽：**表示链路的最大传输速率，单位是 b/s（比特 / 秒），常用的带宽有 1000M、10G、40G、100G 等。

**吞吐量：**表示没有丢包时的最大数据传输速率，单位通常为 b/s （比特 / 秒）或者 B/s（字节 / 秒），吞吐量受带宽的限制，吞吐量 / 带宽也就是该网络链路的使用率。

**延时：**表示从网络请求发出后，一直到收到远端响应，所需要的时间延迟，表示建立连接需要的时间（比如 TCP 握手延时），或者一个数据包往返所需时间（比如 RTT）。

通常，我们更常用的是双向的往返通信延迟，比如 ping 测试的结果，就是往返延时 RTT（Round-Trip Time）。除了网络延迟外，另一个常用的指标是应用程序延迟，它是指，从应用程序接收到请求，再到发回响应，全程所用的时间。应用程序延迟也指的是往返延迟，是网络数据传输时间加上数据处理时间的和。

**PPS：**Packet Per Second（包 / 秒）的缩写，表示以网络包为单位的传输速率。PPS 通常用来评估网络的转发能力，而基于 Linux 服务器的转发，很容易受到网络包大小的影响（交换机通常不会受到太大影响，即交换机可以线性转发）。PPS，是以网络包为单位的网络传输速率，通常用在需要大量转发的场景中。对 TCP 或者 Web 服务来说，更多会用并发连接数和每秒请求数（QPS，Query per Second）等指标，它们更能反应实际应用程序的性能。

网络的可用性（网络能否正常通信）、并发连接数（TCP 连接数量）、丢包率（丢包百分比）、重传率（重新传输的网络包比例）等也是常用的性能指标。

（**备注**：网络接口层和网络层，它们主要负责网络包的封装、寻址、路由以及发送和接收。在这两个网络协议层中，每秒可处理的网络包数 PPS，就是最重要的性能指标，特别是 64B 小包的处理能力，值得我们特别关注。）

### **1.1 本机内部的流量处理知识整理：**

Linux 内核中的网络栈，类似于 TCP/IP 的四层结构，如下所示：

![img](https://pic3.zhimg.com/80/v2-9455f474f20e70478fd7b16a2de37a76_720w.webp)

对于操作系统中的数据包收发流程，如下所示：

![img](https://pic2.zhimg.com/80/v2-61eddcc318ca695b6239ac19f370bfc5_720w.webp)

### **1.2 本机入口和出口流量的处理知识点整理：**

1）对于机器的入口流量来说，主要涉及到的知识便是C10K、C1000K、C10M的场景处理。

C10K 、C1000K、 C10M的首字母 C 是 Client 的缩写，C10K 是单机同时处理 1 万个请求（并发连接 1 万）的问题，C1000K 是单机支持处理 100 万个请求（并发连接 100 万）的问题，C10M是1000万个请求（并发连接1000万）的问题。

**1>I/O 模型的优化**，特别是Linux 2.6 中引入的 epoll完美解决了 C10K 的问题。

I/O 模型相关知识略

**2>从 C10K 到 C100K，我们只需要增加系统的物理资源**，就可以满足要求。

**3>**从 C100K 到 C1000K ，光增加物理资源就不够了。这时，**就要对系统的软硬件进行统一优化**，从硬件的中断处理，到网络协议栈的文件描述符数量、连接状态跟踪、缓存队列，再到应用程序的工作模型等的整个网络链路，都需要深入优化。**4>**要实现 C10M，就不是增加物理资源、调优内核和应用程序可以解决的问题了，这时内核中冗长的网络协议栈就成了最大的负担。需要用 **XDP 方式**，在内核协议栈之前，先处理网络包；或基于 **DPDK** ，直接跳过网络协议栈，在用户空间通过轮询的方式处理。

**DPDK：**是用户态网络的标准，它跳过内核协议栈，直接由用户态进程通过轮询的方式，来处理网络接收。对于C10M场景，基本上每时每刻都有新的网络包需要处理，轮询的优势就很明显了。

```text
1. 在 PPS 非常高的场景中，查询时间比实际工作时间少了很多，绝大部分时间都在处理网络包；
2. 跳过内核协议栈后，就省去了繁杂的硬中断、软中断再到 Linux 网络协议栈逐层处理的过程，
应用程序可以针对应用的实际场景，有针对性地优化网络包的处理逻辑，而不需要关注所有的细节。
3. DPDK 还通过大页、CPU 绑定、内存对齐、流水线并发等多种机制，优化网络包的处理效率。
```

![img](https://pic1.zhimg.com/80/v2-d6d8ed06a38f8e24c7892668d74bf114_720w.webp)

（**备注**：DPDK 是目前最主流的高性能网络方案，不过，这需要能支持 DPDK 的网卡配合使用。）

**XDP**（eXpress Data Path）：则是 Linux 内核提供的一种高性能网络数据路径，它允许网络包，在进入内核协议栈之前，就进行处理，也可以带来更高的性能，XDP 底层都是基于 Linux 内核的 eBPF 机制实现的。

![img](https://pic3.zhimg.com/80/v2-576738195188839d373f5cacd2f0f382_720w.webp)

（**备注**：XDP 对内核的要求比较高，需要的是 Linux 4.8 以上版本，并且它也不提供缓存队列，基于 XDP 的应用程序通常是专用的网络应用，常见的有 IDS（入侵检测系统）、DDoS 防御、 cilium 容器网络插件等。）



**2) iptables 与 NAT：**

NAT 技术可以重写 IP 数据包的源 IP 或者目的 IP，被普遍地用来解决公网 IP 地址短缺的问题。它的主要原理就是，网络中的多台主机，通过共享同一个公网 IP 地址，来访问外网资源，同时，由于 NAT 屏蔽了内网网络，自然也就为局域网中的机器提供了安全隔离。

NAT 的主要目的，是实现地址转换，根据实现方式的不同，NAT 可以分为三类：

```text
静态 NAT，即内网 IP 与公网 IP 是一对一的永久映射关系；
动态 NAT，即内网 IP 从公网 IP 池中，动态选择一个进行映射；
网络地址端口转换 NAPT（Network Address and Port Translation），即把内网 IP 映射到公网 IP 的不同端口上，让多个内网 IP 可以共享同一个公网 IP 地址。
```

**iptables**:
Linux 内核提供的 Netfilter 框架，允许对网络数据包进行修改（比如 NAT）和过滤（比如防火墙）。在这个基础上，iptables、ip6tables、ebtables 等工具，又提供了更易用的命令行接口，以便系统管理员配置和管理 NAT、防火墙的规则，其中，iptables 就是最常用的一种配置工具。

### **1.3 非本机相关的网络知识：**

DNS（Domain Name System），即域名系统，是互联网中最基础的一项服务，主要提供域名和 IP 地址之间映射关系的查询服务。

## **2.工具和命令介绍：**

### 2.1 查看网络配置 ifconfig、ip

```text
# RUNNING,LOWER_UP: 表示物理网络是连通的
# RX:receive,表示的是接收数据，从开启到现在接收封包的情况，是下行流量（Downlink）
# TX:Transmit,表示的是发送数据，从开启到现在发送封包的情况，是上行流量（Uplink）。
# errors 表示发生错误的数据包数，比如校验错误、帧同步错误等；
# dropped 表示丢弃的数据包数，即数据包已经收到了 Ring Buffer，但因为内存不足等原因丢包；
# overruns 表示超限数据包数，即网络 I/O 速度过快，导致 Ring Buffer 中的数据包来不及处理（队列满）而导致的丢包；
# carrier 表示发生 carrirer 错误的数据包数，比如双工模式不匹配、物理电缆出现问题等；
# collisions 表示碰撞数据包数。
$ ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
      inet 10.240.0.30 netmask 255.240.0.0 broadcast 10.255.255.255
      inet6 fe80::20d:3aff:fe07:cf2a prefixlen 64 scopeid 0x20<link>
      ether 78:0d:3a:07:cf:3a txqueuelen 1000 (Ethernet)
      RX packets 40809142 bytes 9542369803 (9.5 GB)
      RX errors 0 dropped 0 overruns 0 frame 0
      TX packets 32637401 bytes 4815573306 (4.8 GB)
      TX errors 0 dropped 0 overruns 0 carrier 0 collisions 0

$ ip -s addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
  link/ether 78:0d:3a:07:cf:3a brd ff:ff:ff:ff:ff:ff
  inet 10.240.0.30/12 brd 10.255.255.255 scope global eth0
      valid_lft forever preferred_lft forever
  inet6 fe80::20d:3aff:fe07:cf2a/64 scope link
      valid_lft forever preferred_lft forever
  RX: bytes packets errors dropped overrun mcast
   9542432350 40809397 0       0       0       193
  TX: bytes packets errors dropped carrier collsns
   4815625265 32637658 0       0       0       0
```

### 2.2 查看套接字、网络栈、网络接口以及路由表的信息 netstat、ss

1）当套接字处于连接状态（Established）：

Recv-Q 表示套接字缓冲还没有被应用程序取走的字节数（即接收队列长度）。

Send-Q 表示还没有被远端主机确认的字节数（即发送队列长度）。

2）当套接字处于监听状态（Listening）：

Recv-Q 表示全连接队列的长度。

Send-Q 表示全连接队列的最大长度。

```text
# head -n 3 表示只显示前面3行
# -l 表示只显示监听套接字
# -n 表示显示数字地址和端口(而不是名字)
# -p 表示显示进程信息
$ netstat -nlp | head -n 3
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      840/systemd-resolve

# -l 表示只显示监听套接字
# -t 表示只显示 TCP 套接字
# -n 表示显示数字地址和端口(而不是名字)
# -p 表示显示进程信息
$ ss -ltnp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:(("systemd-resolve",pid=840,fd=13))
LISTEN   0         128                 0.0.0.0:22    
           0.0.0.0:*        users:(("sshd",pid=1459,fd=3))
           
 # 协议栈统计信息
$ netstat -s
#主动连接、被动连接、失败重试、发送和接收的分段数量等各种信息
...
Tcp:
    3244906 active connection openings
    23143 passive connection openings
    115732 failed connection attempts
    2964 connection resets received
    1 connections established
    13025010 segments received
    17606946 segments sent out
    44438 segments retransmitted
    42 bad segments received
    5315 resets sent
    InCsumErrors: 42
...

$ ss -s
Total: 186 (kernel 1446)
TCP:   4 (estab 1, closed 0, orphaned 0, synrecv 0, timewait 0/0), ports 0

Transport Total     IP        IPv6
*    1446      -         -
RAW    2         1         1
UDP    2         2         0
TCP    4         3         1
...
```

### 2.3 查看系统当前的网络吞吐量和 PPS：sar

```text
# 数字1表示每隔1秒输出一组数据
$ sar -n DEV 1 // -n 参数就可以查看网络的统计信息，DEV是接口
#rxpck/s 和 txpck/s 分别是接收和发送的 PPS，单位为包 / 秒。
# rxkB/s 和 txkB/s 分别是接收和发送的吞吐量，单位是 KB/ 秒。
# rxcmp/s 和 txcmp/s 分别是接收和发送的压缩数据包数，单位是包 / 秒。
# %ifutil 是网络接口的使用率，即半双工模式下为 (rxkB/s+txkB/s)/Bandwidth，而全双工模式下为 max(rxkB/s, txkB/s)/Bandwidth。
Linux 4.15.0-1035 (ubuntu)   01/06/19   _x86_64_  (2 CPU)

13:21:40        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
13:21:41         eth0     18.00     20.00      5.79      4.25      0.00      0.00      0.00      0.00
13:21:41      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
13:21:41           lo      0.00      0.00      0.00      0.00 
```

Bandwidth 可以用 ethtool 来查询，它的单位通常是 Gb/s 或者 Mb/s，这里小写字母 b ，表示比特而不是字节。

```text
$ ethtool eth0 | grep Speed
  Speed: 1000Mb/s
```

### 2.4 连通性和延迟 ping

```text
# -c3表示发送三次ICMP包后停止
# 第一部分，是每个 ICMP 请求的信息，包括 ICMP 序列号（icmp_seq）、TTL（生存时间，或者跳数）以及往返延时。
$ ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=54 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=47 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=3 ttl=67 time=244 ms
# 第二部分，则是三次 ICMP 请求的汇总。
--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 244.023/244.070/244.105/0.034 ms
```

### 2.5 抓包工具tcpdump

安装：

```text
# Ubuntu
apt-get install tcpdump wireshark
# CentOS
yum install -y tcpdump wireshark
```

使用介绍：

```text
# -nn ，表示不解析抓包中的域名（即不反向解析）、协议以及端口号。
# udp port 53 ，表示只显示 UDP 协议的端口号（包括源端口和目的端口）为 53 的包。
# host 35.190.27.188 ，表示只显示 IP 地址（包括源地址和目的地址）为 35.190.27.188 的包。
# 这两个过滤条件中间的“ or ”，表示或的关系，也就是说，只要满足上面两个条件中的任一个，就可以展示出来。
# -w ping.pcap 表示输出到本地的ping.pcap存储起来
$ tcpdump -nn udp port 53 or host 35.190.27.188 -w ping.pcap
```

下载到本地机器，使用wireshark进行分析

```text
$ scp host-ip/path/ping.pcap .
```

### 2.6 路由相关工具：mtr、route、traceroute

```text
# --tcp表示使用TCP协议，-p表示端口号，-n表示不对结果中的IP地址执行反向域名解析
$ traceroute --tcp -p 80 -n baidu.com
traceroute to baidu.com (123.125.115.110), 30 hops max, 60 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  123.125.115.110  20.684 ms *  20.798 ms
```

## **3.压测工具介绍**

### **3.1 转发性能：hping3**

网络接口层和网络层，它们主要负责网络包的封装、寻址、路由以及发送和接收，在这两个网络协议层中，每秒可处理的网络包数 PPS，就是最重要的性能指标。

hping3 可以作为一个测试网络包处理能力的性能工具。

```text
# -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u10表示每隔10微秒发送一个网络帧
$ hping3 -S -p 80 -i u10 192.168.0.30
```

**（备注：**Linux 内核自带的高性能网络测试工具 pktgen**）**

```text
# -c表示发送3次请求，-S表示设置TCP SYN，-p表示端口号为80
$ hping3 -c 3 -S -p 80 baidu.com
HPING baidu.com (eth0 123.125.115.110): S set, 40 headers + 0 data bytes
len=46 ip=123.125.115.110 ttl=51 id=47908 sport=80 flags=SA seq=0 win=8192 rtt=20.9 ms
len=46 ip=123.125.115.110 ttl=51 id=6788  sport=80 flags=SA seq=1 win=8192 rtt=20.9 ms
len=46 ip=123.125.115.110 ttl=51 id=37699 sport=80 flags=SA seq=2 win=8192 rtt=20.9 ms

--- baidu.com hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 20.9/20.9/20.9 ms
```

### **3.2 TCP/UDP 性能：iperf 或者 netperf**

iperf 和 netperf 是最常用的网络性能测试工具，测试 TCP 和 UDP 的吞吐量，它们都以客户端和服务器通信的方式，测试一段时间内的平均吞吐量。

```text
# iperf 安装方式
# Ubuntu
apt-get install iperf3
# CentOS
yum install iperf3
```

iperf测试方式：

```text
# 被测试机器上启动服务端进行
# -s表示启动服务端，-i表示汇报间隔，-p表示监听端口
$ iperf3 -s -i 1 -p 10000
```

测试的客户端

```text
# -c表示启动客户端，192.168.0.30为目标服务器的IP
# -b表示目标带宽(单位是bits/s)
# -t表示测试时间，15 表示15秒
# -P表示并发数，-p表示目标服务器监听端口
$ iperf3 -c 192.168.0.30 -b 1G -t 15 -P 2 -p 10000
```

测试结果报告分析：

```text
[ ID] Interval           Transfer     Bandwidth
...
[SUM]   0.00-15.04  sec  0.00 Bytes  0.00 bits/sec                  sender
[SUM]   0.00-15.04  sec  1.51 GBytes   860 Mbits/sec                  receiver
# 吞吐量是：860 Mb/s，传递的消息量是：1.51 GBytes
```

### **3.3 HTTP 的性能： ab、webbench**

ab 是 Apache 自带的 HTTP 压测工具，主要测试 HTTP 服务的每秒请求数、请求延迟、吞吐量以及请求延迟的分布情况等。

```text
# ab的安装
# Ubuntu
$ apt-get install -y apache2-utils
# CentOS
$ yum install -y httpd-tools
```

目标机器上面运行被测试的服务。

客户端运行ab进行测试：

```text
# -c表示并发请求数为1000，-n表示总的请求数为10000，http://192.168.0.30/ 测试的测试服务的接口
$ab -c 1000 -n 10000 http://192.168.0.30/
...
Server Software:        nginx/1.15.8
Server Hostname:        192.168.0.30
Server Port:            80

...
# 请求汇总、
Requests per second:    1078.54 [#/sec] (mean)
Time per request:       927.183 [ms] (mean)
Time per request:       0.927 [ms] (mean, across all concurrent requests)
Transfer rate:          890.00 [Kbytes/sec] received
# 连接时间汇总
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   27 152.1      1    1038
Processing:     9  207 843.0     22    9242
Waiting:        8  207 843.0     22    9242
Total:         15  233 857.7     23    9268
# 还有请求延迟汇总
Percentage of the requests served within a certain time (ms)
  50%     23
  66%     24
  75%     24
  80%     26
  90%    274
  95%   1195
  98%   2335
  99%   4663
 100%   9268 (longest request)
```

### **3.4 应用负载性能：wrk**

wrk是一个 HTTP 性能测试工具，内置了 LuaJIT，方便你根据实际需求，生成所需的请求负载payload，或者自定义响应的处理方法。

安装：

```text
$ https://github.com/wg/wrk
$ cd wrk
$ apt-get install build-essential -y
$ make
$ sudo cp wrk /usr/local/bin/
```

启动被测试服务之后，客户端运行下面的命令进行测试：

```text
# -c表示并发连接数1000，-t表示线程数为2，被测试服务的接口路径：http://192.168.0.30/
$wrk -c 1000 -t 2 http://192.168.0.30/
Running 10s test @ http://192.168.0.30/
  2 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    65.83ms  174.06ms   1.99s    95.85%
    Req/Sec     4.87k   628.73     6.78k    69.00%
  96954 requests in 10.06s, 78.59MB read
  Socket errors: connect 0, read 0, write 0, timeout 179
Requests/sec:   9641.31
Transfer/sec:      7.82MB
```

tcpdump 和 Wireshark 就是最常用的网络抓包和分析工具，更是分析网络性能必不可少的利器。tcpdump 仅支持命令行格式使用，常用在服务器中抓取和分析网络包。Wireshark 除了可以抓包外，还提供了强大的图形界面和汇总分析工具，在分析复杂的网络情景时，尤为简单和实用。因而，在实际分析网络性能时，先用 tcpdump 抓包，后用 Wireshark 分析，也是一种常用的方法。

原文地址：https://zhuanlan.zhihu.com/p/572183950

作者：linux