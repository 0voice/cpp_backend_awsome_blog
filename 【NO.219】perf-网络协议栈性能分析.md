# 【NO.219】perf-网络协议栈性能分析

分析 Linux 网络协议栈性能有多种方式和工具。本文主要通过 Perf 生成 On-CPU 火焰图的方式，分析 Linux 内核网络协议栈在特定场景下的性能瓶颈，从而知晓当前协议栈的网络状况。

## 1.关于 On/Off-CPU

### 1.1.概念定义

![img](https://pic4.zhimg.com/80/v2-140bbf8313007f59c92008d0cf6d87c3_720w.webp)

### 1.2.On/Off-CPU 选择

在工程实践中，如果是 CPU 消耗型使用 On-CPU 火焰图，如果是 IO 消耗型则使用 Off-CPU 火焰图。如果无法确定, 可以通过压测工具来确认: 通过压测工具看能否让 CPU 使用率趋于饱和，从而判断是否为 CPU 消耗型。

### 1.3.分析方法

Perf 火焰图整个图形看起来就像一团跳动的火焰，这也正是其名字的由来。燃烧在火苗尖部的就是 CPU 正在执行的操作，不过需要说明的是颜色是随机的，本身并没有特殊的含义，纵向表示调用栈的深度，横向表示消耗的时间。因为调用栈在横向会按照字母排序，并且同样的调用栈会做合并，所以一个格子的宽度越大越说明其可能是瓶颈。综上所述，主要就是看那些比较宽大的火苗，特别留意那些类似平顶山的火苗。

## 2.火焰图原理

火焰图是基于 stack 信息生成图片, 用来展示 CPU 调用栈。

- y 轴表示调用栈, 每一层都是一个函数。调用栈越深, 火焰就越高, 顶部就是正在执行的函数, 下方都是它的父函数。
- x 轴表示抽样数, 如果一个函数在x轴占据越宽, 则表示它被抽到的次数多, 即执行的时间长。

火焰图就是看顶层的哪个函数占据的宽度最大。只要有“平顶”(plateaus)，就表示该函数可能存在性能问题。

## 3.On-CPU 采集原理

![img](https://pic1.zhimg.com/80/v2-a682f2717063b9dc0ca22c4da9989d74_720w.webp)

![img](https://pic1.zhimg.com/80/v2-081e26d5bef572c4e585d05ca522990c_720w.webp)

Linux 内核网络协议栈分析

server

```
yum install iperfyum install perfgit clone https://github.com/brendangregg/FlameGraph.git
```

运行 iperf server

```
[root@bogon ~]# iperf -s -u -DRunning Iperf Server as a daemon[root@bogon ~]#
```

查看 CPU 以及 softirqd 进程号

```
[root@bogon ~]# lscpuArchitecture:          aarch64Byte Order:            Little EndianCPU(s):                64On-line CPU(s) list:   0-63Thread(s) per core:    1Core(s) per socket:    4Socket(s):             16NUMA node(s):          4Model:                 2BogoMIPS:              100.00NUMA node0 CPU(s):     0-15NUMA node1 CPU(s):     16-31NUMA node2 CPU(s):     32-47NUMA node3 CPU(s):     48-63Flags:                 fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
```

![img](https://pic4.zhimg.com/80/v2-a4150c37c195fd209c70f582b0959d9b_720w.webp)

查看网卡中断亲和性

![img](https://pic2.zhimg.com/80/v2-12bd0c929fad16b85392da3063e4f001_720w.webp)

![img](https://pic2.zhimg.com/80/v2-a2d31aad24d547a15829d926cb6bbdcd_720w.webp)

```
VM001: 通过如下方式采集信息(采集原理见上)perf record -F 1000 -a -g -p 162 -- sleep 60
```

发送 UDP 数据报文

```
iperf  -c 10.254.2.161 -i 1 -P 10 -t 10 -u -b 1000M
```

## 4.生成 On-CPU 火焰图

![img](https://pic1.zhimg.com/80/v2-b088f3056d212b56812a3b60285b3a04_720w.webp)

![img](https://pic1.zhimg.com/80/v2-bddd0d6ad7f632560c3f4ae6dfc8ff48_720w.webp)

![img](https://pic1.zhimg.com/80/v2-503a3f2918d0b2e0474fc90de58b6558_720w.webp)

原文链接：https://linuxcpp.0voice.com/?id=442

作者：[HG ](https://linuxcpp.0voice.com/?auth=10)