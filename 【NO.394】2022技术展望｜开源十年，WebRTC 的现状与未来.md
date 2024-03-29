# 【NO.394】2022技术展望｜开源十年，WebRTC 的现状与未来

2020 年，WebRTC 发生了很多变化。WebRTC 其实就是一个客户端库。大家都知道它是开源的。尽管 Google 大力地在支持 WebRTC，但社区的力量同样功不可没。

WebRTC 对于桌面平台、浏览器端实现音视频交互来讲十分重要。因为在你可以再浏览器上运行任何一种服务，并进行安全检查，无需安装任何应用。这是此前开发者使用该开源库的主要方式。

但 2020 年，浏览器的发展方向变了。首先讲讲 Microsoft，它将自研的浏览器引擎替换为基于 Chromium 的引擎，同时它们也成为了 WebRTC 的积极贡献者。Microsoft 贡献之一是 perfect negotiation，它使得两端以更稳妥的形式协商。而且，它们还改良了屏幕捕获，使其效率更高。

另一方面，还有 Safari。苹果的 Safari 还在继续改进他们 WebRTC API。激动人心的是，最新一版的 Safari Tech Preview 中已支持了 VP9，而且还支持硬件加速，大家可以在 Safari 的“开发者设置”中启用它。

火狐浏览器增加了重传以及 transport-cc，这有助于更好地估计可用带宽，从而改善媒体质量。

另一方面，Project Zero——Google 负责产品安全性的团队，通过寻找漏洞，帮助提高 WebRTC 的安全性。这意味着如果你的库不基于浏览器，及时更新 WebRTC 库、遵守说明就更加重要了。

------

另一件激动人心的事情就是，2020 年，云游戏已经上线了。它的实现有赖于 WebRTC。Stadia（Google 的云游戏平台）已于 2019 年底推出，但 2020 年初才正式在浏览器得以支持。其云游戏搭载 VP9，提供 4k、HDR 图像和环绕声体验。这些都会通过 WebRTC 进行传输。

数月前，NVIDIA 也发布了适用于 Chromebook 的 GeForce Now，同样使用了 WebRTC。最近，Microsoft 和亚马逊也宣布支持基于浏览器的云游戏开发。这确实促使 WebRTC 从数百毫秒延迟降低到了数十毫秒延迟，同时开启了全新的应用场景。但最重要的是， 2020 年，实时通讯（RTC）对于每个人来说都是必不可少的一部分。因此，许多网络服务的使用率暴涨，涨幅从十倍到几百倍不等。大家打语音电话的次数更多了，时间更久了，群组数量和成员人数也增加了， 线上交流越来越多。所以我们需要更丰富的互动方式。

从 Google 的角度来看， 在疫情爆发的头 2、3 个月内，我们的最大需求容量增长了 30 倍。所以即使是 Google，要确保后端和所有系统功能都可以应对这么大的增长，我们也付出了很多努力。

在变化面前， WebRTC 和实时通信使用量激增。大众的日常习惯也在变化。现在不只在公司能工作， 自己的卧室、厨房里都是工作场所了。由于“社交距离”，面对面交流变得不再现实，我们需要其它与他人社交的方法。我们只能通过视频，依据别人的表情猜测他的意图，此时高清的视频质量就显得更加重要了。

每个人协作的方式不同，可能是因为我们用的设备不一样。如果你在公司， 你用的可能是台式机，那你可能会用它在会议室里开会。而下班之后，你可能会带你的笔记本电脑回家。但现在人们都在用笔记本处理各种事宜，比如同时运行应用、视频会议和文字聊天。这种场景下，电脑的使用率非常高。我们看到学校里的孩子们也在用笔记本电脑，比如 Chromebook， 但他们电脑的性能相对差一点。社交、学习线上化之后，电脑的任务处理量突然增大， 所以开展该 WebRTC 项目的意义在于我们需要帮助扩展 WebRTC，确保其运行良好。

其次，我们需要为 Web 开发者和实时通讯开发者提供更大的灵活度，让他们可以在当下开发出新的互动体验。当疫情爆发时，它阻碍我们了在 Chrome 中开展的所有实验，于是我们所做的一件事情就是专注于服务的扩展、维护。但这远远不够，特别是在提高性能方面，我们需要做得更好。

------

大家可以猜一猜，如果你要在任何使用 WebRTC 的浏览器中开展实时服务， 最耗性能的部分会是什么呢？是视频编码？音频编码？网络适配？（因为你会考虑到可能会有丢包和网络变化）又或者是渲染？

当你想在屏幕显示摄像头采集的画面时，我们可以来看看浏览器中发生了什么。我们假设你有一个通过 USB 驱动程序输入的摄像头， 驱动运行，开始处理，摄像头可能还会进行人脸检测、亮度调整等操作。这要经过浏览器内的系统，Chrome 和其它浏览器都是多进程的。多进程有助于浏览器的稳定性和安全性，比如一个组件或一个页面崩溃，或存在安全漏洞，那么它就会与其他沙盒中的组件隔离。但这也意味着进程间有大量的通信。所以如果你有一帧视频数据从摄像头被采集，它可能是 MJPEG 格式。当它开始渲染你定义媒体流的页面时， 格式可能为 I420。当从渲染进程转到 GPU 进程（需要实际在屏幕上绘制）时，需要提供最高质量的数据，此时数据可能是 RGB 格式。当它再次进入操作系统，在屏幕上进行合成时， 可能需要一个 alpha 层， 格式又要变。这中间涉及到大量转换和复制步骤。由此可见， 无论内容来自摄像头还是某一终端，仅仅把它放到屏幕上的视频帧中就要花费大量的处理时间。所以这就是 WebRTC 服务中最复杂的部分——渲染。

![img](https://pic3.zhimg.com/80/v2-f7abd945ef89a1a43b6987accb9cad8a_720w.webp)

这也是我们做了很多优化的地方。渲染变得更加高效了，可以确保我们不会在每次更新视频帧时都重新绘制。如果同时有多个视频，我们会把他们同步，再做其他操作。Chrome 团队优化了内存分配，确保每个视频帧都以最有效的方式得到分配。我们还改进了 Chrome OS 上的操作系统调度，以确保视频服务即使负载过重也能保证交互和响应。接下来的几个月里，我们将致力于从摄像头采集到视频帧渲染到屏幕这个过程的“零拷贝”。我们希望不会出现一次拷贝或转换，但所有信息都会以最高效的方式保存在图片内存里的。

同时，我们也致力于使刷新率适应视频帧率。所以在没有任何变化的情况下，我们不需要 60Hz 的屏幕刷新率，但要适应视频的帧速率，例如 25 秒一次。以下是我们觉得有用的建议：

1、避免耗时耗力的扩展操作，在 incongnito 模式下进行测试。

避免耗时耗力的扩展操作很难，它可以干扰你的服务进程，减缓你的服务速度。

2、避免安全程序干扰浏览器运行

杀毒软件若要做深度数据包检查或阻止数据包，会占用大量 CPU。

3、通过 Intel Power Gadgets 来测试

我们建议你用 Intel Power Gadgets 看看你的服务用了多少能耗。它会比只看 CPU 百分比直观的多。

4、花哨的视频效果会占用更多性能

如果你用一些花哨的动画， 比如会动的圆点来装饰你的视频帧，就会占用更多性能。尽管看起来不错，但它可能会导致视频帧卡顿一番才能渲染在屏幕上。

5、摄像头分辨率设置与传输分辨率一致

如果你使用摄像头采集，请确保打开摄像头时将其分辨率的设置，与你调用 getUserMedia 时的设置保持一致。如果你打开摄像头，设置了高清画质，格式为 VGA，那么势必需要转换很多字节的信息都会被扔掉。

6、要留意 WebAudio 的使用

WebAudio 可能比预期需要更多 CPU 来处理。

------

**关于视频编解码**

视频编解码器可用于构建更高性能服务器。因为不仅 CPU 资源很重要， 若你构建网络相关的服务，视频编解码器就显得重要起来了。如果你要把业务拓展一百倍， Google 提供一款免费的编解码器，VP8、VP9、AV1，并且他在所有浏览器中都可用。

![img](https://pic4.zhimg.com/80/v2-d81aa327ba0c930e4fcae985c889116b_720w.webp)

VP8 是目前为止浏览器内部使用最多的版本，所有浏览器都支持它。VP9 同样在市场中流通很多年了，也一直在改进。它具备 30%-50% 的节约率，以及如支持 HDR 和 4K 的高质量功能。同时，它广泛应用于谷歌内部，支持 Stadia 及其他内部服务。因为它有 VP8 中没有的功能，即能帮助你更好地适应高低带宽连接的转换。然后是 AV1。AV1 也即将在 WebRTC、一些开源实现和浏览器中使用。大多数浏览器已经可以使用它进行流式传输。希望明年能正式启用它。实际上，微软刚刚宣布他们的操作系统将支持硬件加速 AV1。性能的提升给予了开发者更大空间。

------

**WebRTC NV（Next Version）**

发布 WebRTC 1.0 之后，我们就和社区一起研发下一个版本, 该版本叫“NV”。该版本意在支持当前 WebRTC API 不可能或很难实现的新用例，比如虚拟现实。对于虚拟现实特效，就像前面提到过的笔记本电脑和机器学习的例子一样， 为了能够使用 WebRTC API 运行，我们需要更好地掌握媒体处理的技能， 比如更好控制传输和拥塞，使用编解码器进行更多自定义操作等等。

在以上这些目标上，WebRTC NV 的思路是不定义全新 API。目前已经有两个 API 和 WebRTC，PeerConnetion 和 getUserMedia 了。我们不会重新定义它们，从头开始研发。相反，我们正在做的是：允许我们使用称为“HTML 流”的接口访问端对 peer connection 内部，以及允许访问浏览器中的编解码器的接口。再加上诸如 Web Assembly 和 workers threads 的新技术，你可以在浏览器，以及集成的端对端连接中使用 Javascript 进行实时处理。

如果看一下现在的 WebRTC 的内部，你会发现媒体流就像是从网络传入时一样被拆包（depacketized）。这里会有一些丢失或延迟的适配。因此，我们对此进行了重构。

另一方面， 摄像头输入或麦克风输入已经经过编解码器如 Opus 或 VP8，去除了回声。比特率已经根据网络情况进行了适配，然后将其打包为 RTP 数据包并通过网络发送。我们想做到在 WebRTC NV 中拦截该管道，所以要从媒体框架开始。因此，我们希望能够在媒体帧从网络到达显示器，以及从摄像机麦克风到网络回到媒体帧时对其进行监听。我们希望能够更好地管理这些流。目前我们提出两个流方案，也正是我致力研究的两个 API。

![img](https://pic3.zhimg.com/80/v2-6247cd4c566d30aec7f1484bfca8df3e_720w.webp)

第一个是可插入媒体流（Insertable Media Stream）。当前的 Chrome 浏览器 86 中已提供此功能。Google 服务和其他外部服务已使用了此功能。你可以使用它来实现端到端加密，或者可以使用它向框架中添加自定义元数据（meta-data）。你要做的是在 PeerConnection 中定义此编码的可插入媒体流，并且你也可以创建流。之后，当你从摄像头获取视频帧时，它首先被编码，比如 VP8 格式，之后你可以访问它并进行流式处理。你还可以对其进行加密或标记其中一些元数据。

另一个是原始媒体流 API（Raw Media Stream）。这是标准委员会正在讨论的标准工作。目前已经有一些确切的建议了。从 Google 的角度来说，我们正在尝试这种实现。该 API 允许我们访问原始帧。它意味着，当原始帧从摄像头采集后，在还未进行视频编码前，你就可以访问这些数据了。然后你可以对其进行处理，比如实现 AR 效果。你还可以运行多个滤镜来删除背景，然后应用一些效果。比如我想把我现在的视频背景设成一个热带岛屿。这还可以应用到自定义的编解码器中，比如你此前使用的一些编解码器与现在的浏览器不兼容，那么你可以利用这个接口将数据直接传给编解码器来处理。原始媒体流 API 可以提供一种非常有效的方式来访问此原始媒体。

总结一下。虽然 WebRTC 作为 W3C 正式标准已经发布，但仍在继续改进。新的视频编解码器 AV1 可节省多达 50% 的带宽，正在 WebRTC 和网络浏览器中使用。开源代码的持续改进有望进一步减少延迟，提高视频流的质量。WebRTC NV 收集了创建补充 API 的倡议，以实现新的用例。这些 API 包括对现有 API 的扩展，以提供更多对现有功能的控制，例如可扩展视频编码，以及提供对 low-level 组件的访问的 API。后者通过集成高性能的定制 WebAssembly 组件，为网络开发者提供了更多的创新灵活性。随着新兴的 5G 网络和对更多交互式服务的需求，我们预计在未来一年内，持续增强在 WebRTC 的服务端建设。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/448159960