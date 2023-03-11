# 【NO.170】五分钟了解互联网Web技术发展史

> 1991年8月，第一个静态页面诞生了，这是由Tim Berners-Lee发布的，想要告诉人们什么是万维网。从静态页面到Ajax技术，从Server Side Render到React Server Components，历史的车轮滚滚向前，一个又一个技术诞生和沉寂。

## 0.**前言**

1994年，万维网联盟（W3C，World Wide Web Consortium）成立，超文本标记语言（HTML，Hyper Text Markup Language）正式确立为网页标准语言，我们的旅途从此开始。

本文将沿着时间线，从“**发现问题-解决问题**”的角度，带领大家了解 Web 技术发展的关键历程，了解典型技术的诞生以及技术更迭的缘由，思考技术发展的原因。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I7ZdRa6qwIPGoaZAfaIboDjnIMFWbxEKsoK5gYtibmy6pjaA3jpyDOVg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 1.**Tim Berners-Lee**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IJ0SRczTJFYFE4LR0hJXPmXkmpiamsVGr0Mdk95kick7ugIibPiaxtOYo6Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

Tim Berners-Lee（蒂姆·伯纳斯·李），英国科学家，万维网之父，于1989年在欧洲核子研究组织（CERN）正式提出万维网的设想。该网络最初是**为了满足世界各地大学和研究所的科学家之间对自动信息共享**的需求而设计和开发的，这也是为什么HTML的顶层声明是 `document`，标签名、文档对象模型的名称也是由此而来。

1990年12月，他开发出了世界上第一个网页浏览器。1993年4月30日，欧洲核子研究组织将万维网软件置于公共领域，把万维网推广到全世界，让万维网科技获得迅速的发展，深深改变了人类的生活面貌。

他创造了超文本标记语言（HTML），并创建了历史上第一个网站。当然，现在只剩下了由 CERN 恢复的网站副本：[info.cern.ch](http://info.cern.ch/).

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IAo9QmsQYMLMI0hib00yRQ92gbOCsMA2ojbhkRsNHJq7o8lyzp1P2PGg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.**静态网页时代**

早期的静态网页，只有最基本的单栏布局，HTML所支持的标签也仅有`<h1>`、`<p>`、`<a>`。后来为了丰富网页的内容，`<img>`、`<table>`标签诞生了。

这一阶段，Web服务器基本上只是一个静态资源服务器，每当客户端浏览器发来访问请求，它都来者不拒的建立连接，查找URL指向的静态页面，再返回给客户端。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IespBSpLf6JkjnibkzicM3o2ibqxN7qHDxlFukicrOYXQsiaXaT6G9WSTaUQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

随着网页的飞速发展，人们发现要**人工实现所有信息的编写是非常困难**的，而且非常耗时。

设想一下，假如一个页面有两块区域展示的内容是互相独立的，那么你需要涵盖所有的可能，需要编写的页面数量是两块区域的内容数量的乘积！

此外，静态网站只能够根据用户的请求返回指向的网页，除了进行超链接跳转，没办法实现任何交互。

此时，人们想要

- **网页能够动态显示**
- **直接使用数据库里的数据**
- **网页实现一些用户交互**
- **让页面更美观**

## 3.**JavaScript的诞生**

1994年，网景公司发布了 Navigator 浏览器，但他们急需一种网页脚本语言，以使浏览器可以与网页互动。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I94iclLsWWgn71g9Az0Hv9R71GqQZO7BjNULcLDPedRDSV9lS8yW116Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

1995年，网景公司的 Brendan Eich 迫于公司的压力，只花了十天就设计了 JS 的最初版本，并命名为 Mocha。后来网景公司为了蹭 Java 的热度，把 JS 最终改名为 JavaScript。但实际情况是，网景公司和Sun公司结成联盟，才更名为 JavaScript。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IDz8bq0bofr6pXAoTiaKQMTEJianql6UNicrjlZbBFnPrn5ZLibMuQqc4Sg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**从此网页有了一些简单的用户交互**，比如表单验证；也**有了一些JS为基础的动效**，如走马灯。

但是让网页真正开始进入动态网页时代的却是以 PHP 为代表的后端网站技术。

### 3.1 扩展资料：第一次浏览器大战

在网景公司推出JavaScript的时候，微软以JS为基础，编写了JScript和VBScript作为浏览器语言，并在 1995 年的 8 月推出了 IE 1.0。

由于微软在系统里捆绑浏览器，而 90% 的人都在使用 Windows 操作系统，大量用户被动地选择了IE。面对微软快速抢占浏览器份额，网景公司无奈之下只能快速将 JavaScript 向 ECMA 提交标准，制定了 ECMAScript 标准。

> 在这段时间，还发生过一件趣事，IE 4.0 发布当天 Netscape 的员工们发现公司的草坪上出现了一个大大的 IE 图标，这明显是一个挑衅的举动。作为回应，Netscape 把自己的吉祥物 “Mozilla” 放在 IE 的图标上，并挂上胸牌，写着 “Netscape 72，Microsoft 18”——在当时， IE 的市场份额确实不如 Netscape Navigator。
>
> ![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IacHxZcqianZ8g3SOmLk3HGb9T0X1uic1ug3077q6XnWnENpM0dAOaIGg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

但这无法解决份额的问题，网景公司最终在第一次浏览器大战中落败，于1998年，被美国在线（AOL）以42亿美元收购。

在1998年网景公司被收购前，网景公司公开了 Navigator 源代码，想通过广大程序员的参与重新获得市场份额。Navigator 改名为 Mozilla。这也是火狐浏览器的由来，也是第二次浏览器大战的伏笔。

## 4.**CSS**

1994年，Hkon Wium Lie 最初提出了 CSS 的想法。1996年12月，W3C 推出了 CSS 规范的第一版本。

美观是所有人的追求。HTML诞生以来，网页基本上就是一个简陋的富文本容器。由于缺少布局和美化手段，早期网页流行用table标签进行布局。为了解决网页“丑”的问题，Hkon Wium Lie 和 Bert Bos 共同起草了 CSS 提案，同期的 W3C 也对这个很感兴趣。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IWtwEeiaMbReicogrTTYIPyqe8m6Gy4ENjQQ5OJo8WicE6syNWLsmOiaTvw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)早期网页外观

早期的 CSS 存在多种版本，在PSL96版本你甚至可以在里面使用逻辑表达式。但因为它太容易扩展，浏览器厂商那么多，会变得很难统一，最终被放弃。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I6bCH6s9PIehNmtGcVq1emrdYAvChDjdGEUg5fY9A9KMfOsKxPrM6tg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在众多提案中，Håkon W Lie 的 CHSS（Cascading HTML Style Sheets）最早提出了**样式表可叠加**的概念。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IVKBqa80XNnjPYcCyXxs2OptjElpQd6oDNU0k173fFwqicicwM26pwL9g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

行尾的百分比表示这条样式的权重，最终将根据权重计算最终值。图中将会计算 `30pt * 40% + 20pt * 60%` 作为h2字体大小的最终值。

为了解决 CSS 兼容性的问题，网景公司甚至还将 CSS 用 JS 来编写。

CSS 从诞生开始就伴随着大量的bug，不同浏览器表现不同坑害了无数的程序员。今天我们能用上相对靠谱的 css，不得不说这是一个奇迹。

## 5.**动态网页技术**

1995年，Rasmus Lerdof 创造的 PHP 开始活跃在各大网站，它让 Web 可以访问数据库了，PHP 实现了人们渴望的动态网页。

这里的动态网页不是指网页动效，而是指内容的动态展示、丰富的用户交互。PHP 就像给网络世界打开了一扇窗，各种动态网页技术（如ASP、JSP）雨后春笋般的冒了出来，万维网也因此开始高速发展，MVC模式也开始出现在后端网站技术中。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IOFdia6jMafxNRrviagOHlO4wbq94LGmibuXiaO9cic3Pc1VYMBvKbfQwlXg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

动态网页技术解决了以前各种令人无法呼吸的痛，生活总会越来越好的：

- **可以用数据库作为基础来展示网页内容**
- **可以实现表单和一些简单交互**
- **再也不用编写一大堆静态页面了**

PHP等动态网页技术的原理，大体上都是根据客户端的请求，从数据库里获取相对应的数据，然后塞到网页里去，返回给客户端一个填充好内容的网页。**这个阶段也是前后端耦合的**。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6Inia4DBsDlKOE4iciaaoYbe4bQ2cwyrbXj7vOrh8PfxJrXYDDICpSvbibdw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)网页开发流程

而当一些基础的需求被满足之后，**动态网页技术带来的不足也渐渐暴露出来**：

- **网页总是刷新**。用户名密码校验需要刷新以展示错误提示；因下拉选择器选择不同而展示的内容需要刷新才能展示；每次数据交互必然会刷新一次页面。
- **网页和后端逻辑混合**。相信老前端们都有过这样的经历：开发完HTML后，会把页面发给后端修改，加上数据注入逻辑；联调或者debug的时候两个人坐在一块看，查问题的效率很低。
- **有大量重复代码无法复用**。举一个典型的例子，论坛。很多时候只有内容有变化，菜单、侧边栏等几乎不会有改变，但每次请求的时候还是得再将整个网页传输一遍。不仅页面会刷新，速度慢，还挺耗流量（这个年代上网也是一种奢侈）。

然后AJAX站了出来。

## 6.**AJAX**

AJAX，Async JavaScript And XML，于1998年开始初步应用，2005年开始普及。AJAX的广泛使用，标志着Web2.0时代的开启。这同时也是各大浏览器争锋的时代。

现在，我们可以通过AJAX来动态获取数据，利用DOM操作动态更新网页内容了。来看看加入了AJAX的网页是怎么工作的：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IzrdN2b8M42ibrhUWuFT99pDmXp2IGUYZYL2OH1BaYbzu12Wics0q9CibQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这个时候前端路由还没有兴起，大多数情况下还是后端返回一整个页面，部分内容通过AJAX进行获取。

随着智能手机的出现，APP开始萌芽。相比起网页，APP编写好之后只需要数据接口就能工作；而网页不仅需要后端写业务逻辑，控制跳转，还要写一部分接口用于AJAX请求。

这个阶段前端能做的事情还是很少，还背负着“切图仔”的绰号。随着HTML5草案的提出，前端能做的交互越来越多，程序员们急需解决以下问题：

- **后端业务代码和数据接口混合**，还得兼容APP的接口（很多企业既有APP又有网站）
- **前端的代码复杂度急剧增加**

能不能让前端也像APP一样，只需要请求数据接口即可展现内容呢？

### 6.1 扩展资料：第二次浏览器大战

2004年 Firefox 发布，拉开了第二次浏览器大战的序幕。同期市面上诞生的各种新兴浏览器，如Safari、Chrome等，也加入了战争。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IpuVgictTujkGNxzkLPKWGZoU9tcMic3PDubQ9JEzXZPmMhpC3icYoQwTA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

此前由于 XP 系统实在过于火爆，导致 IE 6 无任何竞争对手，微软甚至解散了浏览器的大部分员工，只留下几个人象征性地维护顺便修补一下 bug。这让开发人员非常痛苦。

此时 Firefox 以优越于IE的性能和非常友好的编程工具，迅速将那些被 IE6 搞得焦头烂额的网页开发人员们，从水火之中救出，导致先让前端工程师成为忠实的第一批用户，然后，经由这些有经验的开发人员们推广到了普通的用户群体。

基于webkit内核的Safari，借助自家产品（iOS、MacOS）的垄断快速收割移动端和mac端市场份额；同样基于webkit内核的Chrome，趁着微软放松警惕，凭借优越于市场上所有浏览器的性能，如同中国历史上的成吉思汗一样大杀四方，快速扩展市场份额。

微软知道，自己已经失去了最初能称霸的机会，这次它不想失去，IE再次开始迭代，各大浏览器厂商又开始不顾标准，迭代再次开始，为了统一化标准，W3C开发了HTML5，但是迟迟得不到微软的认可。在其他浏览器纷纷支持HTML5后，微软发现，自己又成了孤家寡人，份额不断缩水。

2016年，Chrome浏览器份额超越IE，第二次浏览器大战结束。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I80nrFpnjwQaQEsszLP6ASkQicpU6xgHuice950cYRgbksm360ODsovmg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

浏览器大战极大的推动了技术进步，正是 Google 研发出的 V8 引擎极大的提升了 JS 的运行效率，NodeJS 才有机会诞生，前端才能走向全栈。**JS其实没有你想象的那么慢**。

## 7.**SPA**

2008年HTML5草案提出，各大浏览器开启良性竞争，争先实现HTML5功能。由于HTML5带来前端**代码复杂度的增加**，前端为了寻求良好的可维护性和可复用性，也不得不参考后端MVC进行了设计和拆分，后来出现了三大前端框架：Vue（2014）、React（2010）、AngularJS（2009）。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I1NrNum63v0YGSFhnLMibl9zzxCicIHCw9UTL1via9HQQGpibic4t1nhEpXw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

单页应用返回一个空白的HTML，并通过JS脚本进行动态生成内容，从此和页面刷新说拜拜。

后端不再负责模板渲染，前端和APP开始对等，后端的API也可以通用化了。**前后端终于得以分离**。（PS: 最终目标是成为后端）

但SPA因为返回的是空HTML，所有JS也被打包为一个文件，需要在一开始就加载完所有的资源，

- 请求网页后**白屏时间比传统网页要长**
- 爬虫爬到的是空白页面，**没办法做SEO**
- 在业务复杂的情况下，**请求文件很大，渲染非常慢**。

这使得前端不得不拆分过于庞大的单页应用，出现了框架的多页面概念，也出现了多种解决方案。

很多网页首次加载的时候其实并不需要太多的东西，比如论坛首页与贴子详情页，完全可以将其拆开，用户在新打开的页面阅读反而体验更好（**多页应用**）。

又比如管理后台，可以在页面框架内，将每个菜单对应的管理页拆出来**动态加载**（import）。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I5S6gMLqDmjenKhnVTMsiarGquCY1PIVNibAtQJxo5X4nfvncC9jWd6cw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 8.**Server Side Render**

Server Side Render，服务端渲染，简称SSR，又称服务端同构、直出，一般使用NodeJS实现。

这里的服务端渲染和以前的不一样，SSR会利用已经“脱水”的首屏数据来渲染首屏页面返回给客户端，到了浏览器再注入浏览器事件，并且保留单页应用的能力，对SEO非常友好。但学习成本高，限制较多。

让我们看看传统SPA和加入了SSR的SPA在请求上的区别：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I6ricTtAsfD7LeGBcaaJ3m0gsT61TRvZP8P9bl0MX11UC131ZMneQ19g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)客户端渲染示意

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IRvlUJTlL3WicQzSd3WMRBTIogLic35dvkECRG0H3AX0SdiccxwEHibibficQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)服务端渲染示意

传统SPA可以更快的返回页面，请求响应时间更短；加载JS后才开始渲染，白屏时间更长，loading结束后用户感知到的相对可交互时间更早。

而SSR在接到浏览器请求时，先从后端拉取首屏数据渲染在页面内才返回，请求响应时间更长；因为节约了一段浏览器请求首屏数据的时间，白屏时间更短。由于JS异步加载，用户感知的相对可交互时间变晚。但体验上SSR一般更好。

在极端情况下，用户眼中传统SPA会一直显示loading，使用了SSR的页面则会出现“点不动”的情况。

大多数时候SSR体验会更佳，因为服务端承担了大部分渲染工作，这也导致服务端负载变高。但在业务复杂的情况下，SSR首屏请求的接口数很多，导致返回HTML变慢。

归根结底，SSR不能很好的应付业务复杂的情况，首屏**要加载的东西还是太多了**。所以我们要怎样让用户感知到的白屏时间变短呢？

- **减小加载体积**
- **减少接口请求数**
- **PWA缓存**
- **分块渲染**
- …

IMWEB的企鹅辅导落地了 SSR + PWA 之后，达到了几乎秒开的程度。

## 9.**NodeJS**

说完了 SSR，必须说一下 NodeJS。2010年 NodeJS 正式立项到现在已经11个年头了，NodeJS 的诞生来自于 Ryan Dahl（下图） 的灵感。他想**以非阻塞的方式做所有事情**，用完全异步方式可以处理非常多的请求（高并发）。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IIetS62OMxc3kJjTbMCRViafdHZowibUAdTs9YV1SRPeJzhEzXDhjxNrg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

NodeJS 的出现让前端向全栈的发展迈出了重大的一步。很多公司开始用 NodeJS 搞 BFF（backend for frontend），我们也开始把 Controller 层放到 NodeJS 来处理，后端只负责基础业务数据。也就是现在的三层架构：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IzJRG41uVTawGzjZiaibZKqmN5QYtOeYy2vs5NfMXxqCYYy2h7YYHyfjQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这种架构在跨端的时候具有良好的适配性，我们可以根据业务需求，为不同端设计不同的 Controller 和 View，而后台可以不做变更。这种架构省去了很多沟通成本，前端专注页面的展示，后端专注业务逻辑。

当然，NodeJS 还可以对后端数据进行预处理，前端根据自己的需要自己设计数据结构，**页面开发与接口调试形成闭环**，还为后端分担了压力。

### 9.1 扩展资料：第三次浏览器大战

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IT1DgEJp5bJicqVQk3mbLRZiaqCH5SicGfMFbU7hppkBXZkKbQ7ntsO1Ww/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)智能手机的飞速发展，这张图表现的淋漓尽致。第三次浏览器大战是争夺移动端市场份额的一战，也是当下正在进行的一战。

> Benedict Evans: “Mobile is eating the world.”（移动设备正在蚕食世界） “Mobile remakes the Internet.”（移动设备正在重构Internet）

而未来，浏览器真正的对手不再是浏览器，而是小程序这样结合了APP和网页优点的新兴技术。

## 10.**未来**

早在2009年，Facebook的工程师就开发了[bigPipe](https://zhuanlan.zhihu.com/p/82398532)，让Facebook页面打开速度提高了两倍。bigPipe使用 **分块渲染** 的思想，将网页的渲染变成了一小块一小块的，服务端渲染好一块页面就发送给客户端。他们直接把木桶拆了，打破了**短板效应**。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/j3gficicyOvavO9wC2Qp77snsNmCibPmX6I7yyvguYXVdN0JY4MpO51QMTRT9WQV2MvqCPI3iaxqSzjjgrKjtJFpvg/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)服务端渲染 VS 流式分块渲染

时隔11年，也就是2020年12月，React 团队提出了 React Server Components，算是一个可扩展的前后端融合方案。其理念和 bigPipe 类似，把组件放在服务端渲染，节省了从浏览器进行数据请求的开支，一些运行时也可以不用放到浏览器，减小了包大小（如 markdown 在服务端渲染好了，也就不再需要把工具库发送给浏览器了）。React Server Components 的引入，也同步做到了自动的 Code Split。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IyZnszQEcgtMLJpF5391vP1ZolXYw7qqQ3Vo3Y1ZtOXKWXVFcSbicn8w/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

React Server Components 原理

不同的是 React Server Components 返回的不是 HTML，而是带有结构和数据的自定义类JSON数据。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IhmnQUvhnt4pUxN6SAPz5aamtPIfVmHDuOTaZibcs3JGojNVktj5NXpA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这种结构，是对服务端渲染的核心（结构+数据）进行抽象，结合 React 的工作方式（如Suspense），平缓的从服务端过渡到了客户端，维持了组件状态，并且可以更自由的拼装服务器组件和客户端组件。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvavO9wC2Qp77snsNmCibPmX6IDSqaBBVmjF5xQpdIAXO48hg8dxLEMZ7pastLL1ChTXP7YWZHp8MWAw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)客户端组件和服务端组件混用

关于拆分这条思路，让我想到微前端，虽然现在微前端还有很多问题，但微应用即服务也不乏为一条解决之道。未来前端或许会往“小而美”的方向发展，甚至形成一个以服务端组件为单位的包管理器，网页打包大小会越来越小，更多的组件是从网络上直接获取。

此外，我也很期待 Web Components 的发展，有了原生的支持，0kb runtime也不是不可能了。合久必分分久必合，现存很多前端框架也可以得到统一了。当然现在 Web Components 想要投入使用，首先离不开浏览器的支持，而且必须有一个平缓的过渡，此外兼容性也是一个大问题（最后还是苦了程序员们）。

## 11.**结语**

从 JavaScript 的诞生一路走来，从“发现问题-解决问题”的角度，我们看到了技术发展的原因和必然性。2021年的今天，Web APP 仍然距离原生 APP 体验有一定的差距。或许以后会出现一个小程序桌面APP，小程序能够得到统一；或许会在 PWA 的道路上越走越远；又或者浏览器开放更多原生系统 API，利用各种加载方式，再模拟 APP 的各种体验，达到近似 APP 的效果。

每个时代都诞生了许多的技术，大浪淘沙，留下的却也只是只存在于这个时代的王者。技术总是不断的更迭，重要的不是慌慌张张的追赶技术的脚步，而是去思考技术为什么这么如此演变，思考这样的演变方式的利与弊。如果是你，又会怎么解决当代技术的问题呢？

欢迎在评论区各抒己见。

原文作者：charryhuang，腾讯 CSIG 前端开发工程师

原文链接：https://mp.weixin.qq.com/s/HUknNfaxNULc4Yvf5ajRBA