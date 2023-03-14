# 【NO.236】万字详解：腾讯如何自研大规模知识图谱 Topbase

> Topbase 是由 TEG-AI 平台部构建并维护的一个专注于通用领域知识图谱，其涉及 226 种概念类型，共计 1 亿多实体，三元组数量达 22 亿。在技术上，Topbase 支持图谱的自动构建和数据的及时更新入库。此外，Topbase 还连续两次获得过知识图谱领域顶级赛事 KBP 的大奖。目前，Topbase 主要应用在微信搜一搜，信息流推荐以及智能问答产品。本文主要梳理 Topbase 构建过程中的技术经验，从 0 到 1 的介绍了构建过程中的重难点问题以及相应的解决方案，希望对图谱建设者有一定的借鉴意义。

### **1.简介**

知识图谱（ Knowledge Graph）以结构化的形式描述客观世界中概念、实体及其关系，便于计算机更好的管理、计算和理解互联网海量信息。通常结构化的知识是以图形式进行表示，图的节点表示语义符号（实体，概念），图的边表示符号之间的语义关系（如图 1 所示），此外每个实体还有一些非实体级别的边（通常称之为属性），如：人物的出生日期，主要成就等。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIQr7Kx5z63es9PsyQVaCohBNFFCfUPE81g37ePOdInHmTUmSHMCA68g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图1 知识图谱的示列

TEG-AI 平台部的 Topbase 是专注于通用领域知识。数据层面，TopBase 覆盖 51 个领域的知识，涉及 226 种概念类型，共计 1 亿多个实体，三元组数量达 22 亿多。技术层面，Topbase 已完成图谱自动构建和更新的整套流程，支持重点网站的监控，数据的及时更新入库，同时具备非结构化数据的抽取能力。此外，Topbase 还连续两次获得过知识图谱领域顶级赛事 KBP 的大奖，分别是 2017 年 KBP 实体链接的双项冠军，以及 2019 年 KBP 大赛第二名。在应用层面，Topbase 主要服务于微信搜一搜，信息流推荐以及智能问答产品。本文主要梳理 Topbase 构建过程中的重要技术点，介绍如何从 0 到 1 构建一个知识图谱，内容较长，建议先收藏。

### **2.知识图谱技术架构**

TopBase 的技术框架如图 2 所示，主要包括知识图谱体系构建，数据生产流程，运维监控系统以及存储查询系统。其中知识图谱体系是知识图谱的骨架，决定了我们采用什么样的方式来组织和表达知识，数据生产流程是知识图谱构建的核心内容，主要包括下载平台，抽取平台，知识规整模块，知识融合模块，知识推理模块，实体重要度计算模块等。Topbase 应用层涉及知识问答（基于 topbase 的 KB-QA 准确率超 90%），实体链接（2017 图谱顶级赛事 KBP 双料冠军），相关实体推荐等。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIeria6AMjSUL9vTKyobt9Q84tutPnMhbu7USvmLTUQUnvHwib18o4wrUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图2 知识图谱Topbase的技术框架

1. **下载平台-知识更新**：下载平台是知识图谱获取源数据平台，其主要任务包括新实体的发现和新实体信息的下载。
2. **抽取平台-知识抽取**：下载平台只负责爬取到网页的源代码内容，抽取平台需要从这些源码内容中生成结构化的知识，供后续流程进一步处理。
3. **知识规整**：通过抽取平台以及合作伙伴提供的数据我们可以得到大量的多源异构数据。为了方便对多源数据进行融合，知识规整环节需要对数据进行规整处理，将各路数据映射到我们的知识体系中。
4. **知识融合**：知识融合是对不同来源，不同结构的数据进行融合，其主要包括实体对齐和属性融合。
5. **知识推理**：由于处理数据的不完备性，上述流程构建的知识图谱会存在知识缺失现象（实体缺失，属性缺失）。知识推理目的是利用已有的知识图谱数据去推理缺失的知识，从而将这些知识补全。此外，由于已获取的数据中可能存在噪声，所以知识推理还可以用于已有知识的噪声检测，净化图谱数据。
6. **实体知名度计算**：最后，我们需要对每一个实体计算一个重要性分数，这样有助于更好的使用图谱数据。比如：名字叫李娜的人物有网球运动员，歌手，作家等，如果用户想通过图谱查询“李娜是谁”那么图谱应该返回最知名的李娜（网球运动员）。

### **3.知识体系构建**

知识体系的构建是指采用什么样的方式来组织和表达知识，核心是构建一个本体（或 schema）对目标知识进行描述。在这个本体中需要定义：1）知识的类别体系（如：图 1 中的人物类，娱乐人物，歌手等）；2）各类别体系下实体间所具有的关系和实体自身所具有的属性；3）不同关系或者属性的定义域，值域等约束信息（如：出生日期的属性值是 Date 类型，身高属性值应该是 Float 类型，简介应该是 String 类型等）。我们构建 Topbase 知识体系主要是以人工构建和自动挖掘的方式相结合，同时我们还大量借鉴现有的第三方知识体系或与之相关的资源，如：Schema.org、Dbpedia、大词林、百科（搜狗）等。知识体系构建的具体做法：

1. 首先是定义概念类别体系：概念类别体系如图 1 的概念层所示，我们将知识图谱要表达的知识按照层级结构的概念进行组织。在构建概念类别体系时，必须保证上层类别所表示的概念完全包含下层类别表示的概念，如娱乐人物是人物类的下层类别，那么所有的娱乐人物都是人物。在设计概念类别体系时，我们主要是参考 schema.org、DBpedia 等已有知识资源人工确定顶层的概念体系。同时，我们要保证概念类别体系的鲁棒性，便于维护和扩展，适应新的需求。除了人工精心维护设计的顶层概念类别体系，我们还设计了一套上下位关系挖掘系统，用于自动化构建大量的细粒度概念（或称之为上位词），如：《不能说的秘密》还具有细粒度的概念：“青春校园爱情电影”，“穿越电影”。
2. 其次是定义关系和属性：定义了概念类别体系之后我们还需要为每一个类别定义关系和属性。关系用于描述不同实体间的联系，如：夫妻关系（连接两个人物实体），作品关系（连接人物和作品实体）等；属性用于描述实体的内在特征，如人物类实体的出生日期，职业等。关系和属性的定义需要受概念类别体系的约束，下层需要继承上层的关系属性，例如所有歌手类实体应该都具有人物类的关系和属性。我们采用半自动的方式生成每个概念类别体系下的关系属性。我们通过获取百科 Infobox 信息，然后将实体分类到概念类别体系下，再针对各类别下的实体关系属性进行统计分析并人工审核之后确定该概念类别的关系属性。关系属性的定义也是一个不断完善积累的过程。
3. 定义约束：定义关系属性的约束信息可以保证数据的一致性，避免出现异常值，比如：年龄必须是 Int 类型且唯一（单值），演员作品的值是 String 类型且是多值。

### **4.下载平台-知识更新**

知识更新主要包括两方面内容，一个是新出现的热门实体，需要被及时发现和下载其信息，另一个是关系属性变化的情况需要对其值进行替换或者补充，如明星的婚姻恋爱关系等。知识更新的具体流程如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIVYJia9S9YV3mpKM41vc59PjZXRJx7QCeAvUz6HpIKt2NesWvnPd7xow/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图3 Topbase知识更新流程图

1. 针对热门实体信息的更新策略主要有：

- 从各大站点主页更新，定时遍历重点网站种子页，采用广搜的方式层层下载实体页面信息；
- 从新闻语料中更新，基于新闻正文文本中挖掘新实体，然后拼接实体名称生成百科 URL 下载；
- 从搜索 query log 中更新，通过挖掘 querylog 中的实体，然后拼接实体生成百科 URL 下载。基于 querylog 的实体挖掘算法主要是基于实体模板库和我们的 QQSEG-NER 工具；
- 从知识图谱已有数据中更新，知识图谱已有的重要度高的实体定期重新下载；
- 从人工运营中更新，将人工（业务）获得的 URL 送入下载平台获取实体信息；
- 从相关实体中更新，如果某个热门实体信息变更，则其相关实体信息也有可能变更，所以需要获得热门实体的相关实体，进行相应更新。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIic2ov3jLQHIvaOKibpa9QMEez6Mwt8ibfGNiaSpsoEv25mwJzEsfOvdx6g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

表 1 最近 7 日下载数据统计情况

2.针对其他关系属性易变的情况，我们针对某些重要关系属性进行专项更新。如明星等知名人物的婚姻感情关系我们主要通过事件挖掘的方式及时更新，如：离婚事件会触发已有关系“妻子”“丈夫”变化为“前妻”“前夫”，恋爱事件会触发“男友”“女友”关系等。此外，基于非结构化抽取平台获得的三元组信息也有助于更新实体的关系属性。

### **5.抽取平台 - 知识抽取**

Topbase 的抽取平台主要包括结构化抽取，非结构化抽取和专项抽取。其中结构化抽取主要负责抽取网页编辑者整理好的规则化知识，其准确率高，可以直接入库。由于结构化知识的局限性，大量的知识信息蕴含在纯文本内容中，因此非结构化抽取主要是从纯文本数据中挖掘知识弥补结构化抽取信息的不足。此外，某些重要的知识信息需要额外的设计专项策略进行抽取，比如：事件信息，上位词信息（概念），描述信息，别名信息等。这些重要的知识抽取我们统称专项抽取，针对不同专项的特点设计不同的抽取模块。

**1. 结构化抽取平台**

许多网站提供了大量的结构化数据，如（图 4 左）所示的百科 Infobox 信息。这种结构化知识很容易转化为三元组，如：“<姚明，妻子，叶莉>”。针对结构化数据的抽取，我们设计了基于 Xpath 解析的抽取平台，如（图 4 右）所示，我们只需要定义好抽取网页的种子页面如：baike.com,然后从网页源码中拷贝 Infobox 中属性的 xpath 路径即可实现结构化知识的自动抽取，入库。通过结构化抽取平台生成的数据准确率高，因此无需人工参与审核即可直接入库，它是知识图谱的重要数据来源。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFI08IvI3JpGg5FlnBcZyksfNyTXDMuAhm1oiag5O0HD7KicuIu2jszzPMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图4 Topbase结构化抽取平台的xpath配置界面

1. **非结构化抽取平台**

由于大量的知识是蕴含在纯文本中，为了弥补结构化抽取信息的不足，我们设计了非结构化抽取平台。非结构化抽取流程如图 5 所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIbC6I8YhOnh6pOD8BIXAmtPqXfyqDwHFX7yTBoQibdw4ZBatsOicoiaNNw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图5 Topbase非结构化抽取平台的技术框架

首先我们获取知识图谱中重要度高的实体名构建 Tri 树，然后回标新闻数据和百科正文数据，并将包含实体的句子作为候选抽取语料（新闻和百科数据需要区别对待，新闻数据往往包含最及时和最丰富的三元组信息，百科数据质量高，包含准确的知识，且百科摘要或正文描述相对简单，抽取结果的准确率高）。

然后，我们利用 Topbase 的实体链接服务，将匹配上的实体链接到知识库的已有实体中，避免了后期的数据融合。比如：实体“李娜”匹配到一句话是“歌手李娜最终归一了佛门”，那么这句话中的李娜会对应到知识库中的歌手李娜，而不是网球李娜，从这句话中抽取的结果只会影响歌手李娜的。实体链接之后，我们将候选语料送入我们的抽取服务，得到实体的三元组信息。

最后，三元组结果会和知识库中已有的三元组数据进行匹配并给每一个抽取得到的三元组结果进行置信度打分，如果知识库已经存在该三元组信息则过滤，如果知识库中三元组和抽取得到的三元组发生冲突则进入众包标注平台，如果三元组是新增的知识则根据他们的分值决定是否可以直接入库或者送入标注平台。此外，标注平台的结果数据会加入到抽取服务中 Fine-tune 模型，不断提升抽取模型的能力。

上述流程中的核心是抽取服务模块，它是非结构化抽取策略的集合。抽取服务构建流程如图 6 所示，其主要包括离线模型构建部分以及在线服务部分。离线模型构建的重点主要在于如何利用远监督的方式构建抽取模型的训练数据以及训练抽取模型。在线流程重点是如何针对输入的文本进行预处理，走不同的抽取策略，以及抽取结果的后处理。针对不同属性信息的特点，抽取策略主要可以简单归纳为三大类方法：

- 基于规则的抽取模块：有些属性具有很强的模板（规则）性质，所以可以通过人工简单的配置一些模板规则就可以获得高准确率的三元组结果。一般百科摘要文本内容描述规范，适合于规则抽取的输入数据源。此外，适用于规则抽取的属性主要有上位词，别名，地理位置，人物描述 tag 等。当然，规则模块召回有限往往还得搭配模型抽取模块，但是规则模块结果适合直接入库，无需标注人员审核。
- 基于 mention 识别+关系分类模块：基本思想是先用 NER 或者词典匹配等方式识别出句子中的 mention，然后利用已有的实体信息以及识别出来的 mention 进行属性分类。举例：给定识别出 mention 的句子“<org>腾讯</org>公司是由<per>马化腾</per>创立的。”,用 schema 对输入进行调整，一种情况是 org 作为头实体，per 作为尾实体，那么该样本的分类结果是关系“创始人”，另一种情况是 per 作为头实体，org 作为尾实体，那么该样本的分类结果是“所属公司”，所以最终可以得到三元组<腾讯，创始人，马化腾>和<马化腾，所属公司，腾讯>。一般人物，地点，机构，影视剧，时间等实体可以利用 qqseg-ner 识别。词典性质的实体如：职业，名族，国籍，性别等适合于词典匹配的方式识别。
- 基于序列标注模块：此外，还有许多属性值是无法进行 mention 识别，因此针对这类属性，我们采用一种序列标注的联合抽取方式来同时识别实体的属性值以及属性。这类属性主要有人物的“主要成就”信息，人物的描述 tag 信息，以及一些数值型属性信息。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIWibKjzickykjliajCUj26S09eAqDZsH1UFklwE6sNmB32XickSia09a2BJQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图6 Topbase的非结构化抽取服务

**3. 专项抽取**

专项抽取模块主要是针对一些重要知识的抽取。目前知识图谱设计的专项抽取内容主要有：上位词抽取（概念），实体描述抽取，事件抽取，别名抽取等。

**1 ) 上位词抽取**: 上位词可以理解为实体细粒度的概念，有助于更好的理解实体含义。图 7 是构建上位词图谱的一个简要流程图，其中主要从三路数据源中抽取上位词数据，主要包括：知识图谱的属性数据，百科人工标注 Tag，纯文本语料。由于抽取得到的上位词表述多样性问题，所以需要在抽取后进行同义上位词合并。此外，抽取生成的上位词图谱也会存在着知识补全的问题，所以需要进一步的进行图谱的连接预测，进行上位词图谱的补全。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFI2ObqZbOcaUJEmuw2dGIOM9CwW0GHgw8DMuXRg8VdwXeasLM6SKiatEQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图7 上位词抽取流程

**2) 实体描述 tag 抽取**: 实体描述 tag 是指能够描述实体某个标签的短句，图 7 是从新闻文本数据中挖掘到的实体“李子柒”的部分描述 tag。描述 tag 目前主要用于相关实体推荐理由生成，以及搜索场景中实体信息展示。描述 tag 抽取的核心模块以 QA-bert 为主的序列标注模型，query 是给定的实体信息，答案是句子中的描述片段。此外，还包括一系列的预处理过滤模块和后处理规整过滤模块。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFI2PqSPH8Y06UblmLt3t2uKMFWZVicqXLhJIE6NlJzs5GxMsfibrUmrVhQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图8  描述tag的示列说明

**3)事件抽取:** 事件抽取的目的是合并同一事件的新闻数据并从中识别出事件的关键信息生成事件的描述。事件抽取的基本流程如图 8 所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIUwt9m0STIpL8BEPoVTHpK4qttTozJibL4gicbh31cGsyUu7vnsUDFWqA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图9  Topbase的事件抽取流程框图

- 预处理阶段主要是对新闻流数据按照实体进行分堆处理。
- 事件聚类阶段主要是对每一堆的新闻数据进行关键词的提取等操作，将堆内的新闻进一步的聚类。
- 事件融合主要包括同批次事件融合和增量事件融合。事件抽取流程是分批次对输入数据进行处理。同批次事件融合主要解决不同实体属于同一事件的情况，将前一步得到的类簇进行合并处理。增量事件融合是将新增的新闻数据和历史 Base 的事件库进行增量融合。
- 最后，我们需要识别每一个事件类簇中的事件元素，过滤无效事件，生成事件的描述。

### **6.知识规整 - 实体分类**

知识规整目的是将实体数据映射到知识体系，并对其关系属性等信息进行去噪，归一化等预处理。如图 9 所示，左侧是从百科页面获取的武则天人物信息，右侧是从电影相关网站中获得的武则天信息，那么左侧的“武则天”应该被视为“人物类--历史人物--帝王”，右侧“武则天”应该被视为“作品--影视作品--电影”。左侧人物的“民族”属性的原始名称为“民族族群”，所以需要将其规整为 schema 定义的“民族”，这称之为属性归一。此外，由于不同来源的数据对实体名称会有不同的注释，如豆瓣的“武则天”这部电影后面加了一个年份备注，所以我们还需要对实体名进行还原处理等各种清洗处理。知识规整的核心模块是如何将实体映射到知识体系，即实体分类。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIKPTROibiazMfNJtIYYZ5D50AibdiaH2SKxajoC7PCwgeBp9z87BiabVYDvA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图10 数据规整的示列说明

**1. 实体分类的挑战**：

- 概念类别多（200+类），具有层次性，细分类别差异小（电影，电视剧）；
- 实体属性存在歧义：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIYVqmbP6c5nxN9uqVfTmAywlvq8rEmUuiccqaQ9Vbib3UiapoZ9Gt534jw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图11 实体分类中属性歧义问题

- 实体名称或者实体简介信息具有迷惑性：例如实体"菅直人"是一个政治家，其名称容易和民族类别混淆，电影“寄生虫”简介如下图所示，其内容和人物概念极其相似。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFI84cDMZCGtDLPPEtdWY2KKRSgumLm8NGBq3C0lpAC3Sc5t4hlibkKQ8g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图12 实体分类中简介迷惑性问题

**2.实体分类方法**：实体分类本质是一个多分类问题。针对知识库的特点以及上述挑战，我们分别从训练样本构建，特征选择以及模型设计三方面实现实体分类模块。

**1 ）实体分类的训练样本构建**：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFICqy6JYUbfXgWBTicSRSOTh7ib5bvob77WjjXIsYogBnEUEj60ictFvfwA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图13 实体分类训练数据构建流程

- **属性规则模块**：每个实体页面包含了实体结构化属性信息，利用这些属性字段可以对实体进行一个规则的分类。如：人物类别的实体大多包含民族，出生日期，职业等字段，歌手类实体的职业字段中可能有“歌手”的属性值。通过构建正则式规则，可以批量对实体页面进行分类。基于规则模块得到的类别信息准确率高，但是泛化能力弱，它的结果既可以作为后续分类模型的训练数据 1 也可以作为实体分类的一路重要分类结果。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIGQ59RIWVnvKEpC18VnTJ07qDiaQulAu6AlBBU91eicVoXhb2SIJwWib8A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图14 Topbase中用于实体分类的属性规则配置页面

- **简介分类模块**：简介分类模块以规则模块的数据作为训练数据，可以得到一个以简介为实体分类依据的分类模型，然后基于该模型预测属性规则模块无法识别的实体，选择高置信度的结果作为训练数据 2。
- **自动构建的训练数据去噪模块**：基于规则和简介分类模块可以得到部分分类样本，但是这些训练样本不可避免的会引入噪声，所以我们引入 N-折交叉训练预测自清洗数据，进一步保留高置信的训练样本，清洗思路如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIsvyYtasJUMRibLU4RLwicFPfctEyjib8OREc9whkKfkTEU94hkNB9o3Xg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图15 实体分类训练数据自清洗流程

- **运营模块**：运营模块主要包括日常 badcase 收集以及标注人员审核的预测置信度不高的样本。运营数据会结合自动构建数据，联合训练最终的实体分类模型。

**2） 实体分类的特征选择**：

- 属性名称：除了通用类的属性名称，如：中文名，别名，正文，简介等，其他属性名称都作为特征；
- 属性值：不是所有的属性值都是有助于实体分类，如性别的属性值“男”或者“女”对区分该实体是“商业人物”和“娱乐人物”没有帮助，但是职业的属性值如“歌手”“CEO”等对于实体的细类别则有很强的指示作用，这些属性值可以作为实体细分类的重要特征。一个属性值是否需要加入他的属性值信息，我们基于第一部分得到的训练数据，利用特征选择指标如卡方检验值，信息增益等进行筛选。
- 简介：由于简介内容相对较长且信息冗余，并非用得越多越好。针对简介的利用我们主要采用百科简介中头部几句话中的主语是该实体的句子。

**3） 实体分类模型**

- 模型架构：基于 bert 预训练语言模型的多 Label 分类模型

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIo31UXXCm3QhCO3Bet7dJsItWYEnLDVdITib6OuCQ5VIccMdjCddmbsg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图16 实体分类基础模型

- 模型输入：我们针对上述特征进行拼接作为 bert 的输入，利用[sep]隔开实体的两类信息，每一类信息用逗号隔开不同部分。第一类信息是实体名称和实体简介，刻画了实体的一个基本描述内容，第二类信息是实体的各种属性，刻画了实体的属性信息。例如，刘德华的输入形式如下：

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIicGj0Qu7AibDEDSEzDLicjpCWYZmA90JfN130rqcib7hx3nm9u1Pr7zr2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  图17  实体分类模型的输入形式

- 模型 loss：基于层次 loss 方式，实体 Label 是子类：父类 Label 要转换为正例计算 loss；实体 Label 是父类：所有子类 label 以一定概率 mask 不产生负例 loss，避免训练数据存在的细类别漏召回问题。

### 7.知识融合 - 实体对齐

知识融合的目的是将不同来源的数据进行合并处理。如从搜狗百科，体育页面以及 QQ 音乐都获取到了"姚明"信息，首先需要判断这些来源的"姚明"是否指同一实体，如果是同一个实体（图 18 中的搜狗和虎扑的姚明页面）则可以将他们的信息进行融合，如果不是（QQ 音乐的姚明页面）则不应该将其融合。知识融合的核心是实体对齐，即如何将不同来源的同一个实体进行合并。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIehnVRyExeb3oacaxkCMFBzkgrbVB99wlr96A42afHSw38xrdqJqbcg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIgFCPw7QHEiaOAL4q5rUbL0Fef1J6lv1icvAXkCluWTMumIicpficgFPkTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



图18  知识融合示列说明

\1. **实体对齐挑战**

- 不同来源实体的属性信息重叠少，导致相似度特征稀疏，容易欠融合；

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIf8XpTWHQM5F0MGrCv97BUWElzSeE8EyN90VK4DgoElmmicAqxxRaAMg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFISHPUJZThOPVdlf0ZDwFwabx5h7jUcnTA12LV6QAuPueWoygd94naSg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图19  来自于百科和旅游网站的武夷山页面信息

- 同系列作品（电影，电视剧）相似度高，容易过融合，如两部还珠格格电视剧

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIciauWHbgjghNPvR5RlhBsng1YJAbJZjIcIBSTAcFp19yQFsKriaJTCMA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIV2XLAGgYKNn4drFEl1SAFBa0pn5puJrw8Kwtywicj1hiaEM2OLCOvzlA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图20  两部还珠格格的信息内容

- 多路来源的实体信息量很大（亿级别页面），如果每次进行全局融合计算复杂度高，而且会产生融合实体的 ID 漂移问题。

\2. **实体对齐的解决思路**

实体对齐的整体流程如图所示，其主要环节包括数据分桶，桶内实体相似度计算，桶内实体的聚类融合。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIDoLoU9zWiaj0xIuLy0lA8e16O4d1PTHicsSfNGficqB631ZlOqyJGcpUA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图21  Topbase实体对齐流程图

**1)数据分桶**：数据分桶的目的是对所有的多源实体数据进行一个粗聚类，粗聚类的方法基于简单的规则对数据进行分桶，具体规则主要是同名（原名或者别名相同）实体分在一个桶内，除了基于名称匹配，我们还采用一些专有的属性值进行分桶，如出生年月和出生地一致的人物分在一个桶。

**2)实体相似度计算**：实体相似度直接决定了两个实体是否可以合并，它是实体对齐任务中的核心。为了解决相似属性稀疏导致的欠融合问题，我们引入异构网络向量化表示的特征，为了解决同系列作品极其相似的过融合问题，我们引入了互斥特征。

- 异构网络向量化表示特征：每个来源的数据可以构建一个同源实体关联网络，边是两个实体页面之间的超链接，如下图所示，百科空间可以构建一个百科实体关联网络，影视剧网站可以构建一个影视剧网站的实体关联网络。不同空间的两个实体，如果存在高重合度信息，容易判别二者相似度的两个实体，可以建立映射关系（如影视剧网站的梁朝伟页面和百科的梁朝伟页面信息基本一致，则可以认为二者是同一个实体，建立链接关系），这样可以将多源异构网络进行合并，梁朝伟和刘德华属于连接节点，两个无间道重合信息少，则作为两个独立的节点。然后基于 deepwalk 方式得到多源异构网络的节点向量化表示特征。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIDmmB6VLue7Q2EwMd5DMc85Gw0gJxLFIzkUSBBARra49cbJY75LZicJQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图22 多源异构网络关联图

- 文本相似特征：主要是针对存在简介信息的实体，利用 bert 编码得到向量，如果两个实体都存在简介信息，则将两个简介向量进行点乘得到他们的文本相似度特征；
- 基本特征：其他属性的相似度特征，每一维表示属性，每一维的值表示该属性值的一个 Jaccard 相似度；
- 互斥特征：主要解决同系列作品及其相似的问题，人工设定的重要区分度特征，如电视剧的集数，系列名，上映时间。
- 最后，按照下图结构将上述相似度特征进行融合预测两两实体是否是同一实体；

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIU1fo3uIxeeeNOAeLcg9zBJ2y1MsxYgkqicbXA30YEEJPUKEoZpic5xug/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图23 实体对相似度打分模块

**3) 相似实体的聚类合并：**

- **Base 融合**：在上述步骤的基础上，我们采用层次聚类算法，对每一个桶的实体进行对齐合并，得到 base 版的融合数据，然后赋予每一个融合后的实体一个固定的 ID 值，这就得到了一个 Base 的融合库；
- **增量融合**：对于每日新增的实体页面信息，我们不再重新进行聚类处理，而是采用“贴”的模式，将每一个新增实体页面和已有的融合实体进行相似度计算，判断该实体页面应该归到哪一个融合实体中，如果相似度都低于设置的阈值，则该新增实体独立成一堆，并设置一个新的融合实体 ID。增量融合的策略可以避免每次重复计算全量实体页面的融合过程，方便数据及时更新，同时保证各个融合实体的稳定性，不会轻易发生融合实体 ID 的漂移问题；
- **融合拆解**：由于 Base 融合可能存在噪声，所以我们增加了一个融合的修复模块，针对发现的 badcase，对以融合成堆的实体进行拆解重新融合，这样可以局部修复融合错误，方便运营以及批量处理 badcase。

### 8.知识关联和推理

**知识关联（链接预测）**是将实体的属性值链接到知识库的实体中，构建一条关系边，如图 24 所示“三国演义”的作者属性值是“罗贯中”字符串，知识关联需要将该属性值链接到知识库中的实体“罗贯中”，这样实体“三国演义”和“罗贯中”之间存在一条“作者”的关系边。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIfjQ6BsVT9GlAW3BO52tnicgRJtRqrr9tke4tALqWttdYiaQ9Xic1wTib1w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图24  基于超链接关联的示列说明

Topbase 的知识关联方案分为基于超链接的关联和基于 embedding 的文本关联两种方式。超链接关联是 Topbase 进行关联和推理的第一步，它是利用网页中存在的超链接对知识图谱中的实体进行关联，如百科“三国演义”页面中，其“作者”属性链接到“罗贯中”的百科页面（如图 24 所示），基于这种超链接的跳转关系，可以在 Topbase 的实体之间建立起一条边关系，如该示列会在实体“三国演义”与“罗贯中”之间生成一条“作者”关系，而“曹操”并没有该超链接，所以三国演义的主要人物属性中的字符串“曹操”不会关联到具体的实体页面中。在进行超链接关联之前，Topbase 中的实体是一个个孤立的个体，超链接关联为知识图谱补充了第一批边关系，但是超链接关联无法保证链接的覆盖率。

基于此，Topbase 提出基于 embedding 的文本关联。基于 embedding 的文本关联是在已知头实体、关系的基础上，在候选集中对尾实体进行筛选，尾实体的候选集是通过别名匹配召回。如上述百科示列中的“主要人物”属性，我们利用其属性值字符串”曹操“去 Topbase 库里匹配，召回所有和”曹操”同名称的实体作为建立链接关系的候选。然后利用知识库 embedding 的方法从候选实体中选择最相似的实体作为他的链接实体。基于文本名称的匹配召回候选可以大大提高知识库 embeding 方法的链接预测效果。基于 embedding 的链接关系预测是通过模型将实体和关系的属性信息、结构信息嵌入到一个低维向量中去，利用低维向量去对缺失的尾实体进行预测。

当前采用的嵌入模型是 TextEnhanced+TransE，模型结构如图 25 所示。TransE 是将实体与关系映射到同一向量空间下，它是依据已有的边关系结构对实体之间的边关系进行预测，对孤立实体或链接边较少的实体预测效果较差。为了引入文本信息，解决模型对孤立实体预测的难题，模型使用 TextEnhanced 对文本信息进行嵌入。TextEnhanced 通过 NN 模型对文本信息嵌入后，利用 Attention 机制将文本信息嵌入到 Trans 系列的实体向量中，进而对尾实体进行预测。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIznCR1ZCPgpvjLLHj1siaA28lPYUlXJpUq6EOloXibBBvVwf3N1VrDgQA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图25  TextEnhanced+TransE结构图

由于知识关联是在已知属性值的前提下，通过名称匹配的方式得到关联实体的候选集，所以知识关联无法补充缺失属性值的链接关系。如上图中“三国演义”的信息中并没有“关羽”，知识推理目的是希望能够挖掘“三国演义”和“关羽”的潜在关系。为了保证图谱数据的准确率，Topbase 的知识推理主要以规则推理为主，具体的规则方法可以归纳为以下几类：

- **伴随推理**是在已经被链接的两个实体之间，根据两个实体的属性信息，发现两者间蕴含的其它关系。例如实体 A 已经通过“配偶”关系与实体 B 相连，实体 A 的性别为“男”，实体 B 的性别为“女”，则伴随推理会生成一条“妻子”关系边，将实体 A 与实体 B 链接在一起，代表 B 为 A 的妻子。伴随推理的规则可以通过统计同时关联起两个实体的属性共现比例得到。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFI0qgK8samnuiclxr6tqXhsnuKNibE4fqK94BT5wlIALkVDc3tnjKyIwKA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图26  伴随推理的示列说明

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIPibuzCrQFcN5ePpeCJDzYkDxp3CAvDKXSJpfSHgcHY9l9icG5Ap50FhA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

表2 Topbase的伴随推理规则库示列

- **反向推理**是依据边之间的互反关系，为已经链接的两个实体再添加一条边。比如实体 A 通过“作者”边与实体 B 相连，代表实体 B 是实体 A 的作者，则可以直接生成一条从实体 B 指向实体 A 的“作品”边，代表实体 A 是实体 B 的作品，因为“作品”与“作者”是一条互反关系。反向推理与伴随推理类似，都是在已经存在边关系的实体之间，挖掘新的边关系，不同的是，伴随推理在生成边关系时需要满足一定的属性条件，如上例中的“性别”限制，而反向推理直接通过已有的边关系，无需参考其它属性值，直接生成一条互反边关系。反向推理规则可以通过统计 A-B，B-A 的属性共现数量筛选。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIugcoRt2CxvtgAVCvOHGibiaWzBUibUWSv27z798naer2JYibNERY6GetLw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图27  反向推理的示列说明

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIbiabsZ771BdSLyODJuLMuFvRUtVvXkHhV4UjW5mm4cxkJpukyx7ZyDQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

表3 Topbase的反向关联规则库示列

- **多实体推理**是在多个实体之间挖掘蕴含的边关系，是一种更复杂的关联规则，如第一种形式：A 的父亲是 B，B 的母亲是 C，则 A 的奶奶是 C，该形式通过统计 A+PATH = C，A+R0=C，情况得到规则  [PATH(R1R2)=R0]；第二种形式是 A 的母亲是 B，A 的儿子 C，则 B 的孙子是 C，该形式通过统计：A+R1 = B，A+R2=C，B+R0=C 的情况，得到规则[R1 &R2 = R0]。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFISFvibM0NBywwzbEI1Qus3aSE1ib7O1a0kFCEYwmg372lpQ4CsCuwia0UA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIiaYpOialwkX0prU4bibiag0ZpdGyvS7ojx038nQWkVkElXAfKfP7pCFdZg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图28 多实体推理的两种形式示列说明

### 9.实体知名度计算

实体的知名度（Popularity）指标可以用于量化不同实体的重要性程度，方便我们更好的使用图谱数据。Topbase 知识库的 popularity 计算以基于实体链接关系的 pagerank 算法为核心，以对新热实体的 popularity 调整为辅，并配以直接的人工干预来快速解决 badcase。具体地，首先抽取实体页面之间的超链接关系，以此为基础通过修改后的 pagerank 算法来计算所有实体的 popularity；对于难以通过 pagerank 算法计算的新热实体的 popularity，再进行规则干预。最后对于仍然难以解决的 case，则直接对其 popularity 值进行人工赋值。Popularity 计算模块的整体流程如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIupGBMAZsuchrLiaOM7Lv5dLP9tUpJCMyJxnKF2SlPEcBnyssr3cMk9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图29  Topbase实体知名度计算流程

- **多类型边关系的 pagerank 算法：** 基于链接关系的 popularity 计算方法的出发点在于：一个实体 A 对另一个实体 B 的引用（链接），表示实体 A 对于实体 B 的认可，链接到 B 的实体越多，表示 B 受到的认可越多，由此推断它的知名度也就越高。但实际上有很多的链接关系并不是出于“认可”而产生的，只是简单的表示它们之间有某种关系。比如歌手与专辑、音乐之间的各种关系。一个专业的音乐网站会收录歌手、专辑、音乐之间的完整从属关系，这会导致同一个歌手或同一张专辑之内的热门歌曲与其它歌曲之间没有任何区分性。并且由于这几类实体之间高密度的链接关系，会导致它们的计算结果比其它类别的实体的都高出很多。

  因此有必要对实体之间不同的链接关系进行区别对待。与最基础的 pagerank 算法的不同在于：实体之间可以有多条边，且有多种类型的边。在进行迭代计算的过程中，不同类型的边对流经它的概率分布会有不同程度的拟制作用。之所以进行这样的修改，是因为知识库中实体的信息有多种不同的来源。有的实体来源于通用领域百科，有的实体来源于垂类领域网站等。甚至同一个实体内部，不同的属性信息也会有不同的来源。由此，实体之间的链接关系也会属于不同的来源。比如“刘德华”与“朱丽倩”之间的“夫妻”关系可能抽取自百科，而与“无间道”之间的“参演”关系可能来自于电影网站。不同来源的信息有着不同的可信度，有的经过人工的审核编辑，可信度很高；而有的则属于算法自动生成，会有不同程度的错误。

  因此链接关系之间也有可信度的差别，无法做到将它们一视同仁地看待。其次，有的链接关系即使在可靠性方面完全正确，但它们对于 popularity 的正确计算不仅没有太大帮助，反而会导致 popularity 的计算结果与预期不符。修改后的 pagerank 算法的计算过程与基础 pagerank 算法基本一致，只是在进行分布概率的流转时有所区别。下面进行举例说明：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIYozR3iaczaodQic8JCuvYsbEKudBIY9COTg5wialfobSSDDvDmJiakfVgQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图30  多类型边的PageRank算法说明

实体 A 指向实体 B、C、D。其与 B 之间的链接类型为 X，与 C 之间的链接类型为 Y，与 D 之间的为 Z。通过先验知识或实验总结，我们认为链接类型 Y 可信性不高，相比于 X，对 rank 值的流转有拟制作用，因此对其赋予一个系数 0.8，Z 的可信度很准确，但其性质与上述的音乐网站的关系类似，因此对于其赋予一个系数 0.2，而 X 类型的完全可行，其系数则为 1.0。在某一迭代阶段，实体 A 的 rank 值为 3，B、C、D 的 rank 值分别为 4、2、3。由于 A 有 3 条出边，因此到 B、C、D 的初始流出值均为 3/ 3 = 1。加上系数的影响，实际到 C、D 的流出值分别为 0.8 和 0.2，未流出的剩余值为(1 -0.8) + (1 - 0.2) = 1.0。

因此迭代过后，B、C、D 的 rank 值分别为 4 + 1.0 = 5，2 + 0.8= 2.8，3 + 0.2 =3.2，而 A 的 rank 值需要在所有指向它的实体流入到它的值之和的基础上，再加上未流出的 1.0。

- **新热实体的 Popularity 调整：**新热实体的含义为最新出现的热门实体。这类实体需要较高的 popularity 值。但由于是新近出现的实体，其与其它实体的链接关系非常匮乏，因此无法通过基于实体链接关系的这类方法来计算。对此我们采取的方案侧重于对新热实体的发现，然后对发现的新热实体的 popularity 进行调整，使其 popularity 值在同名实体中处于最高的位置。新热实体的发现目前基于两类方法：一类方法发现的热门实体可以直接对应到知识库中的某个实体，另一个方法只能发现热门的实体名，需要通过一些对齐方法与知识库中的某个实体关联起来。

  第一种方法从 Topbase 监控的重点网站页面中直接获取最近热门的实体。这种方法获取的实体可以直接通过 url 与知识库中的某个实体准确无误地关联起来。第二类方法首先发现一些热门的实体名，包括：一、从微博热搜榜中爬取热门话题，通过命名实体识别方法识别其中的人名和机构名，将其作为热门实体名；二、将新闻中每天曝光的高频次标签作为实体名。以上两种方法发现的实体名带有一定的附加信息，通过实体链接可以将其对齐到知识库中的某个实体。

### 10.知识库的存储和查询

知识图谱是一种典型的图结构数据集合，实体是图中的节点，关系（属性）是带有标签的边。因此，基于图结构的存储方式能够直接正确地反映知识图谱的内部结构，有利于知识的查询。如下图所示，红色圈代表实体，实线是边（妻子），表示实体间的关系，如“刘德华的妻子是朱丽倩”，*虚线*是属性（出生日期），表示实体具有的属性，如“刘德华的出生日期是 1961 年 9 月 27 日”。

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIEmOjlh67LKWLo7828VZIJgkSEXaH3FhxMqP81DJnNRwU6UGK3M6vBA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图31 图数据说明

Topbase 知识图谱的存储是基于分布式图数据库 JanusGraph，选择 JanusGraph 的主要理由有：1）JanusGraph 完全开源，像 Neo4j 并非完全开源；2）JanusGraph 支持超大图，图规模可以根据集群大小调整；3）JanusGraph 支持超大规模并发事务和可操作图运算，能够毫秒级的响应在海量图数据上的复杂的遍历查询操作等。

**Topbase**基于**JanusGraph**存储查询架构如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvav5tj5KtuTS1I7BMtgFSSFIEyxYWjwdtknPqw1xdXfLib6RNoZIE9NBtcicB19BxxH1uENjaIicv4Qvg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

图32  基于JanusGraph的存储查询系统

- Graph_Loader 模块主要是将上述数据生产流程得到的图谱数据转换为 JanusGraph 存储要求的格式，批量的将图谱数据写入图数据库存储服务中，以及相关索引建立。
- 图数据库存储服务：**JanusGraph**数据存储服务可以选用 ScyllaDb、HBase 等作为底层存储，topbase 选用的是 ScyllaDb。Graph_loader 会每天定时的将数据更新到图数据库存储服务。
- 图数据库索引：由于 JanusGraph 图数据库存储服务只支持一些简单查询，如：“刘德华的歌曲”，但是无法支持复杂查询，如多条件查询：“刘德华的 1999 年发表的粤语歌曲”。所以我们利用 Es 构建复杂查询的数据索引，graph_loader 除了批量写入数据到底层存储之外，还会建立基于复杂查询的索引。
- 图数据库主服务：主服务通过 Gremlin 语句对图数据库的相关内容进行查询或者改写等操作。

### **11.总结**

由于知识图谱的构建是一项庞大的数据工程，其中各环节涉及的技术细节无法在一篇文档中面面俱到。本文主要梳理 Topbase 构建过程中的技术经验，从 0 到 1 的介绍了图谱构建流程，希望对图谱建设者有一定的借鉴意义。

原文作者：郑孙聪，腾讯 TEG 应用研究员

原文链接：https://mp.weixin.qq.com/s/Qp6w7uFcgqKXzM7dWhYwFg