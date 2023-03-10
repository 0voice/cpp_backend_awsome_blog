# 【NO.26】【引路者】学习DPDK，须知多队列网卡的原理

网卡多队列，顾名思义，也就是传统网卡的`DMA`队列有多个，网卡有基于多个`DMA`队列的分配机制。多队列网卡已经是当前高速率网卡的主流。

## 1.RPS

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212122034487115657.png)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203546_10420.jpg)

`Linux`内核中，`RPS`（`Receive Packet Steering`）在接收端提供了这样的机制。RPS主要是把软中断的负载均衡到`CPU`的各个`core`上，网卡驱动对每个流生成一个`hash`标识，这个`hash`值可以通过四元组（源IP地址`SIP`，源四层端口`SPOR`T，目的IP地址`DIP`，目的四层端口`DPORT`）来计算，然后由中断处理的地方根据这个`hash`标识分配到相应的`core`上去，这样就可以比较充分地发挥多核的能力了。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203547_55361.jpg)

## 2. DPDK多队列支持

`DPDK Packet I/O`机制具有与生俱来的多队列支持功能，可以根据不同的平台或者需求，选择需要使用的队列数目，并可以很方便地使用队列，指定队列发送或接收报文。由于这样的特性，可以很容易实现`CPU`核、缓存与网卡队列之间的亲和性，从而达到很好的性能。从`DPDK`的典型应用`l3fwd`可以看出，在某个核上运行的程序从指定的队列上接收，往指定的队列上发送，可以达到很高的cache命中率，效率也就会高。

除了方便地做到对指定队列进行收发包操作外，DPDK的队列管理机制还可以避免多核处理器中的多个收发进程采用自旋锁产生的不必要等待。

以`run to completion`模型为例，可以从核、内存与网卡队列之间的关系来理解DPDK是如何利用网卡多队列技术带来性能的提升。

- 将网卡的某个接收队列分配给某个核，从该队列中收到的所有报文都应当在该指定的核上处理结束。
- 从核对应的本地存储中分配内存池，接收报文和对应的报文描述符都位于该内存池。
- 为每个核分配一个单独的发送队列，发送报文和对应的报文描述符都位于该核和发送队列对应的本地内存池中。

可以看出不同的核，操作的是不同的队列，从而避免了多个线程同时访问一个队列带来的锁的开销。但是，如果逻辑核的数目大于每个接口上所含的发送队列的数目，那么就需要有机制将队列分配给这些核。不论采用何种策略，都需要引入锁来保护这些队列的数据。

网卡是如何将网络中的报文分发到不同的队列呢？常用的方法有微软提出的RSS与英特尔提出的`Flow Director`技术，前者是根据哈希值希望均匀地将包分发到多个队列中。后者是基于查找的精确匹配，将包分发到指定的队列中。此外，网卡还可以根据优先级分配队列提供对`QoS`的支持。

## 3. DPDK重点学习内容如下：

### **3.1 DPDK环境与testpmd/l3fwd/skeleton**

1. DPDK环境参数讲解
2. **多队列网卡的工作原理**
3. CPU亲和性
4. Burst数据包的优缺点
5. DPDK轮询模式 异步中断，轮询模式，混合中断轮询模式
6. virtio与vhost

### **3.2 DPDK的用户态协议栈实现**

1. 内核网络接口 KNI的实现原理
2. 接收线程/发送线程/KNI线程
3. 内存数据结构 rte_mbuf, rte_mempool
4. 端口数据结构 rte_kni, rte_kni_conf, rte_kni_ops, rte_eth_conf
5. 协议数据结构 rte_ether_hdrrte_ipv4_hdr, rte_udp_hdr
6. 数据处理接口 rte_eth_rx_burst,rte_kni_tx_burst,rte_pktmbuf_mtod

### **3.3 千万级流量并发的DNS处理**

1. udp协议包处理
2. dns协议实现
3. 配置文件解析
4. 数据结构 rte_ring
5. trex数据包性能测试

### **3.4 高性能数据处理框架VPP**

1. vpp使用vmxnet3
2. DPDK ACL实现数据过滤
3. vpp web应用
4. vpp基础库 VPPInfra
5. 高速查找路由表，CAM表

### **3.5 DPDK的虚拟交换机框架 OvS**

1. OvS三大组件 ovs-vswitchd, ovsdb-server, openvswitch.ko
2. OvS报文处理机制
3. OvS 4种数据路径
4. VXLAN数据协议

PS：系统学习DPDK，**[我要进大厂](https://link.zhihu.com/?target=https%3A//jq.qq.com/%3F_wv%3D1027%26k%3DkV5PWhiO)**

## 4. 流分类

高级的网卡设备（比如`Intel XL710`）可以分析出包的类型，包的类型会携带在接收描述符中，应用程序可以根据描述符快速地确定包是哪种类型的包。`DPDK`的`Mbuf`结构中含有相应的字段来表示网卡分析出的包的类型。

### 4.1 RSS（Receive-Side Scaling，接收方扩展）

`RSS`就是根据关键字通过哈希函数计算出哈希值，再由哈希值确定队列。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203547_95631.jpg)

关键字是如何确定的呢？

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203548_92080.jpg)

哈希函数一般选取微软托普利兹算法（`Microsoft Toeplitz Based Hash`）或者对称哈希。

### 4.2 Flow Director

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203549_83387.jpg)

`Flow Director`技术是`Intel`公司提出的根据包的字段精确匹配，将其分配到某个特定队列的技术：网卡上存储了一个`Flow Director`的表，表的大小受硬件资源限制，它记录了需要匹配字段的关键字及匹配后的动作；驱动负责操作这张表，包括初始化、增加表项、删除表项；网卡从线上收到数据包后根据关键字查`Flow Director`的这张表，匹配后按照表项中的动作处理，可以是分配队列、丢弃等。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203550_78445.jpg)

相比`RSS`的负载分担功能，它更加强调特定性。比如，用户可以为某几个特定的TCP对话（`S-IP+D-IP+S-Port+D-Port`）预留某个队列，那么处理这些`TCP`对话的应用就可以只关心这个特定的队列，从而省去了`CPU`过滤数据包的开销，并且可以提高`cache`的命中率。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203550_29281.jpg)

### 4.3 服务质量

多队列应用于服务质量（`QoS`）流量类别：把发送队列分配给不同的流量类别，可以让网卡在发送侧做调度；把收包队列分配给不同的流量类别，可以做到基于流的限速。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203551_34380.jpg)

### 4.4 流过滤

来自外部的数据包哪些是本地的、可以被接收的，哪些是不可以被接收的？可以被接收的数据包会被网卡送到主机或者网卡内置的管理控制器，其过滤主要集中在以太网的二层功能，包括`VLAN`及`MAC`过滤。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203552_74459.jpg)

## 5. 应用

针对`Intel®XL710`网卡，PF使用`i40e Linux Kernel`驱动，`VF`使用`DPDK i40e PMD`驱动。使用`Linux`的`Ethtool`工具，可以完成配置操作`cloud filter`，将大量的数据包直接分配到`VF`的队列中，交由运行在`VF`上的虚机应用来直接处理。

```
echo 1 > /sys/bus/pci/devices/0000:02:00.0/sriov_numvfsmodprobe pci-stubecho "8086 154c" > /sys/bus/pci/drivers/pci-stub/new_idecho 0000:02:02.0 > /sys/bus/pci/devices/0000:2:02.0/driver/unbindecho 0000:02:02.0 > /sys/bus/pci/drivers/pci-stub/bindqemu-system-x86_64 -name vm0 -enable-kvm -cpu host -m 2048 -smp 4 -drive file=dpdk-vm0.img -vnc :4 -device pci-assign,host=02:02.0ethtool -N ethx flow-type ip4 dst-ip 2.2.2.2 user-def 0xffffffff00000000 action 2 loc 1
```

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/12/20221212203553_71886.jpg)

原文链接：https://zhuanlan.zhihu.com/p/384256684

原文作者：零声Github整理库