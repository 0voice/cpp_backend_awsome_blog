# 【NO.454】DPDK技术系统学习一（接收，发送，arp，icmp功能测试）

***\*如何技术不去手动做练习实践，就总有一种无从下手的感觉\****

## 1.准备环境并启动，使用dpdk接管其中一个网卡。

ubuntu虚拟机环境配置多队列网卡，安装dpdk。

在环境已经配置ok的前提下，每次重启环境后需要重新配置环境变量，并且绑定网卡。

```
export RTE_SDK=/home/hlp/dpdk/dpdk-stable-19.08.2
export RTE_TARGET=x86_64-native-linux-gcc
ifconfig    #注意保存要绑定的网卡的ip和mac地址，理解是mac地址比较重要
#这里我dpdk要绑定eth0网卡，其对应的ip和mac为  192.168.50.59和00-0c-29-4d-f0-d3
sudo ifconfig eth0 down  #关闭要绑定的网卡
./usertools/dpdk-setup.sh #通过脚本绑定网卡，使dpdk接管网卡数据。 这里用49

```

## 2.测试dpdk接管网卡数据，测试对udp数据的接收。

### 2.1.描述预计准备

通过第0步，dpdk已经接管了网卡，个人理解是这里与mac地址。==》dpdk接管网卡

获取老师提供的已有的基于dpdk实现的测试接收功能的demo代码。==》准备demo

demo实现原理 ==》通过dpdk提供的接口获取到网卡数据，对数据进行过滤，观察udp数据

参考dpdk examples目录，用makefile进行编译。 ===》编译测试代码，使用make命令

查看生成的可执行文件，目录如下：

```
root@ubuntu:/home/hlp/dpdk/dpdk-stable-19.08.2/examples/01_recv# tree
├── build                    #这个目录都是编译生成的相关文件
│   ├── app
│   │   ├── dpdk_recv
│   │   └── dpdk_recv.map
│   ├── dpdk_recv            #生成的可执行文件
│   ├── dpdk_recv.map
│   ├── _install
│   ├── _postbuild
│   ├── _postinstall
│   ├── _preinstall
│   └── recv.o
├── Makefile                #编译makefile配置文件
└── recv.c                    #我们的demo代码
2 directories, 12 files
```

**运行测试进行查看，**

===》网卡接收到的数据过多

===》使用测试代码对接收到数据进行过滤，解析udp的相关数据，通过打印观察现象，

===》运行测试demo，使用串口调试工具模拟udp的发送，观察demo打印信息。

### 2.2.正确测试结果如下：

**1：测试demo运行如下：**

```
root@ubuntu:/home/hlp/dpdk/dpdk-stable-19.08.2/examples/01_recv# ./build/dpdk_recv EAL: Detected 8 lcore(s)EAL: Detected 1 NUMA nodesEAL: Multi-process socket /var/run/dpdk/rte/mp_socketEAL: Selected IOVA mode 'PA'EAL: No available hugepages reported in hugepages-1048576kBEAL: Probing VFIO support...EAL: VFIO support initializedEAL: PCI device 0000:02:06.0 on NUMA socket -1EAL:   Invalid NUMA socket, default to 0EAL:   probe driver: 8086:100f net_e1000_emEAL: PCI device 0000:03:00.0 on NUMA socket -1EAL:   Invalid NUMA socket, default to 0EAL:   probe driver: 15ad:7b0 net_vmxnet3
```

**2：启动网络调试助手进行数据发送测试：==》中间可能有发送不成功，下文分析**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215171904_81123.jpg)

**描述：**

====》从图中可以看到，测试发送后，基于dpdk实现的测试demo运行ok

====》demo可以正常接收到我们的数据，并正常分析出我们的报文中的原ip，目的ip，以及发送内容

**遗留问题：**

====》 这里的端口打印可能有问题，后期通过自己实现解决

====》这里除了接收我们的消息外，还会接收到相关其他的udp数据，为什么？

**3：测试中流程分析：**

我的测试场景是：使用物理机+虚拟机（linux环境进行测试）

在物理机上用串口模拟工具下发，目标ip填写的是上文保存的dpdk 绑定网卡前的ip，端口随机。

使用串口工具进行测试时，会发现必然无法发送成功的场景，这是因为这里的发送ip没有找到对应的arp表。

**分析：**

===》要想在物理机发送给虚拟机的链路ok，需要arp表的支持。

===》ip其实是可变的，mac地址（唯一）是寻址的关键，需要配置arp表。

===》配置arp表需要关注，arp -a查出的arp表是多个接口有对应关系，需要配置arp表在对应的接口上。

**配置arp表的相关命令如下：**

```
#注意  这里一定要保存对dpdk绑定的网卡的mac地址# 查看arp 表 arp -a#使用arp命令进行添加 ==》还是不生效，没添加在对应接口中arp -s 192.168.50.59 00-0c-29-4d-f0-d3arp -d 192.168.50.59 #使用netsh进行arp的绑定#1：找到 网线或者网卡对应的idx netsh i i show in#2： 绑定网卡和解绑命令（这里使用有线网或者以太网） 16是查找到的网卡对应的idx值netsh -c i i add neighbors 16 192.168.50.59 00-0c-29-4d-f0-d3#配置后可以通过arp -a查看，发现多了一个静态arp#192.168.50.59         00-0c-29-85-2e-88     静态netsh  i i delete neighbors 16
```

## **3.基于dpdk实现通过网卡发送数据。**

### **3.1.思路描述**

在dpdk 接管网卡数据，解析udp数据的基础上，实现udp数据的回复发送。

通过dpdk相关接口，初始化时要获取到dpdk绑定的本端网卡的mac地址

需要根据获取到的udp的数据报，按照ethhdr协议栈，ip协议栈，udp协议栈依次解析，获取原端口，目的端口，原ip，目的ip，来源数据的mac地址等必要数据

根据相关协议栈规则，构造附后udp报文的结构的数据 eth头，ip头，udp头+body数据的结构进行发送。

### **3.2.测试**

这里直接用老师的demo，02_send.c，编译和运行同上。

参考上文，同样需要手动配置绑定arp，使通路可以识别。

观察：串口通信工具能正常接收到dpdk回复的原数据。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215171905_79049.jpg)

## **4.实现arp的探测回复功能。**

每一次测试时，都需要手动配置arp才能使链路通，需要arp功能支持。

**arp业务处理有三个点：**

===》1：收到arp探测报文，对探测报文的回复（arp response）

===》2：广播发送arp探测报文（arp request）

===》3：arp表的保存

arp主要实现和获取了网络ip和mac地址的对应映射，是局域网网络互通的关键。

### **4.1 .arp描述及demo梳理**

dpdk可以接管网卡接收到的所有数据。

对数据进行解析，过滤arp报文，根据协议类型（0x0806）。

对所有的arp报文进行解析（根据本机ip），因为会收到局域网所有节点的arp探测报文。

已经过滤出是本机ip对应探测的arp报文，根据arp协议（eth头+arp头），构造arp报文进行发送

### **4.2 .测试demo的操作及运行**

获取老师提供的arp.c文件，修改文件中设定的本机ip。

删除测试物理机上已经配置的arp映射关系（netsh i i delete neighbors 16）。

编译并进行运行测试。

===》在没有对物理机arp表做dpdk绑定的网卡映射信息前提下，使用串口工具进行udp发送测试

===》启动后发现，可以收到并过滤出局域网多个环境发来的arp探测请求

===》使用串口工具进行udp发送测试，观察到现象

```
#1： 使用arp -a查物理机arp表，新增了dpdk接收网卡对应mac的一条信息  192.168.50.59         00-0c-29-4d-f0-d3     动态#2: 不用手动配置arp  上文udp的发送和接收都正常。
```

**理解：**在发送udp/tcp等数据前，会查看对应的arp缓存表，如果没有找到映射信息，会先发arp探测报文。

**现象：**可以尝试观察日志/抓包，针对物理机给dpdk发送udp数据的场景，分析这里arp 探测报文的发送时机。（物理机可以抓包查看，dpdk测试机可以通过日志查看，应该是在接收udp报文前有一个物理机ip的arp探测报文）

## 5.icmp协议报文的实现

### **5.1.概述**

icmp协议是ip层的附属协议，

ICMP 相当于网络世界的侦察兵。常用的有两种类型，主动探查的查询报文和异常报告的差错报文。

ping 命令使用查询报文，Traceroute 命令使用差错报文。

这里实现基础的ping命令的功能，

### **5.2.测试**

1：获取已有的测试demo，同上，修改源码中设定本机的ip值，获取网卡mac地址与该ip绑定，提供arp支持（这里的ip设置要注意，注意和测试机同一网段）。

2：对网卡接收到的报文，根据ip头部协议类型解析提取icmp相关的报文，根据icmp协议头部类型，提取为ping命令对应报文，解析后获取必要信息构造icmp报文进行回复。

3：运行该demo的情况下，在测试机器上运行ping 网卡配置的ip，观察返回报文以及我们的运行日志。

### **5.3.测试结果**

可以看到，在没有启动测试代码前，无法ping通，启动后，已经实现ping命令功能，日志上可以看到一些日志：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/15/20221215171906_30011.jpg)



原文链接：https://zhuanlan.zhihu.com/p/497811718

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)