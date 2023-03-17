# 【NO.505】盘点后端开发那些值得学习的优秀开源项目

今天给大家推荐一些值得学习的**开源项目**，包括**C, C++，Golang，Java**等后台开发主流语言的项目，大家工作之余，可以花点时间学习和研究这些项目的优秀**设计和实现，提高自己**。

![img](https://pic2.zhimg.com/80/v2-20533bd51d4ff1c43f9e023562a757e9_720w.webp)

## 1.**学习开源项目好处**

**首先是提升编程技能。**对计算机专业相关的学生而言，在学习编程后，能验证能力的只是一些简单项目，但通过阅读开源项目的源码，你不仅可以学习顶级项目的设计思路，还可以学习顶级开发者的编程思路，比如通过学习提升代码的可读性和简洁性。同时，你也可以提交PR、注释，而社区里的资深工程师会给出直接反馈，这比你自己摸索要成长得更快。

**其次能帮你找到满意的工作。**如果你在开源项目上留下印记，无论是贡献代码、技术文档、应用案例等等，这些都能证明个人能力，甚至，有时你的简历只需放上GitHub个人账号链接就已足够：）

**最后，开源也许会成为你热衷的事业。**处在这样一个开源崛起的时代，尤其在国内很多顶级项目不断催生，现在正是那些热爱开源理念和开源软件的开发者大展鸿图的时候，他们有的在学生时代就已学习和贡献开源，开源世界为他们带来了荣誉和快乐，而他们在未来也致力于开发和运营开源软件。

**总之**，对于高校学生或者已经工作几年同学，只要你能通过开源项目的代码证明自己的实力，这无疑像是拿到了观看球赛的前排门票，你不会再因为“内卷”而发愁，因为你的前方视野足够辽阔。

## 2.**如何学习开源项目**

### 2.1 **首先了解整体架构**

查找和阅读该项目的博客和资料，通过google你能找到某个项目大体介绍的博客，快速阅读一下就能对项目的目的、功能、基本使用有个大概的了解。

### 2.2 **先把项目跑起来**

如果该项目有提供现成的example工程，首先尝试按照开始文档的介绍运行example，如果运行顺利，那么恭喜你顺利开了个好头;如果遇到问题，首先尝试在项目的FAQ等文档里查找答案，再次，可以将问题（例如异常信息）当成关键词去搜索，查找相关的解决办法，你遇到了，别人一般也会遇到，热心的朋友会记录下解决的过程;最后，可以将问题提交到项目的邮件列表，请大家帮你看看。在没有成功运行example之前，不要尝试修改example。

如果时间允许，尝试从源码构建该项目。通常开源项目都会提供一份构建指南，指导你如何搭建一个用于开发、调试和构建的环境。尝试构建一个版本。

### 2.3 **阅读源码建议**

（1）阅读源码之前，查看该项目是否提供架构和设计文档，阅读这些文档可以了解该项目的大体设计和结构，读源码的时候不会无从下手。
（2）阅读源码之前，一定要能构建并运行该项目，有个直观感受。
（3）阅读源码的第一步是抓主干，尝试理清一次正常运行的代码调用路径，这可以通过debug来观察运行时的变量和行为。修改源码加入日志和打印可以帮助你更好的理解源码。
（4）适当画图来帮助你理解源码，在理清主干后，可以将整个流程画成一张流程图或者标准的UML图，帮助记忆和下一步的阅读。
（5）挑选感兴趣的“枝干”代码来阅读，比如你对网络通讯感兴趣，就阅读网络层的代码，深入到实现细节，如它用了什么库，采用了什么设计模式，为什么这样做等。如果可以，debug细节代码。
（6）阅读源码的时候，重视单元测试，尝试去运行单元测试，基本上一个好的单元测试会将该代码的功能和边界描述清楚。
（7）在熟悉源码后，发现有可以改进的地方，有精力、有意愿可以向该项目的开发者提出改进的意见或者issue，甚至帮他修复和实现，参与该项目的发展。

### 2.4 **开启自己的开源项目**

通常在阅读文档和源码之后，你能对该项目有比较深入的了解了，但是该项目所在领域，你可能还想搜索相关的项目和资料，看看有没有其他的更好的项目或者解决方案。在广度和深度之间权衡。

## **3.C经典开源项目**

### 3.1 Libev**

**libev**是一个全功能和高性能的事件驱动库，基于epoll，kqueue等OS提供的基础设施。其以高效出名，它可以将IO事件，定时器，和信号统一起来，统一放在事件处理这一套框架下处理。基于Reactor模式，效率较高，并且代码精简（4.15版本8000多行），是学习事件驱动编程的很好的资源。

**特点**

- 不使用全局变量，而是每个函数都有一个循环上下文。
- 对每种事件类型使用小的观察器(一个I/O观察器在x86_64机器上使用56字节，而用libevent的话使用136字节)。
- 没有http库等组件。libev的功能非常少。
- 允许更多事件类型，例如基于wall clock或者单调时间的定时器、线程间中断等等。

更简单地说，libev的设计遵循UNIX工具箱的哲学，尽可能好地只做一件事。

**整体架构：**

![img](https://pic2.zhimg.com/80/v2-41c7ed2c6fcef13c6291b546fd7d8945_720w.webp)

开源地址：

[https://github.com/enki/libev](https://link.zhihu.com/?target=https%3A//github.com/enki/libev)

### 3.2**** **Redis**

![img](https://pic4.zhimg.com/80/v2-bb085ee23e9cfe4ac1057d9f1d27859f_720w.webp)

**Redis** 是一种经典的开源内存**Key-Value**数据结构存储，用作数据库、缓存和消息代理。Redis 提供了数据结构，例如字符串、散列、列表、集合、带有范围查询的排序集合、位图、超级日志、地理空间索引和流。Redis 内置复制、Lua 脚本、LRU 驱逐、事务和不同级别的磁盘持久化，并通过 Redis Sentinel 和 Redis Cluster 自动分区提供高可用性。

代码架构：

![img](https://pic1.zhimg.com/80/v2-d122b32903c0dc0e3fd85d9ecaff396c_720w.webp)

开源地址：

[https://github.com/redis/redis](https://link.zhihu.com/?target=https%3A//github.com/redis/redis)

### **3.3 Nginx**

![img](https://pic1.zhimg.com/80/v2-9b2265bd95e8663e05142b0b0dddb3e0_720w.webp)

**Nginx**是一款轻量级的Web服务器、反向代理服务器，由于它的内存占用少，启动极快，高并发能力强，在互联网项目中广泛应用。

**特点：**

- Nginx可以部署在网络上使用FastCGI脚本、SCGI处理程序、WSGI应用服务器或Phusion Passenger模块的动态HTTP内容，并可作为软件负载均衡器。
- Nginx使用异步事件驱动的方法来处理请求。Nginx的模块化事件驱动架构可以在高负载下提供更可预测的性能。
- Nginx是一款面向性能设计的HTTP服务器，相较于Apache、lighttpd具有占有内存少，稳定性高等优势。与旧版本（≤2.2）的Apache不同，Nginx不采用每客户机一线程的设计模型，而是充分使用异步逻辑从而削减了上下文调度开销，所以并发服务能力更强。整体采用模块化设计，有丰富的模块库和第三方模块库，配置灵活。在Linux操作系统下，Nginx使用epoll事件模型，得益于此，Nginx在Linux操作系统下效率相当高。同时Nginx在OpenBSD或FreeBSD操作系统上采用类似于epoll的高效事件模型kqueue。

**整体架构:**

![img](https://pic4.zhimg.com/80/v2-bf7b9175b368e0f44fa52d6230252957_720w.webp)

开源地址：

[https://github.com/nginx/nginx](https://link.zhihu.com/?target=https%3A//github.com/nginx/nginx)

### **3.4 SQLite**

**SQLite**是一个开源的嵌入式关系数据库，实现自包容、零配置、支持事务的SQL数据库引擎。其特点是高度便携、使用方便、结构紧凑、高效、可靠。足够小，大致3万行C代码，250K。

**整体架构：**

![img](https://pic2.zhimg.com/80/v2-eb085a37bd856b4c597a76384dd60941_720w.webp)

开源地址：

[http://www.sqlite.org/](https://link.zhihu.com/?target=http%3A//www.sqlite.org/)

### **3.5 Linux**

**Linux** 是一套免费使用和自由传播的类 Unix 操作系统，是一个基于 POSIX 和 UNIX 的多用户、多任务、支持多线程和多 CPU 的操作系统。Linux 能运行主要的 UNIX 工具软件、应用程序和网络协议。它支持 32 位和 64 位硬件。Linux 继承了 Unix 以网络为核心的设计思想，是一个性能稳定的多用户网络操作系统，目前最为流行后台服务器操作系统。

**整体架构：**

![img](https://pic3.zhimg.com/80/v2-126df77e94e6c6aca494cd9f4432839a_720w.webp)

Linux内核学习分为四个阶段。

- 首先，了解操作系统基本概念。
- 其次，了解Linux内核机制（大的框架和架构，不要在乎细节）。
- 其次，研读内核源码（选择自己感兴趣的方向，比如调度（计算），虚拟化，网络，内存，存储等）。

最后，确定个人的发展方向

- 设备驱动开发方向（嵌入式）
- 云网络开发方向（云计算）
- 虚拟化方向（云计算）
- 云存储方向（云计算）
- Linux应用开发方向（Linux后台开发）

开源地址：

[https://www.kernel.org/](https://link.zhihu.com/?target=https%3A//www.kernel.org/)

## 4.**C++开源项目**

### **4.1 TinyWebServer（初学者）**

这是一个帮助初学者快速实现网络编程、搭建属于自己的轻量级Web服务器的小项目。

**项目虽小但真的五脏俱全：**

- 使用线程池、非阻塞Socket、epoll（ET/LT均实现）、事件处理（Reactor及模拟Proactor）的并发模型。
- 使用状态机解析HTTP请求报文，支持解析GET和POST请求
- 访问服务器数据库实现web端用户注册、登录功能，可以请求服务器图片和视频文件
- 实现同步/异步日志系统，记录服务器运行状态
- 经Webbench压力测试可以实现上万的并发连接数据交换

代码地址：

[https://github.com/qinguoyi/TinyWebServer](https://link.zhihu.com/?target=https%3A//github.com/qinguoyi/TinyWebServer)

### **4.2 sylar**

C++高性能分布式服务器框架,功能最全webserver/websocket server,自定义tcp_server（包含日志模块，配置模块，线程模块，协程模块，协程调度模块，io协程调度模块，hook模块，socket模块，bytearray序列化，http模块，TcpServer模块，Websocket模块，Https模块等, Smtp邮件模块, MySQL, SQLite3, ORM,Redis,Zookeeper)。

**优点：**

- 基于epoll的IO复用机制实现Reactor模式，采用边缘触发（ET）模式，和非阻塞模式
- 由于采用ET模式，read、write和accept的时候必须采用循环的方式，直到error==EAGAIN为止，防止漏读等清况，这样的效率会比LT模式高很多，减少了触发次数
- Version-0.1.0基于单线程实现，Version-0.2.0利用线程池实现多IO线程，Version-0.3.0实现通用worker线程池，基于one loop per thread的IO模式，Version-0.4.0增加定时器，Version-0.5.0增加简易协程实现和异步日志实现
- 线程模型将划分为主线程、IO线程和worker线程，主线程接收客户端连接（accept），并通过Round-Robin策略分发给IO线程，IO线程负责连接管理（即事件监听和读写操作），worker线程负责业务计算任务（即对数据进行处理，应用层处理复杂的时候可以开启）基于时间轮实现定时器功能，定时剔除不活跃连接，时间轮的插入、删除复杂度为O(1)，执行复杂度取决于每个桶上的链表长
- 采用智能指针管理多线程下的对象资源增加简易协程实现，目前版本基于ucontext.h（供了解学习，尚未应用到本项目中）From:
- simple-coroutine增加简易C++异步日志库 From: simple-log
- 支持HTTP长连接
- 支持优雅关闭连接
- 通常情况下，由客户端主动发起FIN关闭连接客户端发送FIN关闭连接后，服务器把数据发完才close，而不是直接暴力close,如果连接出错，则服务器可以直接close.

代码地址：

[https://github.com/sylar-yin/sylar](https://link.zhihu.com/?target=https%3A//github.com/sylar-yin/sylar)

### **4.3 OpenSSL**

一个强大的安全套接字层密码库，加密HTTPS，加密SSH都贼好用，同时它还可以用于跨平台密码工具。

**OpenSSL实现了以下功能：**

- 数据保密性：信息加密就是把明码的输入文件用加密算法转换成加密的文件以实现数据的保密。加密的过程需要用到密钥来加密数据然后再解密。
- 数据完整性：加密也能保证数据的一致性。例如：消息验证码（MAC），能够校验用户提供的加密信息，接收者可以用MAC来校验加密数据，保证数据在传输过程中没有被篡改过。
- 安全验证：加密的另外一个用途是用来作为个人的标识，用户的密钥可以作为他的安全验证的标识。SSL是利用公开密钥的加密技术（RSA）来作为用户端与服务器端在传送机密资料时的加密通讯协定。

代码地址：

[https://www.openssl.org/source](https://link.zhihu.com/?target=https%3A//www.openssl.org/source)

### **4.4 LevelDB**

**LevelDB** 是一个由 Google 编写的快速键值存储库，它提供了从字符串键到字符串值的有序映射。

**LevelDB 有以下优点：**

- 提供应用程序运行上下文，方便跟踪调试
- 可扩展的、多种方式记录日志，包括命令行、文件、回卷文件、内存、syslog服务器、Win事件日志等
- 可以动态控制日志记录级别，在效率和功能中进行调整
- 所有配置可以通过配置文件进行动态调整
- 支持Java、C++、C、python等多种语言

整体架构：

![img](https://pic2.zhimg.com/80/v2-631342e43a7a38060fafdead35125505_720w.webp)

- MemTable：内存数据结构，具体实现是 SkipList。接受用户的读写请求，新的数据修改会首先在这里写入。
- Immutable MemTable：当 MemTable 的大小达到设定的阈值时，会变成 Immutable MemTable，只接受读操作，不再接受写操作，后续由后台线程 Flush 到磁盘上。
- SST Files：Sorted String Table Files，磁盘数据存储文件。分为 Level0 到 LevelN 多层，每一层包含多个 SST 文件，文件内数据有序。Level0 直接由 Immutable Memtable Flush 得到，其它每一层的数据由上一层进行 Compaction 得到。
- Manifest Files：Manifest 文件中记录 SST 文件在不同 Level 的分布，单个 SST 文件的最大、最小 key，以及其他一些 LevelDB 需要的元信息。由于 LevelDB 支持 snapshot，需要维护多版本，因此可能同时存在多个 Manifest 文件。
- Current File：由于 Manifest 文件可能存在多个，Current 记录的是当前的 Manifest 文件名。
- Log Files (WAL)：用于防止 MemTable 丢数据的日志文件。

开源地址：

[https://github.com/google/leveldb](https://link.zhihu.com/?target=https%3A//github.com/google/leveldb)

### **4.5 Chromium**

Chromium是由Google主导开发的网页浏览器。以BSD许可证等多重自由版权发行并开放源代码，Chromium的开发可能早自2006年即开始. Chromium 是Google 的Chrome浏览器背后的引擎，其目的是为了创建一个安全、稳定和快速的通用浏览器.

整体架构：

![img](https://pic2.zhimg.com/80/v2-d511fbc9f53f0f4e587926a21b5c1665_720w.webp)

chromium的代码目录包含这些模块：

base：通用代码集和基础组件实现库，包含字符串、文件、线程、消息队列等工具类集合。

cc：负责渲染绘制，chrome为什么高效就是因为有它。chrome：浏览器界面模块，大量调用了cc提供的接口。

content：多进程沙盒浏览器莫款，管理多进程和多线程。

gpu，OpenGL封装实现：CommandBuffer和OpenGL的兼容支持模块。

net：网络功能实现模块。

media：多媒体封装代码，实现视频播放等功能。

mojo：跨语言（C++ / Java / JavaScript）跨平台的进程间对象通信模块，类似AIDL的功能。

skia：图形库。

third_party：排版引擎。

ui：UI库。

ipc: 网络进程通信模块。

v8，V8 JavaScript 引擎库。

以上每一个模块要想真正理解，都得花很大的功夫，简单用一张图来说明以上模块的关系：

![img](https://pic2.zhimg.com/80/v2-8106af5a2699ee24fcc09d7109f2c311_720w.webp)

开源地址：

[https://chromium.googlesource.com/chromium/src.git](https://link.zhihu.com/?target=https%3A//chromium.googlesource.com/chromium/src.git)

## 5.**Go经典开源项目**

Golang有哪些好像优秀的项目呢？列举一下我收集到的golang开发的优秀项目。

### **5.1 docker**

golang头号优秀项目，通过虚拟化技术实现的操作系统与应用的隔离，也称为容器。

**特点：**

- Docker是世界领先的软件容器平台。
- Docker使用Google公司推出的Go语言进行开发实现，基于Linux内核的cgroup，namespace，以及AUFS类的UnionFS等技术，对进程进行封装隔离，属于操作系统层面的虚拟化技术。由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。Docke最初实现是基于LXC。
- Docker能够自动执行重复性任务，例如搭建和配置开发环境，从而解放了开发人员以便他们专注在真正重要的事情上：构建杰出的软件。
- 用户可以方便地创建和使用容器，把自己的应用放入容器。容器还可以进行版本管理、复制、分享、修改，就像管理普通的代码一样。

整体架构：

![img](https://pic1.zhimg.com/80/v2-66c993a90b61237a774f4014f391ec38_720w.webp)

开源地址：

[https://github.com/docker](https://link.zhihu.com/?target=https%3A//github.com/docker)

### 5.2 kubernetes

**Kubernetes**（常简称为**K8s**）是用于自动部署、扩展和管理“容器化（containerized）应用程序”的开源系统。

**特点：**

- 跨主机编排容器
- 更充分地利用硬件资源来最大化地满足企业应用的需求
- 可移植 : 支持公有云,私有云,混合云,多重云
- 可扩展 : 模块化,插件化,可挂载,可组合,支持各种形式的扩展
- 自动化 : 自动部署,自动重启,自动复制,自动伸缩/扩展,通过声明式语法提供了

**整体架构:**

![img](https://pic1.zhimg.com/80/v2-93c0f887c54ffa08f1ddeb8f51a1d164_720w.webp)

- etcd保存了整个集群的状态；
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

**Kubernetes**设计理念和功能其实就是一个类似Linux的**分层架构**，如下图所示：

![img](https://pic3.zhimg.com/80/v2-321f4e84ed5f66b96d3ea39b8dacac4e_720w.webp)

- 核心层：Kubernetes最核心的功能，对外提供API构建高层的应用，对内提供插件式应用执行环境
- 应用层：部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS解析等）
- 管理层：系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、动态Provision等）以及策略管理（RBAC、Quota、PSP、NetworkPolicy等）
- 接口层：kubectl命令行工具、客户端SDK以及集群联邦
- 生态系统：在接口层之上的庞大容器集群管理调度的生态系统，可以划分为两个范畴Kubernetes外部：日志、监控、配置管理、CI、CD、Workflow、FaaS、OTS应用、ChatOps等Kubernetes内部：CRI、CNI、CVI、镜像仓库、Cloud Provider、集群自身的配置和管理等

开源地址：

[https://github.com/kubernetes/kubernetes](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/kubernetes)

### **5.3 etcd**

etcd 是 CoreOS 团队于 2013 年 6 月发起的开源项目，它的目标是构建一个高可用的分布式键值(key-value)数据库。

**特点：**

- 简单：定义明确、面向用户的 API (gRPC)
- 安全：具有可选客户端证书身份验证的自动 TLS
- 快速：基准测试为 10,000 次写入/秒
- 可靠：使用 Raft 正确分布
- etcd 是用 Go 编写的，使用Raft共识算法来管理高可用的复制日志。
- 许多公司在生产中使用 etcd ，在关键部署场景中，开发团队支持它，在这些场景中，etcd 经常与Kubernetes、locksmith、vulcand、Doorman等应用程序合作。严格的测试进一步确保了可靠性。

整体架构：

![img](https://pic2.zhimg.com/80/v2-81803900b67e866fc5e5885a0faffb41_720w.webp)

- httpserver
  etcd node之间进行通信，接收来自其他node的消息；

- raft
  实现分布式一致性raft协议, raft模块与server模块的通信采用了四个channel：

- - propc：处理client来的命令
  - recvc：处理http消息
  - readyc: 消息经过raft处理之后封装成Ready交给server处理
  - advanceC：server处理一条消息之后通知raft

- WAL
  server为了防止数据丢失而实现的write ahead log，与很多数据库的实现类似

- snapshotter防止wal的无限制增长，定期生成snap文件仅保留 term，index以及key value data；

- mvcc实现多版本的并发控制，使用revision（main和sub）来描述一个key的整个过程，从创建到删除。mvcc中还包含了watcher，用于实现监听key，prefix， range的变化。

- backend & boltdb持久化key value到boltdb数据库

- raftlograftlog模块包含unstable和raft的snapshot，unstable保存log entries，但是entries数量比较多的时候，就需要compact，创建一个snapshot，这里的snapshot还是保存在memory中的。raft模块会定时收集entries交给server处理。

开源地址：

[https://github.com/etcd-io/etcd](https://link.zhihu.com/?target=https%3A//github.com/etcd-io/etcd)

### **5.4 Tidb**

TiDB（“Ti”代表 Titanium）是一个开源的 NewSQL 数据库，支持混合事务和分析处理 (HTAP) 工作负载。它兼容 MySQL，具有水平可扩展性、强一致性和高可用性。

**特点：**

- 水平可扩展性TiDB 通过简单地添加新节点来扩展 SQL 处理和存储。这使得基础设施容量规划比仅垂直扩展的传统关系数据库更容易且更具成本效益。
- MySQL 兼容语法TiDB 就像是您的应用程序的 MySQL 5.7 服务器。您可以继续使用所有现有的 MySQL 客户端库，并且在许多情况下，您不需要更改应用程序中的任何一行代码。由于 TiDB 是从头开始构建的，而不是 MySQL 的 fork，请查看已知兼容性差异列表。
- 分布式事务TiDB 在内部将表分片成基于范围的小块，我们称之为“区域”。每个 Region 默认大小约为 100 MiB，TiDB 使用优化的两阶段提交来确保 Region 以事务一致的方式维护。
- 云原生TiDB 旨在在云中工作——公共、私有或混合——使部署、供应、操作和维护变得简单。TiDB 的存储层，称为 TiKV，是一个Cloud Native Computing Foundation (CNCF) 毕业项目。TiDB 平台的架构还允许 SQL 处理和存储以非常云友好的方式相互独立扩展。
- 最小化 ETLTiDB 旨在支持事务处理 (OLTP) 和分析处理 (OLAP) 工作负载。这意味着，虽然传统上您可能在 MySQL 上进行交易，然后将 (ETL) 数据提取、转换和加载到列存储中以进行分析处理，但不再需要此步骤。
- 高可用性TiDB 使用 Raft 共识算法来确保数据在 Raft 组中的整个存储中的高可用和安全复制。如果发生故障，Raft 组会自动为故障成员选举新的领导者，并在无需任何人工干预的情况下自愈 TiDB 集群。故障和自愈操作对应用程序也是透明的。

**整体架构：**

![img](https://pic4.zhimg.com/80/v2-79ad76aeaaab221a7ecec93b582da97b_720w.webp)

开源地址：

[https://github.com/pingcap/tidb](https://link.zhihu.com/?target=https%3A//github.com/pingcap/tidb)

### **5.5 Netpoll vs gnet**

**Netpoll**是字节跳动内部的 Golang 高性能、I/O 非阻塞的网络库，专注于 RPC 场景。

开源社区目前缺少专注于 RPC 方案的 Go 网络库。类似的项目如：evio、gnet 等，均面向 Redis、Haproxy 这样的场景。因此 Netpoll 应运而生，它借鉴了 evio 和 Netty 的优秀设计，具有出色的性能，更适用于微服务架构。

整体架构：

![img](https://pic3.zhimg.com/80/v2-3a16bb3fd275e33a5cba7e1001248df6_720w.webp)

- netpoll 将 Reactor 以 1:N 的形式组合成主从模式。
- MainReactor 主要管理 Listener，负责监听端口，建立新连接；
- SubReactor 负责管理 Connection，监听分配到的所有连接，并将所有触发的事件提交到协程池里进行处理。
- netpoll 在 I/O Task 中引入了主动的内存管理，向上层提供 NoCopy 的调用接口，由此支持 NoCopy RPC。
- 使用协程池集中处理 I/O Task，减少 goroutine 数量和调度开销。

开源地址：

[https://github.com/cloudwego/netpoll](https://link.zhihu.com/?target=https%3A//github.com/cloudwego/netpoll)

### **5.6 gnet**

![img](https://pic4.zhimg.com/80/v2-0ae62978df369cdf9d0cadd4a0ce29bf_720w.webp)

gnet的卖点在于它是一个高性能、轻量级、非阻塞的纯 Go 实现的传输层（TCP/UDP/Unix Domain Socket）网络框架，开发者可以使用 gnet 来实现自己的应用层网络协议(HTTP、RPC、Redis、WebSocket 等等)，从而构建出自己的应用层网络应用：比如在 gnet 上实现 HTTP 协议就可以创建出一个 HTTP 服务器 或者 Web 开发框架，实现 Redis 协议就可以创建出自己的 Redis 服务器等等。

gnet，在某些极端的网络业务场景，比如海量连接、高频短连接、网络小包等等场景，gnet 在性能和资源占用上都远超 Go 原生的 net 包（基于 netpoller）。

![img](https://pic1.zhimg.com/80/v2-6c1ce8f5f1a3044ff36ca14d62942d60_720w.webp)

主从 Reactors + Goroutine Pool 模型

**功能**

- [x] 高性能的基于多线程/Go程网络模型的 event-loop 事件驱动
- [x] 内置 goroutine 池，由开源库 ants 提供支持
- [x] 内置 bytes 内存池，由开源库 bytebufferpool 提供支持
- [x] 整个生命周期是无锁的
- [x] 简单易用的 APIs
- [x] 基于 Ring-Buffer 的高效且可重用的内存 buffer
- [x] 支持多种网络协议/IPC 机制：TCP、UDP 和 Unix Domain Socket
- [x] 支持多种负载均衡算法：Round-Robin(轮询)、Source-Addr-Hash(源地址哈希) 和 Least-Connections(最少连接数)
- [x] 支持两种事件驱动机制：**「Linux」** 里的 epoll 以及 **「FreeBSD/DragonFly/Darwin」** 里的 kqueue
- [x] 支持异步写操作
- [x] 灵活的事件定时器
- [x] SO_REUSEPORT 端口重用
- [x] 内置多种编解码器，支持对 TCP 数据流分包：LineBasedFrameCodec, DelimiterBasedFrameCodec,FixedLengthFrameCodec和LengthFieldBasedFrameCodec，参考自 netty codec，而且支持自定制编解码器
- [x] 支持 Windows 平台，Go 标准网络库
- [ ] 实现 gnet 客户端

Github:

[https://github.com/panjf2000/gnet](https://link.zhihu.com/?target=https%3A//github.com/panjf2000/gnet)

## **6.Java经典开源项目**

这里推荐一些最值得阅读优秀的Java开源项目。

### 6.1 Netty

**Netty**是一个Java NIO技术的开源异步事件驱动的网络编程框架，用于快速开发可维护的高性能协议服务器和客户端。

![img](https://pic4.zhimg.com/80/v2-0cd55546f651d87c4ad757515225a1ff_720w.webp)

往通俗了讲，可以将Netty理解为：一个将Java NIO进行了大量封装，并大大降低Java NIO使用难度和上手门槛的超牛逼框架。

**特点：**

**设计**

- 各种传输类型的统一 API - 阻塞和非阻塞套接字
- 基于灵活和可扩展的事件模型，允许清晰的关注点分离
- 高度可定制的线程模型——单线程、一个或多个线程池，如 SEDA
- 真正的无连接数据报套接字支持（自 3.1 起）

**便于使用**

- 有据可查的 Javadoc、用户指南和示例

- 没有额外的依赖，JDK 5 (Netty 3.x) 或 6 (Netty 4.x) 就足够了

- - 注意：某些组件（例如 HTTP/2）可能有更多要求。 有关更多信息，请参阅 要求页面。

**表现**

- 更高的吞吐量，更低的延迟
- 更少的资源消耗
- 最小化不必要的内存复制

**安全**

- 完整的 SSL/TLS 和 StartTLS 支

开源地址：

[https://github.com/netty/netty](https://link.zhihu.com/?target=https%3A//github.com/netty/netty)

### **6.2 J2EE框架 Spring**

star:45.1k; fork:31.8k

![img](https://pic2.zhimg.com/80/v2-2dede4cf851558edd585e1c7d4354251_720w.webp)

**Spring Framework** 是一个开源的Java/Java EE全功能栈（full-stack）的应用程序框架，以Apache许可证形式发布，也有.NET平台上的移植版本。该框架基于 Expert One-on-One Java EE Design and Development（ISBN 0-7645-4385-7）一书中的代码，最初由 Rod Johnson 和 Juergen Hoeller等开发。Spring Framework 提供了一个简易的开发方式，这种开发方式，将避免那些可能致使底层代码变得繁杂混乱的大量的属性文件和帮助类。

**Spring 中包含的关键特性：**

- 强大的基于 JavaBeans 的采用控制翻转（Inversion of Control，IoC）原则的配置管理，使得应用程序的组建更加快捷简易。
- 一个可用于从 applet 到 Java EE 等不同运行环境的核心 Bean 工厂。
- 数据库事务的一般化抽象层，允许宣告式(Declarative)事务管理器，简化事务的划分使之与底层无关。
- 内建的针对 JTA 和 单个 JDBC 数据源的一般化策略，使 Spring 的事务支持不要求 Java EE 环境，这与一般的 JTA 或者 EJB CMT 相反。
- JDBC 抽象层提供了有针对性的异常等级(不再从SQL异常中提取原始代码), 简化了错误处理, 大大减少了程序员的编码量. 再次利用JDBC时，你无需再写出另一个 '终止' (finally) 模块. 并且面向JDBC的异常与Spring 通用数据访问对象 (Data Access Object) 异常等级相一致.
- 以资源容器，DAO 实现和事务策略等形式与 Hibernate，JDO 和 iBATIS SQL Maps 集成。利用众多的翻转控制方便特性来全面支持, 解决了许多典型的Hibernate集成问题. 所有这些全部遵从Spring通用事务处理和通用数据访问对象异常等级规范.
- 灵活的基于核心 Spring 功能的 MVC 网页应用程序框架。开发者通过策略接口将拥有对该框架的高度控制，因而该框架将适应于多种呈现(View)技术，例如 JSP，FreeMarker，Velocity，Tiles，iText 以及 POI。值得注意的是，Spring 中间层可以轻易地结合于任何基于 MVC 框架的网页层，例如 Struts，WebWork，或 Tapestry。
- 提供诸如事务管理等服务的面向方面编程框架。

开源地址:

[https://github.com/spring-projects/spring-framework](https://link.zhihu.com/?target=https%3A//github.com/spring-projects/spring-framework)

### 6.3 Android 开源框架 EventBus Android

star:23.1k; fork:4.6k

![img](https://pic2.zhimg.com/80/v2-c2e2778ddcad9f8256594f73f760ddf5_720w.webp)

如果你学习过设计模式，那么当想通知其他组件某些事情发生时你一定会使用观察者模式。好了，既然能想到这个设计模式，那么就来看一个屌爆天的Android开源框架EventBus。主要功能是替代Intent、Handler、BroadCast在Fragment、Activity、Service、线程之间传递消息。他的最牛逼优点是开销小，代码简洁，解耦代码。

**特点：**

- 简化组件之间的通信
- 分离事件发送者和接收者
- 在 UI 工件（例如活动、片段）和后台线程中表现良好
- 避免复杂且容易出错的依赖关系和生命周期问题
- 很快；专为高性能而优化
- 很小（~60k jar）
- 是在实践中被证明通过应用与1,000,000,000+安装
- 具有交付线程、订阅者优先级等高级功能。

开源地址：

[https://github.com/greenrobot/EventBus](https://link.zhihu.com/?target=https%3A//github.com/greenrobot/EventBus)

### **6.4 Java 设计模式 java-design-patterns**

star:71.4k;fork:22.2k

![img](https://pic2.zhimg.com/80/v2-719c53087e9405a05c9544f951261d81_720w.webp)

设计模式是程序员在设计应用程序或系统时解决常见问题的最佳实践，重用设计模式有助于防止可能导致重大问题的细微问题，同时熟悉模式的程序员和架构师的代码也更具可读性。

开源地址：

[https://github.com/iluwatar/java-design-pattern](https://link.zhihu.com/?target=https%3A//github.com/iluwatar/java-design-pattern)