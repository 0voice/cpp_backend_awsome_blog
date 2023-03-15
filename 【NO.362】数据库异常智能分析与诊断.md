# 【NO.362】数据库异常智能分析与诊断

## 1. 现状与问题

### 1.1 规模增长与运维能力发展之间的不平衡问题凸显

伴随着最近几年美团业务的快速发展，数据库的规模也保持着高速增长。而作为整个业务系统的“神经末梢”，数据库一旦出现问题，对业务造成的损失就会非常大。同时，因数据库规模的快速增长，出现问题的数量也大大增加，完全依靠人力的被动分析与定位已经不堪重负。下图是当时数据库实例近年来的增长趋势：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2x1KoT62DWAib7Sw5DticNkmwAYJJd6VN48yA6VqpH6kB7LvXzPicXI4BeQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图1 数据库实例增长趋势

### 1.2 理想很丰满，现实很骨感

美团数据库团队当前面临的主要矛盾是：实例规模增长与运维能力发展之间的不平衡，而主要矛盾体现在数据库稳定性要求较高与关键数据缺失。由于产品能力不足，只能依赖专业DBA手动排查问题，异常处理时间较长。因此，我们决定补齐关键信息，提供自助或自动定位问题的能力，缩短处理时长。

我们复盘了过去一段时间内的故障和告警，深入分析了这些问题的根因，发现任何一个异常其实都可以按时间拆分为异常预防、异常处理和异常复盘三阶段。针对这三阶段，结合MTTR的定义，然后调研了美团内部及业界的解决方案，我们做了一张涵盖数据库异常处理方案的全景图。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xYQTOGCicdWgfQQKQIxhusvpQtm390MJT8AaoamybMu3fvxb6iaTxRVtw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图2 运维能力的现状

通过对比，我们发现：

- 每个环节我们都有相关的工具支撑，但能力又不够强，相比头部云厂商大概20%～30%左右的能力，短板比较明显。
- 自助化和自动化能力也不足，工具虽多，但整个链条没有打通，未形成合力。

那如何解决这一问题呢？团队成员经过深入分析和讨论后，我们提出了一种比较符合当前发展阶段的解决思路。

## 2. 解决的思路

### 2.1 既解决短期矛盾，也立足长远发展

从对历史故障的复盘来看，80%故障中80%的时间都花在分析和定位上。解决异常分析和定位效率短期的ROI（投资回报率）最高。长期来看，只有完善能力版图，才能持续不断地提升整个数据库的稳定性及保障能力。因此，我们当时的一个想法就是既要解决短期矛盾，又要立足长远发展（Think Big Picture, Think Long Term）。新的方案要为未来留足足够的发展空间，不能只是“头痛医头、脚痛医脚”。

在宏观层面，我们希望能将更多的功能做到自动定位，基于自动定位来自助或自动地处理变更，从而提高异常恢复的效率，最终提升用户体验。将异常处理效率提高和用户体验提升后，运维人员（主要是DBA）的沟通成本将会极大被降低，这样运维人员就有更多时间进行技术投入，能将更多“人肉处理”的异常变成自助或自动处理，从而形成“飞轮效应”。最终达成高效的稳定性保障的目标。

在微观层面，我们基于已有的数据，通过结构化的信息输出，提升可观测性，补齐关键数据缺失的短板。同时，我们基于完善的信息输出，通过规则（专家经验）和AI的配合，提供自助或自动定位的能力，缩短处理时长。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xEAdA2UousZudDkm1q3TSR1j1vbQB0DXqHtd5mnqfuxcwzBNZ0xKicPQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图3 宏观和微观

### 2.2 夯实基础能力，赋能上层业务，实现数据库自治

有了明确的指导思想，我们该采取怎样的发展策略和路径呢？就当时团队的人力情况来看，没有同学有过类似异常自治的开发经验，甚至对数据库的异常分析的能力都还不具备，人才结构不能满足产品的终极目标。所谓“天下大事必作于细，天下难事必作于易”。我们思路是从小功能和容易的地方入手，先完指标监控、慢查询、活跃会话这些简单的功能，再逐步深入到全量SQL、异常根因分析和慢查询优化建议等这些复杂的功能，通过这些基础工作来“借假修真”，不断提升团队攻坚克难的能力，同时也可以为智能化打下一个良好的基础。

以下便是我们根据当时人才结构以及未来目标设定的2年路径规划（实现数据自治目标规划在2022以后的启动，下图会省略掉这部分）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xD6NBk9euz9EaOlQZ4HQMe7PBjmx6Ob11Hw1rKp7huVzufep4e93aDg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图4 演进策略

### 2.3 建立科学的评估体系，持续的跟踪产品质量

美国著名管理学者卡普兰说过：“没有度量就没有管理”。只有建立科学的评估体系，才能推进产品不断迈向更高峰，怎样评估产品的质量并持续改善呢？之前我们也做过很多指标，但都不可控，没有办法指导我们的工作。比如，我们最开始考虑根因定位使用的是结果指标准确率和召回率，但结果指标不可控难以指导我们的日常工作。这就需要找其中的可控因素，并不断改善。

我们在学习亚马逊的时候，刚好发现他们有一个可控输入和输出指标的方法论，就很好地指导了我们的工作。只要在正确的可控输入指标上不断优化和提升，最终我们的输出指标也能够得到提升（这也印证了曾国藩曾说过的一句话：“在因上致力，但在果上随缘”）。

以下是我们关于根因定位的指标设计和技术实现思路（在模拟环境不断提升可控的部分，最终线上的实际效果也会得到提升。主要包括“根因定位可控输入和输出指标设计思路”和“根因定位可控输入指标获取的技术实现思路”）。

**根因定位可控输入和输出指标设计思路**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xZX2PhBIJfwd45vOTdnOicoN93dbx8p9htKMVZ2pGIO7ldA1skpzvlDg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图5 可控输入与输出指标设计

**根因定位可控输入指标获取的技术实现思路**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xxBXjuXCLsFfow6x3Yh4lKvBicmyMsHen8IdrYpXtTdPMciakzRpp9Efg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图6 可控输入与输出指标技术设计

在图5中，我们通过场景复现方式，用技术手段来模拟一些用低成本就能实现的异常（绝大部份异常）。在对于复现成本比较高的异常（极少部分），比如机器异常、硬件故障等，我们目前的思路是通过“人肉运营”的方式，发现和优化问题，等到下次线上异常重复发生后，根据优化后诊断的结果，通过和预期比较来确定验收是否通过。

未来我们会建立回溯系统，将发生问题时刻的异常指标保存，通过异常指标输入給回溯系统后的输出结果，判断系统改进的有效性，从而构建更加轻量和更广覆盖的复现方式。图6是复现系统的具体技术实现思路。

有了指导思想，符合当前发展阶段的路径规划以及科学的评估体系后，接下来聊聊技术方案的构思。

## 3. 技术方案

### 3.1 技术架构的顶层设计

在技术架构顶层设计上，我们秉承平台化、自助化、智能化和自动化四步走的演进策略。

首先，我们要完善可观测的能力，通过关键信息的展示，构建一个易用的数据库监控平台。然后我们根据这些关键信息为变更（比如数据变更和索引变更等）提供赋能，将一部分高频运维工作通过这些结构化的关键信息（比如索引变更，可以监测近期是否有访问流量，来确保变更安全性）让用户自主决策，也就是自助化。接下来，我们加入一些智能的元素（专家经验+AI），进一步覆盖自助化的场景，并逐步将部分低风险的功能自动化，最终通过系统的不断完善，走到高级或完全自动化的阶段。

为什么我们将自动化放在智能化之后？因为我们认为智能化的目标也是为了自动化，智能化是自动化的前提，自动化是智能化的结果。只有不断提升智能化，才能达到高级或者完全自动化。下图便是我们的顶层架构设计（左侧是演进策略，右侧是技术架构的顶层设计以及2021年底的现状）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xAJM3WOMJibf0MJL7Z1dAbRqjic5TzxBH62QzYDV7LJ3Sz9gshuDjL2YA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图7 架构顶层设计

顶层设计只是“万里长征第一步”，接下来我们将自底向上逐步介绍我们基于顶层设计开展的具体工作，将从数据采集层的设计、计算存储层的设计和分析决策层的设计逐步展开。

### 3.2 数据采集层的设计

这上面的架构图里，数据采集层是所有链路的最底层和最重要的环节，采集数据的质量直接决定了整个系统的能力。同时，它和数据库实例直接打交道，任何设计上的缺陷都将可能导致大规模的故障。所以，技术方案上必须兼顾数据采集质量和实例稳定性，在二者无法平衡的情况下，宁可牺牲掉采集质量也要保证数据库的稳定性。

在数据采集上，业界都采取基于内核的方式，但美团自研内核较晚，而且部署周期长，所以我们短期的方式是采用抓包的方式做一个过渡，等基于内核的采集部署到一定规模后再逐步切换过来。以下是我们基于抓包思路的技术方案调研：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xVKwxpakIJibW6iaUT44UANocMIib1JgmL2XmY3gGxIssPmYDQhQYlia4Cw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从调研上我们可以看到，基于pf_ring和dpdk的方案都有较大的依赖性，短期难以实现，之前也没有经验。但是，基于pcap的方式没有依赖，我们也有过一定的经验，之前美团酒旅团队基于抓包的方式做过全量SQL数据采集的工具，并经过了1年时间的验证。因此，我们最终采取了基于pcap抓包方式的技术方案。以下是采集层方案的架构图和采集质量以及对数据库性能带来的影响情况。

**Agent的技术设计**



![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xKufnNAdHb469ZdQm3XMO5aDdHJqRdb1s7QAic9l2NDVP7Uw0gGzgRog/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图8 Agent的技术设计

**对数据库的影响**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xI8T6YJPZe8wcgzLZwQ8AnqQU4s7AoqRVoc4zT9QiczO8ek7aby9AJ7Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图9 Agent对数据库的影响测试

### 3.3 计算存储层的设计

由于美团整个数据库实例数量和流量巨大，而且随着业务的快速发展，还呈现出快速增长的态势。所以，我们的设计不仅要满足当前，还要考虑未来5年及更长的时间也能够满足要求。同时，对数据库故障分析来说，数据的实时性和完备性是快速和高效定位问题的关键，而保证数据实时性和完备性需要的容量成本也不容忽视。因此，结合上述要求和其他方面的一些考虑，我们对该部分设计提出了一些原则，主要包括：

- **全内存计算**：确保所有的计算都在单线程内或单进程内做纯内存的操作，追求性能跟吞吐量的极致。
- **上报原始数据**：MySQL实例上报的数据尽量维持原始数据状态，不做或者尽量少做数据加工。
  **数据压缩**：由于上报量巨大，需要保障上报的数据进行极致的压缩。
- **内存消耗可控**：通过理论和实际压测保障几乎不可能会发生内存溢出。
- **最小化对MySQL实例的影响**：计算尽量后置，不在Agent上做复杂计算，确保不对RDS实例生产较大影响。以下是具体的架构图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2x6N50wA2I4qw5zpuLUpvKroyIUnv35poykT6LhibX055c2NH5X3jlJdw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图10 Agent对数据库的影响测试

全量SQL（所有访问数据库的SQL）是整个系统最具挑战的功能，也是数据库异常分析最重要的信息之一，因此会就全量SQL的聚合方式、聚合和压缩的效果和补偿设计做一些重点的介绍。

#### 3.3.1 全量SQL的聚合方式

由于明细数据巨大，我们采取了聚合的方式。消费线程会对相同模板SQL的消息按分钟粒度进行聚合计算，以“RDSIP+DBName+SQL模版ID+SQL查询结束时间所在分钟数”为聚合键。聚合健的计算公式为：Aggkey=RDS_IP_DBName_SQL_Template_ID_Time_Minute （Time_Minute的值取自SQL查询结束时间所在的“年、月、日、时、分钟”）

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xyDSy2ZibsibT8sa18OXM2TDpR53ywGDk1kGqibfYrKb8BhiakUtGsF2Ftg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图11 SQL模版聚合设计

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2x268qqmjAJK3OY55PwsZbITHxzlOAbibQpica1UyibhN8ASPGoU0ibyyOOw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图12 SQL模版聚合方法

#### 3.3.2 全量SQL数据聚合和压缩的效果

在数据压缩方面，遵循层层减流原则，使用消息压缩、预聚合、字典化、分钟级聚合的手段，保证流量在经过每个组件时进行递减，最终达到节省带宽减少存储量的目的。以下是相关的压缩环节和测试数据表现情况（敏感数据做了脱敏，不代表美团实际的情况）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xjksdAR2Jx1HEibNA3icCHl25wg5ggO2ADCfs7HMgwfAzFzrr0PvlY8Tw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图13 全量SQL压缩设计与效果

#### 3.3.3 全量SQL数据补偿机制

如上所述，在数据聚合端是按一分钟进行聚合，并允许额外一分钟的消息延迟，如果消息延迟超过1分钟会被直接丢弃掉，这样在业务高峰期延迟比较严重的场景下，会丢失比较大量的数据，从而对后续数据库异常分析的准确性造成较大的影响。

因此，我们增加了延迟消息补偿机制，对过期的数据发入补偿队列（采用的是美团消息队列服务Mafka），通过过期数据补偿的方式，保证延迟久的消息也能正常存储，通过最终一致性保证了后续的数据库异常分析的准确性。以下是数据补偿机制的设计方案：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2x275uDmh33rD0PKL9KZqU2mEzhzpK2DJ1NDEvqaribNKTnTsmswpOsgw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
图14 全量SQL补全技术设计

### 3.4 分析决策层的设计

在有了比较全的数据之后，接下来就是基于数据进行决策，推断出可能的根因。这部分我们使用了基于专家经验结合AI的方式。我们把演进路径化分为了四个阶段：

- **第一阶段**：完全以规则为主，积累领域经验，探索可行的路径。
- **第二阶段**：探索AI场景，但以专家经验为主，在少量低频场景上使用AI算法，验证AI能力。
- **第三阶段**：在专家经验和AI上齐头并进，专家经验继续在已有的场景上迭代和延伸，AI在新的场景上进行落地，通过双轨制保证原有能力不退化。
- **第四阶段**：完成AI对大部分专家经验的替换，以AI为主专家经验为辅，极致发挥AI能力。

以下是分析决策部分整体的技术设计（我们参考了华为一篇文章：[《网络云根因智荐的探索与实践》](https://mp.weixin.qq.com/s?__biz=MzAwMzcxNDM4Mg==&mid=2649079833&idx=1&sn=f131de51defe1e3a6125f1f0d0f117e7&scene=21#wechat_redirect)）：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xWHqicVzwoIA0fnhMxs9Zh0icYlibYyNcmFrXWHMXf5DIuax3jpLZ1ObIA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图15 分析决策的技术设计

在决策分析层，我们主要采取了专家经验和AI两者方式，接下来会介绍专家经验（基于规则的方式）和AI方式（基于AI算法的方式）的相关实现。

#### 3.4.1 基于规则的方式

专家经验部分，我们采取了GRAI（Goal、Result、Analysis和Insight的简称）的复盘方法论来指导工作，通过持续、大量、高频的复盘，我们提炼了一些靠谱的规则，并通过持续的迭代，不断提升准确率和召回率。下面例举的是主从延迟的规则提炼过程：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xcx9IHy4UJ5qPmcwdian1AdnrIsQzVyiaicDvfibX2qxJAqcj2HhUT13g0g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
图16 专家经验的复盘和改进

#### 3.4.2 基于AI算法的方式

**异常数据库指标检测**

数据库核心指标的异常检测依赖于对指标历史数据的合理建模，通过将离线过程的定期建模与实时过程的流检测相结合，将有助于在数据库实例存在故障或风险的情况下，有效地定位根本问题所在，从而实现及时有效地解决问题。

建模过程主要分为3个流程。首先，我们通过一些前置的模块对指标的历史数据进行预处理，包含缺失数值填充，数据的平滑与聚合等过程。随后，我们通过分类模块创建了后续的不同分支，针对不同类型的指标运用不同的手段来建模。最终，将模型进行序列化存储后提供Flink任务读取实现流检测。

**以下是检测的设计图**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xKLtnTMaArlFlvgAQbic5unp4UdmO3zibvXic6hUq06uczicHgt9iciaRiaKTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图17 基于AI的异常检测设计

**根因诊断（构建中）**

订阅告警消息（基于规则或者异常检测触发），触发诊断流程，采集、分析数据，推断出根因并筛选出有效信息辅助用户解决。诊断结果通过大象通知用户，并提供诊断诊断详情页面，用户可通过标注来优化诊断准确性。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xNuv3kYxHG68nlJ8ynfYFzVMrRNOwicjfdSjgPECO0MIUqNoMs17oUpA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图18 基于AI的异常检测设计

- **数据采集**：采集数据库性能指标、数据库状态抓取、系统指标、硬件问题、日志、记录等数据。
- **特征提取**：从各类数据中提取特征，包括算法提取的时序特征、文本特征以及利用数据库知识提取的领域特征等。
- **根因分类**：包括特征预处理、特征筛选、算法分类、根因排序等部分。
- **根因扩展**：基于根因类别进行相关信息的深入挖掘，提高用户处理问题的效率。具体包括SQL行为分析、专家规则、指标关联、维度下钻、日志分析等功能模块。

**4 建设成果**

**4.1 指标表现**

我们目前主要是通过“梳理触发告警场景->模拟复现场景->根因分析和诊断->改进计划->验收改进质量->梳理触发告警场景”的闭环方法（详情请参考前文**建立科学的评估体系，持续的跟踪产品质量**部分），持续不断的进行优化和迭代。通过可控输入指标的提升，来优化改善线上的输出指标，从而保证系统不断的朝着正确的方向发展。以下是近期根因召回率和准确率指标表现情况。

**用户告警根因反馈准确率**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xlxQ559xUkeRwlgXRYLjuZUSiaCXvvXdWI5ybZWZAxwPs8zyd894FVzw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图19 用户反馈准确率

**告警诊断分析总体召回率**



![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xric7vG40rL9bmswTchjll1v8TpBOayg362M8LvjH1kVqbc998bEJcwg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图20 根因分析召回率

### 4.2 用户案例

在根因结果推送上，我们和美团内部的IM系统（大象）进行了打通，出现问题后通过告警发现异常->大象推送诊断根因->点击诊断链接详情查看详细信息->一键预案处理->跟踪反馈处理的效果->执行成功或者回滚，来完成异常的发现、定位、确认和处理的闭环。以下是活跃会话规则触发告警后根因分析的一个案例。

**自动拉群，并给出根因**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2x2RTOU2KiaRxTGiczfMAZPaHuicib2KfbBEL3zbUFZAaQvjGIekjdGDA5Mg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图21 锁阻塞导致活跃会话过高

**点击诊断报告，查看详情**

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xreoQdkU4uFFrSeuy27pbvbo352RCmqXXNkz3lVibEaO6Exb4gJNGmsA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
图22 锁阻塞导致活跃会话过高

以下是出现Load告警后，我们的一个慢查询优化建议推送案例（脱敏原因，采用了测试环境模拟的案例）。

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVBw9zgLdYXhY0DKVsVOn2xElGiaa38jyCKFYiaWaiaUl6PgTW5o7whptxs5Liaz5rsicNZ1z4iaMZ3QByw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
图23 慢查询优化建议

## 5. 总结与未来展望

数据库自治服务经过2年左右的发展，已基本夯实了基础能力，在部分业务场景上完成了初步赋能（比如针对问题SQL，业务服务上线发布前自动识别，提示潜在风险；针对索引误变更，工单执行前自动检测索引近期访问流量，阻断误变更等）。接下来，我们的目标除了在已完成工作上继续深耕，提升能力外，重点会瞄准数据库自治能力。主要的工作规划将围绕以下3个方向：

**（1）计算存储能力增强**：随着数据库实例和业务流量的持续高速增长，以及采集的信息的不断丰富，亟需对现有数据通道能力进行增强，确保能支撑未来3-5年的处理能力。

**（2）自治能力在少部分场景上落地**：数据库自治能力上，会采取三步走的策略：

- 第一步：建立根因诊断和SOP文档的关联，将诊断和处理透明化；
- 第二步：SOP文档平台化，将诊断和处理流程化；
- 第三步：部分低风险无人干预，将诊断和处理自动化，逐步实现数据库自治。

**（3）更灵活的异常回溯系统**：某个场景根因定位算法在上线前或者改进后的验证非常关键，我们会完善验证体系，建立灵活的异常回溯系统，通过基于现场信息的回放来不断优化和提升系统定位准确率。

## 6. 本文作者

金龙，来自美团基础技术部/数据库研发中心/数据库平台研发组。

原文链接：https://mp.weixin.qq.com/s/PmMVBjAzjeJYWBJI39gf_g