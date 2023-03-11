# 【NO.123】浏览器性能优化实战

当我们在做性能优化的时候，我们究竟在优化什么？浏览器底层是一个什么架构？浏览器渲染的本质究竟是什么？哪些方面对用户的体验影响才是最大的？有没有业内一些通用的标准或标杆参考？都 1202 年了，雅虎军规还有没有用？性能分析工具都有哪些？我们怎么进行打点分析才是合适的？

本文为你一一讲解这些。了解了这些问题，可能你在做性能优化的时候才能更加得心应手。

### **1. 性能优化的本质**

#### 1.1 展示更快，响应更快

性能优化的目的，就是为了提供给用户更好的体验，这些体验包含这几个方面：展示更快、交互响应快、页面无卡顿情况。

更详细的说，就是指，在用户输入 url 到站点完整把整个页面展示出来的过程中，通过各种优化策略和方法，让页面加载更快；在用户使用过程中，让用户的操作响应更及时，有更好的用户体验。

对于前端工程师来说，要做好性能优化，需要理解浏览器加载和渲染的本质。理解了本质原理，才能更好的去做优化。所以我们先来看看浏览器架构是怎样的。

#### 1.2 理解浏览器多进程架构

从大的方面来说，浏览器是一个多进程架构。

它可以是一个进程包含多个线程，也可以是多个进程中，每个进程有多个线程，线程之间通过 IPC 通讯。每个浏览器有不同的实现细节，并没有标准规定浏览器必须如何去实现。

这里我们只谈论 chrome 架构。

下面这张图是目前 chrome 的多进程架构图。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171654506051927.png)

我们来看看这些进程分别对应浏览器窗口中的哪一部分：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171654592066886.png)

那么，怎么看浏览器对应启动了什么进程呢？

chrome 中，我们可以通过更多->More Tools->Task Manager 看到启动的进程。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171655087997842.png)

从 chrome 官网和源码，我们也可以得知，多进程架构中包含这些进程：

- Browser 进程：打开浏览器后，始终只有一个。该进程有 UI 线程、Network 线程、Storage 线程等。用户输入 url 后，首先是 Browser 进程进行响应和请求服务器获取数据。然后传递给 Renderer 进程。
- Renderer 进程：每一个 tab 一个，负责 html、css、js 执行的整个过程。**前端性能优化也与这个进程有关**。
- Plugin 进程：与浏览器插件相关，例如 flash 等。
- GPU 进程：浏览器共用一个。主要负责把 Renderer 进程中绘制好的 tile 位图作为纹理上传到 GPU，并调用 GPU 相关方法把纹理 draw 到屏幕上。

这里的话只是简单介绍一下浏览器的多进程架构，让大家对浏览器整体架构有个初步认识，其实背后的细节还有很多，这里就不一一展开。有兴趣可以细看[这一系列文章](https://developers.google.com/web/updates/2018/09/inside-browser-part1)和[chrome 官网](https://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code)介绍。

#### 1.3 理解页面渲染相关进程

##### **1.3.1 Renderer Process & GPU Process**

从以上的多架构，我们了解到，与前端渲染、性能优化相关的，其实主要是 Renderer 进程和 GPU 进程。那么，它们又是什么架构呢？

来看一下这张我们再熟悉不过的图。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171655204746486.png)

- Renderer 进程：包括 3 个线程。合成线程（Compositor Thread）、主线程（Main Thread）、Compositor Tile Worker。
- GPU 进程：只有 GPU 线程，负责接收从 Renderer 进程中的 Compositor Thread 传过来的纹理，显示到屏幕上。

##### **1.3.2 Renderer Process 详解**

Renderer 进程中 3 个线程的作用为：

- Compositor Thread：首先接收 vsync 信号(vsync 信号是指操作系统指示浏览器去绘制新的帧)，任何事件都会先到达 Compositor 线程。如果主线程没有绑定事件，那么 Compositor 线程将避免进入主线程，并尝试将输入转换为屏幕上的移动。它将更新的图层位置信息作为帧通过 GPU 线程传递给 GPU 进行绘制。

**当用户在快速滑动过程中，如果主线程没有绑定事件，Compositor 线程是可以快速响应并绘制的**，这是浏览器做的一个优化。

- Main Thread：**主线程就是我们前端工程师熟知的线程，这里会执行解析 Html、样式计算、布局、绘制、合成等动作。所以关于性能的问题，都发生在了这里。所以应该重点关注这里**。
- Compositor Tile Worker：由合成线程产生一个或多个 worker 来处理光栅化的工作。

Service Workers 和 Web Workers 可以暂时理解也在 Renderer 进程中，这里不展开讨论。

###### **1.3.2.1 Main Thread**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171655307930369.png)

主线程需要重点讲下。因为这是我们的代码真实存在的环境。

从上一小节 Render 进程和 GPU 进程的图中，我们可以看到有个红色的箭头，从 Recal Styles 和 Layout 指向了 requestAnimationFrame，这意味着有 Forced Synchronous Layout (or Styles)(强制回流和重绘)发生，这一点在性能方面特别要注意。

在 Main Thread 中，有这几个需要注意一下：

- requestAnimationFrame：因为布局和样式计算是在 rAF 之后，所以在 rAF 是进行元素变更的理想时机。如果在这里对一个元素变更 100 个类，不会进行 100 次计算，它们会分批以后处理。需要注意的是，不能在 rAF 中查询任何计算样式和布局的属性（例如：el.style.backgroundImage 或 el.style.offsetWidth），因为这样会导致重绘和回流。
- Layout：布局的计算通常是针对整个文档的，并且与 DOM 元素的大小成正比！（这点特别要注意，如果一个页面 DOM 元素太多，也会导致性能问题）

主线程的顺序始终都是：

```
Input Event Handler->requestAnimationFrame->ParseHtml->ReculateStyles->Layout- >Update Layer Tree->Paint->Composite->commit->requestIdleCallback
```

只能从前往后，例如，必须先是 ReculateStyles，然后 Layout、然后 Paint。但是，如果它只需要做最后一步 Paint，那么这就是它全部要做的事情，不会再发生前面的 ReculateStyles 和 Layout。

这里其实给了我们一个启示：**如果要让 fps 保持 60，即每帧的 js 执行时间少于 16.66ms，那么让这个主线程执行的过程尽可能地少，是我们的性能优化目标**。

根据主线程的这些步骤，理想的情况下，我们只希望浏览器只发生最后一个步骤：Composite(合成)。

CSS 的属性是我们需要关注一下的模块。这里有描述了哪些[CSS 属性](https://csstriggers.com/)会引起重绘、回流和合成。例如，让我们给一个元素进行移动位置时：`transform`和`opacity`可以直接触发合成，但是`left`和`top`却会触发 Layout、Paint、Composite3 个动作。所以显然用 transform 时更好的方案。

**但这并不是说我们不应该用 left 和 top 这些可能引起重绘回流的属性，而是应该关注每个属性在浏览器性能中引起的效果**。

### **2. 看看经典：雅虎军规**

多年前雅虎的 Nicolas C. Zakas 提出 7 个类别 35 条军规，至今为止很多前端优化准则都是围绕着这个展开。如果严格按照这些规则去做，其实我们有很多优化工作可以做，只要认真践行，性能提升不是问题。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171655417084264.png)

我们来看看它 7 个分类都是围绕哪些方面展开：

- Server：与页面发起请求的相关；
- Cookie：与页面发起请求相关；
- Mobile：与页面请求相关；
- Content：与页面渲染相关；
- Image：与页面渲染相关；
- CSS：与页面渲染相关；
- Javascript：与页面渲染和交互相关。

从上面的描述可以看到，其实雅虎军规，是围绕页面发起请求那一刻，到页面渲染完成，页面开始交互这几个方面来展开，提出的一些原则。

很多原则大家也都耳熟能详，就不全部展开了，有兴趣的同学可以去查看[原文](https://developer.yahoo.com/performance/rules.html)。这里主要想提一些忽略但是又值得注意的点：

**减少 DOM 节点数量**

为什么要减少 DOM 节点的数量？

当遍历查询 500 和 5000 个 DOM 节点，进行事件绑定时，会有所差别。

当一个页面 DOM 节点过多，应该考虑使用无限滚动方案来使视窗节点可控。可以看看[google 提的方案](https://developers.google.com/web/updates/2016/07/infinite-scroller)。

**减少 cookie 大小**

cookie 传输会造成带宽浪费，影响响应时间，可以这样做：

消除不必要的 cookies；

静态资源不需要 cookie，可以采用其他的域名，不会主动带上 cookie。

**避免图片 src 为空**

图片 src 为空时，不同浏览器会有不同的副作用，会重新发起一起请求。

### **3. 性能指标**

#### 3.1 什么样的性能指标才能真正代表用户体验？

要衡量性能，我们必须有一些客观的、可衡量的指标来进行监控。**但是客观且定量可衡量的指标不一定能反映用户的真实体验**。

以前，我们会用 load 事件的触发来衡量一个页面是否加载或显示完成。但是设想会不会有这样的情况：一个页面的 load 事件已经被触发，但是却在 load 事件之后几秒才开始加载内容和渲染页面，所以这个时候，load 事件并不能真实反映用户看到内容的时刻。

在过去几年，google 团队和[W3C 性能工作组](https://www.w3.org/webperf/)致力于提供标准的性能 API 来真正衡量用户的体验。主要是从这 4 个方面思考：

| 思考点            | 详细内容                                         |
| :---------------- | :----------------------------------------------- |
| Is it happening?  | 导航是否成功，服务器是否响应了                   |
| Is it useful?     | 是否已经渲染了足够的内容，让用户可以开始参与其中 |
| Is it usable?     | 用户是否可以与页面交互，页面是否处于繁忙状态     |
| Is it delightful? | 交互是否流畅、自然、没有滞后反映或卡顿           |

通常有 2 种途径来衡量性能。

1. 本地实验衡量：本地模拟用户的网络、设备等情况进行测试。通常在开发新功能的时候，实验测量是很重要的，因为我们不知道这个功能发布到线上会有什么性能问题，所以提前进行性能测试，可以进行预防。
2. 线上衡量：实验测量固然可以反映一些问题，但无法反映在用户那里真实的情况。同样的，在用户那里，性能问题会和用户的设备、网络情况有关，而且还跟用户如何与页面进行交互有关。

有这几个类型与用户感知性能相关。

- 页面加载时间：页面以多快的速度加载和渲染元素到页面上。
- 加载后响应时间：页面加载和执行 js 代码后多久能响应用户交互。
- 运行时响应：页面加载完成后，对用户的交互响应时间。
- 视觉稳定性：页面元素是否会以用户不期望的方式移动，并干扰用户的交互。
- 流畅度：过渡和动画是否以一致的帧率渲染，并从一种状态流畅地过渡到另一种状态。

对应上面几种分类，Google 和 W3C 性能工作组提供了对应这几种性能指标：

- **[First contentful paint (FCP)](https://web.dev/fcp/):** 测量页面开始加载到某一块内容显示在页面上的时间。
- **[Largest contentful paint (LCP)](https://web.dev/lcp/):** 测量页面开始加载到最大文本块内容或图片显示在页面中的时间。
- **[First input delay (FID)](https://web.dev/fid/):** 测量用户首次与网站进行交互(例如点击一个链接、按钮、js 自定义控件)到浏览器真正进行响应的时间。
- **[Time to Interactive (TTI)](https://web.dev/tti/):** 测量从页面加载到可视化呈现、页面初始化脚本已经加载，并且可以可靠地快速响应用户的时间。
- **[Total blocking time (TBT)](https://web.dev/tbt/):** 测量从 FCP 到 TTI 之间的时间，这个时间内主线程被阻塞无法响应用户输入。
- **[Cumulative layout shift (CLS)](https://web.dev/cls/):** 测量从页面开始加载到状态变为隐藏过程中，发生不可预期的 layout shifts 的累积分数。

这些指标能从一定程度上衡量页面性能，但不一定都是有效的。举个例子。LCP 指标主要用户衡量页面的主要内容是否完成加载，但会有这样的情况，最大的元素并不是主要内容，那么这个时候 LCP 指标并不是那么重要。

每个不同的站点有自己的特殊性，可以参考以上角度进行衡量，也需要因地制宜。

#### 3.2 Core Web Vitals

在以上列出的指标中，Google 定义了 3 个最核心的指标，作为 Core Web Vitals。它们分别代表着：加载、交互、视觉稳定性。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171656409012206.png)

- **[Largest Contentful Paint (LCP)](https://web.dev/lcp/)**: 测量加载性能。为了能提供较好的用户体验，LCP 指标建议页面首次加载要在 2.5s 内完成。
- **[First Input Delay (FID)](https://web.dev/fid/)**: 测量交互性能。为了提供较好用户体验，交互时间建议在 100ms 或以内。
- **[Cumulative Layout Shift (CLS)](https://web.dev/cls/)**: 测量视觉稳定性。为了提供较好用户体验，页面应该维持 CLS 在 0.1 或以内。

当页面访问量有 75%的数据达到了以上以上 Good 的标准，则认为性能是不错的了。

Core Web Vitals 是作为核心性能指标，但是其他指标也同样在重要，是做为核心指标的一个辅助。例如，TTFB 和 FCP 都可以用来衡量加载性能(服务器响应时间和渲染时间)，它们作为 LCP 的一个问题手段辅助。同样的，TBT 和 TTI 对于衡量交互性能也很重要，是 FID 的一个辅助，但是它们无法在线上进行测量，也无法反映以用户为中心的结果。

Google 官方提供了一个[web-vitals](https://github.com/GoogleChrome/web-vitals)库，线上或本地都可以测量上面提到的 3 个指标：

```
import {getCLS, getFID, getLCP} from 'web-vitals';function sendToAnalytics(metric) {  const body = JSON.stringify(metric);  // Use `navigator.sendBeacon()` if available, falling back to `fetch()`.  (navigator.sendBeacon && navigator.sendBeacon('/analytics', body)) ||      fetch('/analytics', {body, method: 'POST', keepalive: true});}getCLS(sendToAnalytics);getFID(sendToAnalytics);getLCP(sendToAnalytics);
```

下面，分别讲讲这 3 个指标定义的原因、如何测量、如何优化。

##### **3.2.1 Largest Contentful Paint (LCP)**

###### **3.2.1.1 LCP 如何定义**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171656528808285.png)

LCP 是指页面开始加载到最大文本块内容或图片显示在页面中的时间。那么哪些元素可以被定义为最大元素呢？

- `<img>`标签
- `<image>` 在 svg 中的 image 标签
- `<video>` video 标签
- CSS background url()加载的图片
- 包含内联或文本的块级元素

###### **3.2.1.2 如何测量 LCP**

**线上测量工具**

- [Chrome User Experience Report](https://developers.google.com/web/tools/chrome-user-experience-report)
- [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/)
- [Search Console (Core Web Vitals report)](https://support.google.com/webmasters/answer/9205520)
- [`web-vitals` JavaScript library](https://github.com/GoogleChrome/web-vitals)

**实验室工具**

- [Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/)
- [Lighthouse](https://developers.google.com/web/tools/lighthouse/)
- [WebPageTest](https://webpagetest.org/)

**原生的 JS API 测量**

LCP 还可以用 JS API 进行测量，主要使用 PerformanceObserver 接口，目前除了 IE 不支持，其他浏览器基本都支持了。

```
new PerformanceObserver((entryList) => {  for (const entry of entryList.getEntries()) {    console.log('LCP candidate:', entry.startTime, entry);  }}).observe({type: 'largest-contentful-paint', buffered: true});
```

我们看一下结果是怎样的：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171657055420242.png)

**Google 官方 web-vitals 库**

Google 官方也提供了一个[web-vitals](https://github.com/GoogleChrome/web-vitals)库，底层还是使用这个 API，只是帮我们处理了一些需要测量和不需测量的场景、以及一些细节问题。

###### **3.2.1.3 如何优化 LCP**

LCP 可能被这四个因素影响：

- 服务端响应时间
- Javascript 和 CSS 引起的渲染卡顿
- 资源加载时间
- 客户端渲染

更加详细的优化建议就不展开了，可以参考[这里](https://web.dev/optimize-lcp/)。

##### **3.2.2 First Input Delay (FID)**

###### **3.2.2.1 FID 如何定义**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171657167766061.png)

我们都知道第一印象的重要性，比如初次遇到某人形成的印象，会在后续交往中起重要的影响。对于一个网站也是如此。

网站以多快的速度加载完成是其中一项指标，加载后以多快的速度对用户进行响应也同样重要。FID 就是指后者。

可以通过下面的图来更详细了解 FID 处于哪个位置：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171657278557042.png)

从上图可以看出，当主线程处于繁忙的时候，FID 是指从浏览器接收到了用户输入，到浏览器对用户的输入进行响应的延迟时间。

通常，**当我们在写代码的时候，会认为只要用户输入信息，我们的事件回调就会立刻响应，但实际上并不是这样。这是主线程可能处于繁忙，浏览器正忙着解析和执行其他 js。**如上图所示的 FID 时间，主线程正在处理其他任务。

当 FID 的时间为 100ms 或以内，则为 Good。

上面的例子中，用户刚好在主线程最繁忙的时刻进行了交互，但是如果用户在主线程空闲的时候交互，那么浏览器可以立刻响应。所以 FID 的值需要重点查看它的分布情况。

FID 实际上测量的是输入事件被感知到到主线程空闲的这段时间。这意味着即使没有输入事件被注册，FID 也可以测量。因为用户的输入相应并不一定需要事件被执行，但一定需要主线程是空闲的。例如，下面这些 HTML 元素都需要在交互响应之前等待主线程上的正在执行的任务完成：

- 输入框，例如`<input>`、`<textarea>`、`<radio>`、`<checkbox>`
- 下拉框，例如`<select>`
- 链接，例如`<a>`

为什么要考虑测量第一次的输入延迟？有如下原因：

- 因为第一次输入延迟是用户对你的网站形成的第一个印象，网站是否有质量且可靠；
- 在今天，web 中最大的交互问题第一次加载之后；
- 对于网站应该如何解决较高的首次输入延迟(例如代码分割、减少 JavaScript 的预加载)的建议解决方案(TTI 是指衡量这一块)，不一定与在页面加载后解决输入延迟(FID 是指衡量这一块)的解决方案相同。所以 FID 是在 TTI 的基础上更精确的细分。

为什么 FID 只是包含从用户输入到主线程开始相应的时间？而没有包含事件处理到浏览器绘制 UI 的时间？

尽管主线程处理和绘制的这段时间也很重要，但是如果 FID 把这段时间也包含进来，开发者可能会使用异步 API(例如`setTimeout`、`requestAnimationFrame`)来把这个 task 拆分到下一帧，以较少 FID 的时间，这样不仅没有提高用户体验，反而使用户体验降低。

#### 3.2.2.2 如何测量 FID

FID 可以在实验环境也可以在线上环境测量。

**线上测量工具**

- [Chrome User Experience Report](https://developers.google.com/web/tools/chrome-user-experience-report)
- [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/)
- [Search Console (Core Web Vitals report)](https://support.google.com/webmasters/answer/9205520)
- [`web-vitals` JavaScript library](https://github.com/GoogleChrome/web-vitals)

**原生的 JS API 测量**

```
new PerformanceObserver((entryList) => {  for (const entry of entryList.getEntries()) {    const delay = entry.processingStart - entry.startTime;    console.log('FID candidate:', delay, entry);  }}).observe({type: 'first-input', buffered: true});
```

PerformanceObserver 目前除了在 IE 上没有兼容，其他浏览器基本都兼容了。

我们看一下结果是怎样的：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171657505359328.png)
**Google 官方 web-vitals 库**

Google 官方也提供了一个[web-vitals](https://github.com/GoogleChrome/web-vitals)库，底层还是使用这个 API，只是帮我们处理了一些需要测量和不需测量的场景、以及一些细节问题。

#### 3.2.2.3 如何优化 FID

FID 可能被这四个因素影响：

- 减少第三方代码的影响
- 减少 Javascript 的执行时间
- 最小化主线程工作
- 减小请求数量和请求文件大小

更加详细的优化建议可以参考[这里](https://web.dev/optimize-fid/)。

##### **3.2.3 Cumulative Layout Shift (CLS)**

###### **3.2.3.1 CLS 如何定义**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658011068863.png)

CLS 是一个非常重要的、以用户为中心的测量指标。它能衡量页面是否排版稳定。

页面移动会经常发生在资源异步加载、或者 DOM 元素动态添加到已存在的页面元素上面。这些元素有可能是图片、视频、第三方广告或小图标等。

但是我们开发过程中可能不会察觉到这些问题，因为调试过程中刷新页面，图片都已经缓存在本地。调试接口的时候我们使用的是 mock 或者在局域网，接口速度都很快，这些延迟都可能被我们忽略。

CLS 就是帮我们去发现这些真实发生在用户端的问题的指标。

CLS 是测量页面生命周期中，每个发生意外布局移动的分数。当一个可视元素在下一帧移动到另外一个位置，就是指布局移动。

CLS 的分数在 0.1 或以下，则为 Good。

那么意外布局移动的分数如何计算？

浏览器会监控两桢之间发生移动的不稳定元素。布局移动分数由 2 个元素决定：impact fraction 和 distance fraction。

```
layout shift score = impact fraction * distance fraction
```

可视区域内，在前一帧到下一帧之间所有不稳定的元素的并集，会影响当前帧的布局移动分数。

举个例子，下面这张图中，左边是当前帧的一个元素，下一帧中，元素下移了可视区域内 25%的高度。红色虚线框标出了两桢中当前元素的并集，占适口的 75%，所以这个时候，impact faction 是 0.75。

另外一个影响布局移动分数的是 distance fraction，指这个元素相对视口移动的距离。不管是横向还是竖向，取最大值。

下面例子中，竖向距离更大，该元素相对适口移动了 25%的距离，所以 distance fraction 是 0.25。所以布局移动分数是 `0.75 * 0.25 = 0.1875`.

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658129451951.png)

**但是要注意的是，并不是所有的布局移动都是不好的，很多 web 网站都会改变元素的开始位置。只有当布局移动是非用户预期的，才是不好的**。

换句话说，当用户点击了按钮，布局进行了改动，这是 ok 的，CLS 的 JS API 中有一个字段`hadRecentInput`，用来标识 500ms 内是否有用户数据，视情况而定，可以忽略这个计算。

###### **3.2.3.2 如何测量 CLS**

**线上测量工具**

- [Chrome User Experience Report](https://developers.google.com/web/tools/chrome-user-experience-report)
- [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/)
- [Search Console (Core Web Vitals report)](https://support.google.com/webmasters/answer/9205520)
- [`web-vitals` JavaScript library](https://github.com/GoogleChrome/web-vitals)

**实验室工具**

- [Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/)
- [Lighthouse](https://developers.google.com/web/tools/lighthouse/)
- [WebPageTest](https://webpagetest.org/)

**原生的 JS API 测量**

```
let cls = 0;new PerformanceObserver((entryList) => {  for (const entry of entryList.getEntries()) {    if (!entry.hadRecentInput) {      cls += entry.value;      console.log('Current CLS value:', cls, entry);    }  }}).observe({type: 'layout-shift', buffered: true});
```

我们看一下结果是怎样的：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658214521799.png)

**Google 官方 web-vitals 库**

Google 官方也提供了一个[web-vitals](https://github.com/GoogleChrome/web-vitals)库，底层还是使用这个 API，只是帮我们处理了一些需要测量和不需测量的场景、以及一些细节问题。

###### **3.2.3.3 如何优化 CLS**

我们可以根据这些原则来避免非预期布局移动：

- 图片或视屏元素有大小属性，或者给他们保留一个空间大小，设置 width、height，或者使用[unsized-media feature policy](https://github.com/w3c/webappsec-feature-policy/blob/master/policies/unsized-media.md)。
- 不要在一个已存在的元素上面插入内容，除了相应用户输入。
- 使用 animation 或 transition 而不是直接触发布局改变。

更详细的内容可以看[这里](https://web.dev/optimize-cls/)。

### **4. 性能工具：工欲善其事，必先利其器**

Google 开发的[所有工具](https://web.dev/vitals-tools/)都支持 Core Web Vitals 的测量。工具如下：

- [Lighthouse](https://github.com/GoogleChrome/lighthouse)
- [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/)
- [Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools)
- [Search Console](https://search.google.com/search-console/about)
- [web.dev’s 提供的测量工具](https://web.dev/measure/)
- [Web Vitals 扩展](https://chrome.google.com/webstore/detail/web-vitals/ahfhijdlegdabablpippeagghigmibma)
- [Chrome UX Report](https://developers.google.com/web/tools/chrome-user-experience-report) API

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658302259657.png)

这些工具对 Core Web Vitals 的支持如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658404080209.png)

#### 4.1 Lighthouse

打开 F12，就可以看到 Lighthouse，点击 Generate Report，即可生成报告。当然也可以添加 chrome 插件使用。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658511977119.png)

Lighthhouse 是一个实验室工具，本地模拟移动端和 PC 端对这几个方面进行测试。同时 lighthouse 还会针对这几个方面提出建议，在产品上线前值得一测。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171658591915966.png)

Lighthouse 还提供了[Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)，把 Lighthouse 集成到 CI 流水线中。举个例子，每次在上线之前，跑 50 次流水线对 Lighthouse 的各项指标进行测试取平均值，一旦发现异常，立刻进行排查。把性能问题排查提前到发布之前。这块后面会细讲。

#### 4.2 PageSpeed Insights

PageSpeed Insights(PSI)是一个可以分析线上和实验室数据的工具。它是根据线上环境用户真实的数据(在 Chrome UX 报告中)和 Lighthouse 结合出一份报告。和 Lighthouse 类似，它也会给出一些分析建议，可以知道页面的 Core Web Vitals 是否达标。
![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171659097738804.png)

PageSpeed 只是提供对单个页面的性能测试，而 Search Console 是正对整个网站的性能测试。

PageSpeed Insights 也提供了[API](https://developers.google.com/speed/docs/insights/v5/get-started)供我们使用。同样的，我们也可以把它集成到 CI 中。

#### 4.3 CrUX

Chrome UX Report (CrUX)是指汇聚了成千上万条用户体验数据的数据报告集，它是经过用户同意才进行上报的，目前存储在 Google BigQuery 上，可以使用账号登陆进行查询。它测量了所有的 Core Web Vitals 指标。

上面提到的 PageSpeed Insights 工具就是结合 CrUX 的数据进行分析给出的结论。

当然 CrUX 现在也提供了 API 共我们进行查询，可以查询的数据包括：

- Largest Contentful Paint
- Cumulative Layout Shift
- First Input Delay
- First Contentful Paint

原理如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171659191368083.png)

通过 API 的查询的数据每日都更新，并汇集了过去 28 天的数据。

具体的使用方式可以参考官方给出的[demo](https://developers.google.com/web/tools/chrome-user-experience-report/api/guides/getting-started)。

#### 4.4 Chrome DevTools Performance 面板

Performance 是我们最常用的本地性能分析工具。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171659272795468.png)

这里像提几点可以关注下的功能：Frame、Timings、Main、Layers、FPS。下面一一讲解。

##### **4.4.1 Frame**

点击 Frame 展开后，会看到有一个一个红色或绿色小块，这些代表着每帧的消耗时间。目前大多数设备的屏幕刷新率为 60 次/秒，浏览器渲染页面的每一帧的速率如果与设备屏幕的刷新率保持一致，即 60fps 时，我们是不会感知到页面卡的情况的。

我们把鼠标移上去看看：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171659361527158.png)

这种是体验顺畅的情况。

再比如：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171659446344336.png)

提示这一帧耗时了 30.9ms，当前是 32fps 并且是掉帧状态。

##### **4.4.2 Timings**

这里可以看到几个关键指标的时间点。

FP：First Paint；

FCP：First Contentful Paint；

LCP：Largest Contenful Paint；

DCL：DOMContentLoaded Event

L：OnLoad Event。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171659514165792.png)

##### **4.4.3 Main**

Main 是 DevTools 中最常用也是最重要的功能。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171700021394158.png)

通过 record，我们可以查看页面上所有操作在主线程中的执行过程。也就是我们常说的流程：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171700113604346.png)

一旦有任何一个流程时间过长或频繁发生，比如 Update Layer Tree 时间过长、频繁出现 RecalcStyles、Layout(重绘回流)，那么需要引起注意。后面会举一个例子。

##### **4.4.4 Layers**

Layers 是浏览器在绘制过程中生成的一个层。因为浏览器底层渲染的本质是纵向分层、横向分块。这一块的知识点是发生在 Renderer Process 进程中。后面会以一个例子展开讲。

这里想提 Layers 的原因是，**Layer 的渲染也会影响性能问题**，而且有时候还不容易被发现！

Layers 面板一般不会默认展示出来，点击更多->more tools->Layers 即可打开。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171700214127503.png)

点击 Layers 面板，点击左边下三角展开按钮，可以看见页面最终生成的合成层。右边左上角可以选择不同纬度进行查看。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171700294507568.png)
选中某个层，可以查看该层生成的原因。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171700387103475.png)

Chrome 的 Blink 内核给出了 54 种会生成合成层的原因：

```
constexpr CompositingReasonStringMap kCompositingReasonsStringMap[] = {    {CompositingReason::k3DTransform, "transform3D", "Has a 3d transform"},    {CompositingReason::kTrivial3DTransform, "trivialTransform3D",     "Has a trivial 3d transform"},    {CompositingReason::kVideo, "video", "Is an accelerated video"},    {CompositingReason::kCanvas, "canvas",     "Is an accelerated canvas, or is a display list backed canvas that was "     "promoted to a layer based on a performance heuristic."},    {CompositingReason::kPlugin, "plugin", "Is an accelerated plugin"},    {CompositingReason::kIFrame, "iFrame", "Is an accelerated iFrame"},    {CompositingReason::kSVGRoot, "SVGRoot", "Is an accelerated SVG root"},    {CompositingReason::kBackfaceVisibilityHidden, "backfaceVisibilityHidden",     "Has backface-visibility: hidden"},    {CompositingReason::kActiveTransformAnimation, "activeTransformAnimation",     "Has an active accelerated transform animation or transition"},    {CompositingReason::kActiveOpacityAnimation, "activeOpacityAnimation",     "Has an active accelerated opacity animation or transition"},    {CompositingReason::kActiveFilterAnimation, "activeFilterAnimation",     "Has an active accelerated filter animation or transition"},    {CompositingReason::kActiveBackdropFilterAnimation,     "activeBackdropFilterAnimation",     "Has an active accelerated backdrop filter animation or transition"},    {CompositingReason::kXrOverlay, "xrOverlay",     "Is DOM overlay for WebXR immersive-ar mode"},    {CompositingReason::kScrollDependentPosition, "scrollDependentPosition",     "Is fixed or sticky position"},    {CompositingReason::kOverflowScrolling, "overflowScrolling",     "Is a scrollable overflow element"},    {CompositingReason::kOverflowScrollingParent, "overflowScrollingParent",     "Scroll parent is not an ancestor"},    {CompositingReason::kOutOfFlowClipping, "outOfFlowClipping",     "Has clipping ancestor"},    {CompositingReason::kVideoOverlay, "videoOverlay",     "Is overlay controls for video"},    {CompositingReason::kWillChangeTransform, "willChangeTransform",     "Has a will-change: transform compositing hint"},    {CompositingReason::kWillChangeOpacity, "willChangeOpacity",     "Has a will-change: opacity compositing hint"},    {CompositingReason::kWillChangeFilter, "willChangeFilter",     "Has a will-change: filter compositing hint"},    {CompositingReason::kWillChangeBackdropFilter, "willChangeBackdropFilter",     "Has a will-change: backdrop-filter compositing hint"},    {CompositingReason::kWillChangeOther, "willChangeOther",     "Has a will-change compositing hint other than transform and opacity"},    {CompositingReason::kBackdropFilter, "backdropFilter",     "Has a backdrop filter"},    {CompositingReason::kBackdropFilterMask, "backdropFilterMask",     "Is a mask for backdrop filter"},    {CompositingReason::kRootScroller, "rootScroller",     "Is the document.rootScroller"},    {CompositingReason::kAssumedOverlap, "assumedOverlap",     "Might overlap other composited content"},    {CompositingReason::kOverlap, "overlap",     "Overlaps other composited content"},    {CompositingReason::kNegativeZIndexChildren, "negativeZIndexChildren",     "Parent with composited negative z-index content"},    {CompositingReason::kSquashingDisallowed, "squashingDisallowed",     "Layer was separately composited because it could not be squashed."},    {CompositingReason::kOpacityWithCompositedDescendants,     "opacityWithCompositedDescendants",     "Has opacity that needs to be applied by compositor because of composited "     "descendants"},    {CompositingReason::kMaskWithCompositedDescendants,     "maskWithCompositedDescendants",     "Has a mask that needs to be known by compositor because of composited "     "descendants"},    {CompositingReason::kReflectionWithCompositedDescendants,     "reflectionWithCompositedDescendants",     "Has a reflection that needs to be known by compositor because of "     "composited descendants"},    {CompositingReason::kFilterWithCompositedDescendants,     "filterWithCompositedDescendants",     "Has a filter effect that needs to be known by compositor because of "     "composited descendants"},    {CompositingReason::kBlendingWithCompositedDescendants,     "blendingWithCompositedDescendants",     "Has a blending effect that needs to be known by compositor because of "     "composited descendants"},    {CompositingReason::kPerspectiveWith3DDescendants,     "perspectiveWith3DDescendants",     "Has a perspective transform that needs to be known by compositor because "     "of 3d descendants"},    {CompositingReason::kPreserve3DWith3DDescendants,     "preserve3DWith3DDescendants",     "Has a preserves-3d property that needs to be known by compositor because "     "of 3d descendants"},    {CompositingReason::kIsolateCompositedDescendants,     "isolateCompositedDescendants",     "Should isolate descendants to apply a blend effect"},    {CompositingReason::kFullscreenVideoWithCompositedDescendants,     "fullscreenVideoWithCompositedDescendants",     "Is a fullscreen video element with composited descendants"},    {CompositingReason::kRoot, "root", "Is the root layer"},    {CompositingReason::kLayerForHorizontalScrollbar,     "layerForHorizontalScrollbar",     "Secondary layer, the horizontal scrollbar layer"},    {CompositingReason::kLayerForVerticalScrollbar, "layerForVerticalScrollbar",     "Secondary layer, the vertical scrollbar layer"},    {CompositingReason::kLayerForScrollCorner, "layerForScrollCorner",     "Secondary layer, the scroll corner layer"},    {CompositingReason::kLayerForScrollingContents, "layerForScrollingContents",     "Secondary layer, to house contents that can be scrolled"},    {CompositingReason::kLayerForSquashingContents, "layerForSquashingContents",     "Secondary layer, home for a group of squashable content"},    {CompositingReason::kLayerForForeground, "layerForForeground",     "Secondary layer, to contain any normal flow and positive z-index "     "contents on top of a negative z-index layer"},    {CompositingReason::kLayerForMask, "layerForMask",     "Secondary layer, to contain the mask contents"},    {CompositingReason::kLayerForDecoration, "layerForDecoration",     "Layer painted on top of other layers as decoration"},    {CompositingReason::kLayerForOther, "layerForOther",     "Layer for link highlight, frame overlay, etc."},    {CompositingReason::kBackfaceInvisibility3DAncestor,     "BackfaceInvisibility3DAncestor",     "Ancestor in same 3D rendering context has a hidden backface"},};
```

##### **4.4.5 Rendering**

Rendering 面板也隐藏了很多好用的功能。

###### **4.4.5.1 Paint flashing**

勾选了 Paint flashing 后，我们就会看到页面上有哪些内容被重绘了：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171700554576876.png)

###### **4.4.5.2 Layout Shift6 Regions**

勾选了 Layout Shift Regions 后，进行交互，就可以看到哪些元素进行了布局移动：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171701032304771.png)

###### **4.4.5.3 Frame Rendering Stats**

这个工具有一个小插曲。

Frame Rendering Stats 的前身是 FPS meter，在 Google 版本[85.0.4181.0](https://chromium.googlesource.com/chromium/src/+log/85.0.4180.0..85.0.4181.0?pretty=fuller&n=10000)改成了 Frame Rendering Stats，但是迫于[用户抱怨](https://www.bleepingcomputer.com/news/google/google-chrome-rolls-back-fps-meter-changes-after-user-complaints/?__cf_chl_jschl_tk__=cc8bc924a9cecaaeac51306be72fd84e9e9497e4-1620787727-0-AcZbEMncBYzkJEfBKoagUU_Lx_usv9QYDmr9LcL011Asjxa2HprCsaE1dwQcSl-OQQ82hMfZEpmBKtrfSDn1AXmdocNGLn3WqQePxYnFpYr5xPlLRYCzEDejvJYmGWRW09ceBNb4LKnAqh-5uKGnE5DccFkd1CY_t_Omo-l8SaAQqrEcOboHnNIehm6-UOHfwHJ6nvyrOUTDAHtzi0x1izcrT73ECtRUNkvg3l48cNkm2NNZVcq7mIpnWqTfJEEdZ9oSmZOU9AjWkNETVkiUdZ7m3ylBJ_tXVptpa01quzWE9xhdRYyLM6dbeHGpxcqaNHKdbkPBki7jZYpZiKza1hFELUG5RXceHwQPqAjZLuY0StG7nQnIcoPdYh4339dB4883flK-OXEB4O1xykIF_HEQ6Kz3cnvLTrpaxuDVwxUhV3S7rdhXxj4wiNPju2t6gsXiHgGfuNUxu2c9sMbM6888k9jSIMBUMhLHgDYn6lnBxPcQO-MoIzUf0NalXSZqyw)，在 90 版本的时候又改回来了。

Frame Rendering Stats 主要显示不掉帧率。而 FPS 侧重于显示每秒的刷新率 fps。

Chrome 为什么要改成不掉帧率，是因为认为不掉帧率更能反映页面的顺畅度。而 FPS 显示每一秒渲染的帧数虽然能一定程度反映页面顺畅度，但是在一些特殊情况例如没有激活或空闲的页面，fps 会比较低，这样并不能反映真实情况。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171701147681846.png)

##### **4.4.6 Memory**

在大型项目中，内存问题也是有发生。DevTools 也提供了内存分析工具供我们使用。

点击 Memory 面板，点击录制按钮。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171701211749102.png)
点击录制后，会看到当前状态下内存的占用情况，根据大小排序，我们可以定位到内存占用过多的地方。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171701583562951.png)

#### 4.5 Search Console

Google Search Console 其实就是监控和维护网站在 Google 搜索结果中的展示情况以及排查问题的平台。数据来源是 CrUX。

它会展示 3 个 Core Web Vitals metrics: LCP, FID, CLS。如果发现有问题，可以配合 PageSpeed 一起使用，分析问题。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171701447244702.png)

#### 4.6 web.dev

[web.dev/measure](https://web.dev/measure/)是 google 官方提供的测量性能工具，也会提供类似 PageSpeed Insight 的指标，还会提供一些具体代码更改建议。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171702095330832.png)

#### 4.7 Web Vitals extension

Google 也提供了扩展工具去测量 Core Web Vitals。可以从[Store](https://chrome.google.com/webstore/detail/web-vitals/ahfhijdlegdabablpippeagghigmibma?hl=en)中进行安装。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171702202480424.png)

#### 4.8 工具：思考与总结

当我们了解了这么多工具之后，琳琅满目，我们该如何选择？如何使用好这些工具进行分析？

- 首先我们可以使用 Lighthouse，在本地进行测量，根据报告给出的一些建议进行优化；
- 发布之后，我们可以使用 PageSpeed Insights 去看下线上的性能情况；
- 接着，我们可以使用 Chrome User Experience Report API 去捞取线上过去 28 天的数据；
- 发现数据有异常，我们可以使用 DevTools 工具进行具体代码定位分析；
- 使用 Search Console’s Core Web Vitals report 查看网站功能整体情况；
- 使用 Web Vitals 扩展方便的看页面核心指标情况；

### **5. 谈谈监控**

最后一个章节想来谈谈监控。

我们在做性能优化的时候，常常会通过各种线上打点，来收集用户数据，进行性能分析。没错，这是一种监控手段，更精确的说，这是一种”事后”监控手段。

“事后”监控固然重要，但我们也应该考虑”事前”监控，否则，每次发布一个需求后，去线上看数据。咦，发现数据下降了，然后我们去查代码，去查数据，去查原因。这样性能优化的同学永远处于”追赶者”的角色，永远跟在屁股后面查问题。

举个例子，我们可以这样去做”事前”监控。

建立流水线机制。流水线上如何做呢？

- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) 或 [PageSpeed Insights API](https://developers.google.com/speed/docs/insights/v5/get-started)：把 Lighthouse 或 PageSpeed Insights API 集成到 CI 流水线中，输出报告分析。
- [Puppeteer](https://github.com/puppeteer/puppeteer/blob/main/docs/api.md#class-tracing) 或 [Playwright](https://playwright.dev/docs/api/class-browser?_highlight=tracing#browserstarttracingpage-options)：使用 E2E 自动化测试工具集成到流水线模拟用户操作，得到 Chrome Trace Files，也就是我们平常录制 Performance 后，点击左上角下载的文件。Puppeteer 和 Playwright 底层都是基于[Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171702388250820.png)

- Chrome Trace Files：根据[规则](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/edit)分析 Trace 文件，可以得到每个函数执行的时间。如果函数执行时间超过了一个临界值，可以抛出异常。如果一个函数每次的执行时间都超过了临界值，那么就值得注意了。但是还有一点需要思考的是：函数执行的时间是否超过临界值固然重要，但更重要的是这是不是用户的输入响应函数，与用户体验是否有关。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171702481831930.png)

- 输出报告。定义异常临界值。如果异常过多，考虑是否卡发布流程。

### **6. 总结**

我们来回顾一下前面的内容：

**第一部分，讲了浏览器整体架构和渲染相关进程**.为什么把这个章节也放到这篇性能优化的文章中？浏览器对于我们前端开发来说，是一个 sandbox 或者 darkbox。我们知道 js、html、css 结合起来就能实现我们的需求，但如果知道它是如何去渲染、执行、处理我们的代码，不管是对做需求还是性能优化，都能更知其然和所以然。

**第二部分，雅虎军规是多年前提出的非常经典的优化建议**。至今对于我们异常有很强的指导作用。你会发现它是从页面加载、页面渲染、到页面交互全面的一个指导建议。与今天 Chrome 和 W3c 提出的 Web Vitals 思路依然类似。

**第三部分，性能指标**。参考标准与业内标杆的建议，能更好地指导我们进行优化。

**第四部分，性能工具**。工欲善其事，必先利其器。这个道理大家都懂，运用好工具，才能让我们更加事半功倍。

**第五部分，监控在性能优化中占很重要的部分，”事前”监控更重要，防患于未然。让性能优化成为一个预防者而不是追赶者。**

罗里吧嗦说了很多，当然还有很多性能优化的细节没有讲到，如果有错误的地方欢迎指正。或者有什么好方法好建议也强烈欢迎私聊交流一下。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212171702583609362.png)

参考文章：

- https://web.dev/learn-web-vitals/
- https://developers.google.com/web/updates/2018/09/inside-browser-part1
- https://aerotwist.com/blog/the-anatomy-of-a-frame/
- https://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code
- https://medium.com/punching-performance/jank-you-can-measure-what-your-users-can-feel-e5713df2845f

原文作者：rosefang，腾讯 PCG 前端开发工程师

原文链接：https://mp.weixin.qq.com/s/RCJftzmhQbc-b89pU5d32w