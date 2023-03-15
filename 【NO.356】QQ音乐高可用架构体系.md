# 【NO.356】QQ音乐高可用架构体系

### **1. QQ音乐高可用架构体系全景**

故障无处不在，而且无法避免。（[分布式计算谬误](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)）

在分布式系统建设的过程中，我们思考的重点不是避免故障，而是拥抱故障，通过构建高可用架构体系来获得优雅应对故障的能力。QQ音乐高可用架构体系包含三个子系统：架构、工具链和可观测性。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZe6zbQANoB1icltLwzqsS8GTps2KXYjPXdu9OehD4JYUqibrjkpDmMngA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **架构**：架构包括冗余架构、自动故障转移和稳定性策略。高可用架构的基础是通过冗余架构来避免单点故障。其中，基础设施的冗余包括集群部署、多机房部署，多中心部署等；软件架构上要支持横向扩展、负载均衡和自动故障转移。这样系统就具备了基础的容灾容错能力。在冗余架构的基础之上，可以通过一系列稳定性策略来进一步提升架构的可用性。稳定性策略包括分布式限流，熔断，动态超时等。
- **工具链**：工具链指一套可互相集成和协作的工具，从外围对架构进行实验和测试以达到提升架构可用性的目的，包括混沌工程和全链路压测等。混沌工程通过注入故障的方式，来发现系统的脆弱环节并针对性地加固，帮助我们提升系统的可用性。全链路压测通过真实、高效的压测，来帮助业务完成性能测试任务，进行容量评估和瓶颈定位，保障系统稳定。
- **可观测性**：通过观测系统的详细运行状态来提升系统的可用性，包括日志、指标、全链路跟踪、性能分析和panic分析。可观测性可以用于服务生命周期的所有阶段，包括服务开发，测试，发布，线上运行，混沌工程，全链路压测等各种场景。



### **2. 容灾架构：**

业内主流的容灾方案，包括异地冷备，同城双活，两地三中心，异地双活/多活等。

- **异地冷备**：冷备中心不工作，成本浪费，关键时刻不敢切换。
- **同城双活**：同城仍然存在很多故障因素(如自然灾害)导致整体不可用。
- **异地双活/多活**：双写/多写对数据一致性是个极大挑战。有些做法是按用户ID哈希，使用户跟数据中心绑定，避免写冲突问题，但这样一来舍弃了就近接入的原则，而且灾难发生时要手动调度流量，也无法做到API粒度的容灾。

容灾架构的选型我们需要衡量投入产出比，不要为了预防哪些极低概率的风险事件而去投入过大的成本，毕竟业务的规模和收入才是重中之重。QQ音乐的核心体验是听歌以及围绕听歌的一系列行为，这部分业务以只读服务为主。而写容灾需要支持双写，带来的数据一致性风险较大，且有一定的实施成本。综合权衡，我们舍弃写容灾，采用**一写双读的异地双活**模式。

#### 2.1. 异地双中心

在原有深圳中心的基础上，建设上海中心，覆盖接入层、逻辑层和存储层，形成异地双中心。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZQ1SBfRSEHxmeL67PXWnKjwgicunOToPcMbtXhjy3MiaBSOhTR18sdtDQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **接入层**：深圳中心和上海中心各部署一套STGW和API网关。通过GSLB把流量按就近原则划分为两份，分配给深圳中心和上海中心，深圳中心和上海中心做流量隔离。
- **逻辑层**：服务做读写分离。深圳部署读/写服务，上海部署只读服务，上海的写请求由API网关路由到深圳中心处理。
- **存储层**：深圳中心和上海中心各有一套存储。写服务从深圳写入存储，通过同步中心/存储组件同步到上海，同步中心/存储组件确保两地数据的一致性。同步方式有两种，对于有建设异地同步能力的组件Cmongo和CKV+，依赖存储组件的异地同步能力，其他不具备异地同步能力的，如ckv，tssd等老存储，使用同步中心进行同步。

#### 2.2. 自动故障转移

异地容灾支持自动故障转移才有意义。如果灾难发生后，我们在灾难发现、灾难响应以及评估迁移风险上面浪费大量时间，灾难可能已经恢复。

我们最初的方案是，客户端对两地接入IP进行动态评分（请求成功加分，请求失败减分），取最优的IP接入来达到容灾的效果。经过近两年多的外网实践，遇到一些问题，动态评分算法敏感度高会导致流量在两地频繁漂移，算法敏感度低起不了容灾效果。而算法调优依赖客户端版本更新也导致成本很高。

后来我们基于异地自适应重试对容灾方案做了优化，核心思想是在API网关上做故障转移，降低客户端参与度。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZZHuHDDDZA5gtVa4oef2NXLOs7Fa0sqehTIbhLIJ5zjIrUK3ibGTG2bw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



方案主要有两点：

- **API网关故障转移**：当本地中心API返回失败时（包括触发熔断和限流），API网关把请求路由到异地处理。以此解决API故障的场景。
- **客户端故障转移**：当API网关发生超时的时候，客户单进行异地重试。如果网关有回包，即使API返回失败，客户端也不重试。解决API网关故障的场景。

使用最新方案后，API网关重试比客户端调度更可控，双中心流量相对稳定，一系列自适应限流和熔断策略也抵消重试带来的请求量放大问题。接下来介绍方案细节。

#### 2.3. API网关故障转移

API网关故障转移需要考虑重试流量不能压垮异地，否则会造成双中心同时雪崩。这里我们做了一个自适应重试的方案，在异地成功率下降的时候，取消重试。

**自适应重试方案**：

- 引入重试窗口：如果当前周期窗口为10，则最多只能重试10次，超过的部分丢弃。
- 网关请求服务失败，判断重试窗口是否耗光。如果耗光则不重试，如果还有余额，重试异地。

上述方案中的重试窗口，由**探测及退避策略**决定：

- **探测策略**：当探测成功率正常时，增大下一次窗口并继续探测。通过控制窗口大小，避免重试流量瞬间把异地打垮。
- **退避策略**：在探测成功率出现异常时，重试窗口快速退避。
- **增加重试开关**，控制整体及服务两个维度的重试。

**探测策略及退避策略图示：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZnrPjHA4KECPw28RXF1Nuza94JgicRDRsCJicksf2zyYkaSB1O1datXTg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





**探测策略及退避策略的算法描述：**

```
// 设第 i 次探测的窗口为 f(i)，实际探测量为 g(i)，第 i 次探测的成功率为 s(i)，第 i 次本地总请求数为 t。
// 那么第 i+1 次探测的窗口为 f(i+1)，取值如下：
if  s(i) = [98%, 100%]       // 第 i 次探测成功率 >= 98%，探测正常
    if  g(i) >= f(i)      // 如果第 i 次实际探测量等于当前窗口，增大第 i+1 次窗口大小
        f(i+1) = f(i) + max (min ( 1% * t, f(i) ) , 1)    
    else
        f(i+1) = f(i)        // 如果第 i 次实际探测量小于当前窗口，第 i+1 次探测窗口维持不变
else   
    f(i+1) = max(1,f(i)/2)        // 如果第 i 次探测异常，第 i+1 次窗口退避置 1
// 其中，重试窗口即 f(i) 初始大小为 1。算法中参数及细节，根据实际测试和线上效果进行调整。
```

**自适应重试效果：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZYngHib3AAm32O7mU2hmenricEZrX7PGQibqyyiaYprUOrnJVIxmibFo2wHQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





#### 2.4. 客户端故障转移

- 当客户端未收到响应时，说明API网关异常或者网络不通，客户端重试异地。
- 当客户端收到响应，而http状态码为5xx，说明API网关异常，客户端重试异地。当http状态码正常，说明API网关正常，此时即使API失败也不重试。
- 当双中心均超时，探测网络是否正常，如果网络正常，说明两地API网关均异常，所有客户端请求冻结一段时间。



### **3. 稳定性策略：**

#### 3.1. 分布式限流

即使我们在容量规划上投入大量精力，根据经验充分考虑各种请求来源，评估出合理的请求峰值，并在生产环境中准备好足够的设备。但预期外的突发流量总会出现，对我们规划的容量造成巨大冲击，极端情况下甚至会导致雪崩。我们对容量规划的结果需要坚守不信任原则，做好防御式架构。

限流可以帮助我们应对突发流量，我们有几个选择：

- **固定窗口计算器**优点是简单，但存在临界场景无法限流的情况。
- **漏桶**是通过排队来控制消费者的速率，适合瞬时突发流量的场景，面对恒定流量上涨的场景，排队中的请求容易超时饿死。
- **令牌桶**允许一定的突发流量通过，如果下游（callee）处理不了会有风险。
- **滑动窗口计数器**可以相对准确地完成限流。

我们采用的是滑动窗口计数器，主要考虑以下几点：

- 超过限制后微服务框架直接丢弃请求。
- 对原有架构不引入关键依赖，用分布式限流的方式代替全局限流。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZpqRmeluEhvEVyMUsUL1rhdZtlEfxHUhfZGdNemjUdnh4cibUibk7DyWg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



上图描述ServceA到ServiceB之间的RPC调用过程中的限流：

- Sidecar用于接管微服务的信令控制，限流器则以插件的方式集成到Sidecar。限流器通过限流中心Agent，从限流中心获取限流结果数据。
- ServiceA发起请求到ServiceB前，通过Sidecar的限流器判断是否超出限制，如果超出，则进入降级逻辑。
- 限流中心采用滑动窗口的方式，计算每个Service的限流数据。

限流算法的选择，还有一种可行的方案是，框架提供不同的限流组件，业务方根据业务场景来选择，但也要考虑成本。社区也有Sentinel等成熟解决方案，新业务可以考虑集成现成的方案。

#### 3.2. 自适应限流

上一节的分布式限流是在Client-side限制流量，即请求量超出阈值后在主调直接丢弃请求，被调不需要承担拒绝请求的资源开销，可最大程度保护被调。然而，Client-side限制流量强依赖主调接入分布式限流，这一点比较难完全受控。同时，分布式限流在集群扩缩容后需要及时更新限流阈值，而全量微服务接入有一定的维护成本。而且分布式限流直接丢弃请求更偏刚性。作为分布式限流的互补能力，自适应限流是在Server-side对入口流量进行控制，自动嗅探负载、入口QPS、请求耗时等指标，让系统的入口QPS和负载达到一个平衡，确保系统以高水位QPS正常运行，而且不需要人工维护限流阈值。相比分布式限流，自适应限流更偏柔性。

**指标说明：**

| 指标名称  |               指标含义                |
| :-------: | :-----------------------------------: |
| CPU usage | 当系统CPU使用率超过阈值启动自适应限流 |
| inflight  |       系统中正在处理的请求数量        |
|    qps    |    窗口内每个桶的请求处理成功的量     |
|  MaxQPS   |         滑动窗口中QPS的最大值         |
|    rt     |   窗口内每个桶的请求成功的响应耗时    |
|   MinRt   |      滑动窗口中响应延时的最小值       |

**算法原理：**

根据Little's Law，inflight = 延时 *QPS。则最优inflight为MaxPass* MinRt，当系统当前inflight超过最优inflight，执行限流动作。

用公式表示为：cpu > 800 AND InFlight > (MaxQPS * MinRt)

其中MaxQPS和MinRt的估算需要增加平滑策略，避免秒杀场景下最优inflight的估算失真。

**限流效果：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZo6yzGW8nDB0jJjianvKn4sLdeia1aniaLKBTWjJRibZsKl3tV7aA847P2g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





#### 3.3. 熔断

在微服务系统中，服务会依赖于多个服务，并且有一些服务也依赖于它。如下图，“统一权限”服务，依赖歌曲权限配置、购买信息、会员身份等服务，综合判断用户是否拥有对某首歌曲进行播放/下载等操作的权限，而这些权限信息，又会被歌单、专辑等歌曲列表服务依赖。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZo8DXa3Y0AvoqWLc9Llj8DibmvYNxRvt1XsAwPz8OKriaiaoYXudiajg9CA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



当“统一权限”服务的其中一个依赖服务（比如歌曲权限配置服务）出现故障，“统一权限”服务只能被动的等待依赖服务报错或者请求超时，下游连接池会逐渐被耗光，入口请求大量堆积，CPU、内存等资源逐渐耗尽，导致服务宕掉。而依赖“统一权限”服务的上游服务，也会因为相同的原因出现故障，一系列的级联故障最终会导致整个系统宕掉。

合理的解决方案是断路器和优雅降级，通过尽早失败来避免局部不稳定而导致的整体雪崩。

传统熔断器实现Closed、Half Open、Open三个状态，当进入Open状态时会拒绝所有请求，而进入Closed状态时瞬间会有大量请求，服务端可能还没有完全恢复，会导致熔断器又切换到Open状态，一种比较刚性的熔断策略。SRE熔断只有打Closed和Half-Open两种状态，根据请求成功率自适应地丢弃请求，尽可能多地让请求成功请求到服务端，是一种更弹性的熔断策略。QQ音乐采用更弹性的SRE熔断器：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZ47danuia5ANN1oiamLlDEViccAw3CDk1JE013LlCJIQg4ktPmribIDC6qQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **requests**：窗口时间内的请求总数。
- **accepts**：正常处理的请求数量。
- **K**：敏感度，K越小丢弃概率越大，一般在1.5~2之间。

正常情况下，requests 等于 accepts，所以丢弃概率为0。随着正常处理的请求减少，直到 requests 等于 K * accepts ，一旦超过这个限制，熔断器就会打开，并按照概率丢弃请求。

#### 3.4. 动态超时

超时是一件很容易被忽视的事情。在早期架构发展阶段，相信大家都有因为遗漏设置超时或者超时设置太长导致系统被拖慢甚至挂起的经历。随着微服务架构的演进，超时逐渐被标准化到RPC中，并可通过微服务治理平台快捷调整超时参数。但仍有不足，传统超时会设定一个固定的阈值，响应时间超过阈值就返回失败。在网络短暂抖动的情况下，响应时间增加很容易产生大规模的成功率波动。另一方面，服务的响应时间并不是恒定的，在某些长尾条件下可能需要更多的计算时间，为了有足够的时间等待这种长尾请求响应，我们需要把超时设置足够长，但超时设置太长又会增加风险，超时的准确设置经常困扰我们。

其实我们的微服务系统对这种短暂的延时上涨具备足够的容忍能力，可以考虑基于EMA算法动态调整超时时长。EMA算法引入“平均超时”的概念，用平均响应时间代替固定超时时间，只要平均响应时间没有超时即可，而不是要求每次都不能超时。主要算法：总体情况不能超标；平均情况表现越好，弹性越大；平均情况表现越差，弹性越小。

如下图，当平均响应时间(EMA)大于超时时间限制(Thwm)，说明平均情况表现很差，动态超时时长(Tdto)就会趋近至超时时间限制(Thwm)，降低弹性。当平均响应时间(EMA)小于超时时间限制(Thwm)，说明平均情况表现很好，动态超时时长(Tdto)就可以超出超时时间限制(Thwm)，但会低于最大弹性时间(Tmax)，具备一定的弹性。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZIY1ibuskViccd3NU5vUm4NAbTBg3U0Ax1YnOus9qMib5Duq1gwOESldoQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



为降低使用门槛，QQ音乐微服务只提供超时时间限制(Thwm)和最大弹性时间(Tmax)两个参数的设置，并可在微服务治理平台调整参数。算法实现参考：https://github.com/jiamao/ema-timeout

#### 3.5. 服务分级

我们做了很多弹性能力，比如限流，我们是否可以根据服务的重要程度来决策丢弃请求。此外，在架构迭代的过程中，有许多涉及整体系统的大工程，如微服务改造，容器化，限流熔断能力落地等项目，我们需要根据服务的重要程度来决策哪些服务先行。

如何为服务确定级别：

- **1级**：系统中最关键的服务，如果出现故障会导致用户或业务产生重大损失，比如登录服务、流媒体服务、权限服务、数专服务等。
- **2级**：对于业务非常重要，如果出现故障会导致用户体验受到影响，但是不会完全无法使用我们的系统，比如排行榜服务、评论服务等。
- **3级**：会对用户造成较小的影响，不容易注意或很难发现，比如用户头像服务，弹窗服务等。
- **4级**：即使失败，也不会对用户体验造成影响，比如红点服务等。

服务分级的应用场景：

- 核心接口运营质量日报：每日邮件推送1级服务和2级服务的观测数据。
- SLA：针对1级服务和2级服务，制定SLO。
- API网关根据服务分级限流，优先确保1级服务通过。
- 重大项目参考服务重要程度制定优先级计划，如容灾演练，大型活动压测等。

#### 3.6. API网关分级限流

API网关既是用户访问的流量入口，也是后台业务响应的最终出口，其可用性是QQ音乐架构体系的重中之重。除了支持自适应限流能力，针对服务重要程度，当触发限流时优先丢弃不重要的服务。

效果如下图，网关高负载时，2级、3级、4级服务丢弃，只有1级服务通过。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZ0nZkdkTJsn6ySGB6pqo0t2VjG6k3f9YQILnNqIicfepb1yYhmkA6hLg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





### **4. 工具链：**

随着产品的迭代，系统不断在变更，性能、延时、业务逻辑、用户量等因素的变化都有可能引入新的风险，而现有的架构弹性能力可能不足以优雅地应对新的风险。事实上，即使架构考虑的场景再多，外网仍然存在很多未知的风险，在某些特殊条件满足后就会引发故障。一般情况下，我们只能等着告警出现。当告警出现后，复盘总结，讨论规避方案，进行下一轮的架构优化。应对故障的方式比较被动。

那么，我们有没有办法变被动为主动？在故障触发之前，尽可能多地识别风险，针对性地加固和防范，而不是等着故障发生。业界有比较成熟的理论和工具，混沌工程和全链路压测。

#### 4.1. 混沌工程

混沌工程通过在生产环境上进行实验，注入网络超时等故障，主动找出系统中的脆弱环节，不断提升系统的弹性。

TMEChaos 以ChaosMesh为底层故障注入引擎，结合TME微服务架构、mTKE容器平台打造成云原生混沌工程实验平台。支持丰富的故障模拟类型，具有强大的故障场景编排能力，方便研发同学在开发测试中以及生产环境中模拟现实世界中可能出现的各类异常，帮助验证架构设计是否合理，系统容错能力是否符合预期，为组织提供常态化应急响应演练，帮助业务推进高可用建设。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZbOW74FFNNoseJOV90XlJ0xtibhc7343ax5dOLacuH0C8CZwsdYLsibxQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **TMEChaos Dashboard Web**：TMEChaos的可视化组件，提供了一套用户友好的 Web 界面，用户可通过该界面对混沌实验进行操作和观测。
- **TMEChaos Dashboard Backend**：NodeJS实现的Dashboard中间层，为Web提供Rest API接口，并进行TMEOA权限/微服务权限验证。
- **TMEChaos APIServer**：TMEChaos的逻辑组件，主要负责服务维度的爆炸半径设置，ChaosMesh多集群管理、实验状态管理、Mock行为注入。
- **Status Manager**：负责查询各ChaosMesh集群中的实验状态同步到TMEChaos的存储组件。
- **SteadyState Monitor**：稳态指标组件，负责监控混沌实验运行过程中IAAS层/服务相关指标，如果超出阈值自动终止相关实验。
- **ChaosMesh Dashboard API**：ChaosMesh 对外暴露的Rest API接口层，用于实验的增删改查，直接跟K8S APIServer交互。
- **Chaos Controller Manager**：ChaosMesh 的核心逻辑组件，主要负责混沌实验的调度与管理。该组件包含多个 CRD Controller，例如 Workflow Controller、Scheduler Controller 以及各类故障类型的 Controller。
- **Chaos Daemon**：ChaosMesh 的主要执行组件，以 DaemonSet 的方式运行。该组件主要通过侵入目标 Pod Namespace 的方式干扰具体的网络设备、文件系统、内核等。

#### 4.2. 全链路压测

上一节的混沌工程是通过注入故障的方式，来发现系统的脆弱环节。而全链路压测，则是通过注入流量给系统施加压力的方式，来发现系统的性能瓶颈，并帮助我们进行容量规划，以应对生产环境各种流量冲击。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZ5AV1Cd2JkxMtTT2J7DMsgzXdXvYYUmib3myTibxia99qUm4exXYdibRXXg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



全链路压测的核心模块有4个：流量构造、流量染色、压测引擎和智能监控。

- **流量构造**：为了保证真实性，从正式环境API网关抽样采集真实流水，同时也提供自定义流量构造。
- **流量染色**：全链路压测在生产环境进行。生产环境需要具备数据与流量隔离能力，不能影响到原有的用户数据、BI报表以及推荐算法等等。要实现这个目标，首先要求压测流量能被识别，我们采用的做法是在RPC的context里面增加染色标记，所有的压测流量都带有这个染色标记，并且这些标记能够随RPC调用关系进行传递。然后，业务系统根据标记识别压测流量，在存储时，将压测数据存储到影子存储而不是覆盖原有数据。
- **压测引擎**：对接各类协议，管理发压资源，以可调节的速率发送请求。
- **智能监控**：依靠可观测能力，收集链路数据并展示，根据熔断规则智能熔断。



### **5. 可观测性：**

随着微服务架构和基础设施变得越来越复杂，传统监控已经无法完整掌握系统的健康状况。此外，服务等级目标要求较小的故障恢复时间，所以我们需要具备快速发现和定位故障的能力。可观测性可以帮助我们解决这些问题。

指标、日志和链路追踪构成可观测的三大基石，为我们提供架构感知、瓶颈定位、故障溯源等能力。借助可观测性，我们可以对系统有更全面和精细的洞察，发现更深层次的系统问题，从而提升可用性。

在实践方面，目前业界已经有很多成熟的技术栈，包括Prometheus，Grafana，ELK，Jaeger等。基于这些技术栈，我们可以快速搭建起可观测系统。

#### 5.1. Metrics

指标监控能够从宏观上对系统的状态进行度量，借助QPS、成功率、延时、系统资源、业务指标等多维度监控来反映系统整体的健康状况和性能。

我们基于Prometheus构建联邦集群，实现千万指标的采集和存储，提供秒级监控，搭配Grafana做可视化展示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZUYfk95heBkAvHTV1mTcoOkGicicU6S01lHfaLR8VicWcsZGMHaNVO61iag/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



我们在微服务框架中重点提供四个黄金指标的观测：

- **Traffic（QPS）**
- **Latency（延时）**
- **Error（成功率）**
- **Staturation（资源利用率）**

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZNOMCe9cQbBsZiaibWPKWQqyqrCgLfI2cb1bribG3mPNucgwRzeIb64ib5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZ2zTLX5TVbHSQ2FwqmaPvouFlGNWoF2Hd3mhxJ6RTTNNebTd67WQd9g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)





![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZIaInAKDVETLFgWMMiaXJ7oaJ9lwVSbE82cnYwIR1yxCl1KhrkibDB9IQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



QQ音乐Metrics解决方案优势：

- **秒级监控**：QQ音乐存在多个高并发场景，数专、演唱会直播、明星空降评论/社区等，会带来比日常高出几倍甚至数十倍的流量，而且流量集中在活动开始头几秒，容量评估难度极大。我们通过搭建Prometheus联邦，每3秒抓取一次数据，实现了准实时监控。并对活动进行快照采集，记录活动发生时所有微服务的请求峰值，用于下次同级别艺人的峰值评估。
- **历史数据回溯**：QQ音乐海量用户及上万微服务，每天产生的数据量级很大。当我们需要回溯近一个月甚至一年前的指标趋势时，性能是个极大挑战。由于历史数据的精度要求不高，我们通过Prometheus联邦进行阶梯降采样，可以永久存放历史数据，同时也极大降低存储成本。

#### 5.2. Logging

随着业务体量壮大，机器数量庞大，使用SSH检索日志的方式效率低下。我们需要有专门的日志处理平台，从庞大的集群中收集日志并提供集中式的日志检索。同时我们希望业务接入日志处理平台的方式是无侵入的，不需要使用特定的日志打印组件。

我们使用ELK（ElasticSearch、Logstash、Kibana）构建日志处理平台，提供无侵入、集中式的远程日志采集和检索系统。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZdSgXhyibBpBe98KPFSRpVK6Jx0jvxlialZJQnDSdFwQyIcoU2bQ3QlPA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **Filebeat** 作为日志采集和传送器。Filebeat监视服务日志文件并将日志数据发送到Kafka。
- **Kafka** 在Filebeat和Logstash之间做解耦。
- **Logstash** 解析多种日志格式并发送给下游。
- **ElasticSearch** 存储Logstash处理后的数据，并建立索引以便快速检索。
- **Kibana** 是一个基于ElasticSearch查看日志的系统，可以使用查询语法来搜索日志，在查询时制定时间和日期范围或使用正则表达式来查找匹配的字符串。

下图为音乐馆首页服务的远程日志：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZXK6czF8Vpy1IhQQke4pxIvjunOV3j8WV4X1z8bmpv5lnda79llNrBg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 5.3. Tracing

在微服务架构的复杂分布式系统中，一个客户端请求由系统中大量微服务配合完成处理，这增加了定位问题的难度。如果一个下游服务返回错误，我们希望找到整个上游的调用链来帮助我们复现和解决问题，类似gdb的backtrace查看函数的调用栈帧和层级关系。

Tracing在触发第一个调用时生成关联标识Trace ID，我们可以通过RPC把它传递给所有的后续调用，就能关联整条调用链。Tracing还通过Span来表示调用链中的各个调用之间的关系。

我们基于jaeger构建分布式链路追踪系统，可以实现分布式架构下的事务追踪、性能分析、故障溯源、服务依赖拓扑。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZRP3z1QWPUTE6x7MQTf3sS5GbWDjQ0V9ic8Cvm5j23FWshoBjCP1MSKQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



- **jaeger-agent** 作为代理，把jaeger client发送的spans转发到jaeger-collector。
- **jaeger-collector** 接收来自jaeger-agent上报的数据，验证和清洗数据后转发至kafka。
- **jaeger-ingester** 从kafka消费数据，并存储到ElasticSearch。
- **jaeger-query** 封装用于从ElasticSearch中检索traces的APIs。

下图为音乐馆首页服务的链路图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZPCRtFt1LrTZue01viaMEn8vjFYSria7icE8Dpyt1hZtojibMjwNs98icXbg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 5.4. Profiles

想必大家都遇到过服务在凌晨三点出现CPU毛刺。一般情况下，我们需要增加pprof分析代码，然后等待问题复现，问题处理完后删掉pprof相关代码，效率底下。如果这是个偶现的问题，定位难度就更大。我们需要建设一个可在生产环境使用分析器的系统。

建设这个系统需要解决三个问题：

- 性能数据采集后需要持久化，方便回溯分析。
- 可视化检索和分析性能数据。
- 分析器在生产环境采集数据会有额外开销，需要合理采样。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZaPA88iafQMG4IKBRGLafPzZsbnEpxl3teaibqPicZWtNChGBpib7Dibicianw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



我们基于conprof搭建持续性能分析系统：

- 线上服务根据负载以及采样决定采集时机，并暴露profile接口。
- conprof定时将profile信息采集并存储。
- conprof提供统一的可视化平台检索和分析。

如下图，我们可以通过服务名、实例、时间区间来检索profile信息，每一个点对应一个记录。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZNS0iaibDDpktrwILpEPjHwIxk8OuoWv46EZFNP0uxnOnia7ibhRvFN1Elg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZiaTibPzTlFIxgRCkAJOYA9AAP5M9fGZjAmJv1v5tdpgWloia74XHhfDJw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#### 5.5. Dumps

传统的方式是在进程崩溃时把进程内存写入一个镜像中以供分析，或者把panic信息写到日志中。core dumps的方式在容器环境中实施困难，panic信息写入日志则容易被其他日志冲掉且感知太弱。QQ音乐使用的方式是在RPC框架中以拦截器的方式注入，发生panic后上报到sentry平台。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvatT4zrm9QGfBt43Pd4uicIVZtic76Mfm2RTvv6qTwqEbMb0RNTTApAHrUmJaSNVt5EONmZ1SkWEfIRw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



### **6. 总结**

本文从架构、工具链、可观测三个维度，介绍了QQ音乐多年来积累的高可用架构实践。先从架构出发，介绍了双中心容灾方案以及一系列稳定性策略。再从工具链维度，介绍如何通过工具平台对架构进行测试和风险管理。最后介绍如何通过可观测来提升架构可用性。这三个维度的子系统紧密联系，相互协同。架构的脆弱性是工具链和可观测性的建设动力，工具链和可观测性的不断完善又会反哺架构的可用性提升。

此外，QQ音乐微服务建设、Devops建设、容器化建设也是提升可用性的重要因素。单体应用不可用会导致所有的功能不可用，而微服务化按单一职责拆分服务，可以很好地处理服务不可用和功能降级问题。Devops把服务生命周期的管理自动化，通过持续集成、持续测试、持续发布等来降低人工失误的风险。容器化最大程度降低基础设施的影响，让我们能够将更多精力放在服务的可用性上，此外，资源隔离，HPA，健康检查等，也在一定程度上提升可用性。

至此，基础架构提供了各种高可用的能力，但可用性最终还是要回归业务架构本身。业务系统需要根据业务特性选择最优的可用性方案，并在系统架构中遵循一些原则，如最大限度减少关键依赖；幂等性等可重试设计；消除扩容瓶颈；预防和缓解流量峰值；过载时做好优雅降级等等。而更重要的一点是，我们需要时刻思考架构如何支撑业务的长期增长。



原文作者：brightnfeng，腾讯 QQ 音乐后台开发工程师

原文链接：https://mp.weixin.qq.com/s/G00cwGYAr6l2Px6-DiwXLA