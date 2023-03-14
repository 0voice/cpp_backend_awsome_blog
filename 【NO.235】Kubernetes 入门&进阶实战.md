# 【NO.235】Kubernetes 入门&进阶实战

### **0.写在前面**

笔者今年 9 月从端侧开发转到后台开发，第一个系统开发任务就强依赖了 K8S，加之项目任务重、排期紧，必须马上对 K8S 有概念上的了解。然而，很多所谓“K8S 入门\概念”的文章看的一头雾水，对于大部分新手来说并不友好。经历了几天痛苦地学习之后，回顾来看，K8S 根本不复杂。于是，决心有了这一系列的文章：一方面希望对新手同学有帮助；另一方面，以文会友，希望能够有机会交流讨论技术。

本文组织方式：

\1. K8S 是什么，即作用和目的。涉及 K8S 架构的整理，Master 和 Node 之间的关系，以及 K8S 几个重要的组件：API Server、Scheduler、Controller、etcd 等。
\2. K8S 的重要概念，即 K8S 的 API 对象，也就是常常听到的 Pod、Deployment、Service 等。
\3. 如何配置 kubectl，介绍kubectl工具和配置办法。
\4. 如何用kubectl 部署服务。
\5. 如何用kubectl 查看、更新/编辑、删除服务。
\6. 如何用kubectl 排查部署在K8S集群上的服务出现的问题

### **1.K8S 概览**

##### **1.1 K8S 是什么？**

K8S 是[Kubernetes](https://kubernetes.io/zh/docs/concepts/overview/what-is-kubernetes/)的全称，官方称其是：

> Kubernetes is an open source system for managing [containerized applications](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) across multiple hosts. It provides basic mechanisms for deployment, maintenance, and scaling of applications.
>
> 用于自动部署、扩展和管理“容器化（containerized）应用程序”的开源系统。

翻译成大白话就是：“**K8S 是 负责自动化运维管理多个 Docker 程序的集群**”。那么问题来了：Docker 运行可方便了，为什么要用 K8S，它有什么优势？

插一句题外话：

- 为什么 Kubernetes 要叫 Kubernetes 呢？[维基百科](https://zh.wikipedia.org/wiki/Kubernetes#cite_note-3)已经交代了（老美对星际是真的痴迷）：

  > Kubernetes（在希腊语意为“舵手”或“驾驶员”）由 Joe Beda、Brendan Burns 和 Craig McLuckie 创立，并由其他谷歌工程师，包括 Brian Grant 和 Tim Hockin 等进行加盟创作，并由谷歌在 2014 年首次对外宣布 。该系统的开发和设计都深受谷歌的 Borg 系统的影响，其许多顶级贡献者之前也是 Borg 系统的开发者。在谷歌内部，Kubernetes 的原始代号曾经是[Seven](https://zh.wikipedia.org/wiki/九之七)，即[星际迷航](https://zh.wikipedia.org/wiki/星际迷航)中的 Borg（[博格人](https://zh.wikipedia.org/wiki/博格_(星际旅行))）。Kubernetes 标识中舵轮有七个轮辐就是对该项目代号的致意。

- 为什么 Kubernetes 的缩写是 K8S 呢？我个人赞同[Why Kubernetes is Abbreviated k8s](https://medium.com/@rothgar/why-kubernetes-is-abbreviated-k8s-905289405a3c#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6ImQ5NDZiMTM3NzM3Yjk3MzczOGU1Mjg2YzIwOGI2NmU3YTM5ZWU3YzEiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJuYmYiOjE2MDUxNjk3MjAsImF1ZCI6IjIxNjI5NjAzNTgzNC1rMWs2cWUwNjBzMnRwMmEyamFtNGxqZGNtczAwc3R0Zy5hcHBzLmdvb2dsZXVzZXJjb250ZW50LmNvbSIsInN1YiI6IjExMDc5MTA1ODc0OTMzMDE5NDUwOCIsImVtYWlsIjoibWFvamlhbmd5dW45OTk5QGdtYWlsLmNvbSIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJhenAiOiIyMTYyOTYwMzU4MzQtazFrNnFlMDYwczJ0cDJhMmphbTRsamRjbXMwMHN0dGcuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJuYW1lIjoiSmlhbmd5dW4gTWFvIiwicGljdHVyZSI6Imh0dHBzOi8vbGgzLmdvb2dsZXVzZXJjb250ZW50LmNvbS9hLS9BT2gxNEdpclJJYnVrcEltb0dHemt4Q01xbzJsMWFtUjdlX1pSSTdEZVZjWD1zOTYtYyIsImdpdmVuX25hbWUiOiJKaWFuZ3l1biIsImZhbWlseV9uYW1lIjoiTWFvIiwiaWF0IjoxNjA1MTcwMDIwLCJleHAiOjE2MDUxNzM2MjAsImp0aSI6IjYwYjc1ZjczYjkwNzBlZDYwODY2MzFiN2RmZjY2ZGQ1YjE0YzNlZGYifQ.Z2jxeJpyVs_hKdXirBUaM1B_llDVFmWX3M4Yb--VM2wpd0WwTXQ_48g88ShWsAqGuoVP0nOlTXFktg2DZKn5wj7H7W_URgE5nxxiXOBZAqAxpoiPN-_Uup73PaATVvDHg-dKuWWRIZQ21E8nyhSnFAQA2tHQenTIh6UpQBMPUpcI7v6M-c_b1X8n4_EB0KEPOeFeJb3Yz8xFpm9hqb0D6B6L8ovZBFa6d576S56D6f_9RdJS67vDDf4wOjqr1aIxSEgOTV_m-nJJgdCZEr3OgGLuTXm86mh9jg1d8PdMbcxoRjG9LVeQz68-lUxnxN798zYavZjnLsmtV9QYeM0Nfw)中说的观点“嘛，写全称也太累了吧，不如整个缩写”。其实只保留首位字符，用具体数字来替代省略的字符个数的做法，还是比较常见的。

##### **1.2 为什么是 K8S?**

试想下传统的后端部署办法：把程序包（包括可执行二进制文件、配置文件等）放到服务器上，接着运行启动脚本把程序跑起来，同时启动守护脚本定期检查程序运行状态、必要的话重新拉起程序。

有问题吗？显然有！最大的一个问题在于：**如果服务的请求量上来，已部署的服务响应不过来怎么办？**传统的做法往往是，如果请求量、内存、CPU 超过阈值做了告警，运维马上再加几台服务器，部署好服务之后，接入负载均衡来分担已有服务的压力。

问题出现了：从监控告警到部署服务，中间需要人力介入！那么，**有没有办法自动完成服务的部署、更新、卸载和扩容、缩容呢？**

**这，就是 K8S 要做的事情：自动化运维管理 Docker（容器化）程序**。

##### **1.3 K8S 怎么做？**

我们已经知道了 K8S 的核心功能：自动化运维管理多个容器化程序。那么 K8S 怎么做到的呢？这里，我们从宏观架构上来学习 K8S 的设计思想。首先看下图，图片来自文章[Components of Kubernetes Architecture](https://medium.com/@kumargaurav1247/components-of-kubernetes-architecture-6feea4d5c712)：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwwXk0rxy9CzHyow9mHPQiaNm0VB6R8R1Amz9q18cQt9KcDicVNQyBjiaVA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

K8S 是属于**主从设备模型（Master-Slave 架构）**，即有 Master 节点负责核心的调度、管理和运维，Slave 节点则在执行用户的程序。但是在 K8S 中，主节点一般被称为**Master Node 或者 Head Node**（本文采用 Master Node 称呼方式），而从节点则被称为**Worker Node 或者 Node**（本文采用 Worker Node 称呼方式）。

要注意一点：Master Node 和 Worker Node 是分别安装了 K8S 的 Master 和 Woker 组件的实体服务器，每个 Node 都对应了一台实体服务器（虽然 Master Node 可以和其中一个 Worker Node 安装在同一台服务器，但是建议 Master Node 单独部署），**所有 Master Node 和 Worker Node 组成了 K8S 集群**，同一个集群可能存在多个 Master Node 和 Worker Node。

首先来看**Master Node**都有哪些组件：

- **API Server**。**K8S 的请求入口服务**。API Server 负责接收 K8S 所有请求（来自 UI 界面或者 CLI 命令行工具），然后，API Server 根据用户的具体请求，去通知其他组件干活。
- **Scheduler**。**K8S 所有 Worker Node 的调度器**。当用户要部署服务时，Scheduler 会选择最合适的 Worker Node（服务器）来部署。
- **Controller Manager**。**K8S 所有 Worker Node 的监控器**。Controller Manager 有很多具体的 Controller，在文章[Components of Kubernetes Architecture](https://medium.com/@kumargaurav1247/components-of-kubernetes-architecture-6feea4d5c712)中提到的有 Node Controller、Service Controller、Volume Controller 等。Controller 负责监控和调整在 Worker Node 上部署的服务的状态，比如用户要求 A 服务部署 2 个副本，那么当其中一个服务挂了的时候，Controller 会马上调整，让 Scheduler 再选择一个 Worker Node 重新部署服务。
- **etcd**。**K8S 的存储服务**。etcd 存储了 K8S 的关键配置和用户配置，K8S 中仅 API Server 才具备读写权限，其他组件必须通过 API Server 的接口才能读写数据（见[Kubernetes Works Like an Operating System](https://thenewstack.io/how-does-kubernetes-work/)）。

接着来看**Worker Node**的组件，笔者更赞同[HOW DO APPLICATIONS RUN ON KUBERNETES](https://thenewstack.io/how-do-applications-run-on-kubernetes/)文章中提到的组件介绍：

- **Kubelet**。**Worker Node 的监视器，以及与 Master Node 的通讯器**。Kubelet 是 Master Node 安插在 Worker Node 上的“眼线”，它会定期向 Worker Node 汇报自己 Node 上运行的服务的状态，并接受来自 Master Node 的指示采取调整措施。
- **Kube-Proxy**。**K8S 的网络代理**。私以为称呼为 Network-Proxy 可能更适合？Kube-Proxy 负责 Node 在 K8S 的网络通讯、以及对外部网络流量的负载均衡。
- **Container Runtime**。**Worker Node 的运行环境**。即安装了容器化所需的软件环境确保容器化程序能够跑起来，比如 Docker Engine。大白话就是帮忙装好了 Docker 运行环境。
- **Logging Layer**。**K8S 的监控状态收集器**。私以为称呼为 Monitor 可能更合适？Logging Layer 负责采集 Node 上所有服务的 CPU、内存、磁盘、网络等监控项信息。
- **Add-Ons**。**K8S 管理运维 Worker Node 的插件组件**。有些文章认为 Worker Node 只有三大组件，不包含 Add-On，但笔者认为 K8S 系统提供了 Add-On 机制，让用户可以扩展更多定制化功能，是很不错的亮点。

总结来看，**K8S 的 Master Node 具备：请求入口管理（API Server），Worker Node 调度（Scheduler），监控和自动调节（Controller Manager），以及存储功能（etcd）；而 K8S 的 Worker Node 具备：状态和监控收集（Kubelet），网络和负载均衡（Kube-Proxy）、保障容器化运行环境（Container Runtime）、以及定制化功能（Add-Ons）。**

到这里，相信你已经对 K8S 究竟是做什么的，有了大概认识。接下来，再来认识下 K8S 的 Deployment、Pod、Replica Set、Service 等，但凡谈到 K8S，就绕不开这些名词，而这些名词也是最让 K8S 新手们感到头疼、困惑的。

### **2.K8S 重要概念**

##### **2.1 Pod 实例**

官方对于**Pod**的解释是：

> **Pod**是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。

这样的解释还是很难让人明白究竟 Pod 是什么，但是对于 K8S 而言，Pod 可以说是所有对象中最重要的概念了！因此，我们**必须首先清楚地知道“Pod 是什么”**，再去了解其他的对象。

从官方给出的定义，联想下“最小的 xxx 单元”，是不是可以想到本科在学校里学习“进程”的时候，教科书上有一段类似的描述：资源分配的最小单位；还有”线程“的描述是：CPU 调度的最小单位。什么意思呢？**”最小 xx 单位“要么就是事物的衡量标准单位，要么就是资源的闭包、集合**。前者比如长度米、时间秒；后者比如一个”进程“是存储和计算的闭包，一个”线程“是 CPU 资源（包括寄存器、ALU 等）的闭包。

同样的，**Pod 就是 K8S 中一个服务的闭包**。这么说的好像还是有点玄乎，更加云里雾里了。简单来说，**Pod 可以被理解成一群可以共享网络、存储和计算资源的容器化服务的集合**。再打个形象的比喻，在同一个 Pod 里的几个 Docker 服务/程序，好像被部署在同一台机器上，可以通过 localhost 互相访问，并且可以共用 Pod 里的存储资源（这里是指 Docker 可以挂载 Pod 内的数据卷，数据卷的概念，后文会详细讲述，暂时理解为“需要手动 mount 的磁盘”）。笔者总结 Pod 如下图，可以看到：**同一个 Pod 之间的 Container 可以通过 localhost 互相访问，并且可以挂载 Pod 内所有的数据卷；但是不同的 Pod 之间的 Container 不能用 localhost 访问，也不能挂载其他 Pod 的数据卷**。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xw8tMnpYpyNAQn3o2FicHmuayH2do04NuzfQlvsrP66EyjE8O8omthHBA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

对 Pod 有直观的认识之后，接着来看 K8S 中 Pod 究竟长什么样子，具体包括哪些资源？

K8S 中所有的对象都通过 yaml 来表示，笔者从[官方网站](https://kubernetes.io/zh/docs/tasks/configure-pod-container/assign-memory-resource/)摘录了一个最简单的 Pod 的 yaml：

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

看不懂不必慌张，且耐心听下面的解释：

- `apiVersion`记录 K8S 的 API Server 版本，现在看到的都是`v1`，用户不用管。

- `kind`记录该 yaml 的对象，比如这是一份 Pod 的 yaml 配置文件，那么值内容就是`Pod`。

- `metadata`记录了 Pod 自身的元数据，比如这个 Pod 的名字、这个 Pod 属于哪个 namespace（命名空间的概念，后文会详述，暂时理解为“同一个命名空间内的对象互相可见”）。

- `spec`记录了 Pod 内部所有的资源的详细信息，看懂这个很重要：

- - `containers`记录了 Pod 内的容器信息，`containers`包括了：`name`容器名，`image`容器的镜像地址，`resources`容器需要的 CPU、内存、GPU 等资源，`command`容器的入口命令，`args`容器的入口参数，`volumeMounts`容器要挂载的 Pod 数据卷等。可以看到，**上述这些信息都是启动容器的必要和必需的信息**。
  - `volumes`记录了 Pod 内的数据卷信息，后文会详细介绍 Pod 的数据卷。

**2.2 Volume 数据卷**

K8S 支持很多类型的 volume 数据卷挂载，具体请参见[K8S 卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/)。前文就“如何理解 volume”提到：“**需要手动 mount 的磁盘**”，此外，有一点可以帮助理解：**数据卷 volume 是 Pod 内部的磁盘资源**。

其实，单单就 Volume 来说，不难理解。但是上面还看到了`volumeMounts`，这俩是什么关系呢？

**volume 是 K8S 的对象，对应一个实体的数据卷；而 volumeMounts 只是 container 的挂载点，对应 container 的其中一个参数**。但是，**volumeMounts 依赖于 volume**，只有当 Pod 内有 volume 资源的时候，该 Pod 内部的 container 才可能有 volumeMounts。

##### **2.3 Container 容器**

本文中提到的镜像 Image、容器 Container，都指代了 Pod 下的一个`container`。关于 K8S 中的容器，在 2.1Pod 章节都已经交代了，这里无非再啰嗦一句：**一个 Pod 内可以有多个容器 container**。

在 Pod 中，容器也有分类，对这个感兴趣的同学欢迎自行阅读更多资料：

- **标准容器 Application Container**。
- **初始化容器 Init Container**。
- **边车容器 Sidecar Container**。
- **临时容器 Ephemeral Container**。

一般来说，我们部署的大多是**标准容器（ Application Container）**。

##### **2.4 Deployment 和 ReplicaSet（简称 RS）**

除了 Pod 之外，K8S 中最常听到的另一个对象就是 Deployment 了。那么，什么是 Deployment 呢？官方给出了一个要命的解释：

> 一个 *Deployment* 控制器为 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 和 [ReplicaSets](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicaset/) 提供声明式的更新能力。
>
> 你负责描述 Deployment 中的 *目标状态*，而 Deployment 控制器以受控速率更改实际状态， 使其变为期望状态。你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment，并通过新的 Deployment 收养其资源。

翻译一下：**Deployment 的作用是管理和控制 Pod 和 ReplicaSet，管控它们运行在用户期望的状态中**。哎，打个形象的比喻，**Deployment 就是包工头**，主要负责监督底下的工人 Pod 干活，确保每时每刻有用户要求数量的 Pod 在工作。如果一旦发现某个工人 Pod 不行了，就赶紧新拉一个 Pod 过来替换它。

新的问题又来了：那什么是 ReplicaSets 呢？

> ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

再来翻译下：ReplicaSet 的作用就是管理和控制 Pod，管控他们好好干活。但是，ReplicaSet 受控于 Deployment。形象来说，**ReplicaSet 就是总包工头手下的小包工头**。

笔者总结得到下面这幅图，希望能帮助理解：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xw4ickITAbMdSSn2JYyhlT8n7rf3TOMZ3Th1vdniab4p4nwTGJogVytPbA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

新的问题又来了：**如果都是为了管控 Pod 好好干活，为什么要设置 Deployment 和 ReplicaSet 两个层级呢，直接让 Deployment 来管理不可以吗？**

回答：不清楚，但是私以为是因为先有 ReplicaSet，但是使用中发现 ReplicaSet 不够满足要求，于是又整了一个 Deployment（**有清楚 Deployment 和 ReplicaSet 联系和区别的小伙伴欢迎留言啊**）。

但是，从 K8S 使用者角度来看，用户会直接操作 Deployment 部署服务，而当 Deployment 被部署的时候，K8S 会自动生成要求的 ReplicaSet 和 Pod。在[K8S 官方文档](https://www.kubernetes.org.cn/replicasets)中也指出用户只需要关心 Deployment 而不操心 ReplicaSet：

> This actually means that you may never need to manipulate ReplicaSet objects: use a Deployment instead, and define your application in the spec section.
>
> 这实际上意味着您可能永远不需要操作 ReplicaSet 对象：直接使用 Deployments 并在规范部分定义应用程序。

补充说明：在 K8S 中还有一个对象 --- **ReplicationController（简称 RC）**，[官方文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/replicationcontroller/)对它的定义是：

> *ReplicationController* 确保在任何时候都有特定数量的 Pod 副本处于运行状态。换句话说，ReplicationController 确保一个 Pod 或一组同类的 Pod 总是可用的。

怎么样，和 ReplicaSet 是不是很相近？在[Deployments, ReplicaSets, and pods](https://www.ibm.com/cloud/architecture/content/course/kubernetes-101/deployments-replica-sets-and-pods/)教程中说“ReplicationController 是 ReplicaSet 的前身”，官方也推荐用 Deployment 取代 ReplicationController 来部署服务。

##### **2.5 Service 和 Ingress**

吐槽下 K8S 的概念/对象/资源是真的多啊！**前文介绍的 Deployment、ReplicationController 和 ReplicaSet 主要管控 Pod 程序服务；那么，Service 和 Ingress 则负责管控 Pod 网络服务**。

我们先来看看[官方文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/)中 Service 的定义：

> 将运行在一组 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 上的应用程序公开为网络服务的抽象方法。
>
> 使用 Kubernetes，您无需修改应用程序即可使用不熟悉的服务发现机制。Kubernetes 为 Pods 提供自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名， 并且可以在它们之间进行负载均衡。

翻译下：K8S 中的服务（Service）并不是我们常说的“服务”的含义，而更像是网关层，是若干个 Pod 的流量入口、流量均衡器。

那么，**为什么要 Service 呢**？

私以为在这一点上，[官方文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/)讲解地非常清楚：

> Kubernetes [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 是有生命周期的。它们可以被创建，而且销毁之后不会再启动。如果您使用 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/) 来运行您的应用程序，则它可以动态创建和销毁 Pod。
>
> 每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。
>
> 这导致了一个问题：如果一组 Pod（称为“后端”）为群集内的其他 Pod（称为“前端”）提供功能， 那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用工作量的后端部分？

补充说明：K8S 集群的网络管理和拓扑也有特别的设计，以后会专门出一章节来详细介绍 K8S 中的网络。这里需要清楚一点：K8S 集群内的每一个 Pod 都有自己的 IP（是不是很类似一个 Pod 就是一台服务器，然而事实上是多个 Pod 存在于一台服务器上，只不过是 K8S 做了网络隔离），在 K8S 集群内部还有 DNS 等网络服务（一个 K8S 集群就如同管理了多区域的服务器，可以做复杂的网络拓扑）。

此外，笔者推荐[k8s 外网如何访问业务应用](https://www.jianshu.com/p/50b930fa7ca3)对于 Service 的介绍，不过对于新手而言，推荐阅读前半部分对于 service 的介绍即可，后半部分就太复杂了。我这里做了简单的总结：

**Service 是 K8S 服务的核心，屏蔽了服务细节，统一对外暴露服务接口，真正做到了“微服务”**。举个例子，我们的一个服务 A，部署了 3 个备份，也就是 3 个 Pod；对于用户来说，只需要关注一个 Service 的入口就可以，而不需要操心究竟应该请求哪一个 Pod。优势非常明显：**一方面外部用户不需要感知因为 Pod 上服务的意外崩溃、K8S 重新拉起 Pod 而造成的 IP 变更，外部用户也不需要感知因升级、变更服务带来的 Pod 替换而造成的 IP 变化，另一方面，Service 还可以做流量负载均衡**。

但是，Service 主要负责 K8S 集群内部的网络拓扑。那么集群外部怎么访问集群内部呢？这个时候就需要 Ingress 了，[官方文档](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)中的解释是：

> Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。
>
> Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。

翻译一下：Ingress 是整个 K8S 集群的接入层，复杂集群内外通讯。

最后，笔者把 Ingress 和 Service 的关系绘制网络拓扑关系图如下，希望对理解这两个概念有所帮助：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwaaRkshAhKGmeJuRwZNbz7aWQh5f9ftA33iaBarA62rfKQ34VKNpuxzg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### **2.6 namespace 命名空间**

和前文介绍的所有的概念都不一样，namespace 跟 Pod 没有直接关系，而是 K8S 另一个维度的对象。或者说，前文提到的概念都是为了服务 Pod 的，而 namespace 则是为了服务整个 K8S 集群的。

那么，namespace 是什么呢？

上[官方文档](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)定义：

> Kubernetes 支持多个虚拟集群，它们底层依赖于同一个物理集群。这些虚拟集群被称为名字空间。

翻译一下：**namespace 是为了把一个 K8S 集群划分为若干个资源不可共享的虚拟集群而诞生的**。

也就是说，**可以通过在 K8S 集群内创建 namespace 来分隔资源和对象**。比如我有 2 个业务 A 和 B，那么我可以创建 ns-a 和 ns-b 分别部署业务 A 和 B 的服务，如在 ns-a 中部署了一个 deployment，名字是 hello，返回用户的是“hello a”；在 ns-b 中也部署了一个 deployment，名字恰巧也是 hello，返回用户的是“hello b”（要知道，在同一个 namespace 下 deployment 不能同名；但是不同 namespace 之间没有影响）。前文提到的所有对象，都是在 namespace 下的；当然，也有一些对象是不隶属于 namespace 的，而是在 K8S 集群内全局可见的，官方文档提到的可以通过命令来查看，具体命令的使用办法，笔者会出后续的实战文章来介绍，先贴下命令：

```
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

不在 namespace 下的对象有：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xw40s4vIe4cxeFjx9J7WVT92MCuo48ImLUic2HW3SsgbOmDyH2EcFvibrQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在 namespace 下的对象有（部分）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xw7WujZuJjhdNqYK2DCCmC9Y2vutHqyYibzKT6j7530hKn1fcvZhTEoGQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### **2.7 其他**

K8S 的对象实在太多了，2.1-2.6 介绍的是在实际使用 K8S 部署服务最常见的。其他的还有 Job、CronJob 等等，在对 K8S 有了比较清楚的认知之后，再去学习更多的 K8S 对象，不是难事。

### **3. 配置 kubectl**

##### **3.1 什么是 kubectl？**

[官方文档](https://kubernetes.io/zh/docs/reference/kubectl/overview/)中介绍 kubectl 是：

> Kubectl 是一个命令行接口，用于对 Kubernetes 集群运行命令。Kubectl 的配置文件在$HOME/.kube 目录。我们可以通过设置 KUBECONFIG 环境变量或设置命令参数--kubeconfig 来指定其他位置的 kubeconfig 文件。

也就是说，可以通过 kubectl 来操作 K8S 集群，基本语法：

> 使用以下语法 `kubectl` 从终端窗口运行命令：
>
> ```
> kubectl [command] [TYPE] [NAME] [flags]
> ```
>
> 其中 `command`、`TYPE`、`NAME` 和 `flags` 分别是：
>
> - `command`：指定要对一个或多个资源执行的操作，例如 `create`、`get`、`describe`、`delete`。
> - `TYPE`：指定[资源类型](https://kubernetes.io/zh/docs/reference/kubectl/overview/#resource-types)。资源类型不区分大小写，可以指定单数、复数或缩写形式。例如，以下命令输出相同的结果:
>
> ```
> ```shell
> kubectl get pod pod1
> kubectl get pods pod1
> kubectl get po pod1
> - `NAME`：指定资源的名称。名称区分大小写。如果省略名称，则显示所有资源的详细信息 `kubectl get pods`。
> 
> 在对多个资源执行操作时，您可以按类型和名称指定每个资源，或指定一个或多个文件：
> 
> - 要按类型和名称指定资源：
>   - 要对所有类型相同的资源进行分组，请执行以下操作：`TYPE1 name1 name2 name<#>`。
>  例子：`kubectl get pod example-pod1 example-pod2`
>   - 分别指定多个资源类型：`TYPE1/name1 TYPE1/name2 TYPE2/name3 TYPE<#>/name<#>`。
>  例子：`kubectl get pod/example-pod1 replicationcontroller/example-rc1`
> - 用一个或多个文件指定资源：`-f file1 -f file2 -f file<#>`
>   - [使用 YAML 而不是 JSON](https://kubernetes.io/zh/docs/concepts/configuration/overview/#general-config-tips) 因为 YAML 更容易使用，特别是用于配置文件时。
>  例子：`kubectl get -f ./pod.yaml`
> - `flags`: 指定可选的参数。例如，可以使用 `-s` 或 `-server` 参数指定 Kubernetes API 服务器的地址和端口。
> ```

就如何使用 kubectl 而言，官方文档已经说得非常清楚。不过对于新手而言，还是需要解释几句：

1. kubectl 是 K8S 的命令行工具，并不需要 kubectl 安装在 K8S 集群的任何 Node 上，但是，需要确保安装 kubectl 的机器和 K8S 的集群能够进行网络互通。
2. kubectl 是通过本地的配置文件来连接到 K8S 集群的，默认保存在$HOME/.kube 目录下；也可以通过 KUBECONFIG 环境变量或设置命令参数--kubeconfig 来指定其他位置的 kubeconfig 文件【官方文档】。

接下来，一起看看怎么使用 kubectl 吧，切身感受下 kubectl 的使用。

请注意，如何安装 kubectl 的办法有许多非常明确的教程，比如《[安装并配置 kubectl](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/)》，本文不再赘述。

##### **3.2 怎么配置 kubectl？**

**第一步，必须准备好要连接/使用的 K8S 的配置文件**，笔者给出一份杜撰的配置：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: thisisfakecertifcateauthoritydata00000000000
    server: https://1.2.3.4:1234
  name: cls-dev
contexts:
- context:
    cluster: cls-dev
    user: kubernetes-admin
  name: kubernetes-admin@test
current-context: kubernetes-admin@test
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    token: thisisfaketoken00000
```

解读如下：

- `clusters`记录了 clusters（一个或多个 K8S 集群）信息：

- - `name`是这个 cluster（K8S 集群）的名称代号
  - `server`是这个 cluster（K8S 集群）的访问方式，一般为 IP+PORT
  - `certificate-authority-data`是证书数据，只有当 cluster（K8S 集群）的连接方式是 https 时，为了安全起见需要证书数据

- `users`记录了访问 cluster（K8S 集群）的账号信息：

- - `name`是用户账号的名称代号
  - `user/token`是用户的 token 认证方式，token 不是用户认证的唯一方式，其他还有账号+密码等。

- `contexts`是上下文信息，包括了 cluster（K8S 集群）和访问 cluster（K8S 集群）的用户账号等信息：

- - `name`是这个上下文的名称代号
  - `cluster`是 cluster（K8S 集群）的名称代号
  - `user`是访问 cluster（K8S 集群）的用户账号代号

- `current-context`记录当前 kubectl 默认使用的上下文信息

- `kind`和`apiVersion`都是固定值，用户不需要关心

- `preferences`则是配置文件的其他设置信息，笔者没有使用过，暂时不提。

**第二步，给 kubectl 配置上配置文件**。

1. **`--kubeconfig`参数**。第一种办法是每次执行 kubectl 的时候，都带上`--kubeconfig=${CONFIG_PATH}`。给一点温馨小提示：每次都带这么一长串的字符非常麻烦，可以用 alias 别名来简化码字量，比如`alias k=kubectl --kubeconfig=${CONFIG_PATH}`。

2. **`KUBECONFIG`环境变量**。第二种做法是使用环境变量`KUBECONFIG`把所有配置文件都记录下来，即`export KUBECONFIG=$KUBECONFIG:${CONFIG_PATH}`。接下来就可以放心执行 kubectl 命令了。

3. **$HOME/.kube/config 配置文件**。第三种做法是把配置文件的内容放到$HOME/.kube/config 内。具体做法为：

4. 1. 如果$HOME/.kube/config 不存在，那么`cp ${CONFIG_PATH} $HOME/.kube/config`即可；
   2. 如果如果 $HOME/.kube/config已经存在，那么需要把新的配置内容加到 $HOME/.kube/config 下。单单只是`cat ${CONFIG_PATH} >> $HOME/.kube/config`是不行的，正确的做法是：`KUBECONFIG=$HOME/.kube/config:${CONFIG_PATH} kubectl config view --flatten > $HOME/.kube/config` 。解释下这个命令的意思：先把所有的配置文件添加到环境变量`KUBECONFIG`中，然后执行`kubectl config view --flatten`打印出有效的配置文件内容，最后覆盖$HOME/.kube/config 即可。

请注意，上述操作的优先级分别是 1>2>3，也就是说，kubectl 会优先检查`--kubeconfig`，若无则检查`KUBECONFIG`，若无则最后检查$HOME/.kube/config，如果还是没有，报错。但凡某一步找到了有效的 cluster，就中断检查，去连接 K8S 集群了。

**第三步：配置正确的上下文**

按照第二步的做法，如果配置文件只有一个 cluster 是没有任何问题的，但是对于有多个 cluster 怎么办呢？到这里，有几个关于配置的必须掌握的命令：

- `kubectl config get-contexts`。列出所有上下文信息。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwBG2ZBE0ONEhcltc7tbhAPUu3nxcyEyezEuvbsO5FeR1tOODgibmoRibg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- `kubectl config current-context`。查看当前的上下文信息。其实，命令 1 线束出来的*所指示的就是当前的上下文信息。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xw2YVf2MObzvFCibfxJmgVTibY7SiaZ0vtSE4mDhUYM01vVCTLJ36gytckQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- `kubectl config use-context ${CONTEXT_NAME}`。更改上下文信息。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xwpn7yHyKIX9Onh2hBc3HEID0iaBTOTmYZrwv3dEERiaXUR8ibgPCMiabCPw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- `kubectl config set-context ${CONTEXT_NAME}|--current --${KEY}=${VALUE}`。修改上下文的元素。比如可以修改用户账号、集群信息、连接到 K8S 后所在的 namespace。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xwaiayk7TTibIRccxs8jMXib1UrCCOthzibkiaZGGWy1h1lm3n2WgL3NExl6g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  关于该命令，还有几点要啰嗦的：

- - `config set-context`可以修改任何在配置文件中的上下文信息，只需要在命令中指定上下文名称就可以。而--current 则指代当前上下文。

  - 上下文信息所包括的内容有：cluster 集群（名称）、用户账号（名称）、连接到 K8S 后所在的 namespace，因此有`config set-context`严格意义上的用法：

    `kubectl config set-context [NAME|--current] [--cluster=cluster_nickname] [--user=user_nickname] [--namespace=namespace] [options]`

    （备注：[options]可以通过 kubectl options 查看）

综上，如何操作 kubectl 配置都已交代。

### **4.kubectl 部署服务**

K8S 核心功能就是部署运维容器化服务，因此最重要的就是如何又快又好地部署自己的服务了。本章会介绍如何部署 Pod 和 Deployment。

##### **4.1 如何部署 Pod？**

通过 kubectl 部署 Pod 的办法分为两步：1). 准备 Pod 的 yaml 文件；2). 执行 kubectl 命令部署

**第一步：准备 Pod 的 yaml 文件**。关于 Pod 的 yaml 文件初步解释，本系列上一篇文章《K8S 系列一：概念入门》已经有了初步介绍，这里再复习下：

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

继续解读：

- `metadata`，对于新入门的同学来说，需要重点掌握的两个字段：

- - `name`。这个 Pod 的名称，后面到 K8S 集群中查找 Pod 的关键字段。
  - `namespace`。命名空间，即该 Pod 隶属于哪个 namespace 下，关于 Pod 和 namespace 的关系，上一篇文章已经交代了。

- `spec`记录了 Pod 内部所有的资源的详细信息，这里我们重点查看`containers`下的几个重要字段：

- - `name`。Pod 下该容器名称，后面查找 Pod 下的容器的关键字段。

  - `image`。容器的镜像地址，K8S 会根据这个字段去拉取镜像。

  - `resources`。容器化服务涉及到的 CPU、内存、GPU 等资源要求。可以看到有`limits`和`requests`两个子项，那么这两者有什么区别吗，该怎么使用？在[What's the difference between Pod resources.limits and resources.requests in Kubernetes?](https://stackoverflow.com/questions/55047093/whats-the-difference-between-pod-resources-limits-and-resources-requests-in-kub)回答了：

    **`limits`是 K8S 为该容器至多分配的资源配额；而`requests`则是 K8S 为该容器至少分配的资源配额**。打个比方，配置中要求了 memory 的`requests`为 100M，而此时如果 K8S 集群中所有的 Node 的可用内存都不足 100M，那么部署服务会失败；又如果有一个 Node 的内存有 16G 充裕，可以部署该 Pod，而在运行中，该容器服务发生了内存泄露，那么一旦超过 200M 就会因为 OOM 被 kill，尽管此时该机器上还有 15G+的内存。

  - `command`。容器的入口命令。对于这个笔者还存在很多困惑不解的地方，暂时挖个坑，有清楚的同学欢迎留言。

  - `args`。容器的入口参数。同上，有清楚的同学欢迎留言。

  - `volumeMounts`。容器要挂载的 Pod 数据卷等。请务必记住：**Pod 的数据卷只有被容器挂载后才能使用**！

**第二步：执行 kubectl 命令部署**。有了 Pod 的 yaml 文件之后，就可以用 kubectl 部署了，命令非常简单：`kubectl create -f ${POD_YAML}`。

随后，会提示该命令是否执行成功，比如 yaml 内容不符合要求，则会提示哪一行有问题：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwPhD4JBdG4RScd3epywzJMFneKGRL29WkofibcWfFZ73cmXKGQU7k7gA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

修正后，再次部署：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwNEVqlrV83dAbc37tZX7z11XJhy3tXGW0BVA54d25X0pchfEEQHdKLg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### **4.2 如何部署 Deployment？**

**第一步：准备 Deployment 的 yaml 文件**。首先来看 Deployment 的 yaml 文件内容：

```
 apiVersion: extensions/v1beta1
 kind: Deployment
 metadata:
   name: rss-site
   namespace: mem-example
 spec:
   replicas: 2
   template:
     metadata:
       labels:
         app: web
     spec:
      containers:
       - name: memory-demo-ctr
         image: polinux/stress
         resources:
         limits:
           emory: "200Mi"
         requests:
           memory: "100Mi"
         command: ["stress"]
         args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
         volumeMounts:
         - name: redis-storage
           mountPath: /data/redis
     volumes:
     - name: redis-storage
       emptyDir: {}
```

继续来看几个重要的字段：

- `metadata`同 Pod 的 yaml，这里提一点：如果没有指明 namespace，那么就是用 kubectl 默认的 namespace（如果 kubectl 配置文件中没有指明 namespace，那么就是 default 空间）。

- `spec`，可以看到 Deployment 的`spec`字段是在 Pod 的`spec`内容外“包了一层”，那就来看 Deployment 有哪些需要注意的：

- - `metadata`，新手同学先不管这边的信息。
  - `spec`，会发现这完完全全是上文提到的 Pod 的`spec`内容，在这里写明了 Deployment 下属管理的每个 Pod 的具体内容。
  - `replicas`。副本个数。也就是该 Deployment 需要起多少个相同的 Pod，**如果用户成功在 K8S 中配置了 n（n>1）个，那么 Deployment 会确保在集群中始终有 n 个服务在运行**。
  - `template`。

**第二步：执行 kubectl 命令部署**。Deployment 的部署办法同 Pod：`kubectl create -f ${DEPLOYMENT_YAML}`。由此可见，**K8S 会根据配置文件中的`kind`字段来判断具体要创建的是什么资源**。

这里插一句题外话：**部署完 deployment 之后，可以查看到自动创建了 ReplicaSet 和 Pod**，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwlVqHX3aqo6xm1ibqNaibZ3rKOicn5noaAEnDfaSQXTj43lQ8NjkZEBzqA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

还有一个有趣的事情：**通过 Deployment 部署的服务，其下属的 RS 和 Pod 命名是有规则的**。读者朋友们自己总结发现哦。

综上，如何部署一个 Pod 或者 Deployment 就结束了。

### **5. kubectl 查看、更新/编辑、删除服务**

作为 K8S 使用者而言，更关心的问题应该是本章所要讨论的话题：如何通过 kubectl 查看、更新/编辑、删除在 K8S 上部署着的服务。

##### **5.1 如何查看服务？**

请务必记得一个事情：**在 K8S 中，一个独立的服务即对应一个 Pod。即，当我们说要 xxx 一个服务的就是，也就是操作一个 Pod**。而与 Pod 服务相关的且需要用户关心的，有 Deployment。

通过 kubectl 查看服务的基本命令是：

```
$ kubectl get|describe ${RESOURCE} [-o ${FORMAT}] -n=${NAMESPACE}
# ${RESOURCE}有: pod、deployment、replicaset(rs)
```

在此之前，还有一个需要回忆的事情是：Deployment、ReplicaSet 和 Pod 之间的关系 - 层层隶属；以及这些资源和 namespace 的关系是 - 隶属。如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwRFoqfcv3mvhDbrtDianUF0CGZ69iaZTia7J4fvRyw3JvcNO8Xwe0stAmg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

因此，**要查看一个服务，也就是一个 Pod，必须首先指定 namespace**！那么，如何查看集群中所有的 namespace 呢？`kubectl get ns`：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0Xwkg4iaQuibpBI9Q02nI6plCqHy0vwt1LsNvq5NmLT8ibBgnZToyjgOEicNA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

于是，只需要通过`-n=${NAMESPACE}`就可以指定自己要操作的资源所在的 namespace。比如查看 Pod：`kubectl get pod -n=oona-test`，同理，查看 Deployment：`kubectl get deployment -n=oona-test`。

问题又来了：**如果已经忘记自己所部属的服务所在的 namespace 怎么办？这么多 namespace，一个一个查看过来吗？**

```
kubectl get pod --all-namespaces
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwvuLlao7Mmx4Q2wx8RiaUnpRK6NgaiaicsxWCjyTQTb0JOrqMgJj6wjx3g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这样子就可以看到所有 namespace 下面部署的 Pod 了！同理，要查找所有的命名空间下的 Deployment 的命令是：`kubectl get deployment --all-namespaces`。

于是，就可以开心地查看 Pod：`kubectl get pod [-o wide] -n=oona-test`，或者查看 Deployment：`kubectl get deployment [-o wide] -n=oona-test`。

哎，这里是否加`-o wide`有什么区别吗？实际操作下就明白了，其他资源亦然：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwDsTrWRffgGkGSicEwTZZQD1zaDFLJAJ0u2Q44U2wticNWOe3LywN07NA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

哎，我们看到之前部署的 Pod 服务 memory-demo 显示的“ImagePullBackOff”是怎么回事呢？先不着急，我们慢慢看下去。

##### **5.2 如何更新/编辑服务？**

两种办法：1). 修改 yaml 文件后通过 kubectl 更新；2). 通过 kubectl 直接编辑 K8S 上的服务。

**方法一：修改 yaml 文件后通过 kubectl 更新**。我们看到，创建一个 Pod 或者 Deployment 的命令是`kubectl create -f ${YAML}`。但是，如果 K8S 集群当前的 namespace 下已经有该服务的话，会提示资源已经存在：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwXrpqsyug4JkXJIibdeAgGj6v5I4eibINwK808avEFCrLFicGBtL2ciajnA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

通过 kubectl 更新的命令是`kubectl apply -f ${YAML}`，我们再来试一试：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwclTRDSZUVbosAKJIjMCHlAIbSIMEV4EPqGXOTicibmD9ONWvEPVvB6gw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

（备注：命令`kubectl apply -f ${YAML}`也可以用于首次创建一个服务哦）

**方法二：通过 kubectl 直接编辑 K8S 上的服务**。命令为`kubectl edit ${RESOURCE} ${NAME}`，比如修改刚刚的 Pod 的命令为`kubectl edit pod memory-demo`，然后直接编辑自己要修改的内容即可。

但是请注意，无论方法一还是方法二，能修改的内容还是有限的，从笔者实战下来的结论是：**只能修改/更新镜像的地址和个别几个字段**。如果修改其他字段，会报错：

> The Pod "memory-demo" is invalid: spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds` or `spec.tolerations` (only additions to existing tolerations)

如果真的要修改其他字段怎么办呢？恐怕只能删除服务后重新部署了。

##### **5.3 如何删除服务？**

在 K8S 上删除服务的操作非常简单，命令为`kubectl delete ${RESOURCE} ${NAME}`。比如删除一个 Pod 是：`kubectl delete pod memory-demo`，再比如删除一个 Deployment 的命令是：`kubectl delete deployment ${DEPLOYMENT_NAME}`。但是，请注意：

- **如果只部署了一个 Pod，那么直接删除该 Pod 即可**；

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwbHHLGOltdOASRbQ9Bt5hrZmhmkg0v9yc76mgRC1WUoOhicdTUnQ38vw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- **如果是通过 Deployment 部署的服务，那么仅仅删除 Pod 是不行的，正确的删除方式应该是：先删除 Deployment，再删除 Pod**。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwcTxc1QtDRZNeESiazh6TsESZx9AibeRj8dIHeVfay75nGwp2HwJP3N3A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

关于第二点应该不难想象：仅仅删除了 Pod 但是 Deployment 还在的话，Deployment 定时会检查其下属的所有 Pod，如果发现失败了则会再拉起。因此，会发现过一会儿，新的 Pod 又被拉起来了。

另外，还有一个事情：有时候会发现一个 Pod 总也删除不了，这个时候很有可能要实施强制删除措施，命令为`kubectl delete pod --force --grace-period=0 ${POD_NAME}`。

### **6.kubectl 排查服务问题**

上文说道：部署的服务 memory-demo 失败了，是怎么回事呢？本章就会带大家一起来看看常见的 K8S 中服务部署失败、服务起来了但是不正常运行都怎么排查呢？

首先，祭出笔者最爱的一张 K8S 排查手册，来自博客《[Kubernetes Deployment 故障排除图解指南](https://tonybai.com/2019/12/08/k8s-deployment-troubleshooting/)》：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwAsWj5hj9A6xfMmUWWRJLznObQZicxf2JjGwb8ibZEqRlGIFeydOfibKCg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

哈哈哈，对于新手同学来说，上图还是不够友好，下面我们简单来看两个例子：

##### **6.1 K8S 上部署服务失败了怎么排查？**

请一定记住这个命令：`kubectl describe ${RESOURCE} ${NAME}`。比如刚刚的 Pod 服务 memory-demo，我们来看：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvauLok8egCcGJbUzUQjEd0XwYRwE63AA98ibln2shEyWaU0wUKlmlHD5S2tge0AXTPiaogC0BhNdIdFg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

拉到最后看到`Events`部分，会显示出 K8S 在部署这个服务过程的关键日志。这里我们可以看到是拉取镜像失败了，好吧，大家可以换一个可用的镜像再试试。

一般来说，通过`kubectl describe pod ${POD_NAME}`已经能定位绝大部分部署失败的问题了，当然，具体问题还是得具体分析。大家如果遇到具体的报错，欢迎分享交流。

##### **6.2 K8S 上部署的服务不正常怎么排查？**

如果服务部署成功了，且状态为`running`，那么就需要进入 Pod 内部的容器去查看自己的服务日志了：

- 查看 Pod 内部某个 container 打印的日志：`kubectl log ${POD_NAME} -c ${CONTAINER_NAME}`。
- 进入 Pod 内部某个 container：`kubectl exec -it [options] ${POD_NAME} -c ${CONTAINER_NAME} [args]`，嗯，这个命令的作用是通过 kubectl 执行了`docker exec xxx`进入到容器实例内部。之后，就是用户检查自己服务的日志来定位问题。

显然，线上可能会遇到更复杂的问题，需要借助更多更强大的命令和工具。

### 7.**写在后面**

本文希望能够帮助对 K8S 不了解的新手快速了解 K8S。笔者一边写文章，一边查阅和整理 K8S 资料，过程中越发感觉 K8S 架构的完备、设计的精妙，是值得深入研究的，K8S 大受欢迎是有道理的。

原文作者：oonamao毛江云，腾讯 CSIG 应用开发工程师

原文链接：https://mp.weixin.qq.com/s/mUF0AEncu3T2yDqKyt-0Ow