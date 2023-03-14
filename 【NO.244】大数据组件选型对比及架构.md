# 【NO.244】大数据组件选型对比及架构

## 1.MQ架构设计及选型对比

### 1.1 **RocketMQ vs Kafka vs Pulsar**

RocketMQ: **后台业务开发、高性能及高可靠场景**，如阿里双十一电商业务，阿里开源,现在为apache rocketmq；queue模式，支持dead letter queue可延迟投递,pull +push皆可支持。可靠同步+可靠异步传输

Kafka: 分布式日志流传输系统，更多用于大数据领域，顺序磁盘写入、zero-copy等特大幅提升kafka性能，特别适合**海量数据传输**。计算和存储耦合；streaming模式。仅支持pull模式。

**Pulsar**： **Kafka+RocketMQ**，支持streaming+queue模式，既可以用于后台业务开发，如腾讯计费系统-米大师，又可以使用到大数据领域用于数据传递; 支持pull+push模式;支持**多租户管理、跨地域复制；存算分离;支持dead-letter-queue，实现延迟发送**



### 1.2 **Queue vs Topic**

![img](https://pic3.zhimg.com/80/v2-2dd2ecb8eb5ebf19032b633d641a01a2_720w.webp)

![img](https://pic1.zhimg.com/80/v2-0692c8eb2f9013965528b0ba0a3c6aa8_720w.webp)

### 1.3 **RocketMQ架构图**

![img](https://pic3.zhimg.com/80/v2-d57e6b0ca968f8e94733845df69323ba_720w.webp)

发送方式及可靠性：

![img](https://pic1.zhimg.com/80/v2-ae6fe49a95516520d103226feeb8914c_720w.webp)



### 1.4 **Kafka架构图**

![img](https://pic4.zhimg.com/80/v2-7117f7aaaf1e9df4b92c59d8c013685b_720w.webp)

![img](https://pic3.zhimg.com/80/v2-61c735fb5dd2e08d30c7fcbe8f71fa8e_720w.webp)



### 1.5 **Pulsar架构**

![img](https://pic1.zhimg.com/80/v2-3cfc7145f08d3336e96b5d1bf162d228_720w.webp)

**Pulsar消费方式:Streaming + Queuing**

![img](https://pic2.zhimg.com/80/v2-6b9310cddc8983e6eaac9d88fdb5a235_720w.webp)

## 2.**KV架构设计及选型对比**

KV存储使用场景高性能缓存，配合数据库完成海量服务的后台设计。具体的KV有多种，具体有如下几种：

**mongodb**: 可以存储文档doc的kv数据库，在博客系统使用很广泛

**redis**：延迟极低（5ms），经常用于后台开发的缓存系统，基于内存数据存储，通过持久化化策略AOF、RDB 将数据固化到磁盘；断电有丢失数据风险。常用的结构有hash、map、list、sortedset

**hbase**: 延迟还可以（500ms），适合海量数据的存储，如日增量200G数据的场景;一般来说hbase运维比较复杂。

**rocksdb**：高性能KV存储，有广泛的用途，一般以基础建设方式存在，如tidb底层基于rocksdb存储、flink的状态存储也采用rocksdb、阿里高性能的TairDB、美团的Cellar的KV存储等都是基于rocksdb修改和完善



### 2.1 **mongodb集群**

![img](https://pic3.zhimg.com/80/v2-509e446a63e75d69c141297062d070de_720w.webp)

### 2.2 **redis集群**

**redis sentinel 哨兵模式**

![img](https://pic3.zhimg.com/80/v2-51f52f749927dc21b60f084d2e68d922_720w.webp)



**redis codis集群模式（豌豆荚开放，刘奇负责开发，后续创建了pingcap公司）**

![img](https://pic2.zhimg.com/80/v2-fabd4d53145203e741457cfeb05ba1e9_720w.webp)



**redis 3.0 官方支持，最多不能超过1000个节点**

![img](https://pic4.zhimg.com/80/v2-cea754ba02ebbd48047fdd2a08f6cafb_720w.webp)



### 2.3 **Hbase集群**

![img](https://pic1.zhimg.com/80/v2-0b7946e0e49b7b87c8cd27d98fbac710_720w.webp)

## 3.**OLAP架构设计及选型对比**

海量数据自由聚合和分析，目前分为两个派系，一类是预聚合好，提供服务的；另外一类是MPP数据库，按需执行计算和聚合。

**预聚合类：**

kylin: 离线T+1的报表分析，将数仓的数据导出到kylin，做分析和使用

**apahe druid**: 偏向于实时的olap多维分析，做实时报表和风控；一般可做实时+离线报表；而且维度支持上万列

**MPP数据库：**

**apache doris**：MPP数据库，集存储和计算于一体，多表join性能不错。可以做实时+离线的MPP数据库，支持高并发的写入和查询；可以做线性扩容。最初百度开源，商业化公司starrocks

clickhouse：MPP数据库，百亿数据做group by统计秒级返回。俄罗斯公司yandex开放使用。

**MPP计算引擎：**

**presto**/impala: 不存储数据，直接以MPP的方式查询存储到Hadoop里面的数据，presto 使用java编写，impala使用c++编写。美团、哈啰等使用presto查询HDFS数据，要比hive自带的mr更快速。



**Apache druid架构：**

按节点角色分类架构图：

![img](https://pic4.zhimg.com/80/v2-f2a81ab8f7d55a87ebc05007273a724f_720w.webp)

数据摄入及服务提供：

![img](https://pic2.zhimg.com/80/v2-9e07441032d223165994eddf371a2589_720w.webp)

**MPP架构：**

大规模并行处理

![img](https://pic4.zhimg.com/80/v2-707343adf98e2cffbd88749c73af969b_720w.webp)









**Doris架构：**

整体架构：

![img](https://pic4.zhimg.com/80/v2-424a6c0abf7e37d3eb18806a4783ea23_720w.webp)

doris上下游生态

![img](https://pic2.zhimg.com/80/v2-7be7c372b2d09c2490f46b3fe3da03a1_720w.webp)

**presto架构：**

![img](https://pic3.zhimg.com/80/v2-6f156ae3db3a8f015a064807605f1f4e_720w.webp)



1. **流式计算框架（flink、kafka streaming、apache nifi、apache beam）对比**

**flink**：分布式低延时、海量数据实时处理框架，生态十分完善，如cep、图计算、flink ml等，是实时计算的标杆

kafka streaming：是一个lib,可以嵌入到程序里面进行实时数据流的处理，不适合处理亿级甚至千万级的数据

apache nifi：很好支持了**物联网MQTT协议**，可以做物联网边缘端数据的清洗和处理，有图像化操作界面，方便易用，可将数据处理完成后传递到大数据中心如flink集群进行最终处理

apache beam:是一个不强绑定的大数据实时计算、离线计算的SDK，它的代码可以打包部署到flink集群、spark集群、google data cloud platform集群等等，它指定的大数据计算的api，底层运行可以兼容多种框架。



### 3.1 **flink架构：**

![img](https://pic1.zhimg.com/80/v2-4e395a64889c14d720417f36c4421bd4_720w.webp)

### 3.2 **Kafka Stream**

后台程序配合kafka stream lib的程序应用架构

![img](https://pic2.zhimg.com/80/v2-52ef65b936af3bec3c1ebce2528a20a9_720w.webp)

后台程序分布式处理：

![img](https://pic3.zhimg.com/80/v2-061c024754276cadbf00c371287ca292_720w.webp)



### 3.3 **Apache NIFI**

架构图：

![img](https://pic4.zhimg.com/80/v2-3fa8a3ba474ec06de28d4b96c80ef20b_720w.webp)

界面化pipeline操作

![img](https://pic3.zhimg.com/80/v2-f5902016cfa04d5ce9abde4230b37b7e_720w.webp)



### 3.4 **Apache Beam:**

![img](https://pic1.zhimg.com/80/v2-e86d06ab87adb0e3dab2dbd42ec75c90_720w.webp)



编程语言、批流模式、编程模型、运行Runner

![img](https://pic3.zhimg.com/80/v2-e486cb5a6129b7704fb7718df55e2ffe_720w.webp)



## 4.**数据湖选型对比（deltalake、hudi、iceberg）**

**deltalake**：spark后面商业公司databricks开源，跟spark强耦合，是比较早提出数据湖概念，目前市场热度不高，很大一部分公司实时计算使用flink，所以集成度不高。

**iceberg**: 较为标准的数据湖公司，由uber公司开源，目前有腾讯、阿里、网易、去哪儿网内部使用iceberg构建数据湖；

**hudi**; uber公司开源的数据湖解决方案，目前使用比较广泛；在googgle云、微软云、IBM云阿里云、腾讯云EMR、亚马逊云EMR等大数据套件提供了对hudi的集成支持，目前像T3出行；hudi背后成立商业公司OneHouse，整体来说，hudi有商业公司加持，发展更好



### 4.1 **hudi 上下游生态：**

![img](https://pic3.zhimg.com/80/v2-cd4be6e970f96604e5841e2dca5c1836_720w.webp)



### 4.2 **Flink+iceberg构建实时数据湖：**

![img](https://pic2.zhimg.com/80/v2-0fcd9926d40cc8df6758104a54256449_720w.webp)

### 4.3 **HTAP选型及对比（Google Spanner+F1、TIDB、CocksRoachDB）**

TP和AP数据库很难兼容统一完成，只能不断组合提供更完备的混合型TP+AP的数据库；TiDB和CocksRoachDB都是模仿google的论文Spanner、F1论文完成的。目前TIDB在国内发展很快，tidb很好打造成功了金融级解决方案。cocksroachdb在国外发展不错，主打跨洲际数据复制。

无论tidb还是cockroachdb，从开发之处就支持k8s云原生部署。

Spanner 是Google的全球级的分布式数据库 (Globally-Distributed Database) 。Spanner的扩展性达到了令人咋舌的全球级，可以扩展到数百万的机器，数已百计的数据中心，上万亿的行。

### 4.4 **Google Spanner +F1架构**



![img](https://pic4.zhimg.com/80/v2-eecf91d4257198e137470ab3b1ef16b3_720w.webp)

F1+Spanner分布式架构全景图：

![img](https://pic3.zhimg.com/80/v2-b2f68deaa31a64bbec5e9cf4149c5f96_720w.webp)



### 4.5 **TiDB架构**

tidb整体架构

![img](https://pic4.zhimg.com/80/v2-be496de2529401ee1fb9f81cf9ca174f_720w.webp)

tidb结合后台程序的整体架构

![img](https://pic2.zhimg.com/80/v2-bad58abc1df848bf23ce3d819d7969d5_720w.webp)

### 4.6 **CockRoachdb 架构：**

整体架构：

![img](https://pic2.zhimg.com/80/v2-4a4c6ce0e333a65e26130d5bd6aea579_720w.webp)



cockroachdb跨洲际数据同步复制：

![img](https://pic4.zhimg.com/80/v2-b66d6db41304a783046b4560f004caf7_720w.webp)

原文作者：鹅厂架构师

原文链接：https://zhuanlan.zhihu.com/p/510249281