# 【NO.366】即时通信IM核心能力及应用场景

本次分享的内容分为三块，一是腾讯云即时通信IM的产品概述，二是IM的核心功能特点，三是IM的应用场景介绍。

## **1.即时通信IM是什么**



即时通信IM是一款PaaS产品，以提供SDK的形式，集成至用户的APP或业务系统中，帮助用户快速实现类似QQ、微信那样的聊天能力。利用IM，用户可以实现APP内的单聊、群聊等稳定的消息传输能力；实现好友与黑名单等关系链管理能力；实现群成员与群资料等群组管理能力；实现聊天会话置顶、未读计数等会话管理能力。IM还开放了丰富的DEMO源码，最快1分钟即可跑通，再结合开源UI库（TUIKit），实现 UI 功能的同时调用 IM SDK 相应接口，仅需1天即可帮助用户搭建出自己的专属IM应用。

IM还为出海客户提供新加坡、韩国、德国、印度等海外数据中心，数据存储在当地，满足客户出海合规需求；IM覆盖了全球超过2500个加速节点，自研调度路由算法，“指挥”消息更快到达；支持业内独有的QUIC双通道能力，一条消息经过2路通道进行传输，双份保障，稳定不丢失，在网络质量差的部分国家和地区，消息依旧畅通无阻。IM出海详细指引请见出海专区文档（https://cloud.tencent.com/document/product/269/81906）

IM的底层还与实时音视频、直播等进行了打通，通过集成相应SDK可以轻松应对连麦PK、直播互动、音视频通话等热门音视频场景。除此之外，IM还可适配电商带货、在线课程、互动游戏、客服咨询、社交沟通、企业办公、医疗健康等众多场景，实现泛互、教育、金融、政企、IoT等全行业覆盖。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLgFk8wpEJlYkhIVynBW2d7qqCh7wvKq6BQs1hWlF4jhSch2cYPria6YQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 2.**即时通信IM核心能力**



### **2.1 消息传输&会话管理**

在消息传输中，IM支持多种消息类型，包括图片、文字、语音、短视频、表情、自定义消息等等，可以实现APP内的双人聊天，支持APP管理员在后台模拟其他用户身份发送消息或是下发系统消息。IM也支持类似QQ群、微信群的聊天方式，支持云端的消息存储，用户更换终端依然可以获取其聊天记录。在APP退出后台或进程被kill的情况下，如果有新的消息提醒，IM支持离线推送能力将这条消息推送给客户。IM还具有完备的会话管理能力，用户可以拉取最近会话列表，并可以对会话列表进行置顶、删除、清空等操作。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLquxbX0cws9ic2kXhd7UicwIibswUE2MatYnqLumQhjwJia6rdvql3cic6lQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



### **2.2 群组管理**

与大家平常使用的QQ群、微信群一样，IM可以提供丰富的群组资料管理能力，比如设置群公告、修改群组名称、修改群组简介等等，还支持修改用户群内身份、加群选项、接收群消息选项等等信息。IM还支持对群组及群成员进行管理，可以实现创建、转让、解散群组，可以进行添加/删除群成员、设置管理员、修改群成员资料等操作。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLm34Us0cmTxTPM5cJsdXZtRyoFVfefcHhQju2oG8ic2zrUYYicicnHrzDQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



### **2.3 用户资料及关系链**

IM支持昵称、头像、加好友验证方式等标准资料字段，也支持用户根据自身业务属性，将一些额外的自定义资料字段附加到用户资料上，并通过现有接口进行读写操作。在关系链方面，IM支持3000个好友，支持添加/删除/校验好友，支持添加/删除/拉取/校验黑名单。在非好友情况下，IM也可以支持用户之间相互聊天。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuL6rF3Bjqooj7VNmeLvM9IGEq2WCbud9tWDiaBRVgpFH1UA9bYxv49mzg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



最后，即时通信IM最核心的能力是后台系统的稳定性和抗并发能力。每月服务用户数超过10亿，消息收发成功率、服务可靠性高于99.99%。IM还提供人数无上限的音视频直播群，非常适用于音视频场景，并且支持多级扩散、冷热分离、多地容灾等技术。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuL04SuQibNQSZ5yWicZO7q6ePnb2wKCibfRib8Iicpb7xhInMt0Y9w1Wpl7dQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 3.**即时通信IM核心应用场景**



第一个场景是社交沟通。如果用户想要在APP中实现社交聊天，那么IM可以支持单聊/群聊中的文字、表情、图片、短语音、短视频等多种消息类型，有效提升用户活跃度。IM也支持丰富的群组类型，例如私有群、公开群、聊天室等，满足特定群聊场景。同时，IM还支持支持群头像、群昵称、群简介，群成员头像，群成员昵称等资料管理，并且支持最多20个自定义字段，实现群组等级展示、群组个性化展示等能力，轻松满足社交沟通场景下的用户需求。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLJl04ox0lp2jJtoKmEjcxOcMIU3xVWjlKib2cC8x4P5RiaxK8sqBB3gsw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第二个场景是直播互动。IM为音视频场景提供无人数上限的音视频聊天室，能够为百万级的直播保驾护航。日常直播中的点赞、送礼、打赏等能力，都是通过IM自定义消息实现的。以点赞为例，用户在应用上进行点赞后，点赞操作的次数将通过客户端上报至服务器，之后由服务器下发一条消息至直播群，告知点赞数量达到了多少，客户端接收后更新点赞数显示，就实现了直播点赞功能。IM还提供弹幕聊天功能，实时获取弹幕信息，通过自定义消息还能实现弹幕变色、悬停、加速以及图片弹幕等特殊效果。另外，在直播互动中，抽奖也是不可或缺的能力。IM的自定义消息可以将链接、文本、图片组合成一条消息下发，用户点击后就可以进行抽奖，轻松实现直播场景下的抽奖互动。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLLsQFN21njeUaRp6OBLiaeo84vTH4kmBqNe9wYjL88kyEgLlNgPBVdSg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第三个场景是电商带货。除了点赞送礼、弹幕消息、抽奖互动等能力外，IM的自定义消息和第三方回调能力还能帮助客户打通业务后台，实现领取优惠券，加购商品等能力。用户点击优惠券/加购后，系统自动实现流转。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLAxcjtfaZZgTmp1Q7ZMaQmb3x9iah6vbTeA6UFMYRGSKiaCsoxx4kt9xw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第四个场景是互动游戏。很多游戏的游戏内社交是通过IM实现的，例如游戏内常见的组队聊天、世界聊天等等。IM还能够提供创建游戏内社群的能力，支持群头像、群昵称、群简介，群成员头像、群成员昵称等资料的编辑，还可以通过自定义字段为群成员分配游戏定制版本的身份、等级、勋章……另外，对于很多出海游戏，IM支持全球消息互通，提供包括亚太、北美、欧洲、中东、非洲、拉美等覆盖全球的海外接入点与加速点，实现玩家就近接入，保障全球玩家通讯质量，并且提供了国际站的独立数据存储节点，保证客户数据安全合规。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLa09XJjpatexy7TicNZCyicw5QtXSj8qpPU9VGqPRuq1ZPpgloX2m7g0Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第五个场景是在线教育。IM提供了很多与教育场景适配的能力，比如开课提醒，就可以通过调用IM服务端的API，对所有成员进行消息提醒来实现。同时IM可以为在线教育提供强大的班级管理能力，老师或助教可以邀请成员进入课堂群，针对学员禁言/解禁或踢出课堂群。教师在上课过程中，可以利用IM的消息传输能力，向学生下发课件、PPT、文件、图片、视频等内容。举手发言/随堂测试/互动问答等在线课堂中常用的功能也都可以通过IM的自定义消息能力实现。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLI21QsrkAkHb5Ye4JjlBtItnshoHwNGrFUOuDJrYCxncSKUibokvBbNg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第六个场景是在线客服。这个场景的典型应用就是智能客服，IM通过第三方回调将消息实时抄送至业务或智能客服后台，得到答案后通过Rest API下发，就能够实现用户咨询后自动获取对应的答复，自动化完成客服工作。当然，用户对答复不满意的话也可以要求转人工，人工客服利用IM也可以和客户实现文字/语音/图片等多种形式的实时在线沟通。对于在线客服场景中存在的很多监管需求，IM支持商家或超管随时加入或离开顾客咨询群，实时监督客服服务质量，也支持消息下载与实时抄送，将客服与客户聊天记录保存本地，供监管抽查、考核。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLttdoQbibC4nEOibMcDgnwzcjIcDGJSFPnic4uL7YNK0PG5pick5UMhZkSQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第七个场景是企业通信。IM支持用户自定义字段，可自定义企业通信录信息及用户权限，允许非好友直接通信，满足超大型企业通信需要。通过IM的自定义消息能力，还可实现远程打卡、投票、企业云盘、在线文档、页面分享等多种定制化消息类型，并可将OA、e-HR等内部系统流程，通过系统消息方式发送给指定接收者，实现快速审批，全方位提升企业线上办公效率。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLNpwCiblgR9qP7UZ2QGDDiciboCkJKX3MnkAPwMCMMvZVosVkVgPWgHFkQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



第八个场景是在线会议。IM支持10万人大群，对于超大规模的万人级企业大会也能够满足需求。并且IM为会议提供强大的成员管理能力，支持禁言、踢人、设置联席主持人、邀请入会、禁止入会等多种功能。还可在会议过程中，通过IM的自定义消息能力将图片/文档/投票等会议相关内容分享至会议群内。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLu1ulZq6BTLJURpX9jMQyP7JFKrIAnEt8RUx2j9xriaZ0iaWMaLpWcCBw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



最后一个场景是商业沟通。在日常的打车、配送等服务中，都会涉及到服务双方的简单沟通。IM能够在服务人员接单/抢单后，通过REST API快速创建服务人员与用户的单聊/群聊，实现定向分配对接，同时自动下发订单信息，为双方提供互动沟通的平台。IM还支持发送实时位置信息，通过用户自定义字段，可实时获取并发送服务人员轨迹信息。用户可在应用中实时确认服务人员的位置轨迹，了解服务进度。另外，IM有很强的安全保护能力，能够满足商业沟通场景对信息保护和用户安全的要求。IM能够选择性拉取用户资料，仅展示必要信息，避免用户信息泄露，并通过第三方回调与实名认证服务打通，提高App安全性，还可在服务人员与客户建立群聊后，同步添加安全员，对双方使用者不可见，保护使用者安全。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAGzTolhJia9mclIf2GvByzuLpdS5MiaJvObfIicZCicbSqCV7dcORqrrlAsdNicOU7YZrmsriaav1IW9jGw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



IM还支持私有化部署。对于政务/金融/医疗等数据安全更高的领域，IM 可直接部署在客户本地，数据资产也全部放置在客户本地，保证系统及数据安全。对于想要将聊天能力嵌入到自有系统中的企业，企业微信、钉钉飞书这样的标准产品无法满足他们的需要，也可以通过IM的私有化部署，将IM SDK集成至自有系统，搭建自己的专属企业通信平台。

原文作者：腾讯云音视频产品经理——郑聪兴

原文链接：https://mp.weixin.qq.com/s/fmRkCQHtl-AyBVI0D_EkRg