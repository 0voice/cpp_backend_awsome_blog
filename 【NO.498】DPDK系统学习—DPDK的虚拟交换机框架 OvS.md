# 【NO.498】DPDK系统学习—DPDK的虚拟交换机框架 OvS

## 1.目录：

**多队列网卡**

1. **多队列网卡硬件实现**
2. **内核对多队列网卡的支持**
3. **多队列网卡的结构**
4. **DPDK 与多队列网卡**

**虚拟化**

1. **CPU 虚拟化**
2. **内存虚拟化**
3. **I/O 虚拟化**

**Virtio**

1. **为什么是 virtio?**

## 2.多队列网卡

### 2.1.多队列网卡硬件实现

有四个硬件队列(Queue0, Queue1, Queue2, Queue3),当收到报文时,通过 hash 包头

的(sip, sport, dip, dport)四元组,将一条流总是收到相同队列,同时触发与该队列绑定的中断。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160918_30622.jpg)

### 2.2.内核对多队列网卡的支持

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160919_25103.jpg)

Linux 内核中, RPS ( Receive Packet Steering )在接收端提供了这样的机制。RPS 主要是把软中断的负载均衡到 CPU 的各个 core 上,网卡驱动对每个流生成一个 hash 标识,这个 hash值可以通过四元组(源 IP 地址 SIP ,源四层端口 SPOR T,目的 IP 地址 DIP ,目的四层端口DPORT )来计算,然后由中断处理的地方根据这个 hash 标识分配到相应的 core 上去,这样就可以比较充分地发挥多核的能力了。

NAPI 是 linux 上采用一种提高网络处理效率的技术,其核心概念就是不采用中断的方式读取数据,取而代之是采用中断唤醒数据接收的服务程序,然后 POLL 的方式轮询数据。

### 2.3.NAPI 的优点:

1. 中断缓和,在日常使用中,网卡产生的高达几 k/s,每次中断都需要系统来处理,是一个很大的压力,而 NAPI 使用轮询是禁止了网卡接收中断,减小处理中断的压力。
2. 数据包节流,NAPI 之前的 NIC 总在接收数据包之后产生一个 IRQ,接着在中断服务函数将 skb 加入本地 softnet,然后触发本地 NET_RX_SOFTIRQ 软中断后续处理。如果包速过高,因为 IRQ 的优先级高于 SoftIRQ,导致系统的大部分资源都在响应中断,但 softnet 的队列大小有限,接收到的超额数据包也只能丢掉,所以这时这个模型是在用宝贵的系统资源做无用功。而 NAPI 则在这样的情况下,直接把包丢掉,不会继续将需要丢掉的数据包扔给内核去处理,这样,网卡将需要丢掉的数据包尽可能的早丢弃掉,内核将不可见需要丢掉的数据包,这样也减少了内核的压力。

### 2.4.NAPI 的缺点:

1. 对于上层的应用程序而言,系统不能在每个数据包接收到的时候都可以及时地去处理它,而且随着传输速度增加,累计的数据包将会耗费大量的内存。
2. 另外一个问题是对于大的数据包处理比较困难,原因是大的数据包传送到网络层上的时候耗费的时间比短数据包长很多(即使是采用 DMA 方式)。所以,NAPI 技术适用于对高速率的短长度数据包的处理。

QDisc 是 queueing discipline 的简写,它是理解流量(traffic control)的基础。无论何时,内核如果需要通过某个网络接口发送数据包,需要按照这个接口配置的 qdisc(排队规则)把数据包加入队列。然后,内核会尽可能多的从 qdisc 里面取出数据包,把它们交给网络适配器模块。最简单的 QDisc 是 pfifo,他不对进入的数据包做任何处理,数据包采用先入先出的方式。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160919_37166.jpg)

### 2.5.多队列网卡的结构

Linux 内核的网卡结构体是由 net_device 表示,数据包是由 sk_buff 表示的

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160920_94455.jpg)

### 2.6.接收端

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160921_58741.jpg)

### 2.7.发送端

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160922_42057.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160922_38344.jpg)

### 2.8.DPDK 与多队列网卡

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160923_88685.jpg)

## 3.虚拟化

虚拟化是资源的逻辑表示,不受物理设备的约束。虚拟化技术的实现形式是在系统中加入一个虚拟化层,虚拟化层将下层的资源抽象成另一种形式的资源,提供给上层使用。

### 3.1.CPU 虚拟化

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160924_53198.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160925_55526.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160925_90325.jpg)

### 3.2.内存虚拟化

虚拟机本质上是 Host 机上的一个进程,按理说应该可以使用 Host 机的虚拟地址空间,但由于在虚拟化模式下,虚拟机处于非 Root 模式,无法直接访问 Root 模式下的 Host 机上的内存。

这个时候就需要 VMM 的介入, VMM 需要 intercept (截获)虚拟机的内存访问指令,然后 virtualize(模拟)Host 上的内存,相当于 VMM 在虚拟机的虚拟地址空间和 Host 机的虚拟地址空间中间增加了一层,即虚拟机的物理地址空间,也可以看作是 Qemu 的虚拟地址空间(稍微有点绕,但记住一点,虚拟机是由 Qemu 模拟生成的就比较清楚了)。所以,内存软件虚拟化的目标就是要将虚拟机的虚拟地址(Guest Virtual Address, GVA)转化

为 Host 的物理地址(Host Physical Address, HPA),中间要经过虚拟机的物理地址(GuestPhysical Address, GPA)和 Host 虚拟地址(Host Virtual Address)的转化,即:

GVA -> GPA -> HVA -> HPA

其中前两步由虚拟机的系统页表完成,中间两步由 VMM 定义的映射表(由数据结构kvm_memory_slot 记录)完成,它可以将连续的虚拟机物理地址映射成非连续的 Host 机虚拟地址,后面两步则由 Host 机的系统页表完成。如下图所示。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160926_17253.jpg)

### 3.3.**这样做得目的有两个:**

1. 提供给虚拟机一个从零开始的连续的物理内存空间。
2. 在各虚拟机之间有效隔离、调度以及共享内存资源。

### 3.4.影子页表技术

接上图,我们可以看到,传统的内存虚拟化方式,虚拟机的每次内存访问都需要 VMM 介入,并由软件进行多次地址转换,其效率是非常低的。因此才有了影子页表技术和 EPT 技术。

影子页表简化了地址转换的过程,实现了 Guest 虚拟地址空间到 Host 物理地址空间的直接映射。要实现这样的映射,必须为 Guest 的系统页表设计一套对应的影子页表,然后将影子页表装入 Host 的 MMU 中,这样当 Guest 访问 Host 内存时,就可以根据 MMU 中的影子页表映射关系,完成 GVA 到 HPA 的直接映射。而维护这套影子页表的工作则由 VMM 来完成。

由于 Guest 中的每个进程都有自己的虚拟地址空间,这就意味着 VMM 要为 Guest 中的每个进程页表都维护一套对应的影子页表,当 Guest 进程访问内存时,才将该进程的影子页表装入 Host 的 MMU 中,完成地址转 换。

我们也看到,这种方式虽然减少了地址转换的次数,但本质上还是纯软件实现的,效率还是不高,而且 VMM 承担了太多影子页表的维护工作,设计不好。

为了改善这个问题,就提出了基于硬件的内存虚拟化方式,将这些繁琐的工作都交给硬件来完成,从而大大提高了效率。

### 3.5.EPT 技术

这方面 Intel 和 AMD 走在了最前面,Intel 的 EPT 和 AMD 的 NPT 是硬件辅助内存虚拟化的代表,两者在原理上类似,本文重点介绍一下 EPT 技术。

如下图是 EPT 的基本原理图示,EPT 在原有 CR3 页表地址映射的基础上,引入了 EPT 页表来实现另一层映射,这样,GVA->GPA->HPA 的两次地址转换都由硬件来完成

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160927_53150.jpg)

这里举一个小例子来说明整个地址转换的过程。假设现在 Guest 中某个进程需要访问内存,CPU 首先会访问 Guest 中的 CR3 页表来完成 GVA 到 GPA 的转换,如果 GPA 不为空,则 CPU 接着通过 EPT 页表来实现 GPA 到 HPA 的转换(实际上,CPU 会首先查看硬件 EPT TLB 或者缓存,如果没有对应的转换,才会进一步查看 EPT 页表),如果 HPA 为空呢,则 CPU 会抛出 EPT Violation 异常由 VMM 来处理。如果 GPA 地址为空,即缺页,则 CPU 产生缺页异常,注意,这里,如果是软件实现的方式,则会产生 VM-exit,但是硬件实现方式,并不会发生 VM-exit,而是按照一般的缺页中断处理,这种情况下,也就 是 交 给 Guest 内 核 的 中 断 处 理 程 序 处 理 。 在 中 断 处 理 程 序 中 会 产 生EXIT_REASON_EPT_VIOLATION,Guest 退出,VMM 截获到该异常后,分配物理地址并建立GVA 到 HPA 的映射,并保存到 EPT 中,这样在下次访问的时候就可以完成从 GVA 到HPA 的转换了。

### 3.6.I/O 虚拟化

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160928_42439.jpg)

**I/O 全虚拟化(上图中间部分)**

这种方式比较好理解,简单来说,就是通过纯软件的形式来模拟虚拟机的 I/O 请求。以qemu-kvm 来举例,内核中的 kvm 模块负责截获 I/O 请求,然后通过事件通知告知给用户空间的设备模型 qemu,qemu 负责完成本次 I/O 请求的模拟。

优点:不需要对操作系统做修改,也不需要改驱动程序,因此这种方式对于多种虚拟化技术的「可移植性」和「兼容性」比较好。

缺点:纯软件形式模拟,自然性能不高,另外,虚拟机发出的 I/O 请求需要虚拟机和 VMM之间的多次交互,产生大量的上下文切换,造成巨大的开销。

**I/O 半虚拟化(上图左侧)**

针对 I/O 全虚拟化纯软件模拟性能不高这一点, I/O 半虚拟化前进了一步。它提供了一种机制,使得 Guest 端与 Host 端可以建立连接,直接通信,摒弃了截获模拟这种方式,从而获得较高的性能。

值得注意的有两点:1)采用 I/O 环机制,使得 Guest 端和 Host 端可以共享内存,减少了虚拟机与 VMM 之间的交互;2)采用事件和回调的机制来实现 Guest 与 Host VMM 之间的通信。这样,在进行中断处理时,就可以直接采用事件和回调机制,无需进行上下文切换,减少了开销。

要实现这种方式, Guest 端和 Host 端需要采用类似于 C/S 的通信方式建立连接,这也就意味着要修改 Guest 和 Host 端操作系统内核相应的代码,使之满足这样的要求。为了描述方便,我们统称 Guest 端为前端,Host 端为后端。

前后端通常采用的实现方式是驱动的方式,即前后端分别构建通信的驱动模块,前端实现在内核的驱动程序中,后端实现在 qemu 中,然后前后端之间采用共享内存的方式传递数据。关于这方面一个比较好的开源实现是 virtio

**优点:**性能较 I/O 全虚拟化有了较大的提升

**缺点:**要修改操作系统内核以及驱动程序,因此会存在移植性和适用性方面的问题,导致其使用受限。

**I/O 直通或透传技术(上图右侧)**

上面两种虚拟化方式,还是从软件层面上来实现,性能自然不会太高。最好的提高性能的方式还是从硬件上来解决。如果让虚拟机独占一个物理设备,像宿主机一样使用物理设备,那无疑性能是最好的。

I/O 直通技术就是提出来完成这样一件事的。它通过硬件的辅助可以让虚拟机直接访问物理设备,而不需要通过 VMM 或被 VMM 所截获。

由于多个虚拟机直接访问物理设备,会涉及到内存的访问,而内存又是共享的,那怎么来隔离各个虚拟机对内存的访问呢,这里就要用到一门技术——IOMMU,简单说,IOMMU就是用来隔离虚拟机对内存资源访问的。

I/O 直通技术需要硬件支持才能完成,这方面首选是 Intel 的 VT-d 技术,它通过对芯片级的改造来达到这样的要求,这种方式固然对性能有着质的提升,不需要修改操作系统,移植性也好。但该方式也是有一定限制的,这种方式仅限于物理资源丰富的机器,因为这种方式仅仅能满足一个设备分配给一个虚拟机,一旦一个设备被虚拟机占用了,其他虚拟机时无法使用该设备的。

## 4.Virtio

virtio 是一种 I/O 半虚拟化解决方案,是一套通用 I/O 设备虚拟化的程序,是对半虚拟 化 Hypervisor 中 的 一 组 通 用 I/O 设 备 的 抽 象 。 提 供 了 一 套 上 层 应 用 与 各Hypervisor 虚拟化设备(KVM,Xen,VMware 等)之间的通信框架和编程接口,减少跨平台所带来的兼容性问题,大大提高驱动程序开发效率。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160928_83325.jpg)

### 4.1.为什么是 virtio?

在完全虚拟化的解决方案中,guest VM 要使用底层 host 资源,需要 Hypervisor 来截获所有的请求指令,然后模拟出这些指令的行为,这样势必会带来很多性能上的开销。半虚拟化通过底层硬件辅助的方式,将部分没必要虚拟化的指令通过硬件来完成,Hypervisor只负责完成部分指令的虚拟化,要做到这点,需要 guest 来配合,guest 完成不同设备的前端驱动程序,Hypervisor 配合 guest 完成相应的后端驱动程序,这样两者之间通过某种交互机制就可以实现高效的虚拟化过程。

由于不同 guest 前端设备其工作逻辑大同小异(如块设备、网络设备、 PCI 设备、 balloon驱动等),单独为每个设备定义一套接口实属没有必要,而且还要考虑扩平台的兼容性问题,另外,不同后端 Hypervisor 的实现方式也大同小异(如 KVM、Xen 等),这个时候,就需要一套通用框架和标准接口(协议)来完成两者之间的交互过程,virtio 就是这样一套标准,它极大地解决了这些不通用的问题。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160929_64507.jpg)

### 4.2.virtio 的架构

从总体上看,virtio 可以分为四层,包括前端 guest 中各种驱动程序模块,后端 Hypervisor(实现在 Qemu 上)上的处理程序模块,中间用于前后端通信的 virtio 层和 virtio-ring 层,virtio 这一层实现的是虚拟队列接口,算是前后端通信的桥梁,而 virtio-ring 则是该桥梁的具体实现,它实现了两个环形缓冲区,分别用于保存前端驱动程序和后端处理程序执行的信息。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160930_43552.jpg)

严格来说, virtio 和 virtio-ring 可以看做是一层, virtio-ring 实现了 virtio 的具体通信机制和数据流程。或者这么理解可能更好, virtio 层属于控制层,负责前后端之间的通知机制(kick,notify)和控制流程,而 virtio-vring 则负责具体数据流转发。

### 4.3.virtio 数据流交互机制

vring 主要通过两个环形缓冲区来完成数据流的转发,如下图所示。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160930_77223.jpg)

vring 包含三个部分,描述符数组 desc,可用的 available ring 和使用过的 usedring。

desc 用于存储一些关联的描述符,每个描述符记录一个对 buffer 的描述,available ring 则用于 guest 端表示当前有哪些描述符是可用的,而 used ring 则表示 host 端哪些描述符已经被使用。

Virtio 使用 virtqueue 来实现 I/O 机制,每个 virtqueue 就是一个承载大量数据的队列,具体使用多少个队列取决于需求,例如,virtio 网络驱动程序(virtio-net)使用两个队列(一个用于接受,另一个用于发送),而 virtio 块驱动程序(virtio-blk)仅使用一个队列。

具体的,假设 guest 要向 host 发送数据,首先,guest 通过函数 virtqueue_add_buf将存有数据的 buffer 添加到 virtqueue 中,然后调用 virtqueue_kick 函数,virtqueue_kick 调用 virtqueue_notify 函数,通过写入寄存器的方式来通知到 host。

host 调用 virtqueue_get_buf 来获取 virtqueue 中收到的数据。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160931_68216.jpg)

存放数据的 buffer 是一种分散-聚集的数组,由 desc 结构来承载,如下是一种常用的 desc的结构:

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160932_12199.jpg)

当 guest 向 virtqueue 中写数据时,实际上是向 desc 结构指向的 buffer 中填充数据,完了会更新 available ring,然后再通知 host。当 host 收到接收数据的通知时,首先从 desc 指向的 buffer 中找到 available ring 中添加的 buffer,映射内存,同时更新 used ring,并通知 guest 接收数据完毕。

virtio 是 guest 与 host 之间通信的润滑剂,提供了一套通用框架和标准接口或协议来完成两者之间的交互过程,极大地解决了各种驱动程序和不同虚拟化解决方案之间的适配问题。virtio 抽象了一套 vring 接口来完成 guest 和 host 之间的数据收发过程,结构新颖,接口清晰。

### 4.4.Vhost

vhost 是 virtio 的一种后端实现方案,在 virtio 简介中,我们已经提到 virtio 是一种半虚拟化的实现方案,需要虚拟机端和主机端都提供驱动才能完成通信,通常, virtio主机端的驱动是实现在用户空间的 qemu 中,而 vhost 是实现在内核中,是内核的一个模块 vhost-net.ko。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160932_14819.jpg)

### 4.5.为什么要用 vhost

在 virtio 的机制中,guest 与 用户空间的 Hypervisor 通信,会造成多次的数据拷贝和 CPU 特权级的上下文切换。例如 guest 发包给外部网络,首先,guest 需要切换到host kernel,然后 host kernel 会切换到 qemu 来处理 guest 的请求, Hypervisor通过系统调用将数据包发送到外部网络后,会切换回 host kernel , 最后再切换回guest。这样漫长的路径无疑会带来性能上的损失。

vhost 正是在这样的背景下提出的一种改善方案,它是位于 host kernel 的一个模块,用于和 guest 直接通信,数据交换直接在 guest 和 host kernel 之间通过 virtqueue来进行,qemu 不参与通信,但也没有完全退出舞台,它还要负责一些控制层面的事情,比如和 KVM 之间的控制指令的下发等。

### 4.6.vhost 的数据流程

下图左半部分是 vhost 负责将数据发往外部网络的过程, 右半部分是 vhost 大概的数据交互流程图。其中,qemu 还是需要负责 virtio 设备的适配模拟,负责用户空间某些管理控制事件的处理,而 vhost 实现较为纯净,以一个独立的模块完成 guest 和 host kernel 的数据交换过程。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/17/20221217160933_18122.jpg)

vhost 与 virtio 前端的通信主要采用一种事件驱动 eventfd 的机制来实现,guest 通知 vhost 的事件要借助 kvm.ko 模块来完成,vhost 初始化期间,会启动一个工作线程work 来监听 eventfd,一旦 guest 发出对 vhost 的 kick event,kvm.ko 触发ioeventfd 通知到 vhost,vhost 通过 virtqueue 的 avail ring 获取数据,并设置used ring。同样,从 vhost 工作线程向 guest 通信时,也采用同样的机制,只不过这种情况发的是一个回调的 call envent,kvm.ko 触发 irqfd 通知 guest。

原文链接：https://zhuanlan.zhihu.com/p/510739584

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)