# 【NO.256】FlutterWeb性能优化探索与实践

## 1.背景

### 1.1 关于FlutterWeb

时间回拨到 2018 年，Google 首次公开 FlutterWeb Beta 版，表露出要实现一份代码、多端运行的愿景。经过无数工程师两年多的努力，在今年年初（2021 年 3 月份），Flutter 2.0 正式对外发布，它将 FlutterWeb 功能并入了 Stable Channel，意味着 Google 更加坚定了多端复用的决心。

![图1 FlutterWeb历史](https://p0.meituan.net/travelcube/74f03228b641dda28ac8e3c01f92202436263.jpg)

图1 FlutterWeb历史



当然 Google 的“野心”不是没有底气的，主要体现在它强大的跨端能力上，我们看一下 Flutter 的跨端能力在 Web 侧是如何体现的：

![图2 Flutter跨端能力](https://p0.meituan.net/travelcube/d25b2af72b128a3397334d73bd1b5c01156510.jpg)

图2 Flutter跨端能力



上图分别是 FlutterNative 和 FlutterWeb 的架构图。通过对比可以看出，应用层 Framework 是公用的，意味着在 FlutterWeb 中我们也可以直接使用 Widgets、Gestures 等组件来实现逻辑跨端。而关于渲染跨端，FlutterWeb 提供了两种模式来对齐 Engine 层的渲染能力：Canvaskit Render 和 HTML Render，下方表格对两者的区别进行了对比：

![图3 模式对比](https://p0.meituan.net/travelcube/0adf2c73ec2397521b897c67a5ae1afc142170.png)

图3 模式对比



**Canvaskit Render 模式**：底层基于 Skia 的 WebAssembly 版本，而上层使用 WebGL 进行渲染，因此能较好地保证一致性和滚动性能，但糟糕的兼容性（WebAssembly 从 Chrome 57 版本才开始支持）是我们需要面对的问题。此外 Skia 的 WebAssembly 文件大小达到了 2.5M，且 Skia 自绘引擎需要字体库支持，这意味着需要依赖超大的中文字体文件，对页面加载性能影响较大，因此目前并不推荐在 Web 中直接使用 Canvaskit Render（官方也建议将 Canvaskit Render 模式用于桌面应用）。

**HTML Render 模式**：利用 HTML + Canvas 对齐了 Engine 层的渲染能力，因此兼容性表现优秀。另外，MTFlutterWeb 对滚动性能已有过探索和实践，目前能够应对大部分业务场景。而关于加载性能，此模式下的初始包为 1.2M，是 Canvaskit Render 模式产物体积的 1/2，且我们可对编译流程进行干预，控制输出产物，因此优化空间较大。

基于以上原因，美团外卖技术团队选择在 HTML Render 模式下对 FlutterWeb 页面的性能进行优化探索。

### 1.2 业务现状

美团外卖商家端以 App、PC 等多元化的形态为商家提供了订单管理、商品维护、顾客评价、外卖课堂等一系列服务，且 App、PC 双端业务功能基本对齐。此外，我们还在 PC 上特供了针对连锁商家的多店管理功能。同时，为满足平台运营诉求，部分业务具有外投 H5 场景，例如美团外卖商家课堂，它是一个以文章、视频等形式帮助商家学习外卖运营知识、了解行业发展和跟进经营策略的内容平台，具有较强的传播属性，因此我们提供了站外分享的能力。

![图4 业务形态](https://p0.meituan.net/travelcube/8721e4664a78c49b7367ed1c9de54cd11909982.png)

图4 业务形态



为了实现多端（App、PC、H5）复用，提升研发效率，我们于 2021 年年初开始着手 [MTFlutterWeb](https://mp.weixin.qq.com/s/GjFC5_85pIk9EbKPJXZsXg) 研发体系的建设。目前，我们基于 MTFlutterWeb 完成提效的业务超过了 9 个，在 App 中，能够基于 FlutterNative 提供高性能的服务；在 PC 端和 Mobile 浏览器中，利用 FlutterWeb 做到了低成本适配，提升了产研的整体效率。

然而，加载性能问题是 MTFlutterWeb 应用推广的最大障碍。这里依然以美团外卖商家课堂业务为例，在项目之初页面完全加载时间 TP90 线达到了 6s 左右，距离我们的指标基线值（页面完全加载时间 TP90 线不高于 3s，基线值主要依据美团外卖商家端的业务场景、用户画像等来确定）有些差距，用户访问体验有很大的提升空间，因此 FlutterWeb 页面加载性能优化，是我们亟需解决的问题。

## 2.挑战

不过，想要突破 FlutterWeb 页面加载的性能瓶颈，我们面临的挑战也是巨大的。这主要体现在 FlutterWeb 缺失静态资源的优化策略，以及复杂的架构设计和编译流程。下图展示了 Flutter 业务代码被转换成 Web 平台产物的流程，我们来具体进行分析：

![图5 FlutterWeb 编译流程](https://p1.meituan.net/travelcube/ea32b330219a471097a9b3b2f8e02f6f108678.jpg)

图5 FlutterWeb 编译流程



1. **Framework、Flutter_Web_SDK**（Flutter_Web_SDK 基于 HTML、Canvas，承载 HTML Render 模式的具体实现）等底层 SDK 是可被业务代码直接引入的，帮助我们快速开发出跨端应用；
2. **flutter_tools** 是各平台（Android、iOS、Web）的编译入口，它接收 flutter build web 命令和参数并开始编译流程，同时等待处理结果回调，在回调中我们可对编译产物进行二次加工；
3. **frontend_server** 负责将 Dart 转换为 AST，生成 kernel 中间产物 app.dill 文件（实际上各平台的编译过程都会生成这样的中间产物），并交由各平台 Compiler 进行转译；
4. **Dart2JS Compiler** 是 Dart-SDK 中具体负责转译 JS 的模块，它将上述中间产物 app.dill 进行读取和解析，并注入 Math、List、Map 等 JS 工具方法，最终生产出 Web 平台所能执行的 JS 文件。
5. **编译产物**主要为 main.dart.js、index.html、images 等静态资源，FlutterWeb 对这些静态资源缺少常规 Web 项目中的优化手段，例如：文件 Hash 化、文件分片、CDN 支持等。

可以看出，要完成对 FlutterWeb 编译产物的优化，就需要干预 FlutterWeb 的众多编译模块。而为了提升整体的编译效率，大部分模块都被提前编译成了 snapshot 文件（ 一种 Dart 的编译产物，可被 Dart VM 所运行，用于提升执行效率），例如：flutter_tools.snapshot、frontend_server.snapshot、dart2js.snapshot 等，这又加大了对 FlutterWeb 编译流程进行干预的难度。

## 3.整体设计

如前文所述，为了实现逻辑、渲染跨平台，Flutter 的架构设计及编译流程都具有一定的复杂性。但由于各平台（Android、iOS、Web）的具体实现是解耦的，因此我们的思路是定位各模块（Dart-SDK、Framework、Flutter_Web_SDK、flutter_tools）的 Web 平台实现并寻求优化，整体设计图如下所示：

![图6 整体设计](https://p0.meituan.net/travelcube/f965d1903516dd6ba8613a6b691d7ef7165927.jpg)

图6 整体设计



- **SDK 瘦身**：我们分别对 FlutterWeb 所依赖的 Dart-SDK、Framework、Flutter_Web_SDK 进行了瘦身，并将这些精简版 SDK 集成合入 CI/CD（持续集成与部署）系统，为减小产物包体积奠定了基础；
- **编译优化**：此外，我们在 flutter_tools 中的编译流程做了干预，分别进行了 JS 文件分片、静态资源 Hash 化、资源文件上传 CDN 等优化，使得这些在常规 Web 应用中基础的性能优化手段得以在 FlutterWeb 中落地。同时加强了 FlutterWeb 特殊场景下的资源优化，如：字体图标精简、Runtime Manifest 隔离、Mobile/PC 分平台打包等；
- **加载优化**：在编译阶段进行静态资源优化后，我们在前端运行时，支持了资源预加载与按需加载，通过设定合理的加载时机，从而减小初始代码体积，提升页面首屏的渲染速度。

下面，我们分别对各项优化进行详细的说明。

## 4.设计与实践

### 4.1 精简 SDK

#### 4.1.1 包体积分析

工欲善其事，必先利其器，在开始做体积裁剪之前，我们需要一套类似于 [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer) 的包体积分析工具，便于直观地比较各个模块的体积占比，为优化性能提供帮助。

Dart2JS 官方提供了 [–dump-info](https://dart.dev/tools/dart2js) 命令选项来分析 JS 产物，但其表现差强人意，它并不能很好地分析各个模块的体积占比。这里更推荐使用 [source-map-explorer](https://www.npmjs.com/package/source-map-explorer) ，它的原理是通过 sourcemap 文件进行反解，能清晰地反映出每个模块的占用大小，为 SDK 的精简提供了指引。下图展示了 FlutterWeb JS 产物的反解信息（截图仅包含 Framework 和 Flutter_Web_SDK）：

![图7 反解信息](https://p0.meituan.net/travelcube/a4d3a6c3838bffe372798b1c5a644653732286.png)

图7 反解信息



#### 4.1.2 SDK 裁剪

FlutterWeb 依赖的 SDK 主要包括 Dart-SDK、Framework 和 Flutter_Web_SDK，这些 SDK 对包体积的影响是巨大的，几乎贡献了初始化包的所有大小。虽然在 Release 模式下的编译流程中，Dart Compiler 会利用 [Tree-Shaking](https://dart.dev/tools/dart2js#helping-dart2js-generate-efficient-code) 来剔除那些引入但未使用的 packages、classes、functions 等，很大程度上减少了包体积。但这些 SDK 中仍然存在一些能被进一步优化的代码。

以 Flutter Framework 为例，由于它是全平台公用的模块，因此不可避免地存在各平台的兼容逻辑（通常以 if-else、switch 等条件判断形式出现），而这部分代码是不能被 Tree-Shaking 剔除的，我们观察如下的代码：

```dart
// FileName: flutter/lib/src/rendering/editable.dart
void _handleKeyEvent(RawKeyEvent keyEvent) {
  if (kIsWeb) {
    // On web platform, we should ignore the key.
    return;
  }
  // Other codes ...
}
```

上述代码选自 Framework 中的 RenderEditable 类，当 kIsWeb 变量为真，表示当前应用运行在 Web 平台。受限于 Tree-Shaking 的机制原理，上述代码中，其它平台的兼容逻辑即注释 Other codes 的部分是无法被剔除的，但这部分代码，对 Web 平台来说却是 Dead Code（永远不可能被执行到的代码），是可以被进一步优化的。

![图8 部分功能构成](https://p1.meituan.net/travelcube/49bbeaa272a3044c362f3bbf520d14ff197851.jpg)

图8 部分功能构成



上图展示了 SDK 的一部分功能构成，从图中可以看出，FlutterWeb 依赖的这些 SDK 中包含了一些使用频率较低的功能，例如：蓝牙、USB、WebRTC、陀螺仪等功能的支持。为此，我们提供了对这些长尾功能的定制能力（这些功能默认不开启，但业务可配置），将未被启用长尾的功能进行裁剪。

通过上述分析可得，我们的思路就是对 Dead Code 进行二次剔除，以及对这些长尾功能做裁剪。基于这样的思路，我们深入 Dart-SDK、Framework 和 Flutter_Web_SDK 各个击破，最终将 JS Bundle 产物体积由 1.2M 精简至 0.7M，为 FlutterWeb 页面性能优化打下了坚实的基础。

![图9 精简成果](https://p0.meituan.net/travelcube/7ca944005eb0148f2bd163fc5da1d49815909.jpg)

图9 精简成果



#### 4.1.3 SDK 集成 CI/CD

为了提升构建效率，我们将 FlutterWeb 依赖的环境定制为 Docker 镜像，集成入 CI/CD（持续集成与部署）系统。SDK 裁剪后，我们需要更新 Docker 镜像，整个过程耗时较长且不够灵活。因此，我们将 Dart-SDK、Framework、Flutter_Web_SDK 按版本打包传至云端，在编译开始前读取 CI/CD 环境变量：sdk_version（SDK 版本号），远程拉取相应版本的 SDK 包，并替换当前 Docker 环境中的对应模块，基于以此方案实现 SDK 的灵活发布，具体流程图如下图所示：

![图10 集成CI/CD](https://p0.meituan.net/travelcube/02daa712453380dcdad7dd32b24af016101271.jpg)

图10 集成CI/CD



### 4.2 JS 分片

FlutterWeb 编译之后默认会生成 main.dart.js 文件，它囊括了 SDK 代码以及业务逻辑，这样会引起以下问题：

1. **功能无法及时更新**：为了实现浏览器的缓存优化，我们的项目开启了对静态资源的强缓存，若 main.dart.js 产物不支持 Hash 命名，可能导致程序代码不能被及时更新；
2. **无法使用 CDN**：FlutterWeb 默认仅支持相对域名的资源加载方式，无法使用当前域名以外的 CDN 域名，导致无法享受 CDN 带来的优势；
3. **首屏渲染性能不佳**：虽然我们进行了 SDK 瘦身，但 main.dart.js 文件依然维持在 0.7M 以上，单一文件加载、解析时间过长，势必会影响首屏的渲染时间。

针对文件 Hash 化和 CDN 加载的支持，我们在 flutter_tools 编译流程中对静态资源进行二次处理：遍历静态资源产物，增加文件 Hash（文件内容 MD5 值），并更新资源的引用；同时通过定制 Dart-SDK，修改了 main.dart.js、字体等静态资源的加载逻辑，使其支持 CDN 资源加载。

更详细的方案设计请参考[《Flutter Web在美团外卖的实践》](https://mp.weixin.qq.com/s/GjFC5_85pIk9EbKPJXZsXg)一文。下面我们重点介绍 main.dart.js 分片相关的一些优化策略。

#### 4.2.1 Lazy Loading

Flutter 官方提供 `deferred as` 关键字来实现 Widget 的懒加载，而 dart2js 在编译过程中可以将懒加载的 Widget 进行按需打包，这样的拆包机制叫做 Lazy Loading。借助 Lazy Loading，我们可以在路由表中使用 deferred 引入各个路由（页面），以此来达到业务代码拆离的目的，具体使用方法和效果如下所示：

```dart
// 使用方式
import 'pages/index/index.dart' deferred as IndexPageDefer;
{
  '/index': (context) => FutureBuilder(
    future: IndexPageDefer.loadLibrary(),
    builder: (context, snapshot) => IndexPageDefer.Demo(),
  )
  ... ...
}
```

![图11 效果演示](https://p1.meituan.net/travelcube/01482f3ac8d4efcd991fcc2227d1ca3b58393.png)

图11 效果演示



使用 Lazy Loading 后，业务页面的代码会被拆分到了多个 PartJS（对应图中 xxx.part.js 文件） 中。这样看似解决了业务代码与 SDK 耦合的问题，但在实际操作过程中，我们发现每次业务代码的变动，仍然会导致编译后的 main.dart.js 随之发生变化（文件 Hash 值变化）。经过定位与跟踪，我们发现这个变化的部分是 PartJS 的加载逻辑和映射关系，我们称之为 Runtime Manifest。因此，需要设计一套方案对 Runtime Manifest 进行抽离，来保证业务代码的修改对 main.dart.js 的影响达到最低。

#### 4.2.2 Runtime Manifest抽离

通过对业务代码的抽离，此时 main.dart.js 文件的构成主要包含 SDK 和 Runtime Manifest：

![图12 main.dart.js构成](https://p0.meituan.net/travelcube/26a65505405a5d92a5fd3b392689e19c45090.jpg)

图12 main.dart.js构成



那如何能将 Runtime Manifest 进行抽离呢？对比常规 Web 项目，我们的处理方式是把 SDK、Utils、三方包等基础依赖，利用 Webpack、Rollup 等打包工具进行抽离并赋予一个稳定的 Hash 值。同时，将 Runtime Manifest （分片文件的加载逻辑和映射关系）注入到 HTML 文件中，这样保证了业务代码的变动不会影响到公共包。借助常规 Web 项目的编译思路，我们深入分析了 FlutterWeb 中 Runtime Manifest 的生成逻辑和 PartJS 的加载逻辑，定制出如下的解决方案：

![图13 Runtime Manifest抽离](https://p0.meituan.net/travelcube/90aff428fd23428db2418f22fea51765140812.jpg)

图13 Runtime Manifest抽离



在上图中，Runtime Manifest 的生成逻辑位于 Dart2JS Compiler 模块，在该生成逻辑中，我们对 Runtime Manifest 代码块进行了标记，之后在 flutter_tools 中将标记的 Runtime Manifest 代码块抽离并写入 HTML 文件中（以 JS 常量形式存在）。而在 PartJS 的加载流程中，我们将 manifest 信息的读取方式改为了 JS 常量的获取。按照这样的拆分方式，业务代码的变更只会改变 Runtime Manifest 信息 ，而不会影响到 main.dart.js 公共包。

#### 4.2.3 main.dart.js 切片

经过以上引入 Lazy Loading、Runtime Manifest 抽离，main.dart.js 文件的体积稳定在 0.7M 左右，浏览器对大体积单文件的加载，会有很沉重的网络负担，所以我们设计了切片方案，充分地利用浏览器对多文件并行加载的特性，提升文件的加载效率。

具体实现方案为：将 main.dart.js 在 flutter_tools 编译过程拆分成多份纯文本文件，前端通过 XHR 的方式并行加载并按顺序拼接成 JavaScript 代码置于 < script > 标签中，从而实现切片文件的并行加载。

![图14 并行加载](https://p1.meituan.net/travelcube/44f8b27b16d30725788efbeeb2f21b5a38647.jpg)

图14 并行加载



### 4.3 预加载方案

如上一节所述，虽然我们做了很多工作来稳定 main.dart.js 的内容，但在 Flutter Tree-Shaking 的运行机制下，各个项目引用不同的 Framework Widget，就会导致每个项目生成的 main.dart.js 内容不一致。随着接入 FlutterWeb 的项目越来越多，每个业务的页面互访概率也越来越高，我们的期望是当访问 A 业务时，可以预先缓存 B 业务引用的 main.dart.js，这样当用户真正进入 B 业务时就可以节省加载资源的时间，下面为详细的技术方案。

#### 4.3.1 技术方案

我们把整体的技术方案分为编译、监听、运行三个阶段。

1. 编译阶段，在发布流水线上根据前期定制的匹配规则，筛选出符合条件的资源文件路径，生成云端 JSON 并上传；
2. 监听阶段，在 DOMContentLoaded 之后，对网络资源、事件、DOM 变动进行监听，并对监听结果根据特定规则进行分析加权，得到一个首屏加载完成的状态标识；
3. 运行阶段，在首屏加载完成之后对配置平台下发的云端 JSON 文件进行解析，对符合配置规则的资源进行 HTTP XHR 预加载，从而实现文件的预缓存功能。

下图为预缓存的整体方案设计：

![图15 预缓存方案设计](https://p0.meituan.net/travelcube/2c53abadd2e0fddc44074427dc63f89377786.jpg)

图15 预缓存方案设计



**编译阶段**

编译阶段会扩展现有的发布流水线，在 flutter build 之后增加 prefetch build 作业，这样 build 之后就可以对产物目录进行遍历和筛选，得到我们所需资源进而生成云端 JSON，为运行阶段提供数据基础。下面的流程图为编译阶段的详细方案设计：

![图16 预缓存编译阶段](https://p0.meituan.net/travelcube/4164f54b75562bf6ea3f9578dbf9843084051.jpg)

图16 预缓存编译阶段



编译阶段分为三部分：

1. 第一部分：根据不同的发布环境，初始化线上/线下的配置平台，为配置文件的读写做好准备；
2. 第二部分：下载并解析配置平台下发的资源组 JSON，筛选出符合配置规则的资源路径，更新 JSON 文件并发布到配置平台；
3. 第三部分：通过发布流水线提供的 API，把 PROJECT_ID、发布环境注入HTML文件中，为运行阶段提供全局变量以便读取。

通过对流水线编译期的整合，我们可以生成新的云端 JSON 并上传到云端，为运行阶段的下发提供数据基础。

**监听阶段**

我们知道，浏览器对文件请求的并发数量是有限制的，为了保证浏览器对当前页面的渲染处于高优先级，同时还能完成预缓存的功能，我们设计了一套对缓存文件的加载策略，在不影响当前页面加载的情况下，实现对缓存文件的加载操作。以下为详细的技术方案：

![图17 预缓存监听阶段](https://p0.meituan.net/travelcube/4e62f7df1e747de7baed6615f3e5aef3181257.jpg)

图17 预缓存监听阶段



在页面 DOMContentLoaded 之后，我们会监听三部分的的变化。

1. 第一部分是监听 DOM 的变化。这部分主要是在页面发生 Ajax 请求之后，随着MV模式的变动，DOM 也会随之发生变化。我们使用浏览器提供的 MutationObserver API 对 DOM 变化进行收集，并筛选有效节点进行深度优先遍历，计算每个 DOM 的递归权重值，低于阈值我们就认为首屏已加载完成。
2. 第二部分是监听资源的变化。我们利用浏览提供的 PerformanceObserver API，筛选出 img/script 类型的资源，在 3 秒内收集的资源没有增加时，我们认为首屏已加载完成。
3. 第三部分是监听 Event 事件。当用户发生 click、wheel、touchmove 等交互行为时，我们就认为当前页面处于一个可交互的状态，即首屏加载已完成，这样会在后续进行资源的预缓存。

通过上述步骤，我们就可以得到一个首屏渲染完成的时机，之后就可以实现预缓存功能了。以下为预缓存功能的实现。

**运行阶段**

预缓存的整体流程为：下载编译阶段生成的云端 JSON，解析出需要进行预缓存资源的 CDN 路径，最后通过 HTTP XHR 进行缓存资源进行请求，利用浏览器本身的缓存策略，把其他业务的资源文件写入。当用户访问已命中缓存的页面时，资源已被提前加载，这样可以有效地减少首屏的加载时间。下图为运行阶段的详细方案设计：

![图18 预缓存运行阶段](https://p0.meituan.net/travelcube/c5d529488ce3085a395d1dd6753d1317129511.jpg)

图18 预缓存运行阶段



在监听阶段，我们可以获取到页面的首屏渲染完成的时机，会获取到云端 JSON，首先判断该项目的缓存是否为启用状态。当该项目可用时，会根据全局变量 PROJECT_ID 进行资源数组的匹配，再以 HTTP XHR 方式进行预访问，把缓存文件写入浏览器缓存池中。至此，资源预缓存已执行完毕。

#### 4.3.2 效果展示与数据对比

当有页面间互访问命中预缓存时，浏览器会以 200（Disk Cache）的方式返回数据，这样就节省了大量资源加载的时间，下图为命中缓存后资源加载情况：

![图19 预缓存效果展示](https://p0.meituan.net/travelcube/8c8273203fdced9fff62583ddca1cc9066013.png)

图19 预缓存效果展示



目前，美团外卖商家端业务已有 10+ 个页面接入了预缓存功能，资源加载 90 线平均值由 400ms 下降到 350ms，降低了 12.5%；50 线平均值由 114ms 下降到 100ms，降低了 12%。随着项目接入接入越来越多，预缓存的效果也会越发的明显。

![图20 预缓存数据展示](https://p1.meituan.net/travelcube/7a6fd4860841b106a4b128410ac660a521693.jpg)

图20 预缓存数据展示



### 4.4 分平台打包

如前文所述，美团外卖商家业务大部分都是双端对齐的。为了实现提效的最大化，我们对 FlutterWeb 的多平台适配能力进行加强，实现了 FlutterWeb 在 PC 侧的复用。

在 PC 适配过程中，我们不可避免地需要书写双端的兼容代码，如：为了实现在列表页面中对卡片组件的复用。为此我们开发了一个适配工具 ResponsiveSystem，分别传入 PC 和 App 的各端实现，内部会区分平台完成适配：

```dart
// ResponsiveSystem 使用举例
Container(
  child: ResponsiveSystem(
    app: AppWidget(),
    pc: PCWidget(),
  ),
)
```

上述代码能较方便的实现 PC 和 App 适配，但 AppWidget 或 PCWidget 在编译过程中都将无法被 Tree-Shaking 去除，因此会影响包体积大小。对此，我们将编译流程进行优化，设计分平台打包方案：

![图21 分平台打包](https://p0.meituan.net/travelcube/2fb7fc9166be74f54d7e8b09d3fe0d66156311.jpg)

图21 分平台打包



1. 修改 flutter-cli，使其支持 –responsiveSystem 命令行参数；
2. 我们在 flutter_tools 中的 AST 分析阶段增加了额外的处理：ResponsiveSystem 关键字的匹配，同时结合编译平台（PC 或 Mobile）来进行 AST 节点的改写；
3. 去除无用 AST 节点后，生成各个平台的代码快照（每份快照仅包含单独平台代码）；
4. 根据代码快照编译生成 PC 和 App 两套 JS 产物，并进行资源隔离。而对于 images、fonts 等公用资源，我们将其打入 common 目录。

通过这样的方式，我们去除了各自平台的无用代码，避免了 PC 适配过程中引起的包体积问题。依然以美团外卖商家课堂业务（6 个页面）为例，接入分平台打包后，单平台代码体积减小 100KB 左右。

![图22 效果展示](https://p0.meituan.net/travelcube/710db8f7fd2aa75b4f35761a4492b2f325051.jpg)

图22 效果展示



### 4.5 图标字体精简

当访问 FlutterWeb 页面时，即使在业务代码中并未使用 Icon 图标，也会加载一个 920KB 的图标字体文件：MaterialIcons-Regular.woff。通过探究，我们发现是 Flutter Framework 中一些系统 UI 组件（如：CalendarDatePicker、PaginatedDataTable、PopupMenuButton 等）使用到了 Icon 图标导致，且 Flutter 为了便于开发者使用，提供了全量的 Icon 图标字体文件。

Flutter 官方提供的 `--tree-shake-icons` 命令选项是将业务使用到的 Icon 与 Flutter 内部维护的一个缩小版字体文件（大约 690KB）进行合并，能一定程度上减小字体文件大小。而我们需要的是只打包业务使用的 Icon，所以我们对官方 `tree-shake-icons` 进行了优化，设计了 Icon 的按需打包方案：

![图23 图标字体精简](https://p0.meituan.net/travelcube/51cb5d4bd817d4ca17e0d2b735015757113958.jpg)

图23 图标字体精简



1. 扫描全部业务代码以及依赖的 Plugins、Packages、Flutter Framework，分析出所有用到的 Icon；
2. 把扫描到的所有 Icon 与 material/icons.dart（该文件包含 Flutter Icon 的 unicode 编码集合）进行对比，得到精简后的图标编码列表：iconStrList；
3. 使用 FontTools 工具把 iconStrList 生成字体文件 .woff，此时的字体文件仅包含真正使用到的 Icon。

通过以上的方案，我们解决了字体文件过大带来的包体积问题，以美团外卖课堂业务（业务代码中使用了 5 个 Icon）为例，字体文件从 920KB 精简为 11.6kB。

![图24 效果展示](https://p1.meituan.net/travelcube/e6d006831983e43405eff68e45067f5d24759.jpg)

图24 效果展示



## 5.总结与展望

综上所述，我们基于 HTML Render 模式对 FlutterWeb 性能优化进行了探索和实践，主要包括 SDK（Dart-SDK、Framework、Flutter_Web_SDK）的精简，静态资源产物优化（例如：JS 分片、文件 Hash、字体图标文件精简、分平台打包等）和前端资源加载优化（预加载与按需请求）。**最终使得 JS 产物由 1.2M 减少至 0.7M（非业务代码），页面完全加载时间 TP90 线由 6s 降到了 3s**，这样的结果已能满足美团外卖商家端的大部分业务要求。而未来的规划将聚焦于以下3个方向：

1. **降低 Web 端适配成本**：目前已有 9+ 个业务借助 MTFlutterWeb 实现多端复用，但在 Web 侧（尤其是 PC 侧）的适配效率依然有优化空间，目标是将适配成本降低到 10% 以下（目前大约是 20% ）；
2. **构建 FlutterWeb 容灾体系**：Flutter 动态化包有一定的加载失败概率，而 FlutterWeb 作为兜底方案，能提升整体业务的加载成功率。此外 FlutterWeb 可以提供“免安装更新”的能力，降低 FlutterNative 老旧历史版本的维护成本；
3. **性能优化的持续推进**：性能优化的阶段性成果为 MTFlutterWeb 的应用推广巩固了基础，但依然是有进一步优化空间的，例如：目前我们仅将业务代码和 Runtime Manifest 进行了拆离，而 Framework 及 三方包在一定程度上也影响到了浏览器缓存的命中率，将这部分代码进行抽离，可进一步提升页面加载性能。

美团外卖技术团队正在基于 FlutterWeb 做更多的探索和尝试。如果你对这方面的技术也比较感兴趣，可以在文末留言，跟我们一起讨论。也欢迎大家给提出一些建议，非常感谢。

原文作者：美团技术团队

原文链接：https://tech.meituan.com/2021/12/16/flutterweb-practice-in-meituan-waimai.html