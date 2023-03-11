# 【NO.175】2021 有哪些不容错过的后端技术趋势

## 0.**前言**

2020 年注定是不平凡的一年，虽疫情肆虐，但我国互联网产业展现出巨大韧性，不仅为精准有效防控疫情发挥了关键作用，还在数字基建、数字经济等方面取得了显著进展，成为我国应对新挑战、建设新经济的重要力量。

腾讯在线教育部后台中心团队，作为在线教育行业的从业者，我们尝试整理一下 2020 年后端技术要点，以此窥探后台未来技术的发展趋势：

1. 云计算进程提速，一切皆服务。
2. 云上安全越来越受到企业的重视。
3. 从资源云向业务云化转变，最终全面云原生化。
4. 微服务、DDD、中台技术并非企业技术架构设计的银弹。
5. Python、Go、Rust 成为后端未来最先考虑学习编程语言。
6. Go 语言生态发展稳健，越来越多企业在生产中使用 Go 语言落地业务。
7. 疫情催化在线教育行业产品升级转型，音视频技术不断迭代升级。

## 1.**云原生**

### 1.1 业内趋势

#### 1.1.1 **云原生技术生态日趋完善，细分项目不断涌现**

云原生关键技术正在被广泛采纳，如 43.9%的用户已在生产环境中采纳容器技术，超过七成的用户已经或计划使用微服务架构进行业务开发部署。

#### 1.1.2 **容器云平台将传统云计算的 IaaS 层和 PaaS 层融合**

从技术角度看，容器云平台采用容器、容器编排、服务网格，无服务等技术构建的一种轻量化 PaaS 平台，为应用提供了开发、编排、发布、治理和运维等全生命周期管理。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXricVRsFIOn8gVF1SicZ8HUBIht5UicjRaymUxKSWU6qGsPnJ1iaT0qZvlQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

容器云平台的整体架构，自下而上包括交互(UI)层、接口(API)层、PaaS 服务层、基础层。运维和安全则涵盖了从应用层到容器云引擎层的部分：

- 交互层：提供界面供用户使用
- 接口层：提供 OpenAPI 能力供第三方调用
- PaaS 服务层：提供数据服务、应用服务（微服务、中间件）、DevOps、平台管理、平台运营、应用管理能力，为实现业务应用其自身的生命周期管理
- 基础层：以 Kubernetes 为核心，包括服务网格（ServiceMesh）、无服务计算（Serverless）、容器引擎（Docker）、容器镜像管理等，主要实现对计算、网络和存储资源的池化管理，以及以 Pod 为核心的调度和管理。

#### 1.1.3 **服务网格为微服务带来新的变革**

Mesh 化加速业务逻辑与非业务逻辑的解耦，将非业务功能从客户端 SDK 中分离出来放入独立进程，利用 Pod 中容器共享资源的特性，实现用户无感知的治理接管。

#### 1.1.4 **从资源云向业务云化转变，最终全面云原生化**

云原生技术通过标准化资源，轻量化弹性调度等特征，应用场景较为广泛，随着技术和生态不断成熟和完善，有效缓解企业上云顾虑，拉动全行业的上云程度。

#### 1.1.5 **云原生技术栈的标准统一化**

架构标准统一（微服务之间标准 API 接口通信）、交付标准统一（标准容器化的打包方式实现真正的应用可移植）、研运过程标准统一（DevOps 工具链标准统一），通过标准化后提整体研发运维效能。

### 1.2 在线教育实践

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXcFuUtC3gtENroQLFfiaCticdoC1t3zzJoZPI325oLkibs0hZjBZ92w6gw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)enter image description here

#### 1.2.1 **各类 PaaS 服务上云**

2019 年我们完成了 IaaS、存储层、直播、回放以及各类 PaaS 服务的上云。

#### 1.2.2 **服务全面容器化升级**

2020 年我们重点进行服务全面的容器化升级，目前已经完成企鹅辅导和开心鼠英语两个产品的全面改造，到年底会完成腾讯课堂剩余部分的升级，实现全面完成改造。

#### **1.2.3 完善 DevOps 流程**

完善 CI/CD/CO、蓝盾流水线、容器化、STKE、全链路监控等，提高研发效率，降低现网运营难度

#### 1.2.4 **业务中台架构演进**

在整体架构上，我们依托腾讯云，确定了教育业务中台的架构演进方向，不断的进行重复模块的抽象和整合。我们在腾讯云上实现部署了接入中台、Push 中台、支付中台、音视频中台、运营中台等服务，让各个业务之间的相似能力得以复用。

#### 1.2.5 **存储层上云**

存储层上云后，一方面稳定性提高。

- 异地容灾。通过挂载异地的灾备机器，可以实现 master 主机异地灾备。
- 负载均衡。服务连接 RO 组，RO 组的多个实例会对请求进行负载均衡。
- 数据备份。RO 发生异常，将会被剔除 RO 组，恢复后自动加入 RO 组，保证了 RO 组的可用性。
- 数据加密。提供透明数据加密（Transparent Data Encryption，TDE）功能，透明加密指数据的加解密操作对用户透明，支持对数据文件进行实时 I/O 加密和解密，在数据写入磁盘前进行加密，从磁盘读入内存时进行解密，可满足静态数据加密的合规性要求。

另一方面运营能力也有所提升。

- 可以实时看到数据库连接情况，慢查询、全表扫描、查询、更新、删除、插入情况
- 实时 CPU、内存、磁盘使用情况，并根据设置阈值进行告警优化微服务架构

下面是在线教育上云前后架构对比

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXSibYZcOXHGic8trd84LZCK4rKOZnf0DSPuuibgRNxwGQ9FL7A5GcQUE6g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.**微服务**

微服务,Service Mesh 在过去的一年依旧保持着热度。在已经过去的 2020，微服务可以说有坚守也有破局，有对服务微化共识的形成也有对特殊场景的理性思考。我们可以看到服务框架依然在持续演进，奔向云原生，拥抱云化。越来越多的企业开始跟上服务化云化步伐。

微服务框架：加速奔向云原生

### 2.1 **SpringCloud**

2018 年开始 Hystrix、Ribbon 等核心组件相继进入维护状态，开发者们一度变得忧心忡忡，时至今日我们回过头来再看一下，Spring Cloud 已经针对这些担忧给出了解决方案，Zuul 由 Spring Cloud GateWay 子项目替代，Hystrix 由 Spring Cloud Circuit Breaker 替代，同时也给出了长期的演进方案。在经历了这段小小的波折后，Spring Cloud 也改变了策略，将这些企业贡献的 OSS 库独立出来成为其子项目。目前我们可以看到有 Azure，Alibaba, Amazon 等 3 个带有企业名字的子项目，这种策略在某种程度上可以说解绑了企业开源策略对开源核心组件的影响。截至目前 Spring Cloud 下面的子项目已经新增至 34 个，越来越庞大。供开发者选择的组件越来越多。

### 2.2 **Dubbo**

2019 年 05 月 20 日 Dubbo 毕业，成为 Apache 的顶级项目，在过去的一年社区还是非常努力的，一年 release 5 个版本，加速奔向云原生。在 2.7.5 版本中，其服务模型调整以及协议支持调整带来的新旧版本兼容问题，稳定性等问题值得我们持续关注。

### 2.3 **Istio**

记得 2019 年我们一直在谈 istio 版本难产问题，在 2020 年却出乎意外的因为商标问题上了头条，让我们吃了个大瓜。Google 与 IBM 在商标问题上发生分歧，Istio 商标被 Google 捐献给 Open Usage Commons 组织，而非 CNCF。而这在加速了 Service Mesh 阵营依旧的分化，各大软件厂商纷纷发布了自己的 Service Mesh 产品，如微软发布了 Open Service Mesh，Kong 推出了 Kuma，Nginx 也推出了 NGINX Service Mesh（NSM）。

### 2.4 **企业微服务建设：长期修行，苦练内功**

在微服务框架的演进过程中我们看到都在朝着趋同的方向发展，主要聚焦于微服务治理形态上组件的差异化以及应对场景的方案细化，可能你们家服务中心用的 ZK，我们家就自研。恰逢内源的兴起，似乎在企业内部再造一次轮子，研发一套特定的框架来适配企业业务以及标准化企业内部 IT 治理也是一件很容易的事情，基于这种写实的场景部分企业开始涌现出了内源的服务框架如，腾讯的 tRPC 框架。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXMibsibicg9KtvUo69xbSyEU3bmoE6SYQTt3wicrWKX6LhsvymVicb7bY53w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)enter image description here

**腾讯 tRPC 建设情况** 目前 tRPC 在腾讯内部已经大面积推广使用，覆盖 5 个 BG，40+部门，2700+服务，10000+容器，支持 c++,go,java,js,rust,python 6 种编程语言。其可插拔的插件化架构，高性能，友好的架构兼容特性正在吸引内部越来越多的开发者以及业务用户。

在线教育业务也在积极的拥抱这套框架，逐步将各业务牵引到 tRPC 框架。在解决历史技术架构痛点的过程中，通过微服务构筑，形成微服务群，构筑稳定的支付,音视频等小中台以及面向 C、B 端用户的互联网业务系统。

## 3.**中台**

中台，是最近几年最火热的技术名词之一，关于中台的讨论，甚至是争论，一直都没有停止过。我们尝试结合腾讯在线教育部在中台方向的实践经验，谈谈中台对我们的意义和建设情况。

腾讯在线教育部从 2018 年开始规划部门内的中台建设，2019 年基本完成组织架构和技术架构的中台转型。我们和大多数公司一样，并不是从 0 开始构建中台，而是在保证现有业务快速迭代的前提下，同时完成架构的转型，大家形象的称这个过程是“开着飞机换引擎”。对于一个风险看起来这么高的事情，在决定做之前，我们要回答好几个问题：

### 3.1 我们为什么要做中台，部门需要什么样的中台？

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXEz65VsUwlIx8XqMHHcr5Cj3QcehUFTrNOPiccibG5Aic44icvZg3cIYzvg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

我们部门主要有三款产品：腾讯课堂——职业在线教育学习平台，具有 2B 和 2C 的双重属性。支持教育机构的入驻、直播上课、售卖、结算 等功能。腾讯企鹅辅导——腾讯自营的，主打 K12 名师教学的学习应用，为老师和学生提供了丰富的在线教学的功能。腾讯开心鼠英语——主打 3-8 岁的少儿英语学习，通过生动有趣的交互式学习设计，实现了边玩边学的有趣的学习体验。可以看到，这三个产品有不少类似的功能，例如直播、回放、支付、退款 等，因为这些共性，决定了他们会有很多相同的产品和技术需求。这些形成我们做中台的业务基础。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXKJAQq1hyT7FXsvoMf9rtBNfEwVGXTGV9Kribiaz4zNg9Jp7FuAsGKe9g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在产品发展的初期，由于时间窗口非常紧，需求变化也很频繁。为了快速并行迭代，我们拉起了三个独立的团队进行研发，除了基础设施外，业务逻辑部分完全是独立的。这种组织架构，在当时确实为我们达成了上线时间的目标，帮助产品实现了从 0 到 1 的突破。但是，随着产品形态的成熟，3 个问题越来越突出：第一个问题，功能无法在不同的产品间快速复用。因为独立的代码和架构，复用变得非常的困难，很多开发同学反馈复用代码还不如重写一遍更快。第二个问题，同类的 Bug 的解决和技术优化在不同的产品之间重复进行，非常的浪费人力。第三个问题，不同的时期，我们需要发力的产品方向不一样。当一个产品面临发展窗口期的时候，对研发人力的需求就会成倍的增长。而独立的研发模式，让人力调配非常的困难。基于业务的模型，和团队碰到的痛点，我们提出了中台化的解决方案。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBX2E2mFRO2vicFUbcGXroCwYwJ1ibgrQIkaCA7Ic9pex6sGDSat7DoAa4g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

我们需要的中台应该长什么样子？经过前期的思考，我们总结出在线教育的中台应该包含 4 个部分。首先 是业务支持部分，这个很好理解，包含了各类共性的产品功能，例如前面提到的 直播、回放、支付 等等。其次 是技术支持部分，服务开发过程中 必然会涉及到例如 技术栈怎么选择，高可用怎么做 的共性技术问题，我们希望这部分有一个统一的技术实现。接下来 是数据支持部分，数据上报、计算、汇总、分析 已经是现在互联网产品必不可少的能力，也具备很强的通用性。因此这部分也应该有一个中台来承载。最后，是研发效能部分，我们需要有一整套好用的工具来提高研发效率和保障研发质量。到这里，我们对于中台的模样，应该是比较清晰了。

根据这个规划，我们画出了我们中台的组成图。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBX4Ez7wicyB4xvvKTBupemIQtDj2H3BiaPBzQnvvYttZ7nlK0moFVT1wag/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 3.2 怎么保证中台能做成功，最大的风险是什么？

很多人说，中台是“一把手工程”，意思是一定是自上而下推动的，需要老板的鼎力支持，是因为做中台确实是非常的需要人力，并且很大程度上改变了团队的资源投入模式，在多个业务需求压力都很大的情况下，做这么大的转变，团队短期内一定会碰到各种不适应的问题。虽然我们的中台方案也得到了老板的支持，但是绝不是中台优先，毕竟保障业务的高速发展才是团队追求的结果。一开始我们就意识到了困难的存在，为了不让中台死在半路，我们定了几个原则：

1. 控制人力占比：在人力有限的情况下，为了保障业务需求不受太大的进度影响，中台的人力投入原则上不超过总人力的 30%。
2. 不做过度设计：中台的设计目标只控制在部门内的需求（腾讯课堂、企鹅辅导、开心鼠英语），不面向行业做完全通用化的设计，根据实际需求做决策。
3. 完整规划，逐步实施：在做好完整的技术方案设计之后，我们不追求一次性完成中台的建设，而是结合业务产品需求的情况逐步实施，每半年也会 review 一次方案是否需要调整。

以上的规则，可以说很好的帮助我们保障了中台平滑转型的过程。另外，我们也在一个合适的时间点进行了团队人员组织结构的中台化匹配升级，保证了技术架构和组织架构的一致。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXasT1LlVr2zaSlibUia1qwFib25dJmv9sl7oNaSLOS5YGcicYSAFd4wIia1g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 3.3 中台建设的指导原则是什么，目标是什么？

我们中台服务设计的原则是什么，应该做哪些，什么时候做，做到什么程度。这个在业务部门里面其实是一个非常棘手的问题。很多时候我们需要在“时效性” 和 “扩展性”方面做选择。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXowQPlN53WBoeeBeegkxg4PeKqpW41lywsTLOGKd5OMI4c2ia5bHOSsg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

最后我们总结出来的一套方法是这样：对于一个新的需求，如果看到了可复用性，优先让业务团队自己做，一般这种需求时间紧、不明确，如果这时候讨论宏大的中台设计，往往效率低，耽误上线时间，但是，我会在过方案的时时候问大家“通用性是怎么考虑的”，最终在方案设计上做好通用型的前期 准备即可。后面如果再次收到另外一个产品的类似需求，这时候我们就认真考虑要把这个服务交接到中台组来维护了，由中台组安排人力来进行中台化。最后再有新的业务接入，就直接由中台组来承接了，就会非常的简单，我们很多中台服务都是这么跑出来的。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBX3HibeKf74HXvF3RFr2H0yslZNSbklA6amUFibOpGzFe6qtoL6hO4e3VA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

还有一个需要思考的问题是，中台服务的建设目标是什么。提出这个问题的背景是，我们希望给中台团队一个统一的、清晰的阶段性目标，当然也可以用来做团队考核。我们总结出来三个点：

1. 功能的复用：这个是最基本的，也是提出中台的初衷。具体到落地上，必须是一套代码。
2. 统一运营：要求中台服务能分产品输出标准化的实时监控看板和报表邮件，让业务一目了然。
3. 容灾调度的能力：不同业务的多套部署之间，在紧急情况下可以互备。需要进行实际的线上演习。

我们认为，达成这三个目标之后，才真正的发挥了中台的威力，可以实现 1+1 大于 2 的效果。

## 4.**DevOps**

2020 年，在云原生的浪潮下，devops 相关的技术栈也在稳步地向前演进，下面将从以下几个方面分别进行阐述：

- 敏捷的应用交付流程
- 监控告警系统
- tracing 系统
- 云原生对提升 devops 的展望

### 4.1 敏捷的应用交付流程

#### 4.1.1 **业内趋势**

当前出现了一系列的基于 Kubernetes 的 CI/CD 工具，如 Jenkins-x、Gitkube，它提供了从代码提交、自动编译、打包镜像、配置注入、发布部署到 Kubernetes 平台的一系列自动化流程。甚至出现了像 ballerina 这样的云原生编程语言，它的出现就是为了解决应用开发到服务集成之间的鸿沟。然后结合监控告警系统实时掌握服务运行情况，结合调用链系统进行服务故障定位。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXB5viaQLaialh7f5CHH42znHV013ASs5vPbfJLqILYNyOs18HHibFIh9RQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### 4.1.2 **在线教育的实践**

蓝盾是腾讯从业务安全出发，贯穿产品研发、测试和运营的全生命周期；助力业务平滑过渡到敏捷研发模式，打造的一站式研发运维体系。它助力业务持续快速交付高质量的产品。蓝盾提供了丰富的特性：

- 可视化
- 一键式部署
- 和持续集成无缝集成
- 支持并行部署
- 架构水平扩展，相同逻辑的节点无主备关系
- 数据安全
- 小核心，大扩展！可插件化扩展，优先添加所需要的流程控制部件，同时可方便的扩展其他部件
- 可监控，监控一切异常的构建并告警
- 灰度切换，达到切换时不影响正在构建的流水线

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXGeWelspV6pMtevueROn33g0SUMMMQjK6MY5uQgUjOHjfOn4icGiagWcg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.2 监控告警系统

#### 4.2.1 **监控系统事实上的标准 prometheus**

Prometheus 在度量领域的统治力虽然还暂时不如日志领域中 Elastic Stack 的统治地位那么稳固，但在云原生时代里，基本也已经能算是事实标准了。2020 年，对 prometheus 来说，是忙碌的一年：

grafana cloud agent 发布：为 grafana cloud 优化的的一个轻量级 prometheus 分支，它使用 prometheus 的代码，保证 prometheus 生态依赖的一些属性：数据完整性、数据过期处理等等。允许使用一种更灵活的方式定义从何处以什么方法拉取和传输数据，它已经成为一种更受欢迎的方式将数据写入后端存储。它的 remote_write 方式相比于传统 prometheus agent 的 remote_write，内存使用降低了 40%；

v2.19.0 的发布：最核心的功能是，对于完整的 chunk，在内存中使用 head 结构映射磁盘中的 block，单就这一个点，内存使用就降低了 20%-40%，并且使得 prometheus 的重启速度更快；

v2.20.0 的发布：它为 Docker Swarm 和 DigitalOcean 支持了本地的服务发现；

提升批量插入一大批老数据到 prometheus 的效率；

Grafana Metrics Enterprise 的启动：提供针对大企业的 prometheus-as-a-service 的解决方案

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXvNEicnE71PVs5Bkh2bA5BibrlmqKz3k1pGqhm5pZCgDdQVGCsCp7R1lw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### 4.2.2 **在线教育的实践**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXbO09MNmgeEiatbkHj24R0YoXrTkxtPbVxPadqnFX1JmlLuZuBibhF7PQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

nginx 访问日志和服务间模块调用相关信息都上报到全链路 es 集群

通过一个 exporter 对 es 中的数据按照项目和接口维度进行汇聚，将相关的聚合数据存入 prometheus

全链路 es 和 prometheus 都可以作为 grafana 的数据源进行数据展示和告警判断

alert 模块将告警信息发送到相应的渠道

收到告警之后，去 grafana 查看对应时间段的趋势图，去全链路 es 集群查看对应时间段的原始日志

### 4.3 tracing 系统

比起日志与度量， tracing 这个领域的产品竞争要相对激烈得多。一方面，目前还没有像日志、度量那样出现具有明显统治力的产品。另一方面，几乎市面上所有的追踪系统都是以 Dapper 的论文为原型发展出来的，功能上并没有太本质的差距，却又受制于实现细节，彼此互斥，很难搭配工作。

#### 4.3.1 **OpenTracing 规范**

为了推进追踪领域的产品的标准化，2016 年 11 月，CNCF 技术委员会接受了 OpenTracing 作为基金会第三个项目。OpenTracing 是一套与平台无关、与厂商无关、与语言无关的追踪协议规范。

#### 4.3.2 **OpenCensus 规范**

OpenTracing 规范公布后，几乎所有业界有名的追踪系统，譬如 Zipkin、Jaeger、SkyWalking 等都很快宣布支持 OpenTracing，但是 Google 自己却在此时提出了与 OpenTracing 目标类似的 OpenCensus 规范。OpenCensus 不仅涉及到追踪，还把指标度量也纳入进来。

#### 4.3.3 **OpenTelemetry 规范**

2019 年，OpenTracing 和 OpenCensus 又共同发布了可观测性的终极解决方案 OpenTelemetry，并宣布会各自冻结 OpenTracing 和 OpenCensus 的发展。OpenTelemetry 野心颇大，不仅包括追踪规范，还包括日志和度量方面的规范、各种语言的 SDK、以及采集系统的参考实现。

#### 4.3.4 **在线教育的实践**

腾讯基于 OpenTelemetry 规范，自研了天机阁系统。天机阁主要分为三大部分：

- 分布式追踪（distributed tracing）
- 监控（monitoring，metrics）
- 日志（logging）

与之对应提供七大能力：

- 故障定位：天机阁中能够提供整个请求全链路上下文信息，具体哪个环节出错一目了然
- 耗时分析：天机阁的耗时分布图，可以快速了解全链路耗时情况
- 多维染色：在天机阁基于通用的设计理念下，天机阁提供的染色能力，不会局限在某个业务的具体字段，同样也不会局限单个维度
- 架构治理：天机阁架构治理的核心功能是微服务架构拓扑，基于微服务架构拓扑可以构建更加丰富更加具体的上层分析能力
- 全链路日志：天机阁核心建立在分布式追踪方法论下，提供将分散在各个微服务的日志根据因果和时间有机进行组合，达到提供全链路上下文日志的效果
- 服务监控：天机阁在监控领域将会重点推出如下几大领域监控（机器基础指标监控、数据库监控、进程监控、模调监控）
- 业务看板：主要用于业务定制化的数据指标配置和展示

系统架构图

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXl7L12RLDwwG00w4Ywp6GYiblnJx54bHo5ppaqRsoWyOGx03jsykYPWg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

功能模块图

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXVibz1nKNyR1Nh4x98ia2yWictrWf2uffgNWoPNdhntfGKpMr1nNorc2Aw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.4 云原生对提升 devops 的展望

#### 4.4.1 **serverless 对提升 devops 的展望**

- 降低运维需求
- 缩短迭代周期、上线时间
- 快速试错
- 极致弹性
- 降低运营成本

#### 4.4.2 **service mesh 对提升 devops 的展望**

- 更好发挥掌握不同编程语言的人才优势
- 框架的部分能力平台化，加速应用迭代速度
- 微服务软件架构从解决"拆"到解决"连"的加速落地
- 灵活的流量治理能力改善软件的发布与回滚效率

## 5.**音视频**

### 5.1 音视频技术回顾

随着 AI 技术的兴起、5G 时代的到来，音视频技术不断加速应用发展，像直播、短视频这样的产品遍地开花，火热程度相信大家或多或少都接触过。

音视频技术的加速应用依赖底层编解码标准的发展，当前主流的 H.264 编解码技术已经不能满足未来 4K、8K 的需求，今年年中刚发布的 H266/VCC，与 H.265 相比进一步提高了压缩效率，这项耗时 3 年的标准，主要面向未来的 4K 和 8K，后续的落地应用非常值得期待。

在实时音视频技术领域，不得不提及谷歌的开源项目 WebRTC，可以在浏览器上快速开发出各种音视频应用，目前主流的浏览器包括 Chrome、Firefox、Safari 等都将 WebRTC 作为首选的实时音视频方案。同时也催生了像声网、即构科技这样的专门音视频服务商。从 StackOverflow Trends 和 GoogleTrends 来看，未来关注度仍会持续上升，腾讯也是 WebRTC 应用的主力军。

目前直播后台开发主要分为三大块：

1. 标准直播：我们日常生活中使用频率最高的直播，例如电视节目直播、游戏直播、直播带货。
2. 快直播：标准直播在超低延迟直播场景基础上的延伸，毫秒级超低延迟播放的同时，也兼顾了秒开、卡顿等核心问题，为用户带来超低延迟流畅度的直播体验。
3. 慢直播：能够提供更稳定清晰的直播画面，基本的使用场景都用于监控领域，国内疫情早期，4000 万人同时在线，通过一个固定机位观看雷神山医院建筑工地的现场直播，让云监工迅速火爆。2020 行至年终，各大机构评选的网络热词相继出炉，其中，“云监工”频繁出没于「十大网络热词」榜单中，与之并列的多是“后浪”“网抑云”“打工人”等。

随着网络技术的不断发展，持续提供高质量的视频信号传播已经算不上浪费网络资源，即使一个直播无人观看，未来慢直播具有极大的延伸价值以及发展前景，让我们期待慢直播行业的蓬勃发展。

借助 5G 技术低时延、高速率、大容量等显著优势，音视频的大赛道，从目前的短视频慢慢走向中长视频发展，这是未来的大风口。

### 5.2 平台的新技术点

目前腾讯在线教育音视频直播已完成整体上云，腾讯云的互动直播也从早期的 opensdk 全面升级到 TRTC，TRTC 是腾讯实时音视频[Tencent Real-Time Communication]，源自 QQ 音视频团队，是基于 QQ 十几年来的音视频技术积累。

腾讯云提供 TRTC(全球延时<300ms)+WebRTC 快直播(上行走 RTMP 推流或 FLV、HLS、RTMP 回源，下行支持标准 WebRTC 协议输出，延时 500ms 左右）+标准 LVB 直播(FLV/HLS/DASH，平均延时 3-5 秒)融合解决方案，如下图中用户可以针对自己的业务场景组合不同的直播解决方案。承载大规模带宽、支撑高并发，保证客户业务正常运作，达到 99.9%以上的可用性，整体资源储备及业务突发承接能力行业领先。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXIPibIkPv1weU8ompic9ib1LpibfTme7ZSU6W3BjPo6n3aNEE7YADrYrwMA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

随着全民抗疫，“停课不停学”的号召，在线教育也成为直播的主力军，直播的进房成功率/首帧延迟/卡顿率/音画同步时延/分辨率等指标直接影响用户核心体验。站在云的肩膀上，在线教育直播业务通过组合云上多种直播模式，结合业务流控系统，对各端直播接入进行多级流控及直播模式切换，在保证直播质量的前提下支撑远超互动直播极限的房间容量，下图是具体的直播架构。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXWrQ4hWM9sEWzqX81ymIQVTZXtywVkctW1l9I2snuLBZwE6LVu3m4Bw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 5.3 业务应用新技术的能力扩展

目前直播课普遍采用大班授课方式，老师在上课的时候，跟学生的互动有限，学生的注意力和参与感有限。大班教室人数太多，老师无法提供足量的 presentation 机会，学生与学生之间缺少有效的学习互动。

腾讯在线教育部推出如下图的六人小班课，基于 TRTC 在互动课堂场景下，为学员提供了稳定优质的服务，延迟低至原来的 1/10，互动效果得到很大提升。六人小班课给用户带来更多“被关注”的感觉，相比于大班课，家长的价值感知更高。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXVGTCoEVtRtHMPuKZqzCRKyPdyn5YyfzQglQh2zm0N5nIaKjWCkic6ibg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 6.**接入网关**

### 6.1 网关发展历程

接入网关有四大职能

- API 入口：作为所有 API 接口服务请求的接入点，负责请求的转发。
- 业务聚合：网关封装了系统内部架构，为每个客户端提供一个定制的 API。作为所有后端业务服务的聚合点，所有的业务服务都可以在这里被调用。
- 中介策略：身份验证、路由、过滤、流控、缓存、监控、负载均衡、请求分片与管理、静态响应处理等策略，进行一些必要的中介处理。
- 统一管理：提供统一的管理界面，提供配置管理工具，对所有 API 服务的调用生命周期和相应的中介策略进行统一管理

开源网关发展迅速。从 nginx 横空出世，到 openresty 解放程序员，更加专注解决业务需求，再到 kong 成为 api 网关的独角兽，以及最近出现不久的 apisix，当然也不能少了大名鼎鼎的 envoy。下面介绍主要的几个网关。

#### 6.1.1 **nginx**

2004 年 10 月 4 日发布的第一个公开版本以来，nginx 已成为高性能 web 服务器、反向代理服务器的代名词。相比 Apache，Nginx 使用更少的资源，支持更多的并发连接，能够支持高达 50,000 个并发连接数的响应。模块化和将一个请求分为多个阶段的设计，方便开发人员扩展。

#### 6.1.2 **openresty**

OpenResty® 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。python、js、lua 三种语言中，lua 是解析器最小、性能最高的语言，而 LuaJIT 比 lua 又快数 10 倍，开发人员和系统工程师可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块，快速构造出足以胜任 10K 乃至 1000K 以上单机并发连接的高性能 Web 应用系统。

#### **6.1.3 kong**

kong 是 API 管理的强大效率工具，主要有 4 个特点

- 扩展性：通过增添更多的服务器实例达到横向扩展
- 灵活性：可以部署在单个或多个数据中心环境的私有云或公有云上。支持大多数流行的操作系统，比如 Linux、Mac 和 Windows。包括许多实用技巧，以便针对大多数现代平台完成安装和配置工作
- 模块性：可以与新的插件协同运行，扩展基本功能。可将 API 与许多不同的插件整合起来，以增强安全、分析、验证、日志及/或监测机制
- 生态：开源免费使用，同时也能获得企业版，此外还提供初始安装、从第三方 API 管理工具来迁移、紧急补丁、热修复程序及更多特性。

### 6.2 在线教育网关实践

#### 6.2.1 **在线教育网关发展过程中的包袱**

- 通道 proxy 多语言, 多框架, 多协议功能无法服用，维护成本高

- 配置动态加载能力和插件能力不统一

- 一个接口上线配置多次，验证多次。

- 缺乏完善的监控

- 频控，容灾、熔断、下载等能力缺失

- 非云原生应用，不支持自动扩缩容

  #### 6.2.2 **tiny 网关**

tiny 网关主要的能力如下：

- 提供 app 端, web, pc 端快速接入，统一 sdk 和协议
- 支持智能路由，支持按照 cmd, uid, roomid, cid 字段的路由
- 全房间和多维度组合推送策略
- 可靠 push 保障
- 业务级别的监控告警
- 命令字配置集中管理，支持热加载
- 支持插件化的能力，方便添加业务特性的插件
- 全面落地容器化，支持自动扩缩容

完整的架构如下图所示

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXH63yr5QxgMKGic33cVwxISM2U9GKqzSa2p4fsJpKIul73BNap8FuDvA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 7.**go 语言**

随着云原生在互联网行业的普及,golang 从众多语言中,脱颖而出,成为了云原生时代的新秀。越来越多的开源项目采用 golang 语言来实现。因此学习和掌握 golang 语言,越来越成为一种趋势。本篇文章,主要围绕 golang 语言的主要特性来展开讲解,希望对大家有帮助。

### 7.1 **语法简单**

golang 素以简单著称,总共 25 个保留字,相比 c++的 82 个,java 语言的 50 个,少的不能再少了。golang 官方也比较吝于新增命令字。常见的结构和判断,对于 golang 来说,就只有 if,for,switch 等非常简单的命令字。自带的 array,slice,map 等数据结构,基本可以 cover 大多数的使用场景。

### 7.2 **部署方便**

编译好的 golang 程序,是一个独立的二进制程序,只依赖操作系统的一些基础库,除此之外,没有任何其他外部依赖。有使用过 golang 语言开发开源软件的同学,应该感触。

### 7.3 **静态编译**

golang 是一门静态强类型的编译型语言,与 c++类似,golang 也是也有一个完整的编译链接过程,并且有严格的编译期的语法检查过程。配合 golang 强大的工具链,在编译期可以提前解决脚本语言运行时才能发现的诸多问题。

### 7.4 **垃圾回收**

一直以来,c++程序员饱受内存问题的困扰,常见的比如内存泄漏,溢出,double free 等问题层出不穷,并且定位起来费时费力。c++官方为了降低使用成本,也在 c++0x 之后,引入了智能指针来解决内存使用的问题。但是内存问题依然存在。golang 跟 java 语言一样,从语言层面提供了 GC 能力,自带的垃圾回收机制,有效解决了内存使用的诸多问题。但是垃圾回收并非完美无缺, 不合理的内存使用方式,依然会导致程序出现严重的 gc 问题，从而导致程序出现性能问题,因此也有一定的 trick 需要遵循。

### 7.5 **工具链支持**

除了 golang 语言自带的编译,安装,单元测试等工具之外,c++的调试神器 gdb 也能够使用。同时第三方提供的 delve 调试工具, 兼容性和易用性更好,同时还提供了远程调试的能力。原生自带的 perf 工具,配合第三方的 go-torch 工具,生成的火焰图,调试的时候非常方便, 让性能瓶颈能够一目了然。

### 7.6 **泛型**

golang 被人诟病的特性之一,就是不支持泛型。官方认为虽然泛型很赞，但会使语言设计复杂度提升，所以并没有把这个泛型支持作为紧急需要增加的特性，也许在不久的将来, 会引入这个特性。现阶段可以通过使用 Interface 作为中间层,起到抽象和适配的作用。一些第三方工具,比如 genny,通过模板化生成代码,也可以作为泛型的一种解决方案。

### 7.7 **错误处理**

golang 语言, 错误处理从语言层面得到了支持, 基于 Error 接口来实现,配合函数的多返回值机制,, ，一般情况下, 错误码也会作为函数的最后一个返回值。在 golang 中, 错误处理非常重要, 语言的设计和规范,也鼓励开发人员显示的检查错误。也正因为如此,golang 的错误处理，也被很多人所诟病，觉得不像其他高级语言,比如 java 的错误处理那么简洁。不过整体来说,golang 作者将错误码显性化,目的是为了让大家能够重视错误处理，所以应该说是各有特色。

### 7.8 **包管理**

golang 语言刚诞生的时候,并不支持版本管理。GOPATH 方式,也只能算是包管理基本的雏形。后来经过一系列的演变，社区先后支持了 dodep,glide 的工具。直到 2016 年,官方才提出采用外部依赖包的方式,引入了 vendor 机制。2017 的时候推出的 dep 工具,基本可以作为准官方的解决方案了。然而,直到 2019,go modules 的推出, golang 的包管理争论才最终尘埃落定。基于 go mod 的版本管理机制,基本上可以说是一统江湖。具体的 golang 包管理演进过程,如下图所示:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXBibicCoVng3ccSAUlpDzZq629gHyElRET6Rqia9OjN8zPR8Leu9640GQw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 7.9 **并发性能**

golang 从语言层面就支持了并发,大大简化了并发程序的编写。这也是 golang 广受大家欢迎的原因之一。goroutine 是 golang 并发设计的核心, 本质上讲就是协程,也叫做用户态线程,它比线程更易用,更高效,更轻便,占用内存更小,并对开发者屏蔽了底层的协程调度细节。提供的 go,select,channel 等关键字,易用性非常好。配合 golang 中提供的 sync 包,可以非常高效的实现并发控制能力。

### 7.10 **常见网站**

- golang 百科全书: https://awesome-go.com/
- golang developer roadmap: https://github.com/Alikhll/golang-developer-roadmap
- sql2go 工具: http://stming.cn/tool/sql2go.html
- toml2go 工具: https://xuri.me/toml-to-go/
- curl2go 工具: https://mholt.github.io/curl-to-go/
- json2go 工具: https://mholt.github.io/json-to-go/
- 泛型工具: https://github.com/cheekybits/genny

### 7.11 **生态**

Go 在未来企业会有更多布道：Go Conference 一直都是 Gopher 关注的技术大会，20 年 11 月国内的 Go Conferernce(https://github.com/gopherchina/conference)分享主题主要包括 Go 语言基础（Go 编程模式、Go Module、Go 编译器、组件包）和架构落地实践（微服务实践、微服务框架、云原生实践、大数据和高并发挑战），侧面也印证了 Go 语言在后端领域具备较强的业务实战落地能力，对打算采用 Go 的互联网企业，具有较强的指导和借鉴意义。

### **7.12 业界认可度**

golang 作为云原生的首选语言,在业界获得广泛的认可。基于 golang 的很多明星项目,包括 docker,k8s,etcd,influxdb,tidb,prometheus,kibana,nsq 等覆盖了容器化,容器编排,存储,监控,消息队列等各个场景, 在各大公司都获得了大量的应用。同时从 github 拉取数据查看语言流行程度, 我们对比了 java,c++,c,go 等语言发现,golang 在 github 开源库的使用上越来越流行。如下图所示的 golang 占比情况:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXayq6QuCFr42cUx5uBBPBm4IeDavOCGOC0FnpXR22xL97tM1u1iaNfYA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat3NmhGhH1ia9XZaVsLmOpBXo2M4ISUBRaHBg7ZNZO5lkrdbTicTrsFHRAiblfoWicmLEl05FkElcqEhA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 7.13 **趋势**

全面转到 Go Module：官方会终止对 GOPATH 的开发支持，全面转到 Go Module。

2021 年,golang 中的泛型还要持续打磨。

随着云原生浪潮，越来越多的企业将会考虑将 Go 作为其主要后端开发语言。

## 8.**总结**

2020 年已经过去，可以确定技术的发展是一分钟也不会停滞。可以预见云原生、微服务等新技术依旧是后台技术发展趋势，2021 年也会有更多创新出现。

原文作者：[腾讯技术工程](javascript:void(0);)

原文链接：https://mp.weixin.qq.com/s/EuqJ2Q9CAWSJ0rTc_p_gjg