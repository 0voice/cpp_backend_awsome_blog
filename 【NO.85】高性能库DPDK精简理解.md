# 【NO.85】高性能库DPDK精简理解

## 1.前言

才开始接触到DPDK，发现概念很多，很难以下了解，在这文章中记录下关键的内容，做到对dpdk的基本东西真正了解了。
这样后面用它来写程序才可能顺利，不能赶进度啊，越赶进度反而可能越慢，慢慢来比较快。
本文主要是自己理解，参考很多文章，有哪里不理解的就查，做不到精深，只了解含义。
文章算是汇编，参考多篇文章，如有侵权，请告知，谢谢！

## 2. 整体理解

历史：
随着计算机核数的增加，网络带宽的增加，对主机进行网络包的处理性能要求越来越高，但是现在的操作系统对网络包处理的方式很低效。
低效表现在：
1）网络数据包来了之后通过中断模式进行通知，而cpu处理中断的能力是一定的，如果网络中有大量的小数据包，造成了网络的拥堵，cpu处理不及时。
【以前cpu的频率远高于网络设备，所以中断很有效】
2）操作系统的协议栈是单核处理，没办法利用现在操作系统的多核。
3）网络数据包从网卡到内核空间，再到用户空间，进行了多次数据拷贝，性能比较差。
DPDK 全称 Data Plane Development Kit 专注于数据面的软件开发套件，是专为Intel的网络芯片开发，运行于Linux和FreeBsd上。
DPDK改变了传统的网络数据包的处理方式，在用户空间直接处理，图示如下：

![img](https://pic4.zhimg.com/80/v2-457c406c6f5f666235ba0aec07e35613_720w.webp)

传统VSDPDK抓包方式

## 3.重要概念理解

这里面说明DPDK文档里面的主要概念，另外如何将概念与实际的我们自己的机器上参数对应起来。

### 3.1 .PPS：包转发率

即1s可以发送多个frame、在以太网里面为以太帧，我们常说的接口带宽为1Gbits/s 、10Gbits/s 代表以太接口能够传输的最高速率，单位为（bit per second 位/秒）
实际上，传输过程中，帧之间有间距（12个字节），每个帧前面还有前导（7个字节）、帧首界定符（1个字节）。
帧理论转发率= BitRate/8 / (帧前导+帧间距+帧首界定符+报文长度）

![img](https://pic3.zhimg.com/80/v2-102a26372bd203e0d55c9f27927d5d1a_720w.webp)

以太帧传输中结构

按照10Gbits/s （没记错的话是万兆光纤）来计算下64个字节下的包的转发率。

![img](https://pic3.zhimg.com/80/v2-89783a27999f5558ee9d4bd78554c9b2_720w.webp)

最短帧大小

10*1024*1024*1024*1024/(12+7+1+64) *8 约等于 1000M*10 /(12+7+1+64) *8 = 14.880952380952381 M/PPS （百万数据包）
也就是1s可以发送 1千400万个数据包。
注意，这里面的Data长度是在46-1500个字节之间，所以最小的帧的长度为 ： 6+6+2+46+4 = 64个字节。
线速：网卡或网络支持的最极限速度。
汇总数据：

![img](https://pic1.zhimg.com/80/v2-b59633a751fbb182d04f050d831e4504_720w.webp)

网卡的限速

arrival为每个数据包之间的时间间隔。
rte：runtime environment 即运行环境。
eal： environment abstraction layer 即抽象环境层。

### **3.2. UIO：用户空间IO**

小的内核模块，用于将设备内存映射到用户空间，并且注册中断。
uio_pci_generic 为linux 内核模块，提供此功能，可以通过 modprobe uio_pci_generic 加载。
但是其不支持虚拟功能，DPDK，提供一个替代模块 igb_uio模块，通过
sudo modprobe uio
sudo insmod kmod/igb_uio.ko
命令加载。

### **3.3. VFIO**

VFIO是一个可以安全的把设备IO、中断、DMA等暴露到用户空间（usespace），从而在用户空间完成设备驱动的框架。用户空间直接访问设备，虚拟设备的分配可以获得更高的IO性能。
参考（[https://blog.csdn.net/wentyoon/article/details/60144824](https://link.zhihu.com/?target=https%3A//blog.csdn.net/wentyoon/article/details/60144824)）
sudo modprobe vfio-pci
命令加载vfio驱动。
1.将两个82599以太网绑定到VFIO ./tools/dpdk_nic_bind.py -b vfio-pci 03：00.0 03：00.1
3.将82599 ehter绑定到IGB_UIO ./tools/dpdk_nic_bind.py -b igb_uio 03：00.0 03：00.1
可参看：[http://www.cnblogs.com/vancasola/p/9378970.html](https://link.zhihu.com/?target=http%3A//www.cnblogs.com/vancasola/p/9378970.html) 进行配置vfio驱动模式。
两者都是用户空间的网卡驱动模块，只是据说UIO依赖IOMMU，VFIO性能更好，更安全，不过必须系统和BSIO支持
通过工具查看现在的绑定情况：

![img](https://pic3.zhimg.com/80/v2-a6518a53b7d2ea0eed9a02c3f991a86a_720w.webp)

### 3.4网卡的限速

说明： 以上driv谁说明在使用的网卡驱动，后面unused为未使用可以兼容的网卡驱动。
绑定命令：
./dpdk-devbind.py –bind=ixgbe 01:00.0

![img](https://pic3.zhimg.com/80/v2-9c45e2976857201a4c376ee462476cde_720w.webp)

绑定网卡和驱动

注意在DPDK的驱动情况下，用ifconfig是看不到网卡的。

### 3.5. PMD

PMD, Poll Mode Driver 即轮询驱动模式 ，DPDK用这种轮询的模式替换中断模式

### 3.6 .RSS

RSS(Receive Side Scaling)是一种能够在多处理器系统下使接收报文在多个CPU之间高效分发的网卡驱动技术。

网卡对接收到的报文进行解析，获取IP地址、协议和端口五元组信息
网卡通过配置的HASH函数根据五元组信息计算出HASH值,也可以根据二、三或四元组进行计算。
取HASH值的低几位(这个具体网卡可能不同)作为RETA(redirection table)的索引
根据RETA中存储的值分发到对应的CPU
DPDK支持设置静态hash值和配置RETA。 不过DPDK中RSS是基于端口的，并根据端口的接收队列进行报文分发的。 例如我们在一个端口上配置了3个接收队列(0,1,2)并开启了RSS，那么 中就是这样的:

{0,1,2,0,1,2,0………}

运行在不同CPU的应用程序就从不同的接收队列接收报文，这样就达到了报文分发的效果。
在DPDK中通过设置rte_eth_conf中的mq_mode字段来开启RSS功能， rx_mode.mq_mode = ETH_MQ_RX_RSS。
当RSS功能开启后，报文对应的rte_pktmbuf中就会存有RSS计算的hash值，可以通过pktmbuf.hash.rss来访问。 这个值可以直接用在后续报文处理过程中而不需要重新计算hash值，如快速转发，标识报文流等。

### 3.7 .对称RSS

在网络应用中，如果同一个连接的双向报文在开启RSS之后被分发到同一个CPU上处理，这种RSS就称为对称RSS。 DPDK的hash算法没办法做到这一点，
对我们需要解析http报文，那么请求和访问如果采用普通的rss就造成了发送和返回报文无法匹配的问题，如果dpdk要支持需要替换其Hash算法。

### 3.8. NUMA架构

NUMA(Non-Uniform Memory Architecture 非一致性内存架构）系统。
特点是每个处理器都有本地内存、访问本地的内存块，访问其他处理器对应的内存需要通过总线，慢。

![img](https://pic1.zhimg.com/80/v2-163afc1a9a00a063721a026a018abddc_720w.webp)

NUMA架构

![img](https://pic1.zhimg.com/80/v2-077e36e59ec9e177af1a1a569e05fc50_720w.webp)

经典计算机架构

### 3.9. Hugepages大页内存

操作系统中，内存分配是按照页为单位分配的，页面的大小一般为4kB，如果页面大小固定内存越大，对应的页项越多，通过多级内存访问越慢，TLB方式访问内存更快，
但是TLB存储的页项不多，所以需要减少页面的个数，那么就通过增加页面大小的办法，增大内存页大小到2MB或1GB等。
DPDK主要分为2M和1G两种页面，具体支持要靠CPU，可以从cpu的flags里面看出来，举个例子：
如果flags里面有pse标识，标识支持2M的大内存页面；
如果有pdge1gb 标识，说明支持1G的大内存页。

![img](https://pic4.zhimg.com/80/v2-3a645123eecb54c2d7075d2a12a6b76f_720w.webp)

cpu的大页支持

![img](https://pic2.zhimg.com/80/v2-cc08954d47b1d8900434b0c5f2a71a65_720w.webp)

查看内存大页信息

## 4.重要模块划分

以下为重要的内核模块划分。

![img](https://pic4.zhimg.com/80/v2-02b6c344a450c75431734e7af571906f_720w.webp)

原文链接：https://zhuanlan.zhihu.com/p/281250848

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)