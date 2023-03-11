# 【NO.86】带你快速了解 Docker 和 Kubernetes

> 从单机容器化技术 Docker 到分布式容器化架构方案 Kubernetes，当今容器化技术发展盛行。本文面向小白读者，旨在快速带领读者了解 Docker、Kubernetes 的架构、原理、组件及相关使用场景。

## 1.**Docker**

### 1.1 什么是 Docker

Docker 是一个开源的应用容器引擎，是一种资源虚拟化技术，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上。虚拟化技术演历路径可分为三个时代：

1. 物理机时代，多个应用程序可能跑在一台物理机器上
   ![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151450107586854.png)
2. 虚拟机时代，一台物理机器启动多个虚拟机实例，一个虚拟机跑多个应用程序
   ![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151450403594316.png)
3. 容器化时代，一台物理机上启动多个容器实例，一个容器跑多个应用程序
   ![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151450571828054.png)

在没有 Docker 的时代，我们会使用硬件虚拟化（虚拟机）以提供隔离。这里，虚拟机通过在操作系统上建立了一个中间虚拟软件层 Hypervisor ，并利用物理机器的资源虚拟出多个虚拟硬件环境来共享宿主机的资源，其中的应用运行在虚拟机内核上。但是，虚拟机对硬件的利用率存在瓶颈，因为虚拟机很难根据当前业务量动态调整其占用的硬件资源，加之容器化技术蓬勃发展使其得以流行。

Docker、虚拟机对比：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151451137564461.png)

另外开发人员在实际的工作中，经常会遇到测试环境或生产环境与本地开发环境不一致的问题，轻则修复保持环境一致，重则可能需要返工。但 Docker 恰好解决了这一问题，它将软件程序和运行的基础环境分开。开发人员编码完成后将程序整合环境通过 DockerFile 打包到一个容器镜像中，从根本上解决了环境不一致的问题。

### 1.2 Docker 的构成

Docker 由镜像、镜像仓库、容器三个部分组成

- 镜像: 跨平台、可移植的程序+环境包
- 镜像仓库: 镜像的存储位置，有云端仓库和本地仓库之分，官方镜像仓库地址（https://hub.docker.com/）
- 容器: 进行了资源隔离的镜像运行时环境

### 1.3 Docker 的实现原理

到此读者们肯定很好奇 Docker 是如何进行资源虚拟化的，并且如何实现资源隔离的，其核心技术原理主要有(内容部分参考自 Docker 核心技术与实现原理)：
![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151451317631587.png)

#### **1.3.1 Namespace**

> 在日常使用 Linux 或者 macOS 时，我们并没有运行多个完全分离的服务器的需要，但是如果我们在服务器上启动了多个服务，这些服务其实会相互影响的，每一个服务都能看到其他服务的进程，也可以访问宿主机器上的任意文件，这是很多时候我们都不愿意看到的，我们更希望运行在同一台机器上的不同服务能做到完全隔离，就像运行在多台不同的机器上一样。

命名空间 (Namespaces) 是 Linux 为我们提供的用于分离进程树、网络接口、挂载点以及进程间通信等资源的方法。Linux 的命名空间机制提供了以下七种不同的命名空间，通过这七个选项我们能在创建新的进程时设置新进程应该在哪些资源上与宿主机器进行隔离。

1. CLONE_NEWCGROUP
2. CLONE_NEWIPC
3. CLONE_NEWNET
4. CLONE_NEWNS
5. CLONE_NEWPID
6. CLONE_NEWUSER
7. CLONE_NEWUTS

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151451544165176.png)

在 Linux 系统中，有两个特殊的进程，一个是 pid 为 1 的 /sbin/init 进程，另一个是 pid 为 2 的 kthreadd 进程，这两个进程都是被 Linux 中的上帝进程 idle 创建出来的，其中前者负责执行内核的一部分初始化工作和系统配置，也会创建一些类似 getty 的注册进程，而后者负责管理和调度其他的内核进程。

当在宿主机运行 Docker，通过`docker run`或`docker start`创建新容器进程时，会传入 CLONE_NEWPID 实现进程上的隔离。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151452065487278.png)

接着，在方法`createSpec`的`setNamespaces`中，完成除进程命名空间之外与用户、网络、IPC 以及 UTS 相关的命名空间的设置。

```
func (daemon *Daemon) createSpec(c *container.Container) (*specs.Spec, error) { s := oci.DefaultSpec() // ... if err := setNamespaces(daemon, &s, c); err != nil {  return nil, fmt.Errorf("linux spec namespaces: %v", err) } return &s, nil}func setNamespaces(daemon *Daemon, s *specs.Spec, c *container.Container) error { // user // network // ipc // uts // pid if c.HostConfig.PidMode.IsContainer() {  ns := specs.LinuxNamespace{Type: "pid"}  pc, err := daemon.getPidContainer(c)  if err != nil {   return err  }  ns.Path = fmt.Sprintf("/proc/%d/ns/pid", pc.State.GetPID())  setNamespace(s, ns) } else if c.HostConfig.PidMode.IsHost() {  oci.RemoveNamespace(s, specs.LinuxNamespaceType("pid")) } else {  ns := specs.LinuxNamespace{Type: "pid"}  setNamespace(s, ns) } return nil}
```

##### **网络**

当 Docker 容器完成命名空间的设置，其网络也变成了独立的命名空间，与宿主机的网络互联便产生了限制，这就导致外部很难访问到容器内的应用程序服务。Docker 提供了 4 种网络模式，通过`--net`指定。

1. host
2. container
3. none
4. bridge

由于后续介绍 Kubernetes 利用了 Docker 的 bridge 网络模式，所以仅介绍该模式。Linux 中为了方便各网络命名空间的网络互相访问，设置了 Veth Pair 和网桥来实现，Docker 也是基于此方式实现了网络通信。

下图中 `eth0` 与 `veth9953b75` 是一个 Veth Pair，`eth0` 与 `veth3e84d4f` 为另一个 Veth Pair。Veth Pair 在容器内一侧会被设置为 `eth0` 模拟网卡，另一侧连接 Docker0 网桥，这样就实现了不同容器间网络的互通。加之 Docker0 为每个容器配置的 iptables 规则，又实现了与宿主机外部网络的互通。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151452231388903.png)

挂载点

> 解决了进程和网络隔离的问题，但是 Docker 容器中的进程仍然能够访问或者修改宿主机器上的其他目录，这是我们不希望看到的。

在新的进程中创建隔离的挂载点命名空间需要在 clone 函数中传入 CLONE_NEWNS，这样子进程就能得到父进程挂载点的拷贝，如果不传入这个参数子进程对文件系统的读写都会同步回父进程以及整个主机的文件系统。当一个容器需要启动时，它一定需要提供一个根文件系统（rootfs），容器需要使用这个文件系统来创建一个新的进程，所有二进制的执行都必须在这个根文件系统中，并建立一些符号链接来保证 IO 不会出现问题。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151452363533407.png)

另外，通过 Linux 的`chroot`命令能够改变当前的系统根目录结构，通过改变当前系统的根目录，我们能够限制用户的权利，在新的根目录下并不能够访问旧系统根目录的结构个文件，也就建立了一个与原系统完全隔离的目录结构。

#### **1.3.2 Control Groups(CGroups)**

Control Groups(CGroups) 提供了宿主机上物理资源的隔离，例如 CPU、内存、磁盘 I/O 和网络带宽。主要由这几个组件构成：

1. **控制组（CGroup）** 一个 CGroup 包含一组进程，并可以在这个 CGroup 上增加 Linux Subsystem 的各种参数配置，将一组进程和一组 Subsystem 关联起来。
2. **Subsystem** 子系统 是一组资源控制模块，比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 CGroup 内存使用量。可以通过`lssubsys -a`命令查看当前内核支持哪些 Subsystem。
3. **Hierarchy** 层级树 主要功能是把 CGroup 串成一个树型结构，使 CGruop 可以做到继承，每个 Hierarchy 通过绑定对应的 Subsystem 进行资源调度。
4. **Task** 在 CGroups 中，task 就是系统的一个进程。一个任务可以加入某个 CGroup，也可以从某个 CGroup 迁移到另外一个 CGroup。

在 Linux 的 Docker 安装目录下有一个 docker 目录，当启动一个容器时，就会创建一个与容器标识符相同的 CGroup，举例来说当前的主机就会有以下层级关系：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151452499860071.png)
每一个 CGroup 下面都有一个 tasks 文件，其中存储着属于当前控制组的所有进程的 pid，作为负责 cpu 的子系统，cpu.cfs_quota_us 文件中的内容能够对 CPU 的使用作出限制，如果当前文件的内容为 50000，那么当前控制组中的全部进程的 CPU 占用率不能超过 50%。

当我们使用 Docker 关闭掉正在运行的容器时，Docker 的子控制组对应的文件夹也会被 Docker 进程移除。

#### **1.3.3 UnionFS**

> 联合文件系统(Union File System)，它可以把多个目录内容联合挂载到同一个目录下，而目录的物理位置是分开的。UnionFS 可以把只读和可读写文件系统合并在一起，具有写时复制功能，允许只读文件系统的修改可以保存到可写文件系统当中。Docker 之前使用的为 AUFS(Advanced UnionFS)，现为 Overlay2。

Docker 中的每一个镜像都是由一系列只读的层组成的，Dockerfile 中的每一个命令都会在已有的只读层上创建一个新的层：

```
FROM ubuntu:15.04COPY . /appRUN make /appCMD python /app/app.py
```

容器中的每一层都只对当前容器进行了非常小的修改，上述的 Dockerfile 文件会构建一个拥有四层 layer 的镜像：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151453055881176.png)
当镜像被 命令创建时就会在镜像的最上层添加一个可写的层，也就是容器层，所有对于运行时容器的修改其实都是对这个容器读写层的修改。容器和镜像的区别就在于，所有的镜像都是只读的，而每一个容器其实等于镜像加上一个可读写的层，也就是同一个镜像可以对应多个容器。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151453177329176.png)

## **2.Kubernetes**

> Kubernetes，简称 K8s，其中 8 代指中间的 8 个字符。Kubernetes 项目庞大复杂，文章不能面面俱到，因此这个部分将向读者提供一种主线学习思路：
>
> - 什么是 Kubernetes？
> - Kubernetes 提供的组件及适用场景
> - Kubernetes 的架构
> - Kubernetes 架构模块实现原理
>
> 有更多未交代或浅尝辄止的地方读者可以查阅文章或书籍深入研究。

### 2.1 为什么要 Kubernetes

尽管 Docker 为容器化的应用程序提供了开放标准，但随着容器越来越多出现了一系列新问题：

- 单机不足以支持更多的容器
- 分布式环境下容器如何通信？
- 如何协调和调度这些容器？
- 如何在升级应用程序时不会中断服务？
- 如何监视应用程序的运行状况？
- 如何批量重新启动容器里的程序？
- …

Kubernetes 应运而生。

### 2.2 什么是 Kubernetes

Kubernetes 是一个全新的基于容器技术的分布式架构方案，这个方案虽然还很新，但却是 Google 十几年来大规模应用容器技术的经验积累和升华的重要成果，确切的说是 Google 一个久负盛名的内部使用的大规模集群管理系统——Borg 的开源版本，其目的是实现资源管理的自动化以及跨数据中心的资源利用率最大化。

Kubernetes 具有完备的集群管理能力，包括多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建的智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制，以及多力度的资源配额管理能力。同时，Kubernetes 提供了完善的管理工具，这些工具涵盖了包括开发、部署测试、运维监控在内的各个环节，不仅是一个全新的基于容器技术的分布式架构解决方案，还是一个一站式的完备分布式系统开发和支撑平台。

### 2.3 Kubernetes 术语

#### **2.3.1 Pod**

Pod 是 Kubernetes 最重要的基本概念，可由多个容器（一般而言一个容器一个进程，不建议一个容器多个进程）组成，它是系统中资源分配和调度的最小单位。下图是 Pod 的组成示意图，其中有一个特殊的 Pause 容器:

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151453301682228.png)
Pause 容器的状态标识了一个 Pod 的状态，也就是代表了 Pod 的生命周期。另外 Pod 中其余容器共享 Pause 容器的命名空间，使得 Pod 内的容器能够共享 Pause 容器的 IP，以及实现文件共享。以下是一个 Pod 的定义：

```
apiVersion: v1  # 分组和版本kind: Pod       # 资源类型metadata:  name: myWeb   # Pod名  labels:    app: myWeb # Pod的标签spec:  containers:  - name: myWeb # 容器名    image: kubeguide/tomcat-app:v1  # 容器使用的镜像    ports:    - containerPort: 8080 # 容器监听的端口    env:  # 容器内环境变量    - name: MYSQL_SERVICE_HOST      value: 'mysql'    - name: MYSQL_SERVICE_PORT      value: '3306'    resources:   # 容器资源配置      requests:  # 资源下限，m表示cpu配额的最小单位，为1/1000核        memory: "64Mi"        cpu: "250m"      limits:    # 资源上限        memory: "128Mi"        cpu: "500m"
```

> EndPoint : PodIP + containerPort，代表一个服务进程的对外通信地址。一个 Pod 也存在具有多个 Endpoint 的情 况，比如当我们把 Tomcat 定义为一个 Pod 时，可以对外暴露管理端口与服务端口这两个 Endpoint。

#### **2.3.2 Label**

Label 是 Kubernetes 系统中的一个核心概念，一个 Label 表示一个 key=value 的键值对，key、value 的值由用户指定。Label 可以被附加到各种资源对象上，例如 Node、Pod、Service、RC 等，一个资源对 象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上。Label 通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。给一个资源对象定义了 Label 后，我们随后可以通过 Label Selector 查询和筛选拥有这个 Label 的资源对象，来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调 度、配置、部署等管理工作。

Label Selector 当前有两种表达式，基于等式的和基于集合的:

- `name=redis-slave`: 匹配所有具有标签`name=redis-slave`的资源对象。
- `env!=production`: 匹配所有不具有标签`env=production`的资源对象。
- `name in(redis-master, redis-slave)`:`name=redis-master`或者`name=redis-slave`的资源对象。
- `name not in(php-frontend)`:匹配所有不具有标签`name=php-frontend`的资源对象。

以 myWeb Pod 为例:

```
apiVersion: v1  # 分组和版本kind: Pod       # 资源类型metadata:  name: myWeb   # Pod名  labels:    app: myWeb # Pod的标签
```

当一个 Service 的 selector 中指明了这个 Pod 时，该 Pod 就会与该 Service 绑定

```
apiVersion: v1kind: Servicemetadata:  name: myWebspec:  selector:    app: myWeb  ports:  - port: 8080
```

#### **2.3.3 Replication Controller**

Replication Controller，简称 RC，简单来说，它其实定义了一个期望的场景，即声明某种 Pod 的副本数量在任意时刻都符合某个预期值。

RC 的定义包括如下几个部分：

- Pod 期待的副本数量
- 用于筛选目标 Pod 的 Label Selector
- 当 Pod 的副本数小于预期数量时，用于创建新 Pod 的模版(template)

```
apiVersion: v1kind: ReplicationControllermetadata:  name: frontendspec:  replicas: 3  # Pod 副本数量  selector:    app: frontend  template:   # Pod 模版    metadata:      labels:        app: frontend    spec:      containers:      - name: tomcat_demp        image: tomcat        ports:        - containerPort: 8080
```

当提交这个 RC 在集群中后，Controller Manager 会定期巡检，确保目标 Pod 实例的数量等于 RC 的预期值，过多的数量会被停掉，少了则会创建补充。通过`kubectl scale`可以动态指定 RC 的预期副本数量。

> 目前，RC 已升级为新概念——Replica Set(RS)，两者当前唯一区别是，RS 支持了基于集合的 Label Selector，而 RC 只支持基于等式的 Label Selector。RS 很少单独使用，更多是被 Deployment 这个更高层的资源对象所使用，所以可以视作 RS+Deployment 将逐渐取代 RC 的作用。

#### **2.3.4 Deployment**

Deployment 和 RC 相似度超过 90%，无论是作用、目的、Yaml 定义还是具体命令行操作，所以可以将其看作是 RC 的升级。而 Deployment 相对于 RC 的一个最大区别是我们可以随时知道当前 Pod“部署”的进度。实际上由于一个 Pod 的创建、调度、绑定节点及在目 标 Node 上启动对应的容器这一完整过程需要一定的时间，所以我们期待系统启动 N 个 Pod 副本的目标状态，实际上是一个连续变化的“部署过程”导致的最终状态。

```
apiVersion: v1kind: Deploymentmetadata:  name: frontendspec:  replicas: 3  selector:    matchLabels:      app: frontend    matchExpressions:      - {key: app, operator: In, values [frontend]}  template:    metadata:      labels:        app: frontend    spec:      containers:      - name: tomcat_demp        image: tomcat        ports:        - containerPort: 8080
```

#### **2.3.5 Horizontal Pod Autoscaler**

除了手动执行`kubectl scale`完成 Pod 的扩缩容之外，还可以通过 Horizontal Pod Autoscaling(HPA)横向自动扩容来进行自动扩缩容。其原理是追踪分析目标 Pod 的负载变化情况，来确定是否需要针对性地调整目标 Pod 数量。当前，HPA 有一下两种方式作为 Pod 负载的度量指标：

- CPUUtilizationPercentage，目标 Pod 所有副本自身的 CPU 利用率的平均值。
- 应用程序自定义的度量指标，比如服务在每秒内的相应请求数(TPS 或 QPS)

```
apiVersion: autoscaling/v1kind: HorizontalPodAutoscalermetadata:  name: php-apache  namespace: defaultspec:  maxReplicas: 3  minReplicas: 1  scaletargetRef:    kind: Deployment    name: php-apache  targetCPUUtilizationPercentage: 90
```

根据上边定义，当 Pod 副本的 CPUUtilizationPercentage 超过 90%时就会出发自动扩容行为，数量约束为 1 ～ 3 个。

#### **2.3.6 StatefulSet**

在 Kubernetes 系统中，Pod 的管理对象 RC、Deployment、DaemonSet 和 Job 都面向无状态的服务。但现实中有很多服务是有状态的，特别是 一些复杂的中间件集群，例如 MySQL 集群、MongoDB 集群、Akka 集 群、ZooKeeper 集群等，这些应用集群有 4 个共同点。

1. 每个节点都有固定的身份 ID，通过这个 ID，集群中的成员可 以相互发现并通信。
2. 集群的规模是比较固定的，集群规模不能随意变动。
3. 集群中的每个节点都是有状态的，通常会持久化数据到永久 存储中。
4. 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功 能受损。

因此，StatefulSet 具有以下特点：

- StatefulSet 里的每个 Pod 都有稳定、唯一的网络标识，可以用来 发现集群内的其他成员。假设 StatefulSet 的名称为 kafka，那么第 1 个 Pod 叫 kafka-0，第 2 个叫 kafka-1，以此类推。

- StatefulSet 控制的 Pod 副本的启停顺序是受控的，操作第 n 个 Pod 时，前 n-1 个 Pod 已经是运行且准备好的状态。

- StatefulSet 里的 Pod 采用稳定的持久化存储卷，通过 PV 或 PVC 来 实现，删除 Pod 时默认不会删除与 StatefulSet 相关的存储卷(为了保证数 据的安全)。

- StatefulSet 除了要与 PV 卷捆绑使用以存储 Pod 的状态数据，还要与 Headless Service 配合使用。

  > Headless Service : Headless Service 与普通 Service 的关键区别在于， 它没有 Cluster IP，如果解析 Headless Service 的 DNS 域名，则返回的是该 Service 对应的全部 Pod 的 Endpoint 列表。

#### **2.3.7 Service**

Service 在 Kubernetes 中定义了一个服务的访问入口地址，前段的应用（Pod）通过这个入口地址访问其背后的一组由 Pod 副本组成的集群实例，Service 与其后端 Pod 副本集群之间则是通过 Label Selector 来实现无缝对接的。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151453558761925.png)

```
apiVersion: v1kind: servicemetadata:  name: tomcat_servicespec:  ports:  - port: 8080   name: service_port  - port: 8005   name: shutdown_port  selector:    app: backend
```

**Service 的负载均衡**

在 Kubernetes 集群中，每个 Node 上会运行着 kube-proxy 组件，这其实就是一个负载均衡器，负责把对 Service 的请求转发到后端的某个 Pod 实例上，并在内部实现服务的负载均衡和绘画保持机制。其主要的实现就是每个 Service 在集群中都被分配了一个全局唯一的 Cluster IP，因此我们对 Service 的网络通信根据内部的负载均衡算法和会话机制，便能与 Pod 副本集群通信。

**Service 的服务发现**

因为 Cluster IP 在 Service 的整个声明周期内是固定的，所以在 Kubernetes 中，只需将 Service 的 Name 和 其 Cluster IP 做一个 DNS 域名映射即可解决。

#### **2.3.8 Volume**

Volume 是 Pod 中能够被多个容器访问的共享目录，Kubernetes 中的 Volume 概念、用途、目的与 Docker 中的 Volumn 比较类似，但不等价。首先，其可被定义在 Pod 上，然后被 一个 Pod 里的多个容器挂载到具体的文件目录下；其次，Kubernetes 中的 Volume 与 Pod 的生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，Volume 中的数据也不会丢失。

```
template:  metadata:    labels:      app: frontend  spec:    volumes:  # 声明可挂载的volume      - name: dataVol       emptyDir: {}    containers:    - name: tomcat_demo      image: tomcat      ports:      - containerPort: 8080      volumeMounts:  # 将volume通过name挂载到容器内的/mydata-data目录        - mountPath: /mydata-data         name: dataVol
```

Kubernetes 提供了非常丰富的 Volume 类型:

- emptyDir，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是 Kubernetes 自动分配的一个目录，当 Pod 从 Node 上移除 emptyDir 中的数据也会被永久删除，适用于临时数据。
- hostPath，hostPath 为在 Pod 上挂载宿主机上的文件或目录，适用于持久化保存的数据，比如容器应用程序生成的日志文件。
- NFS，可使用 NFS 网络文件系统提供的共享目录存储数据。
- 其他云持久化盘等

#### **2.3.9 Persistent Volume**

在使用虚拟机的情况下，我们通常会先定义一个网络存储，然后从中 划出一个“网盘”并挂接到虚拟机上。Persistent Volume(PV) 和与之相关联的 Persistent Volume Claim(PVC) 也起到了类似的作用。PV 可以被理解成 Kubernetes 集群中的某个网络存储对应的一块存储，它与 Volume 类似，但有以下区别:

- PV 只能是网络存储，不属于任何 Node，但可以在每个 Node 上访问。
- PV 并不是被定义在 Pod 上的，而是独立于 Pod 之外定义的。

```
apiVersion: v1kind: PersistentVolumemetadata:  name: pv001  spec:    capacity:      storage: 5Gi    accessMods:      - ReadWriteOnce    nfs:      path: /somePath      server: xxx.xx.xx.x
```

> accessModes，有几种类型，1.ReadWriteOnce:读写权限，并且只能被单个 Node 挂载。2. ReadOnlyMany:只读权限，允许被多个 Node 挂载。3.ReadWriteMany:读写权限，允许被多个 Node 挂载。

如果 Pod 想申请某种类型的 PV，首先需要定义一个 PersistentVolumeClaim 对象，

```
apiVersion: v1kind: PersistentVolumeClaim  # 声明pvcmetadata:  name: pvc001  spec:    resources:      requests:        storage: 5Gi    accessMods:      - ReadWriteOnce
```

然后在 Pod 的 Volume 中引用 PVC 即可。

```
volumes:  - name: mypd   persistentVolumeClaim:     claimName: pvc001
```

PV 有以下几种状态：

- Available：空闲
- Bound：已绑定到 PVC
- Relead：对应 PVC 被删除，但 PV 还没被回收
- Faild：PV 自动回收失败

#### **2.3.10 Namespace**

Namespace 在很多情况下用于实现多租户的资源隔离。分组的不同项目、小组或用户组，便于不同的分组在共享使用整个集群的资源的同时还能被分别管理。Kubernetes 集群在启动后会创建一个名为 default 的 Namespace，通过 kubectl 可以查看:

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151454149230502.png)

#### **2.3.11 ConfigMap**

我们知道，Docker 通过将程序、依赖库、数据及 配置文件“打包固化”到一个不变的镜像文件中的做法，解决了应用的部署的难题，但这同时带来了棘手的问题，即配置文件中的参数在运行期如何修改的问题。我们不可能在启动 Docker 容器后再修改容器里的配置 文件，然后用新的配置文件重启容器里的用户主进程。为了解决这个问题，Docker 提供了两种方式:

- 在运行时通过容器的环境变量来传递参数;
- 通过 Docker Volume 将容器外的配置文件映射到容器内。

在大多数情况下，后一种方式更合 适我们的系统，因为大多数应用通常从一个或多个配置文件中读取参数。但这种方式也有明显的缺陷:我们必须在目标主机上先创建好对应 配置文件，然后才能映射到容器里。上述缺陷在分布式情况下变得更为严重，因为无论采用哪种方式， 写入(修改)多台服务器上的某个指定文件，并确保这些文件保持一致，都是一个很难完成的目标。针对上述问题， Kubernetes 给出了一个很巧妙的设计实现。

首先，把所有的配置项都当作 key-value 字符串，这些配置项可以 作为 Map 表中的一个项，整个 Map 的数据可以被持久化存储在 Kubernetes 的 Etcd 数据库中，然后提供 API 以方便 Kubernetes 相关组件或 客户应用 CRUD 操作这些数据，上述专门用来保存配置参数的 Map 就是 Kubernetes ConfigMap 资源对象。Kubernetes 提供了一种内建机制，将存储在 etcd 中的 ConfigMap 通过 Volume 映射的方式变成目标 Pod 内的配置文件，不管目标 Pod 被调度到哪台服务器上，都会完成自动映射。进一步地，如果 ConfigMap 中的 key-value 数据被修改，则映射到 Pod 中的“配置文件”也会随之自动更新。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151454277097919.png)

### 2.4 Kubernetes 的架构

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151454478600733.png)

Kubernetes 由 Master 节点、 Node 节点以及外部的 ETCD 集群组成，集群的状态、资源对象、网络等信息存储在 ETCD 中，Mater 节点管控整个集群，包括通信、调度等，Node 节点为工作真正执行的节点，并向主节点报告。Master 节点由以下组件构成：

#### **2.4.1 Master 组件：**

1. API Server —— 提供 HTTP Rest 接口，是所有资源增删改查和集群控制的唯一入口。（在集群中表现为名称是 kubernetes 的 service）。可以通过 Dashboard 的 UI 或 kubectl 工具来与其交互。（1）集群管理的 API 入口；
   （2）资源配额控制入口；
   （3）提供完备的集群安全机制。
2. Controller Manager —— 资源对象的控制自动化中心。即监控 Node，当故障时转移资源对象，自动修复集群到期望状态。
3. Scheduler —— 负责 Pod 的调度，调度到最优的 Node。

#### **2.4.2 Node 组件：**

1. kubelet —— 负责 Pod 内容器的创建、启停，并与 Master 密切协作实现集群管理（注册自己，汇报 Node 状态）。
2. kube-proxy —— 实现 k8s Service 的通信与负载均衡。
3. Docker Engine —— Docker 引擎，负责本机容器的创建和管理。

### 2.5 Kubernetes 架构模块实现原理

#### **2.5.1 API Server**

Kubernetes API Server 通过一个名为 kube-apiserver 的进程提供服务，该进程运行在 Master 上。在默认情况下，kube-apiserver 进程在本机的 8080 端口(对应参数–insecure-port)提供 REST 服务。我们可以同时启动 HTTPS 安全端口(–secure-port=6443)来启动安全机制，加强 REST API 访问的安全性。

由于 API Server 是 Kubernetes 集群数据的唯一访问入口，因此安全性与高性能就成为 API Server 设计和实现的两大核心目标。通过采用 HTTPS 安全传输通道与 CA 签名数字证书强制双向认证的方式，API Server 的安全性得以保障。此外，为了更细粒度地控制用户或应用对 Kubernetes 资源对象的访问权限，Kubernetes 启用了 RBAC 访问控制策略。Kubernetes 的设计者综合运用以下方式来最大程度地保证 API Server 的性 能。

1. API Server 拥有大量高性能的底层代码。在 API Server 源码中 使用协程(Coroutine)+队列(Queue)这种轻量级的高性能并发代码， 使得单进程的 API Server 具备了超强的多核处理能力，从而以很快的速 度并发处理大量的请求。
2. 普通 List 接口结合异步 Watch 接口，不但完美解决了 Kubernetes 中各种资源对象的高性能同步问题，也极大提升了 Kubernetes 集群实时响应各种事件的灵敏度。
3. 采用了高性能的 etcd 数据库而非传统的关系数据库，不仅解决 了数据的可靠性问题，也极大提升了 API Server 数据访问层的性能。在 常见的公有云环境中，一个 3 节点的 etcd 集群在轻负载环境中处理一个请 求的时间可以低于 1ms，在重负载环境中可以每秒处理超过 30000 个请求。

#### **2.5.2 安全认证**

**RBAC**

Role-Based Access Control(RBAC)，基于角色的访问控制。

**4 种资源对象**

1. Role
2. RoleBinding
3. ClusterRole
4. ClusterRoleBinding

**Role 与 ClusterRole**

一个角色就是一组权限的集合，都是以许可形式，不存在拒绝的规则。Role 作用于一个命名空间中，ClusterRole 作用于整个集群。

```
apiVersion:rbac.authorization.k8s.io/v1beta1kind:Rolemetadata:  namespace: default #ClusterRole可以省略，毕竟是作用于整个集群  name: pod-readerrules:- apiGroups: [""]  resources: ["pod"]  verbs: ["get","watch","list"]
```

RoleBinding 和 ClusterRoleBinding 是把 Role 和 ClusterRole 的权限绑定到 ServiceAccount 上。

```
kind: ClusterRoleBindingapiVersion: rbac.authorization.k8s.io/v1metadata:    namespace: default    name: app-adminsubjects:-   kind: ServiceAccount    name: app    apiGroup: ""    namespace: defaultroleRef:    kind: ClusterRole    name: cluster-admin    apiGroup: rbac.authorization.k8s.io
```

**ServiceAccount**

Service Account 也是一种账号，但它并不是给 Kubernetes 集群的用户 (系统管理员、运维人员、租户用户等)用的，而是给运行在 Pod 里的进程用的，它为 Pod 里的进程提供了必要的身份证明。在每个 Namespace 下都有一个名为 default 的默认 Service Account 对象，在这个 Service Account 里面有一个名为 Tokens 的可以当作 Volume 被挂载到 Pod 里的 Secret，当 Pod 启动时，这个 Secret 会自动被挂载到 Pod 的指定目录下，用来协助完成 Pod 中的进程访问 API Server 时的身份鉴权。

#### **2.5.3 Controller Manager**

下边介绍几种 Controller Manager 的实现组件

**ResourceQuota Controller**

kubernetes 的配额管理使用过 Admission Control 来控制的，提供了两种约束，LimitRanger 和 ResourceQuota。LimitRanger 作用于 Pod 和 Container 之上(limit ,request)，ResourceQuota 则作用于 Namespace。资源配额，分三个层次：

> 1. **容器级别**，对容器的 CPU、memory 做限制
> 2. **Pod 级别**，对一个 Pod 内所有容器的可用资源做限制
> 3. **Namespace 级别**，为 namespace 做限制，包括：

```
    + pod数量    + RC数量    + Service数量    + ResourceQuota数量    + Secrete数量    + PV数量
```

**Namespace Controller**

管理 Namesoace 创建删除.

**Endpoints Controller**

Endpoints 表示一个 service 对应的所有 Pod 副本的访问地址，而 Endpoints Controller 就是负责生成和维护所有 Endpoints 对象的控制器。

> - 负责监听 Service 和对应 Pod 副本的变化，若 Service 被创建、更新、删除，则相应创建、更新、删除与 Service 同名的 Endpoints 对象。
> - EndPoints 对象被 Node 上的 kube-proxy 使用。

#### **2.5.4 Scheduler**

Kubernetes Scheduler 的作用是将待调度的 Pod(API 新创 建的 Pod、Controller Manager 为补足副本而创建的 Pod 等)按照特定的调 度算法和调度策略绑定(Binding)到集群中某个合适的 Node 上，并将绑定信息写入 etcd 中。Kubernetes Scheduler 当前提供的默认调度流程分为以下两步。

1. 预选调度过程，即遍历所有目标 Node，筛选出符合要求的候 选节点。为此，Kubernetes 内置了多种预选策略(xxx Predicates)供用户选择。
2. 确定最优节点，在第 1 步的基础上，采用优选策略(xxx Priority)计算出每个候选节点的积分，积分最高者胜出。

#### **2.5.5 网络**

Kubernetes 的网络利用了 Docker 的网络原理，并在此基础上实现了跨 Node 容器间的网络通信。

1. 同一个 Node 下 Pod 间通信模型：
   ![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151455076877270.png)
2. 不同 Node 下 Pod 间的通信模型（CNI 模型实现）：
   ![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151455201189114.png)

CNI 提供了一种应用容器的插件化网络解决方案，定义对容器网络 进行操作和配置的规范，通过插件的形式对 CNI 接口进行实现，以 Flannel 举例，完成了 Node 间容器的通信模型。
![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151455294724795.png)

可以看到，Flannel 首先创建了一个名为 flannel0 的网桥，而且这个 网桥的一端连接 docker0 网桥，另一端连接一个叫作 flanneld 的服务进程。flanneld 进程并不简单，它上连 etcd，利用 etcd 来管理可分配的 IP 地 址段资源，同时监控 etcd 中每个 Pod 的实际地址，并在内存中建立了一 个 Pod 节点路由表;它下连 docker0 和物理网络，使用内存中的 Pod 节点 路由表，将 docker0 发给它的数据包包装起来，利用物理网络的连接将 数据包投递到目标 flanneld 上，从而完成 Pod 到 Pod 之间的直接地址通信。

#### **2.5.6 服务发现**

从 Kubernetes 1.11 版本开始，Kubernetes 集群的 DNS 服务由 CoreDNS 提供。CoreDNS 是 CNCF 基金会的一个项目，是用 Go 语言实现的高性能、插件式、易扩展的 DNS 服务端。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151455389563234.png)

## **3.结语**

文章包含的内容说多不多，说少不少，但对于 Docker、Kubernetes 知识原理的小白来说是足够的，笔者按照自己的学习经验，以介绍为出发点，让大家更能了解相关技术原理，所以实操的部分较少。Kubernetes 技术组件还是十分丰富的，文章有选择性地进行了介绍，感兴趣的读者可以再自行从官方或者书籍中学习了解。（附《Kubernetes 权威指南——从 Docker 到 Kubernetes 实践全接触》第四版）

## 4.**相关链接**

- [Docker 核心技术与实现原理](https://draveness.me/docker/)
- [官方镜像仓库地址](https://hub.docker.com/)
- [Kubernetes 官网](https://kubernetes.io/)
- [Kubernetes 中文社区](https://www.kubernetes.org.cn/)

原文作者：honghaohu，腾讯 PCG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/ji0Pj00xeHOeispNhsPKZw