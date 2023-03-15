# 【NO.367】新知 | 腾讯云视立方播放器技术实现与应用

本次分享的主要内容分为三块，一是腾讯云视立方播放器的相关技术背景，二是业务侧经典场景应用方案，三是短视频场景应用的技术实现方案。

## 1.**腾讯云视立方播放器技术背景**



腾讯云视立方播放器基于腾讯视频同款内核打造，完美融合了腾讯视频的能力，视频兼容性、适配能力以及播放稳定性均大幅提升，解决了系统引擎各种播放异常问题。

- **功能全面：**覆盖长短视频点播、直播场景，具备业界领先的自适应技术、画质提升及版权保护等解决方案，满足各项业务诉求。
- **性能可靠：**经过亿级用户验证，性能稳定可靠。具备业界领先的启播解决方案，对启播速度做了深度优化，启播平均速度低至100毫秒。
- **兼容适配：**主流视频格式协议100%覆盖，对于大量在系统播放器中播放异常的非常规编码的视频也能可靠稳定的播放。
- **多平台支持：**支持安卓、iOS、 Web以及Flutter等多种平台。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLCrYygnVqxYtCkd043NKHtrzA1ziaY3fDedUtMqvMYwXibibx2aZ0nX3gQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

在追求卓越内核的同时，腾讯云视立方播放器也非常关注业务的接受成本。为了降低业务侧的开发难度及工作量，所有主流场景均有完整组件&解决方案Demo提供。这些Demo全部开源，本身完整可直接使用且支持自定义修改。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLOPmoOGSJPXL6VD63nltPdgbDGNppyOicUbZ5ewH9uWr9msy6vpTiaAaQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 2.**经典场景应用方案**



播放器UI组件封装了完备的基础操作功能，可以实现小窗播放、切换全屏播放、滑动控制亮度音量、滑动控制进度等业务侧常见的应用操作。同时播放器UI组件也支持弹幕、动态水印、会员试看、剧集播放等进阶功能。弹幕能够在视频播放的同时，于视频上方滑动显示其他用户的评论等信息。动态水印可以实现用户ID在视频上滚动显示，达到防盗录、盗播的效果，提升视频安全性。会员试看能力可以设定非会员可试看时长，超过时长后会弹出会员提示、购买窗等信息。各种业务侧经典的应用场景，都可以通过播放器UI组件实现。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JL2MSu41ALrn9j7DYp71KVniac4aBauNSdFQAETx17HHOIRr9bj2iabDoA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



播放器UI组件的接入也非常简单，仅需极少量的代码即可完成。首先集成播放器UI组件，之后在UI界面加入SuperPlayerView组件。封装SuperPlayerMdel，将播放URL、封面地址等填入。再调用SuperPlayerView的PlayWithModel就可以启动播放。对于进阶场景，比如弹幕、会员试看、动态水印等，开发者也只需在SuperPlayerModel中配置相关信息，例如可观看时长、动态水印文本、大小等等，就可以轻松地实现。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLYibg2KRyPmd7mFicSKPGzhgs1F5aD5sqO1Df3muEkUCP3PY3qQfSpXSA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 3.**短视频场景应用优化**



主流的短视频场景主要分两种，一种是沉浸式播放场景，类似微视、抖音，一次只能看到一个视频，通过上下滑动切换其他的视频。另一种是Feed流场景，一屏页面可同时出现多个视频，第一个完整出现的视频将自动播放。短视频应用场景的界面丰富，所以内存性能是一个很重要的指标，同时为了更加顺滑的观看体验，启播速度也非常重要。

短视频是一个快速消耗内容的场景，很多视频用户可能看都不看就会直接划过。为了降低业务的流量成本消耗，短视频场景需要一些流控策略。常规的流控实现思路是利用列表组件，在播放第一个视频时，对下一个视频进行预播放，以达到滑动至下一个视频时能够马上播放。但如果一个界面可以看到很多视频，这种策略就可能会对多个播放器进行预播放，这会导致内存消耗巨大，且反复销毁创建播放器也会带来比较多的内存碎片。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLGcOy0OppLnR1sQ6myLsuOibqEbOpFDCex7aZSetdjDzOg4LryEKUETg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



那应该怎样优化呢？腾讯云采用的优化思路是使用不超过两个播放器实例，并通过服务去管理播放器的复用与使用。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLgS5vkUQerxKUyzdme74JYiaTNTfNWDKg3y77DQQIxD3G9R4siagMicQog/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



如上图所示，在上层的应用逻辑层，也就是UI和业务层，业务侧可以根据自己的业务特性进行设计，这里可以采用MVC或者MVVM的设计模式。那如何使用两个播放器实例进行复用呢？在应用逻辑层下创建一个服务层，并创建一个类似线程池的管理PlayerPoolManager。服务层还对播放器做了一层封装，命名为TxPlayerWrapper。这层封装，可以便于业务侧统一控制类似playerconfig以及一些业务特性的行为。PlayerPoolManager提供两个接口，其中一个接口是UpdatePoolPlayers，主要运营播放器对复用的管理。具体流程可参考右侧的表格，假设用户停留在第1个视频，之后是序列号2和3的视频。此时PlayerPoolManager可以将1和2的URL放进PoolPlayer去创建播放器，并且设置autoplay为false进行预播放。因为当前要播放的是第1个视频，所以业务侧可以通过PlayerPoolManager提供的另一个接口getPlayer，以URL作为Key去取得要播放的player，然后调用resume进行播放。当用户向下滑动来到视频2时，PoolPlayer中视频1和视频2的URL的播放器，需要更新为视频2和视频3的URL的播放器。因为视频2的URL的播放器仍在PoolPlayer中，所以仅需将原视频1的player复用给视频3去进行预播放，而视频2的player则进行resumeplay。如果用户跳过视频3，快速滑至视频4，此时PoolPlayer中视频2和视频3就需要变成视频4和视频5。这就会产生一个问题，PoolPlayer中的视频2和视频3被清空后，视频4和视频5重新加入，但它们都没有经过预播放，也就导致启播速度受到很大影响。所以针对启播速度还要进一步做优化。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLHjqa4l7Bb4jicTqLv8zI0jpC4uv5zmCKL85Y5lsyXia5QlVico8ia8CY8w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



上图是一个简化的启播流程，从获取数据开始，之后UI展示，封面展示，获取URL/fileID以及配置视频参数，传递到播放器调起播放。通过向服务器请求读取视频文件，把读下的文件进行解封装、解码，到达一定的buffer后，就会启播并回调一个首帧事件。最后业务侧收到首帧事件回调后，进行封面隐藏，整个流程结束。对启播流程的优化，重点在那些引起耗时操作的地方，而耗时操作首先要关注IO的操作。流程中第一个引起耗时操作的地方，便是获取数据。这是一个网络IO相关的操作。这里需要提前进行数据获取，或做好缓存相关管理来减少耗时。流程中第二个引起耗时操作的地方在获取视频链接。在这个步骤中如果使用fileID播放的话，由于fileID仅是一个ID样式并非URL，所以会额外引入换链的过程。换链是一个比较耗时的网络请求过程，所以需要提前换好链来减少耗时。如果配置了防盗链，那么URL还会有一个有效期，同样需要做好管理。如果是多码率视频还需要事先配置好要指定播放的码流，不要等到播放器启动时再做切流，这会引起比较大的耗时。

除了IO操作的耗时外，另外两个优化启播时间的点分别是网络和解封装、解码。对于网络，重点在于视频的CDN部署情况。如果是刚上传的视频，那新增节点是否预热，是否可以保证良好的访问情况都很重要。对端侧来说，可以通过预下载或预播放的方式，把要播放的视频提前下载一部分。对于解封装、解码的耗时，也可以通过预播放机制去解决。

预下载和预播放这两者有什么区别呢？预播放机制就是创建一个播放器实例，并且启动下载和解码环节，首帧解析出来之后再暂停播放器。使用方法就是在startPlay之前设置setAutoPlay为false，要播放的时候调用resume接口来启动。而播放器的预下载机制不需要创建播放器实例，只预先下载视频的部分内容，不启动解码和解封装环节。使用方法就是调用TxVodPreloadManager接口进行预下载，启动播放时跟正常启动播放器一致。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLLgglZ6AS1WmjNoI7M0wqrjwSuibtib0iaT3wCFQoFO5ibKibBImjuw7oLpw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



两种方式都有各自的优缺点。预播放需要启动播放器实例以及解码器的相关操作，会带来一定的性能消耗，所以会占用内存和CPU消耗。但是预下载不需要启动播放器，不需要解码环节，性能消耗相比预播放会低很多。但预下载的启播速度比预播放慢，会有100毫秒左右的差距。因此在实际场景中需要根据业务特性，采用其中一种或者两种都采用的结合方式。

针对M3U8多码流视频，腾讯云也做了针对性的优化。因为在播放器未播放之前，无法知道多码率M3U8中有几个码流，所以开发者可以在启播前指定优先播放的视频分辨率。播放器会查找小于或等于该偏好分辨率的流进行启播，启播后就不必再通过set bitrate index进行切换。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLUWmkysia0ap8vsOwJ9XdV0A981nu4tdD7ic3Cwq0Mm3AfN6XsP0hS2Pw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

最后，通过流控策略还可以进一步精细化降低流量成本。腾讯云视立方播放器提供了三个阶段的流控功能。第一个是预播放流控，设置启播前阶段的最大缓存大小。第二个是预下载流控，预下载阶段流控与预播放类似，也可以控制预下载的大小。这些都可以通过相关接口进行参数设置。第三个是播放阶段的流控，即最大播放缓存的大小，默认是30秒，也可以进行自定义。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/APDZeM2BxAEMo6iaZV1XaBiaCaD2qv39JLdNxxSDaIKDaZBSQ55ovw4cvpA4JXprBM41ibzypJBR0HF3J23hWpMiaQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

原文作者：腾讯云音视频技术导师——李正通

原文链接：https://mp.weixin.qq.com/s/7JCpeuz6CX1hieBSSUuJ0Q