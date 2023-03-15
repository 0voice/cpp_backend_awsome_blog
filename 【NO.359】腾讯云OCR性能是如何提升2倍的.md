# 【NO.359】腾讯云OCR性能是如何提升2倍的

> 腾讯云 OCR 团队近期进行了耗时优化，通用 OCR 优化前平均耗时 1815ms，优化后平均耗时 824ms，提升 2.2 倍。本文旨在让大家了解 OCR 团队在耗时优化中的思路和方法(如工程优化、模型优化、TIACC 加速)，希望能给大家在工作中提供一些新的思路。

### 1.背景介绍

#### **1. 业务背景**

近期某重要客户反馈，受当前正在使用的 OCR 服务可用性(非腾讯云)的影响，业务不可用长达半个小时，而且这样的情况时有发生。为了更好的服务，客户开始调研，主要是从服务可用性，准确率和服务耗时三个方面评估。

通过详细体验和测试，发现腾讯云 OCR 在服务可用性和准确率方面都表现非常不错，但客户对腾讯云 OCR 耗时不满意，希望我们进行耗时优化，在达到要求之后再考虑接入。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1VSxIyl0tba5DOVQwfxdsVoyFebM9ibKpk6wcXVDMw9AJBS8C9vJN9ew/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

除了该客户的反馈，之前也陆陆续续收到其他客户类似的反馈。**因此，优化腾讯云 OCR 服务耗时，提供更快的文字识别服务势在必行。**

#### **2. 问题挑战**

耗时优化是一个系统性工程，需要多方的支持和协作，文字识别服务进行耗时优化，主要有以下挑战：

● 环节多：耗时优化涉及多个环节，包括模型算法、TI-ACC、工程等，多环节都需要分析各自阶段耗时，制定完整的耗时优化目标。

● 时间短：客户耗时优化诉求强烈，但是客户的耐心有限，留给我们优化的时间很短。

● 成本考量：降本增效大背景下，单纯依赖机器的情况一去不复返，需要做成本优化。

我们也成立了专项团队进行攻坚，从工程优化，模型优化，TI-ACC 优化等方面发力，逐步降低服务耗时。优化前平均耗时 1815ms，优化后平均耗时 824ms，耗时性能提升 2.2 倍，并最终得到重要客户的肯定。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf18bU2iaQjZogfiaGRyYcfCVuNJ9gnqukicv5aVkHJ9bicT2OkFkaPB1RvFg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

接下来将介绍我们在耗时优化方面的具体实践。



### 2.分析问题

#### **2.1 框架和链路分析**

OCR 服务通过云 API 接入，内部多个微服务间通过 TRPC 进行调用，基本架构图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1kemibl3YYueq2mznJ8qJ4aNhyapNs77nhibkYicCGaskQgGv21lwlQoZw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从客户发起请求到 OCR 服务处理完成主要链路为：

1. 云 API 层：请求首先会传输到离客户最近的云 API 接入点，云 API 接入点会进行相应的鉴权、寻址、转发等操作
2. 业务逻辑层：文字识别逻辑层服务会对数据做处理、下载、计费、上报等操作
3. 引擎层：算法引擎服务对图片进行处理，识别出文字

#### **2.2 主要阶段耗时**

主要涉及到下面几个阶段的耗时：

● 客户传输耗时：客户请求到云 API 和云 API 响应到客户链路的传输耗时在测试过程中发现波动很大。文字识别业务场景下请求传输的都是图片数据，受客户网络带宽和质量的影响大。而且客户请求的文字识别服务在广州，请求需要跨地域，进一步增加了传输耗时。

● 业务逻辑耗时：业务逻辑中有很多复杂的工程处理，主要耗时点包括数据处理、编解码、图片下载、路由、上报等。

● 算法引擎耗时：为了达到更好的识别效果和泛化性，通用 OCR 模型会比较复杂，其中检测和识别都会涉及到复杂的浮点计算，耗时长。

因此，要优化文字识别服务的整体耗时，就需要从各个阶段进行详细分析和优化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1P2tDISYqPcFAyueicBubcbJMrOvL7acoqib9Z5J5RhtNFUXWfffPFbLw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### **2.3 测试方法和结论**

| 测试机器 | 测试机器地域     | 请求地域参数                          | OCR 部署地域 | 图片大小  | 图片编码 |
| :------- | :--------------- | :------------------------------------ | :----------- | :-------- | :------- |
| 云服务器 | 北京、上海、广州 | ap-beijing、ap-shanghai、ap-guangzhou | 广州         | 200KB~3MB | Base64   |

客户生产环境的机器在云服务器上，为了保持和客户测试条件一致，我们购买了和客户相同环境的云服务器(含北京、上海、广州)进行模拟测试，详细分析文字识别服务各个阶段耗时。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1MZyarNI6mH8H7m82PLX2ryMZmY3UV4ibxTjWjcIloxNVLW2LJ4jzC4g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

端到端耗时为客户请求发起直到收到响应的总耗时，客户到腾讯云 API 接入点，以及云 API 响应给客户存在着公网传输的耗时，从云 API 接入后就是腾讯云内网链路服务的耗时。端到端耗时主要由客户传输耗时、云 API 耗时、业务逻辑层耗时、引擎耗时组成，经过多次请求通用 OCR 后统计各个阶段的平均耗时，具体情况为：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1W2668I5LbAdPD6njCJrUia4tVhmicR6JianwVcICMAiaBRL1JCfXTwQ0kw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

1. 客户的传输耗时占比很大，主要是因为客户在北京发起请求，需要跨地域到广州的服务，存在很大的传输耗时，这部分需要我们通过优化部署来减低耗时。
2. 在服务链路中，业务逻辑处理的耗时也较多，这部分耗时需要我们精益求精，优化工程逻辑和架构，进而降低耗时。
3. 文字识别服务的引擎耗时占比最高，我们需要对算法引擎的耗时进行重点优化。

通过详细的各阶段耗时测试可以发现，引擎耗时占主要部分，所以会重点优化引擎耗时，主要手段是模型优化和 TI-ACC 加速。为了做到极致优化，我们还通过日志分流和编解码优化降低了业务逻辑层耗时，同时服务就近多地部署，优化了客户传输耗时。



### 3.解决问题

#### **3.1 模型优化—引入 TSA 算法减少模型耗时**

优化之前耗时长度会随着文字字数增加显著变大，引入 TSA 算法可以明显改善这种情况，减少模型耗时

##### 3.1.1 OCR 特点与算法过程分析

基于 OCR 的特点与难点，针对 OCR 问题学术界算法可以划分成两大类：

1. 基于 CTC (Connectionist Temporal Classification)的流式文本识别方案
2. 基于 Attention 的机器翻译的框架方案。

在基于深度学习的 OCR 识别算法中，整个流程可以归纳成了四个步骤如下图: ① 几何变换 ② 特征提取 ③ 序列建模 ④ 对齐与输出。CTC 方案与 Attention 方案区别主要是在步骤 ④，它作为衔接视觉特征与语义特征的关键桥梁，可以根据上下文图像特征和语义特征做精确输入、输出的对齐，是 OCR 模型关键的过程。

针对上述特点与耗时问题，我们提出了 TSA 文本识别模型。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1DficWibXUwplOZ4AILZOhbLicr0SMbJgnfjLicuEkjY3wtSUFBFicC9NG3A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

文本识别的主要流程

##### 3.1.2 TSA 研发背景

① 研究表明：图像对 CNN 的依赖是非必需的，当直接应用于图像块序列时，transformer 也能很好地执行图像分类任务 。

② 63%耗时在(BiLSTM+Global Attn，或 attention 形式)序列建模!

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1h1IvSRJwIn8zouD6nHkt4sCbWOyPlsfmalzloMqFf9AanNQDSUy5nw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

OCR 识别模型耗时统计分析

##### 3.1.3 识别模型特点

特点 1: 引入 self-attention 算法，对输入图像进行像素级别建模，并替代 RNN 完成序列建模。相比 CNN 与 RNN，self-attention 不受感受野和信息传递的限制，可以更好地处理长距离信息。

特点 2: 设计 self-attention 计算过程中的掩码 mask。由于 self-attention 天然可以“无视”距离带来的影响，因此需要对输入像素间自注意力进行约束 。其过程如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1omLZQf4Vrib1cb8e6GhAStjcTx33ic5z3VLQM5ibY8zNJAk8lLV55AYUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

掩码注意力计算过程示意图

总结：通过 multi-head self-attention 对序列建模，可以将两个任意时间片的建模复杂度由 lstm 的 O (N)降低到 O(1), 预测速度大幅提升；优化后的算法和文本长度无关，对大文本得到成倍的加速提升。

#### **3.2 TIACC 加速优化—继续减少模型耗时**

为了进一步降低模型的耗时，我们使用了 TI-ACC 进行加速，[TI-ACC](https://cloud.tencent.com/product/tiacc)支持多种框架和复杂场景，面向算法和业务工程师提供一键式推理加速功能。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1WvYqrR26qoC7Tk6y7glodK5rAMpgHiaX9sibISzrcILRAD155KWOo3rg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

TIACC 框架

##### 3.2.1 支持复杂结构模型

OCR 任务可以分为两大类：检测任务和识别任务，检测任务和识别任务一般会保存成两个独立的 TorchScript 模型，每个任务又分为了多个阶段。通用 OCR 检测任务包含了 backbone(RepVGG 等)、neck(FPN 等）、head（Hourglass 等）等部分。

识别任务包含特征提取部分和序列建模部分，其中特征提取部分既有传统 CNN 网络，也有最新 ResT 等基于 Transformer 的网络，序列建模部分既有传统 RNN 网络，也有基于 Attention 的网络。

对 OCR 业务模型的加速，是对推理加速引擎兼容性的考验。下表分别对比了 TIACC 和 Torch-TensorRT（22.08-py3-patch-v1）对本文模型加速能力：

|                | 通用检测 | 通用识别 | 身份证检测 | 身份证识别 | 高精度 encoder | 高精度 decoder |
| :------------- | :------- | :------- | :--------- | :--------- | :------------- | :------------- |
| Torch-TensorRT | √        | √        | ×          | ×          | ×              | ×              |
| TIACC          | √        | √        | √          | √          | √              | √              |

TIACC 和 Torch-TensorRT 兼容性对比

##### 3.2.2 降低请求延迟并提高系统吞吐

OCR 业务不仅模型结构复杂，模型输入也很有特点，下图显示了本文其中一个模型的输入

| 输入名字    | 类型  | 最小 shape | 最大 shape   |
| :---------- | :---- | :--------- | :----------- |
| input_0     | int32 | 1*10       | 1*2000       |
| input_1     | int32 | 1*10       | 1*2000       |
| input_2     | int32 | 1*10*4     | 1*2000*4     |
| input_3     | float | 1*10*10    | 1*2000*4000  |
| input_4     | float | 2*0*768    | 2*2000*768   |
| input_5     | float | 6*12*64*0  | 6*12*64*2000 |
| input_6     | float | 6*12*0*64  | 6*12*2000*64 |
| input_7     | float | 2*0*768    | 2*2000*768   |
| input_8     | float | 6*12*64*0  | 6*12*64*2000 |
| input_9     | float | 6*12*0*64  | 6*12*2000*64 |
| input_10(0) | int64 | 10         | 2000         |
| input_10(1) | int32 | 1*0        | 1*2000       |

上述输入，包括以下特点：1）输入类型多，2）输入 shape 变化范围大， 3）有些最小 shape 维度是 0，4) 输入不仅包含 Tensor, 也包含 Tuple。仅仅上述模型输入格式，就会导致一些推理引擎不可用或加速效果不明显。比如 TensorRT 不支持 int64 输入，Torch-TensorRT 不支持 Tuple 输入。输入变化范围大，会导致推理引擎显存消耗过高，导致某些推理引擎加速失败或无法单卡多路并行推理，进而导致无法有效提升 TPS。对于 OCR 这种大调用量业务，TPS 提升可以有效降低 GPU 卡规模从而带来可观成本降低。

针对 OCR 模型 shape 范围过大，显存占用量高问题，TIACC 通过显存共享机制，有效降低了显存占用。针对动态 shape 模型，推理加速引擎在部分 shape 加速明显，部分 shape 反而会性能降低的问题，TIACC 通过重写关键算子，并且根据模型结构，选择业务全流程最优的算子实现，有效解决了实际业务中部分 shape 加速明显，整体加速不理想问题。

通过 OCR 业务模型的打磨，最终不仅提升模型的推理延迟，即使多路部署也可以有效提升模型的吞吐。下表显示了 TIACC FP16 相比 libtorch FP16 的加速倍数，数字越高越好。

|          | 通用检测 | 通用识别 | 身份证检测 | 身份证识别 | 高精度 encoder | 高精度 decoder |
| :------- | :------- | :------- | :--------- | :--------- | :------------- | :------------- |
| 延迟降低 | 1.67     | 1.58     | 2.2        | 1.6        | 1.6            | 2.1            |
| 吞吐提升 | 1.73     | 1.47     | 1.43       | 1.47       | 1.71           | 1.97           |

TIACC 加速后延迟降低和吞吐提升统计

TIACC 底层使用了 TNN 作为基础框架，性能强大，其中 TNN 是优图实验室结合自身在 AI 场景多年的落地经验，向行业开源的部署框架。TIACC 基于 TNN 最新研发的子图切分功能基础上进行产品化封装，为企业提供 AI 模型训练、推理加速服务，显著提高模型训练推理效率，降低成本。

#### **3.3 工程优化——优化逻辑和传输耗时**

##### 3.3.1 编解码优化

文字识别采用微服务架构设计，整体服务链路长，其中涉及到 TRPC、HTTP、Nginx-PB 协议的转换和调用，所以请求和响应需要有很多的编解码操作。文字识别场景下，请求和响应包体都非常大，RPC 协议之间的转换和编解码增加了计算耗时，为了进一步降低服务耗时，我们对这些编解码操作进行了整体的优化，减少了协议转换和编解码次数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1JyAECTq5GYBwKtEEj1ZAh7P3AgxsqSBkqpoibxj7XtibuUGFd0mhUWQw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### 3.3.2 日志分流处理

在业务中有很多关键节点需要记录日志，便于问题定位。在文字识别业务场景下，日志需要脱敏处理大量识别出的文本数据，写入速度慢。为了避免日志操作影响服务响应耗时，我们设计了日志分流上报服务，将日志操作全部通过异步流程上报到其他微服务完成，减少主逻辑耗时。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1VMRxASjdlkmpmSeIEJqQeBBwyYKfapom7tW8MLIpaSxXR8cdmZvFVQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### 3.3.3 就近部署—降低传输耗时

经过多次详细测试文字识别业务在北京、上海、广州区域机器发起请求的耗时差异，我们对跨地域传输耗时有了比较明确的认识。服务跨地域请求时（比如在北京发起请求，实际服务部署在广州），会存在很大的传输耗时波动，客户的使用体验会下降，因此我们针对通用 OCR 接口进行了就近多地部署，在服务部署的架构上对耗时进行了优化。

在文字识别跨地域请求时，进行多地部署后耗时有明显下降。通用 OCR 接口线上多地部署发布以后，我们对线上云 API 记录的总耗时监控进行了观察和对比，耗时有了很明显的下降，某段时间内耗时从平均 1100ms 下降到 800ms，下降了 300ms。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf11WW9TKDdwGyYibO8pq4vgIBpzsVeQJoBCwZSYcGtdNkjdib9Fgib9xYyg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### 3.3.4 GPU 显存优化-提高系统吞吐

随着 OCR 业务功能点越来越多，业务中使用的 AI 模型越来越多，且更复杂，对显存的要求也越来越大。这时，显卡显存大小影响服务能开启并发数，而并发数影响服务的吞吐，所以显存往往成为业务吞吐量的瓶颈。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf12JktylxyOCgkTqXAKKCDvibFrBjB5QhiaoExE5cbZoMptqEEEjYQ7hEQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

显存优化方案框架

显存管理主要就是解决上述问题，主要思想是解耦合显存占用大小跟服务并发路数之间的关系，提高并发路数不再导致显存增大，进而提升服务整体吞吐量；并且由服务路数实现并行方式转换为不同模型之间并行方式，提高了 GPU 计算的并行度，更好的充分利用 GPU 资源。以通用 OCR 为例，下图可以看使用前后 GPU 利用率变化和显存占用变化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1Z2mFY25OibpHxcwNCcTbTGy7xKrfLWcV7MofMrcHPAbb8duQCplGZnQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

优化后平均 GPU 利用率明显提高



![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1MhX61ibEpNcoTZeuJvfPk4S4tcUC3CgxGUKywAkdbOgdot5EKLzp4MQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

优化后显存明显降低



### 4.最终效果

#### **4.1 通用 OCR 平均耗时优化 54.6%**

通用 OCR 三地的平均耗时，优化前是 1815ms，优化后 824ms，优化比例 54.6%。

| 地区     | 优化前(ms) | 优化后(ms) | 优化比例 |
| :------- | :--------- | :--------- | :------- |
| 华东     | 2110       | 747        | 64.6%    |
| 华北     | 1946       | 1129       | 42.0%    |
| 华南     | 1390       | 596        | 57.1%    |
| 三地平均 | 1815       | 824        | 54.6%    |

#### **4.2 图片文字越多，耗时优化越明显**

1. 随着图片中的字数的增多，耗时优化效果更明显，耗时优化比例从 23%到 69%。
2. 在耗时优化之前，服务耗时和图片文字正相关，相关性强；优化后，相关性降低。

| 华南-测试集(2048P) | 优化前(ms) | 优化后(ms) | 优化比例 |
| :----------------- | :--------- | :--------- | :------- |
| 字数 0-10          | 829        | 637        | 23%      |
| 字数 10-100        | 835        | 565        | 32%      |
| 字数 100-500       | 1493       | 732        | 51%      |
| 字数 500-1000      | 3338       | 820        | 75%      |
| 字数 1000 以上     | 2751       | 847        | 69%      |
| 平均               | 1849       | 720        | 50%      |

#### **4.3 召回率提升 1.1%**

效果：优化后的版本精度有所提升，召回率提升 1.1%，准确率提升 2.3%。

|              | 优化前       | 优化后       | 差值         |              |              |              |              |
| :----------- | :----------- | :----------- | :----------- | :----------- | :----------- | :----------- | :----------- |
| 测试集       | metric       | recall       | precision    | recall       | precision    | recall       | precision    |
| 测试集 1     | char         | 91.94        | 93.82        | 94.87        | 95.59        | 2.93         | 1.77         |
| 测试集 1     | word         | 81.83        | 84.51        | 82.62        | 84.13        | 0.79         | -0.38        |
| 测试集 2     | char         | 91.34        | 89.97        | 91.91        | 95.52        | 0.57         | 5.55         |
| 测试集 2     | word         | 80.03        | 76.83        | 83.64        | 86.9         | 3.61         | 10.07        |
| ............ | ............ | ............ | ............ | ............ | ............ | ............ | ............ |
| 测试集 n     | char         | 97.49        | 97.92        | 97.74        | 98.8         | 0.25         | 0.88         |
| 测试集 n     | word         | 67.25        | 74.4         | 74.67        | 81.43        | 7.42         | 7.03         |
| 平均指标     | 92.5%        | 93.0%        | 93.6%        | 95.4%        | **1.1%**     | **2.3%**     |              |

### 5.成本降低

通过 TI-ACC、工程优化等手段，不仅满足了客户的耗时要求，成功牵引重要客户落地，同时也提升了每个接口的 tps，平均 tps 提升 51.4%。

|          | 通用印刷 OCR | 高精度 OCR | 高精度文档 | 平均      |
| :------- | :----------- | :--------- | :--------- | :-------- |
| tps 提升 | 43%          | 65%        | 60%        | **51.4%** |
| 显存降低 | 降低 30%~80% |            |            |           |

由于 tps 的提升，每月可以节省等比例的机器成本。



### 6.结语

通用全链路优化，整体耗时降低 54.6%，GPU 利用率也从平均 35%优化到 85%。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavJfeqMS3YgOt122Gic9Chf1kB5rS0yoh0Nujp3yXuAEs3FTXqCmUJgAnYJDic2ib7wTykMxScRbAF6w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

本次优化取得了阶段性的成果，但耗时是一个持续不断的过程，通用 OCR pipleline 等环节可能还存在优化空间，后面将继续跟踪。

整个耗时优化涉及到的环节比较多，特别感谢优图实验室、TIACC、测试团队在耗时优化中的支持和给力配合。

注：本文所述数据均为实验测试数据。

相关链接：

1.OCR:https://cloud.tencent.com/product/generalocr

2.TNN:https://github.com/Tencent/TNN

3.TIACC:https://cloud.tencent.com/product/tiacc



原文作者：benpeng，腾讯 CSIG 应用开发工程师

原文链接：https://mp.weixin.qq.com/s/ByQl3oIbxe3sqUzhfQ9rkQ