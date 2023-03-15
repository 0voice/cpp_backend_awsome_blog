# 【NO.385】利用WebTransport进行现场视频流注入

编者按：通过网络支持的实时音视频通话已成为人们日常生活和办公中必不可少的一部分，对于音视频领域的网络技术要求也越来越高。对此，LiveVideoStack 特别邀请到了来自美国 Paramount Global 的张博老师，他以《利用 WebTransport 进行现场视频流注入》为题来进行相关内容分享。

文 / 张博

整理 / LiveVideoStack

 

![图片](https://oscimg.oschina.net/oscnet/up-a2aa309b88a16464f6b6893f133acc9bfa9.jpg)

大家好，我叫张博，目前在美国波士顿，供职于美国 Paramount Global 公司。Paramount 是美国五大电影制片公司之一，国内叫派拉蒙影视。Pluto TV 是它旗下的一个 streaming service 流媒体。我是负责视频编码和播放系统设计的架构师。在此之前，我还供职于其它的视频技术公司，包括 Fubo TV，Brightcove，Ericsson。在供职于 Brightcove 公司期间，我曾担任过多个国际视频制定标准委员会的委员，包括 MPEG，INCITS L3.1，DASH Industry 工业论坛和 CTA-WAVE，并且曾经参与过 MPEG-DASH 和 MPEG-CMAF 标准的制定工作，我还曾经参与 Brightcove 公司著名的 Zencoder 编码系统和开源视频播放器 Video.js 的开发工作。

今天我要讲的话题是利用 WebTransport 进行现场视频流注入，英文叫 Live Video Ingest via WebTransport。Pluto TV 是 Paramount 旗下的一个流媒体服务。Paramount 公司有自己的院线、电影院和 streaming service，因此我们线上线下都有放送的平台。Pluto TV 不需要交会员费，我们是完全通过广告的营收来支持营运。Pluto TV 大概有几百个频道，其中包括 Paramount 下属的其它传统电视频道（比如 CBS 新闻网络，Nickelodean，Showtime），另外也包括一些由众多单个 VoD 内容串联起来的虚拟直播频道。我们基本上都是靠广告营收，在广告上有很多创新。不过今天我要讲的话题跟我的工作其实没有关系。我们也有一部分的现场直播的频道应用，但是现在还没有运用到 WebTransport，因为 WebTransport 是一个比较新的技术，2019 年才正式制定出版协议上线，现在还是在一个定稿阶段。

![图片](https://oscimg.oschina.net/oscnet/up-2f4882e80cc94af6268b92f58b0dd054e79.jpg)

我今天演讲分为三个部分：首先是对 WebTransport 的简单介绍；接下来会分享我提出的一个新的方法：利用 WebTransport 进行 Live Video Ingest 现场视频流的注入；最后我会做一个概念证明，这个 idea 提出来以后需要去证明它真的可以被做出来。

**01 WebTransport 简介**

![图片](https://oscimg.oschina.net/oscnet/up-a669a7be44b2474db83ef2a31da991257b4.jpg)

首先是来简单介绍一下 WebTransport。WebTransport 是一种基于 HTTP/3 的新型网络传输协议，它支持以下功能：包括双向通讯（就通讯双方可以给对方发送 message 和 datagram）、安全传输（所有数据传输都是经过 TLS 加密的），它有两种数据传输模式：一种是基于 stream 的，类似于 TCP 的可靠传输；还有一种是基于 Datagram 的快速低延迟传输，有点类似于 UDP。它还有一个功能是 NAT and firewall traversal，它可以穿透 NAT 和防火墙，支持跨互联网的传输。通常人们把 WebTransport 跟另外两个协议进行对比，一个是 Websocket，一个是 WebRTC。那么 Websocket 是基于 HTTP/2 (第二代 HTTP 协议），WebTransport 是基于 HTTP/3（第三代的 HTTP 协议）。一般认为未来 WebTransport 会取代 Websocket 用在很多游戏和交互比较多的应用上。WebTransport 有很大的发展前景，因为 WebTransport 基于 HTTP/3，所以它比基于 HTTP/2 的 Websocket 拥有更快的传输速度和更低的延迟。另外一个经常对比的协议就是 WebRTC，WebRTC 必须要依靠 ICE（Interactive Connectivity Establishments）协议来让通讯双方知道对方的 IP 地址和网络端口，如果通讯双方没有直接的网络连接的话，它还需要通过中间的通信的基础设施 communication infrastructure 来建立连接。那么在这一点上 Websocket 和 WebRTC 就不如 WebTransport，因为它是直接运行在 443 网络端口上的，所以它天然具备穿透 NAT 和防火墙的能力，现有的 Web Infrastructure 就可以无障碍的支持 WebTransport，所以它相较于 WebRTC 更简单一些，也更易于部署，不需要额外的基础设施投资。

![图片](https://oscimg.oschina.net/oscnet/up-0f11bf5e544886f2cf359bcb0e36ce15a6b.jpg)

WebTransport 是运行在 HTTP/3 和 443 网络端口之上的一种 Client server 协议，和 Websocket 一样，每一个连接双方都会有一个 HTTP/3 的 connection。比如一个传统 Client、一个 browser 浏览器和一个 HTTP 服务器之间，现有的服务器和客户端原本就支持 HTTP，但是我们现在只需要让它额外支持 WebTransport，通讯双方就可以具备 Websocket 和 WebTransport 的能力，它会先建立一个 HTTP/3 的 connection 然后在 HTTP 的 connection 里它会允许建立多个 WebTransport 的 session，每一个 session 都是独立的传输单元，都有自己独立的 session ID。包里面会有一个 session ID 在 header 里，这样的话它就可以区分不同的 session。

连接的建立是由连接的发起方通过 extended CONNECT method 来发起连接的请求，跟 Websocket 是一样的。双方都需要支持 WebTransport 连接才可以建立。在创建 HTTP 连接的时候就需要在 setting frame 里将 SETTINGS_ENABLE_ WEBTRANSPORT 的参数设置为 1。当它看见对方的 setting frame 里参数被设为 1 以后，它就知道对方是支持 WebTransport 的，然后它才会允许连接的建立。流量控制和拥塞控制是由底层的 QUIC 协议来负责，就是 flow control 和 congesting control。

![图片](https://oscimg.oschina.net/oscnet/up-0c18107b28b455a8a8cd58d4a858870d5b1.jpg)

WebTransport 支持单向，也支持双向数据传输，基于 TLS 的安全数据传输可以无障碍穿越 NAT 和防火墙。它有两种传输模式，一个是 stream，一个是 datagram。stream 支持可靠的、有序的数据传输，而 Datagram 就只管发给对方，它不会重发，也不会流量控制的数据传输，所以它的速度会快一些。stream 是比较可靠、有序的传输。

**02 利用 WebTransport 进行现场视频流注入**

第二部分是 WebTransport 在视频方面有哪些应用。首先我想到的就是把 WebTransport 用来进行现场视频流注入 Live Ingest。

![图片](https://oscimg.oschina.net/oscnet/up-0995d4b8b54e4895b67d2c1145539485a24.jpg)

图片是现有传统的现场视频流注入方法，有上、中、下三种现有的模式，第一种是最上面的，用 RTMP 作为视频注入的媒体，从左到右是视频源 video source，把原始视频以 RTMP 流的方式发给 ingester 注入端，注入端拿到后，把数据给转码器和封装器，封装器拿到以后把视频流封装成 DASH 和 HLS 的 stream，然后发给 origin server，再发给 CDN，一直到播放器 player。我们知道 RTNP 是基于 TCP 的，所以它的延迟会比较大，因为它中间需要做一些 buffering，连接的建立也会更费时，一般会有 2 到 3 秒的延迟。

WebRTC 我不知道国内用的多不多，是只用作 live ingest，还是直接对终端用户进行视频直播。但是，美国的一些公司把 WebRTC 作为视频注入协议，它跟 RTMP 类似，把原始的视频流与 WebRTC 流的格式发给 ingester。但是 WebRTC 比较依赖 ICE 和底层的 infrastructure，所以它的协议更复杂一些，需要额外的基础设施部署。

最下面的第三种方法在传统的广电网络里面应用比较多，美国传统的 cable broadcast 公司是直接使用 mpeg-ts 和 multicast，然后直接把数据从视频源发给注入端，这个方法的速度很快而且也很稳定，但这个是基于 broadcaster 也就是广电的电视网，它的网络是独享的，不是共享的公共互联网，它没有任何的网络的 jitter，所以它很快。但是 OTT 的 streaming service 是不可以享受到红利的，所以在互联网中它无法使用，因为互联网一般是不允许 multicast 工作的。

![图片](https://oscimg.oschina.net/oscnet/up-21396e9675b51bde24ad88184b1da72db98.jpg)

基于刚才说的三种模式，我发现它们都有一些各自的问题，那么我想到 WebTransport 来进行现场直播流注入的基本思路就如上图所示这样，首先是视频源 Client serve，注入端就是 server，视频源相对于注入端它是 Client，输入端是 server。Client 和注入端 server 建立一个 WebTransport 的连接，就像中间这样一个管道，然后 Client 通过 WebTransport 管道把 mpeg-ts 的流或其它格式的视频流通过管道传输给 server。WebTransport 本身并不对原始的视频流做任何的修改和优化，它仅仅只是一个管道而已，因为管道具备安全性、较低的延迟，还可以跨越互联网。所以我们就把视频源直接通过管道发给注入端，可以让它更安全、更低延迟地、更及时地传送到另一端。

![图片](https://oscimg.oschina.net/oscnet/up-610faafb6d7c54628055d2c82accb3859cf.jpg)

WebTransport 相对于几个现有的传输方法的优势是易于部署。因为它是基于 HTTP 之上的，现有的 Web Infrastructure 就可以支持，视频的注入无需额外基础设施的投资，然后它是基于 443 端口，所以可以无障碍穿越 NAT、防火墙。低延迟是它一个最大的优势，因为 WebTransport 有 datagram 的传输方式，而现有的 Websocket 是没有的，WebRTC 是有的，但它都是 UDP、RTP 的，需要依赖 ICE，所以它可以支持高效、低延迟的传输，相对于 RTMP 和 HTTP 这两个基于 TCP 的协议要具备优势，因为低延迟对于视频直播尤为重要。

**03 概念证明**

概念是很简单的，也没有很多复杂的概念，但是我需要对我的方法做一个概念证明 Proof of Concept。

![图片](https://oscimg.oschina.net/oscnet/up-3b4947ad65a86c9023af47d90dead475fe0.jpg)

现有对 WebTransport 的支持其实并不多，因为第一版协议 2019 年才提出，那么现在客户端这边支持它的浏览器只有 Chromium，Chromium 是 Chrome 的实验版本产品，通用版的 Chrome 都不支持 WebTransport。Google 有自己的一个简单的 Client 和 server 的 WebTransport 的实现。Client 是用 Javascript 写的，它调用 Chromium 提供的 WebTransport API 来进行连接的建立和传输，然后 server 是用 Python 写的，它调用 AIOQUIC 库，是一个 Python 的 QUIC 的 library，然后我利用 Google 提供的 Client 和 server 的实现做了一个自己的 PoC，程序在 Github 上面可以找到 Live Video Ingest via WebTransport。

![图片](https://oscimg.oschina.net/oscnet/up-87090c1a05ae7360bd5d67f3a81443f7bf2.jpg)

![图片](https://oscimg.oschina.net/oscnet/up-7bd76441169b7078f4e8d5167499b258865.png)

我的实现 PoC 的思路是这样的，首先让 Client 程序和 server 程序建立一个 WebTransport 的连接然后我会让 Client（一个 Javascript 的程序，它是基于 Google 的 WebTransport Client）每隔数秒钟抓取一次摄像头的视频，然后我把摄像头的视频，封装成 WebM 格式，然后 Client 会将 WebM 文件通过 WebTransport 管道发送到 server 那边，server 拿到 WebM 文件后把它用 FFmpeg 转格式成为 MP4 文件，然后存到一个 webserver 目录下，比如说 EngineX 的目录下，然后提供给 video player 进行下载播放。

那么为什么传输格式可以是 mpeg-ts，但我要用 WebM 呢？因为有一些技术上的限制。WebTransport 的客户端仅仅只被浏览器支持，那么 Client 只能是一个 Javascript 程序，我们无法将 FFmpeg 生成的 mpeg-ts 的视频流发给运行在浏览器中的 Client，我没有找到合适的方法来做这件事情，所以我只能用 WebM 格式进行，流传输在我的 PoC 里面是这样的，但是我相信将来 WebTransport 会有更多的本地的 native 的支持，将来我们可以直接把 Web mpeg-ts 流直接通过 WebRTC，而不需要通过浏览器发送到 server 那端。

![图片](https://oscimg.oschina.net/oscnet/up-e44ec6b3fab6a53c6e4384030c8bb96e81b.jpg)

具体的实现是我们让 Client 调用 Chromium 提供的 API 来建立一个与 server 之间的 WebTransport 的 datagram 连接，我之所以采用 datagram 而不采用 stream 是因为 datagram 可以快速地传输数据，然后连接建立以后，Client 会调用 getUserMedia API 来抓取摄像头的视频，并创建一个 video 窗口进行本地的视频播放，然后 Client 会每隔 4 秒钟调用 MediaRecorder API 抓取的视频录制成 WebM 文件，然后将 WebM 文件以 datagram 的形式分段通过 WebTransport 发给 server，每一个 datagram 的长度是 1,200 个字节，这由底层协议的最大报文长度决定的，server 在收到 WebM 文件后用 FFmpeg 把 WebM 文件转格式成为 MP4 格式，然后把它存到 server 的下载链接目录下，并且 server 把新文件的 HTTP 下载链接回复给 Client，Client 拿到链接以后就下载 MP4 文件，然后在另外一个 video 窗口进行播放，如果我把这两个 video 播放窗口并列摆放的话，我们就可以看到整个流程的延迟，即本地视频是直接播放的，下载的视频是经过 WebTransport 和 FFmpeg 的转码再经过 HTTP 的下载，这三个步骤才可以播放，那么这两个摆在一起的话就可以看到整个流程的延迟。

![图片](https://oscimg.oschina.net/oscnet/up-06b19ca84bf55489424a9491357d07c2248.jpg)

下面是一个简单演示 demo。我把 server 部署在 AWS EC2 的机器上，Client 运行在本地的 Chromium 浏览器上。那么我需要打开 443 端口并且允许 UDP traffic 通过。如果以前都是 443 的话，我们只需要打开 TCP。对于 QUIC 和 WebTransport 这种协议来说，我们还需要打开 UDP 的端口以便让 WebTransport datagram 能够到达 server 那边。

![图片](https://oscimg.oschina.net/oscnet/up-2e969e12fa3e766d4074dbcddd7ac2130d5.jpg)

本来的 demo 应该运行在 AWS 上面，但我只有一个免费的那个账户，今天的时间已经到了，所以今天我的 demo 只能运行在本地的机器上面，这是 demo 的截屏。

![图片](https://oscimg.oschina.net/oscnet/up-6f1578584a05d0365dcbd08a18ea6c6c3a2.png)

我现在就要运行一下 demo，窗口是 WebTransport 的 server 的 Python 程序。WebTransport server 要读取本地的 certificate 就是 TLS 的 certificate 和 key，我是让它跑在 443 端口上面，这本身没有区别，然后 chromium 浏览器它需要允许特定的 IP 和端口，然后把 Client 拿出来，本地窗口抓取的视频直接播放，然后我点 connect server 那边就拿到了连接，然后我这边开始抓取视频，ingest 注入到 server 那边 start，然后如果我们等几秒钟就能够看见两个视频可以同时收到。那么我们看一下手机上面的时间，延迟大概是有 4 秒钟左右，有的时候是 4 秒，有的时候是 3 秒，有的时候是 2 秒因为 API 时间并不稳定，每一个 segment 的时间并不稳定。

![图片](https://oscimg.oschina.net/oscnet/up-9b7ae6f6334b824113ad59f38c3e9331bd9.jpg)

因为时间有限，我只完成了一个 PoC 的实现，那么未来我还需要将我的新方法跟其它现有的方法进行比较。那么我很关心的是两个比较参数，一个是它的延迟，一个是丢包率。因为我知道，我用的是 datagram 进行数据传输，用视频流注入的延迟会比较低，但是低多少我还得具体去衡量。再一个就是丢包率，因为用 datagram 的话会有一些丢包，它不保证可靠的传输，那么对视频播放的质量也会有些影响，我要去看具体的影响有多大，然后我再进行一些改进和优化。

![图片](https://oscimg.oschina.net/oscnet/up-608c41f3da5e073aaccae99ad7e25c6df65.jpg)

以上就是本次的所有分享内容，上图是相关的 reference，谢谢大家！

原文作者：张博

原文链接：https://www.livevideostack.cn/news/%e5%88%a9%e7%94%a8webtransport%e8%bf%9b%e8%a1%8c%e7%8e%b0%e5%9c%ba%e8%a7%86%e9%a2%91%e6%b5%81%e6%b3%a8%e5%85%a5/