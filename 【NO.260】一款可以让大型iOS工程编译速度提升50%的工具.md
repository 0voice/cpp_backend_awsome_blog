# 【NO.260】一款可以让大型iOS工程编译速度提升50%的工具

## 1.cocoapods-hmap-prebuilt 是什么？

cocoapods-hmap-prebuilt 是美团平台迭代组自研的一款 cocoapods 插件，以 [Header Map 技术](https://clang.llvm.org/doxygen/classclang_1_1HeaderMap.html) 为基础，进一步提升代码的编译速度，完善头文件的搜索机制。

虽然以二进制组件的方式构建 App 是 HPX （美团移动端统一持续集成/交付平台）的主流解决方案，但在某些场景下（Profile、Address/Thread/UB/Coverage Sanitizer、App 级别静态检查、ObjC 方法调用兼容性检查等等），我们的构建工作还是需要以全源码编译的方式进行；而且在实际开发过程中，大多是以源码的方式进行开发，所以我们将实验对象设置为基于全源码编译的流程。

废话不多说，我们来看看它的实际使用效果！

总的来说，以美团和大众点评的全源码编译流程为实验对象的前提下，cocoapods-hmap-prebuilt 插件能将总链路提升 45% 以上的速度，在 Xcode 打包环节上能提升 50% 以上的速度，是不是有点动心了？

为了更好的理解这个插件的价值和功能，我们不妨先看一下当前的工程中存在的问题。

## 2.为什么现有的项目不够好？

目前，美团内的 App 都是基于 CocoaPods 做包管理方面的工作，所以在实际的开发过程中，CocoaPods 会在 `Pods/Header/` 目录下添加组件名目录和头文件软链，类似于下面的形式：

```sh
/Users/sketchk/Desktop/MyApp/Pods
└── Headers
    ├── Private
    │   └── AFNetworking
    │       ├── AFHTTPRequestOperation.h -> ./XXX/AFHTTPRequestOperation.h
    │       ├── AFHTTPRequestOperationManager.h -> ./XXX/AFHTTPRequestOperationManager.h
    │       ├── ...
    │       └── UIRefreshControl+AFNetworking.h -> ./XXX/UIRefreshControl+AFNetworking.h
    └── Public
        └── AFNetworking
            ├── AFHTTPRequestOperation.h -> ./XXX/AFHTTPRequestOperation.h
            ├── AFHTTPRequestOperationManager.h -> ./XXX/AFHTTPRequestOperationManager.h
            ├── ...
            └── UIRefreshControl+AFNetworking.h -> ./XXX/UIRefreshControl+AFNetworking.h
```

也正是通过这样的目录结构和软链，CocoaPods 得以在 Header Search Path 中添加如下的参数，使得预编译环节顺利进行。

```
$(inherited)
${PODS_ROOT}/Headers/Private
${PODS_ROOT}/Headers/Private/AFNetworking
${PODS_ROOT}/Headers/Public
${PODS_ROOT}/Headers/Public/AFNetworking
```

虽然这种构建 Search Path 的方式解决了预编译的问题，但在某些项目中，例如多达 400+ 组件的巨型项目中，会造成以下几点问题：

1. 大量的 Header Search Path 路径，会造成编译参数中的 `-I` 选项极速膨胀，在达到一定长度后，甚至会造成无法编译的情况
2. 目前美团的工程中，已经有近 5W 个头文件，这意味着不论是头文件的搜索过程，还是软链的创建过程，都会引起大量的文件 IO 操作，进而会产生一些耗时操作。
3. 编译时间会随着组件数量急剧增长，以美团和大众点评有 400+ 个组件的体量为参考，全源码打包耗时均在 1 小时以上。
4. 基于路径顺序查找头文件的方式有潜在的风险，例如重名头文件的情况，排在后面的头文件永远无法参与编译。
5. 由于 `${PODS_ROOT}/Headers/Private` 路径的存在，让引用其他组件的私有头文件变为了可能。

想解决上述的问题，好一点的情况下，可能会浪费 1 个小时，而不好的情况，就是让有风险的代码上线了，你说工程师会不会因此而感到头疼？

## 3.Header Map 是个啥？

还好 cocoapods-hmap-prebuilt 的出现，让这些问题变成了历史，不过要想理解它为什么能解决这些问题，我们得先理解一下什么是 Header Map。

**Header Map 其实是一组头文件信息映射表！**

为了更直观的理解 Header Map，我们可以在 Build Setting 中开启 Use Header Map 选项，真实的体验一下它。

![img](https://p0.meituan.net/travelcube/cae2af91ae8bb1dfeccb6a0b4bd5180325234.jpg)

然后在 Build Log 里获取相应组件里对应文件的编译命令，并在最后加上 `-v` 参数，来查看其运行的秘密：

```sh
$ clang <list of arguments> -c some-file.m -o some-file.o -v
```

在 console 的输出内容中，我们会发现一段有意思的内容：

![img](https://p0.meituan.net/travelcube/19f1298194085bedb77130ba48f46ac2618179.jpg)

通过上面的图，我们可以看到编译器将寻找头文件的顺序和对应路径展示出来了，而在这些路径中，我们看到了一些陌生的东西，即后缀名为 `.hmap` 的文件，后面还有个括号写着 headermap。

没错！它就是 Header Map 的实体。

此时 Clang 已经在刚才提到的 hmap 文件里塞入了一份头文件名和头文件路径的映射表，不过它是一种二进制格式的文件，为了验证这个的说法，我们可以通过 milend 编写的[hmap 工具](https://github.com/milend/hmap)来查其内容。

在执行相关命令（即 `hmap print`）后，我们可以发现这些 hmap 里保存的信息结构大致如下, 类似于一个 Key-Value 的形式，Key 值是头文件的名称，Value 是头文件的实际物理路径：

![img](https://p1.meituan.net/travelcube/1d733e24d900d936ca238a46fff145ec249110.jpg)

需要注意，映射表的键值内容会随着使用场景产生不同的变化，例如头文件引用是在 `"..."` 的形式下，还是 `<...>` 的形式下，又或是在 Build Phase 里 Header 的配置情况。例如，你将头文件设置为 Public 的时候，在某些 hmap 中，它的 Key 值就为 `PodA/ClassA`，而将其设置为 project 的时候，它的 Key 值可能就是 `ClassA`，而配置这些信息的地方，如下图所示：

![img](https://p0.meituan.net/travelcube/6fba39cda51ee74dc16db440976d63c178956.jpg)

至此我想你应该了解到 Header Map 到底是个什么东西了。

当然这种技术也不是一个什么新鲜事儿，在 Facebook 的 [buck](https://buck.build/) 工具中也提供了类似的东西，只不过文件类型变成了 `HeaderMap.java` 的样子。

此时，我估计你可能并不会对 buck 产生太多的兴趣，而是开始思考上一张图中 Headers 的 Public、Private、Project 到底代表着什么意思，好像很多同学从来没怎么关注过，以及为什么它会影响 hmap 里的内容？

## 4.Public，Private，Project 是个啥？

在 Apple 官方的 [Xcode Help - What are build phases?](https://help.apple.com/xcode/mac/current/#/dev50bab713d) 文档中，我们可以看到如下的一段解释：

> Associates public, private, or project header files with the target. Public and private headers define API intended for use by other clients, and are copied into a product for installation. For example, public and private headers in a framework target are copied into Headers and PrivateHeaders subfolders within a product. Project headers define API used and built by a target, but not copied into a product. This phase can be used once per target.

总的来说，我们可以知道一点，就是 Build Phases - Headers 中提到 Public 和 Private 是指可以供外界使用的头文件，而 Project 中的头文件是不对外使用的，也不会放在最终的产物中。

如果你继续翻阅一些资料，例如 [StackOverflow - Xcode: Copy Headers: Public vs. Private vs. Project?](https://stackoverflow.com/questions/7439192/xcode-copy-headers-public-vs-private-vs-project) 和 [StackOverflow - Understanding Xcode’s Copy Headers phase](https://stackoverflow.com/questions/10584936/understanding-xcodes-copy-headers-phase/18910393#18910393)，你会发现在早期 Xcode Help 的 Project Editor 章节里，有一段名为 Setting the Role of a Header File 的段落，里面详细记载了三个类型的区别。

> **Public**: The interface is finalized and meant to be used by your product’s clients. A public header is included in the product as readable source code without restriction. **Private**: The interface isn’t intended for your clients or it’s in early stages of development. A private header is included in the product, but it’s marked “private”. Thus the symbols are visible to all clients, but clients should understand that they’re not supposed to use them. **Project**: The interface is for use only by implementation files in the current project. A project header is not included in the target, except in object code. The symbols are not visible to clients at all, only to you.

至此，我们应该能够彻底了解了 Public、Private、Project 的区别。简而言之，Public 还是通常意义上的 Public，Private 则代表 In Progress 的含义，至于 Project 才是通常意义上的 Private 含义。

此时，你会不会联想到 CocoaPods 中 Podspec 的 Syntax 里还有 `public_header_files` 和 `private_header_files` 两个字段，它们的真实含义是否和 Xcode 里的概念冲突呢？

这里我们仔细阅读一下[官方文档的解释](https://guides.cocoapods.org/syntax/podspec.html)，尤其是 `private_header_files` 字段。

![img](https://p0.meituan.net/travelcube/7af19a16a81f67ad4db968565ee5ef00192569.png)

我们可以看到，`private_header_files` 在这里的含义是说，它本身是相对于 Public 而言的，这些头文件本义是不希望暴露给用户使用的，而且也不会产生相关文档，但是在构建的时候，会出现在最终产物中，只有既没有被 Public 和 Private 标注的头文件，才会被认为是真正的私有头文件，且不出现在最终的产物里。

看起来，CocoaPods 对于 Public 和 Private 的官方解释是和 Xcode 中的描述一致的，两处的 Private 并非我们通常理解的 Private，它的本意更应该是开发者准备对外开放，但又没完全 Ready 的头文件，更像一个 In Progress 的含义。

这一块是不是让你有点大跌眼镜？那么，在现实世界中，我们是否正确的使用了它们呢？

## 5.为什么用原生的 hmap 不能改善编译速度？

前面我们介绍了 hmap 是什么，以及怎么开启它（启用 Build Setting 中的 Use Header Map 选项），也介绍了一些影响生成 hmap 的因素（Public、Private、Project）。

那是不是我只要开启 Xcode 提供的 Use Header Map 就可以提升编译速度了呢?

很可惜，答案是否定的！

至于原因，我们就从下面的例子开始说起，假设我们有一个基于 CocoaPods 构建的全源码工程项目，它的整体结构如下：

- 首先，Host 和 Pod 是我们的两个 Project，Pods 下的 Target 的产物类型为 Static Library。
- 其次，Host 底下会有一个同名的 Target，而 Pods 目录下会有 n+1 个 Target，其中 n 取决于你依赖的组件数量，而 1 是一个名为 Pods-XXX 的 Target，最后，Pods-XXX 这个 Target 的产物会被 Host 里的 Target 所依赖。

整个结构看起来如下所示：

![img](https://p0.meituan.net/travelcube/c2ace3c2979ce3488b32c980425f3414168365.jpg)

当构建的产物类型为 Static Library 的时候，CocoaPods 在创建头文件产物过程中，它的逻辑大致如下：

- 不论 podspec 里如何设置 `public_header_files` 和 `private_header_files`，相应的头文件都会被设置为 Project 类型。
- 在 `Pods/Headers/Public` 中会保存所有被声明为 `public_header_files` 的头文件。
- 在 `Pods/Headers/Private` 中会保存所有头文件，不论是 `public_header_files` 或者 `private_header_files` 描述到，还是那些未被描述的，这个目录下是当前组件的所有头文件全集。
- 如果 podspec 里未标注 Public 和 Private 的时候，`Pods/Headers/Public` 和 `Pods/Headers/Private` 的内容一样且会包含所有头文件。

正是由于这种机制，会导致一些有意思的问题发生。

- 首先，由于所有头文件都被当做最终产物保留下来，在结合 Header Search Path 里 `Pods/Headers/Private` 路径的存在，我们完全可以引用到其他组件里的私有头文件，例如我只要使用 `#import <SomePod/Private_Header.h>` 的方式，就会命中私有文件的匹配路径。
- 其次，就是在 Static Library 的状况下，一旦我们开启了 Use Header Map，结合组件里所有头文件的类型为 Project 的情况，这个 hmap 里只会包含 `#import "ClassA.h"` 的键值引用，也就是说只有 `#import "ClassA.h"` 的方式才会命中 hmap 的策略，否则都将通过 Header Search Path 寻找其相关路径，例如下图中的 PodB，在其 build 的过程中，Xcode 会为 PodB 生成 5 个 hmap 文件，也就是说这 5 个文件只会在编译 PodB 中使用，其中 PodB 会依赖 PodA 的一些头文件，但由于 PodA 中的头文件都是 Project 类型的，所以其在 hmap 里的 Key 全部为 `ClassA.h` ，也就是说我们只能以 `#import "ClassA.h"` 的方式引入。

![img](https://p0.meituan.net/travelcube/bf6ca4218335d92f794a1eb884bd727d369531.jpg)

而我们也知道，在引用其他组件的时候，通常都会采用 `#import <A/A.h>` 的方式引入。至于为什么会用这种方式，一方面是这种写法会明确头文件的由来，避免问题，另一方面也是这种方式可以让我们在是否开启 clang module 中随意切换。当然，还有一点就是Apple 在 WWDC 里曾经不止一次建议开发者使用这种方式来引入头文件。

接着上面的话题来说，所以说在 Static Library 的情况下且以 `#import <A/A.h>` 这种标准方式引入头文件时，开启 Use Header Map 选项并不会帮我们提升编译速度。

但真的就没有办法使用 Header Map 了么？

## 6.cocoapods-hmap-prebuilt 诞生了

当然，总是有办法解决的，我们完全可以自己动手做一个基于 CocoaPods 规则下的 hmap 文件，正是基于这个想法，美团自研的 cocoapods-hmap-prebuilt 插件诞生了。

它的核心功能并不多，大概有以下几点：

- 借助 CocodPods 处理 Header Search Path 和创建头文件 soft link 的时机，构建了头文件索引表并以此生成 n+1 个 hmap 文件（n 是每个组件自己的 Private Header 信息，1 是所有组件公共的 Public Header 信息）。
- 重写 xcconfig 文件里的 Header Search Path 到对应的 hmap 文件上，一条指向组件自己的 private hmap，一条指向所有组件共用的 public hmap。
- 针对 public hmap 里的重名头文件进行了特殊处理，只允许保存`组件名/头文件名`方式的 Key-Value，排查重名头文件带来的异常行为。
- 将组件自身的 Ues Header Map 功能关闭，减少不必要的文件创建和读取。

听起来可能有点绕，内容也有点多，不过这些你都不用关心，你只需要通过以下 2 个步骤就能将其使用起来：

1. 在 Gemfile 里声明插件。
2. 在 Podfile 里使用插件。

```ruby
// this is part of Gemfile
source 'http://sakgems.sankuai.com/' do
  gem 'cocoapods-hmap-prebuilt'
  gem 'XXX'
  ...
end

// this is part of Podfile
target 'XXX' do
  plugin 'cocoapods-hmap-prebuilt'
  pod 'XXX'
  ...
end
```

除此之外，为了拓展其实用性，我们还提供了头文件补丁（解决重名头文件的定向选取）和环境变量注入（无侵入的在其他系统中使用）的能力，便于其在不同场景下的使用。

## 7.总结

至此，关于 cocoapods-hmap-prebuilt 的介绍就要结束了。

回看整个故事的开始，Header Map 是我在研究 Swift 和 Objective-C 混编过程中发现的一个很小的知识点，而且 Xcode 自身就实现了一套基于 Header Map 的功能，在实际的使用过程中，它的表现并不理想。

但幸运的是，在后续的探索的过程中，我们发现了为什么 Xcode 的 Header Map 没有生效，以及为什么它与 CocoaPods 出现了不兼容的情况，虽然它的原理并不复杂，核心点就是将文件查找和读取等 IO 操作编变成了内存读取操作，但结合实际的业务场景，我们发现它的收益是十分可观的。

或许这是在提醒我们，要永远对技术保持一颗好奇的心！

其实，利用 Clang Module 技术也可以解决本文一开始提到的几个问题，但它并不在这篇文章的讨论范围中，如果你对 Clang Module 或者对 Swift 与 Objective-C 混编感兴趣，欢迎阅读参考文档中的 《从预编译的角度理解 Swift 与 Objective-C 及混编机制》一文，以了解更多的详细信息。

## 8.参考文档

- [Apple - WWDC 2018 Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/)
- [Apple 的 HeaderMap.cpp 源码](https://opensource.apple.com/source/lldb/lldb-167.2/llvm/tools/clang/lib/Lex/HeaderMap.cpp.auto.html)

## 9.作者

- 思琦，笔名 [SketchK](https://github.com/SketchK)，美团 iOS 工程师，目前负责移动端 CI/CD 方面的工作及平台内 Swift 技术相关的事宜。
- 旭陶，美团 iOS 工程师，目前负责 iOS 端开发提效相关事宜。
- 霜叶，2015 年加入美团，先后从事过 Hybrid 容器、iOS 基础组件、iOS 开发工具链和客户端持续集成门户系统等工作。

原文作者：美团技术团队

原文链接：https://tech.meituan.com/2021/02/25/cocoapods-hmap-prebuilt.html