# 【NO.238】为什么微信推荐这么快？

# **1. 背景** 

在一些推荐系统、图片检索、文章去重等场景中，对基于特征数据进行 k 近邻检索有着广泛的需求：

- 支持亿级索引的检索，同时要求非常高的检索性能；
- 支持索引的批量实时更新；
- 支持多模型、多版本以灵活开展 ABTest 实验；
- 支持过滤器、过期删除以排除不符合特定条件的数据。

在经过调研后，发现已有的解决方案存在以下问题：

- 在学术界中，已经存在有成熟并开源的 ANN 搜索库，然而这些搜索库仅仅是作为单机引擎存在，而不能作为**高性能、可依赖、可拓展的分布式组件为推荐系统提供服务**；
- 在业界中，大多数的组件都是基于 ANN 搜索库做一层简单的封装，在可拓展、高可用上的表现达不到在线系统的要求；而对于少数在实现上已经较为成熟的分布式检索系统，在**功能上却难以做到紧跟业务发展**；
- 而在更新机制上，很多组件都是要么只支持离线更新、要么只支持在线接口更新，无法满足在微信侧**小至秒级千数量、大至小时级亿数量的索引更新需求**，因此需要可以兼顾近实时更新及离线大批量更新的分布式系统。

基于上述的这些要求以及业内组件的限制，我们借助 WFS 和 Chubby 设计并实现了 SimSvr，它是一个高性能、功能丰富的特征检索组件，具有以下特点：

- 分布式可伸缩的架构，支持亿级以上的索引量，以及索引的并发加速查询，实现了 **10ms 以内检索数亿**的索引；
- 高性能召回引擎，使用了召回性能极佳的 hnswlib 作为首选召回引擎，大部分请求可在 **2ms 内**完成检索；
- 集群化管理，集成了完善的数据调度及动态路由功能；
- 多样的更新机制，支持任务式更新及自动更新，同时也支持全量更新与增量更新，跨越**秒级千数量**到**小时级亿数量**的索引更新；
- 读写分离的机制，在离线利用庞大的计算资源加速构建索引的同时，不影响在线服务的高性能读；
- 丰富的功能特性，支持轻量 embedding kv 库、单表多索引、多版本索引、过滤器、过期删除等特性。

SimSvr 目前已广泛应用于微信视频号、看一看、搜一搜、微信安全、表情搜索等业务，接下来会阐述 SimSvr 的设计以及如何解决来自于业务的难题。

# **2. 检索引擎**

## **2.1 引擎的选择** 

ANN 问题在学术界已被长期研究，并且已有成熟的开源 ANN 搜索库存在，如 nmslib、hnswlib、faiss 等。在 SimSvr 中，**性能及集群的存储容量**是最主要考量的两个指标，因此选择了以下两个检索引擎：

- 在 ann-benchmarks 中检索性能最好的 hnswlib，能够满足在线服务对召回率及检索耗时的高要求（**大于 90% 召回率的情况下，能在 1ms 内完成召回**）；
- faiss 的 IVFx_HNSWy + PQz 算法，支持将向量压缩 10 ~ 30 倍，能够满足资源有限情况下的高维大数据量的索引要求（亿级索引数据，容纳在内存 64G 的机器上）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPfIMXIWjiaHADSibibe5WQh6OnUhACial2GTm19tl7d5picdDNWicAly3KBjw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**ANN检索引擎效果对比**

## **2.2 巧妙利用资源，提升 50% 的数据容纳量**

- hnswlib 是单机检索引擎，在资源使用方面仅考虑了单模型的情况；而 SimSvr 是提供在线服务的组件，一般容纳了多个模型；
- SimSvr 在大部分场景下，拥有读写分离的特点；
  基于以上特点，我们在引入 hnswlib 之后，进行了资源整合，使得 SimSvr 单机情况下可以容纳更多的模型索引：

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPEq9OlibAtIAgtLJGUXCUAW5SjVIe3ib720dnopkAAfbw2tTuJ8UewZuQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- 极限情况下（以 worker 线程数 80、部署 10 张 2kw 索引量的表为例）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPU8JbgtExpKanSok79kCaHycxjU0DGicqgCSCQof1tAGIchibH47HN9sQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- 现网运营中（以某现网模块(11台实例机器，worker 线程 240）为例）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPLiaDcfTLTJ8Cu5BIwSOxJiaxKERq6hmCz45OlmNwhicvnVbORjRDbYUJQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## **2.3 点积距离召回率从 62.6% 到 97.8% 的蜕变心路历程**

- HNSW 算法在**余弦距离**表现优秀，但在**点乘距离**的数据集上存在效果差的情况；

- 点乘距离非度量空间（metric space)，**不满足三角不等式**，距离比较没有传递性；

- - 维基百科中关于度量空间的定义:

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPibLSU08eUZWqxTWF87p4cW92bJb2eRjr5ib8iaibI6C0z4Rl4gIE1pL7nQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- - hnswlib 中说明点积属于非度量空间：

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXP3lFdss02FsuCuVOh9hsia9hiazHctIpZ4WheJEQzWqShwY2xbCEF6egQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- 而在论文 Non-metric Similarity Graphs for
  Maximum Inner Product Search 中，提到了将**点乘距离转换为余弦距离**计算的方法，我们将这种方法简称为 ip2cos；

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPnSkpuUKHAjv8rBNSI67TK02EDUqRKHickER52tjSlp3LCY7dlUVB83g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



在 ip2cos 距离转换的理论基础上，我们使用看一看视频实时 DSSM 模型进行了实际召回情况的效果对比（64 维、ip 距离、100 万索引数据量，进行 1 万次查询取平均耗时），并见证了 ip2cos 的神奇效果：

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPf8pCbhiaqghjMiapvzrrXEmfNo4u6CiaEWIV1LwzJOpu9q4z3cdzrm5wQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## **2.4 如何使用 faiss 省下 2h 的训练时间并提升 30% 的召回率**

- 在 faiss 中增加了 batch kmeans 聚类方法，在保证较好聚类效果的同时大幅加快训练速度。IVF 系类方法训练耗时主要体现在需要从数据中学习 nlist 个聚类中心，对于千万级数据 nlist 的大小在 20 万以上，在 cpu 上使用传统 kmeans 方法训练会非常耗时，下面展示在 128 维、IP 距离、1000 万条数据的情况下 batch kmeans 对训练速度的加速效果：

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPVRbRjywVzZFNzLKSHqUZIX4RBmicES12aqw9JwFmX9MlCENYiaXywhVA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从结果中可以看到，在相同迭代轮次下，不使用 batch kmeans 的方法训练耗时更长，且没有很好收敛，导致召回率不高。

# **3. 总体设计**

## **3.1 数据结构 - 为达成一个小目标，需要做出怎样的改变**

为了满足单模块多模型的需求，SimSvr 使用了表的概念进行**多模型的管理**；另外，为**支持亿级以上 HNSW 索引的表**，并且希望能够并发加速构建索引，我们根据单表的数据情况，将一张表分成了多个 sharding，使得每个 sharding 承担表数据的其中一部分：
tablei 的索引，由 shard0、shard1、…、shardn 构成一份完整的索引数据；而 sect 的数量则决定了表的副本数（可用于伸缩读能力、提供容灾等）。
在 SimSvr 中，我们将一个 shardi_sectj 称之为一个 container，这是 SimSvr 中最小的数据调度和加载单位。

## **3.2 系统架构 - 如何支撑亿级索引、5毫秒级的检索**

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPxJwfp1YcgukFUa2iapNJ8ria4LrdmXJZQMwSvxJksOtNlmWy0D9I1LaA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**SimSvr 架构**

- **SimSvr 与 FeatureKV 一样，涉及的外部依赖也是三个：**

- - Chubby：用来保存元数据、路由信息、worker 资源信息等；SimSvr 中的数据协同、分布式任务执行均是依赖于 Chubby；
  - USER_FS：业务侧存放原始数据的分布式文件系统，可以是 WFS/HDFS，该文件系统的路径及信息保存在表/任务的元信息中；
  - SimSvr_FS：Simsvr 使用的分布式文件系统，用于存放生成的索引文件或者原始的增量数据文件。

- **worker**

- - 负责对外提供检索服务，通过对 Chubby 的轮询检查索引的更新，进而将索引加载至本机以提供服务；
  - 每台 worker 负责的数据，由 master 进行调度，worker 根据 master 保存在 Chubby 上的分配信息进行数据的加载/卸载；
  - worker 的数据是根据 master 分配得来的，除此之外没有其他状态的差别，因此 worker 是易于扩缩容的。

- **master**

- - 数据调度：通过表的元信息及 worker 状态，将未分配的数据或者失效 worker 上的数据调度给其他有效的 worker；
  - 生成路由表：根据 worker 的数据加载情况及状态，生成集群的路由表；
  - 感知数据更新：检查表的自动更新目录，若最大数字目录发生了增长，则建一个任务以供 trainer 进行索引的构建；
  - master 是一个无状态的服务，通过 Chubby 提供的分布式锁保证数据调度以及路由生成的唯一执行。

- **trainer**

- - 负责构建表的索引及资源回收；
  - trainer 单次可构建一张表中一个 sharding 的索引，因此如果表有多个 sharding 时，可通过增加 trainer 的个数实现构建索引的并发加速；
  - trainer 是无状态的服务，通常部署在微信 Yard 系统上，充分了利用微信闲置机器上的资源。

- **数据自动更新**

- - 在建表时，对其指定了一个 fs 的目录，该目录下，是一系列数字递增的目录；
  - 当业务侧需要更新索引时，将最新的数据 dump 到更大的数字目录中；
  - master 感知最大数字目录的更新，从而更新了元信息；
  - trainer 感知元信息的更新并触发建索引；
  - worker 加载索引完成索引的更新。

- **数据任务式更新**

- - 由业务侧主动通过接口的调用，创建一个索引任务；
  - 在索引任务中，指定了数据的配置信息（如 fs 信息及路径等）；
  - trainer 按照表的任务序列，执行任务并构建索引；
  - worker 加载索引完成索引的更新。

## **3.3 数据调度 - 鸡蛋怎么放在多个篮子中**

- SimSvr 在每张表创建时就指定了 sharding 数 n 及 sect 数 m，因此这张表拥有了 n * m 个 Conatiner 以供 master 调度；
- master 会根据 worker 的健康情况及资源使用情况进行数据的调度及路由表的生成；
- 路由表带有递增的版本号，可根据版本号感知路由的变化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPKZenkTQO3kSicgMZMKeClGIiaOuFvRHibFpGC0ER2YB2aY84lWFWmL9TA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- worker 定期轮询 Chubby 获取数据的调度情况及最新的路由表信息；
- client 首次请求时，将随机请求一台 worker 获取最新的路由表信息并将其缓存在本地；
- client 在本地有路由表的情况下，将根据表的数据分布情况，带上版本号并发地向目标 worker 发起请求，最终合并所有 sharding 的结果，将其返回给业务端。

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXP99eKIOibMNLnfrkrmAz9CVa331xCSzumOtAxzlXic8nK5X5QU8uibLh6w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## **3.4 系统拓展 - 篮子装满了该怎么办**

- SimSvr 将表拆分成了更小粒度的数据调度单位，且不要求每台机器上的数据一样，因此可以用拓展机器的方式，将集群的存储容量扩大；
- 对于单表而言，当读能力达到瓶颈时，可以单独扩展此表的读副本数；

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXP8qMZ0g2zAqIJZhRFqfRt8maJIjibmk9eElVHgiaGu47N5BI3su4OAHibw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

# **4. 近实时增量更新的挑战 - 十秒内完成索引的更新**

- 数据一致性与持久化

- - 对于大多数的分布式存储组件来说，都是使用 raft 或者 paxos 等一致性协议保证数据一致性并持久化至本机上；
  - 对于 SimSvr 来说，每张表会被分为多个 sharding，且 sharding 数不保证为奇数；
  - 在 worker 中加入一致性组件及额外的存储引擎，会使得整体的结构变得复杂；
  - 最终在考量后，结合业务的批量增量更新的特点，选择了先将数据落地 fs，再由 worker 拉取数据加载的方案；在这种方案下，1000 以内数量的 key 插入，能够在 10s 内完成，达到了业务的要求。

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPKReraaeenzKkowEe8ukjGsRxSpmw3lUGaOFvcmic5XgFqokjicQ5Q0uA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**增量持久化**

- 增量更新的性能保障

- - 由于在线建索引是非常消耗 cpu 资源的过程，因此为了不影响现网的读服务，worker 仅提供少量的 cpu 资源用于增量数据的更新；
  - 对于小批量的增量数据，worker 可以直接加载存放在 fs 上的数据并直接进行索引的在线插入；
  - 对于大批量的增量数据，为了避免影响读服务及大增量更新慢的问题，SimSvr 将大批量数据在 trainer 进行合并且并发重建索引，最后再由 worker 直接加载建好的索引。

![图片](https://mmbiz.qpic.cn/mmbiz_png/0Az9wtrYXDVvJshxncI3RWvQiaMbtdUXPA6ADk29yN4lAf9Lh2SKOGYr3un98vNbbrqU0g8nibZW51iaI5NsKpn2A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**增量更新**

# **5. 丰富的功能特性**

## **5.1 支持额外的特征存储库**

- 在推荐系统中，同一个模型，产生的数据除了用于检索的索引库，常常还有视频特征/用户画像的特征数据；
- 这类数据，仅仅只需要查询的功能，并且与同个模型同个版本产出的索引库相互作用，产生正确的召回效果；
- 基于这种原子性更新的特性，SimSvr 支持了额外的特征存储库，用于存储与模型一同更新且仅用于查询的特征数据，帮助业务省去了数据同步与对齐的烦恼。

## **5.2 支持原子性更新的单表多索引**

- 在推荐系统中，ABTest 是非常常见的，多个模型的实验往往也是需要同时进行的；
- 另外，在某些场景下，同一个模型会产生不同的索引数据，在线上使用时要求同模型的索引要同时生效；
- 对于以上两种情况，如果使用多表支持多模型，在索引更新上存在生效时间的差异从而无法支持；
- SimSvr 对于这种情况，支持了同一张表多份索引的原子性更新，保证了索引能够同时生效。

## **5.3 多版本索引**

- 在 ABTest 场景下，除了有多模型间的实验，还有相同模型不同版本数据的实验；
- 在相同模型中，版本迭代/不同版本进行实验的场景是广泛存在的；
- 如果使用多表支持这样的多版本索引，不管在业务方的使用上，还是在 SimSvr 的管理上，都显得不是那么地优雅；
- 对此，SimSvr 支持了同一张表的多版本管理，并且多版本支持在现网下同时进行服务，业务可以按需请求目标版本，进行灵活的实验。

## **5.4 支持布隆过滤器、阈值过滤器等**

- 在视频号场景中，业务使用 SimSvr 对视频进行索引；
- 在使用某个用户的特征进行召回时，常常召回了许多用户已看过的视频，影响用户体验；
- 通过增加召回结果并在结果中进行过滤，对于重度用户，一样存在上述问题，并且还会导致不必要的性能开销；
- SimSvr 改造 hnswlib，嵌入了过滤器的逻辑，使得其支持在检索过程中实时对符合特定条件的 key 进行过滤，保证了召回结果的有效性。

## **5.5 支持过期删除**

- 对于一些推荐系统来说，对于数据的时效性要求是非常高的，在数据过了其最佳召回时间段之后，就不应该出现在召回结果中，以免出现不合时宜的尴尬；
- SimSvr 支持导入带过期时间的数据，在现网召回过程中，实时淘汰过期的 key 以达到准确的召回要求。

# **6. 现网运营情况**

- SimSvr 目前已部署 160+ 个模型索引，使用逻辑核 8000+，总索引量超过 20 亿特征向量，广泛应用于视频号、看一看、搜一搜等推荐业务中。
- 搜一搜基于 SimSvr 建立小程序优质文章的向量索引，提升小程序文章搜索的优质结果召回率。新方案相比旧方案，优质结果召回率提升 7%；
- 搜一搜使用 SimSvr 检索视频指纹，进行相似视频去重；单表索引量高达 1.7 亿 * 128 维，检索平均耗时小于 8ms，日检索量 12.5 亿。

# **7. 总结**

随着推荐系统的强势发展，特征检索的使用场景越来越广泛。而作为基础组件，除了要拥有支持亿级索引的基本素养外，在功能特性上也需要不断迎合业务的发展。因此我们开发了 SimSvr，搭配特征存储 FeatureKV，在视频号、看一看、搜一搜等推荐系统中发挥了重要的作用。

原文作者：sauronzhang、flashlin、fengshanliu，微信后台开发工程师