# 【NO.466】【网络通信 -- WebRTC】WebRTC 基础知识 -- 基础知识总结【1】WebRTC 简介

## 1.WebRTC 简介

WebRTC (Web Real­Time Communication，网页实时通信) 是 Google 于 2010 以 6829 万美元从 Global IP Solutions 公司购买，并于 2011 年将其开源，旨在建立一个互联网浏览器间的实时通信的平台，让 WebRTC 技术成为 H5 标准之一；

WebRTC 是一个基于浏览器的实时多媒体通信技术，该项技术旨在使 Web 浏览器具备实时通信能力；同时，通过将这些能力封装并以 JavaScript API 的方式开放给 Web 应用开发人员，使得 Web 应用开发人员能够通过 HTML 标签和 JavaScript API 快速地开发出基于 Web 浏览器的实时音视频应用，而无需依赖任何第三方插件；

WebRTC 由 IETF (Internet Engineering Task Force，互联网工程任务组) 和 W3C (World Wide Web Consortium，万维网联盟) 联合负责其标准化工作；

- IETF 定制 WebRTC 的互联网基础协议标准，该标准也被称为 RTC Web (Real-Time Communication in Web-browsers)
- W3C 则负责定制 WebRTC 的客户端 JavaScript API 接口的标准

文章福利：[分享一个每晚8-10点都有的免费技术直播的链接，订阅即可学习~宝藏地址！！](https://link.zhihu.com/?target=https%3A//ke.qq.com/course/3202131%3FflowToken%3D1040743)

下面收藏的一些视频可以加我群领取

![img](https://pic4.zhimg.com/80/v2-f1fddd71350ab3261823c3defb1d3263_720w.webp)

## 2.WebRTC 框架简介

**WebRTC 整体框架图示**

![img](https://pic4.zhimg.com/80/v2-2a02ae4a6a0df5193bcde0f64f3b723b_720w.webp)

### 2.1 Your Web App

Web 开发者开发的程序，Web 开发者可以基于集成 WebRTC 的浏览器提供的 Web API 开发基于视频、音频的实时通信应用

### 2.2 Web API

面向第三方开发者的 WebRTC 标准 API(Javascript)，使开发者能够容易地开发出类似于网络视频聊天的 Web 应用，这些 API 提供三个功能接口，分别是 MediaStream、RTCPeerConnection 和 RTCDataChannel；

MediaStream 接口用于捕获和存储客户端的实时音视频流，便于客户端进行音视频采集和渲染

RTCPeerConnection 接口是 WebRTC 的核心接口，封装了 WebRTC 连接的管理，负责 WebRTC 连接机制的接口

RTCDataChannel 接口是进行 WebRTC 连接数据传输的数据通道接口

### 2.3 WebRTC Native C++ API

本地 C++ API 提供给浏览器厂商、平台 SDK 开发者使用的 C++ API，不同的平台可以通过各自的 C++ 接口调用能力，对其进行上层封装以满足跨平台的需求

Session management/Abstract signaling

会话管理 / 抽象信令，WebRTC 的会话层，主要用于进行信令交互和管理 RTCPeerConnection 的连接状态

### 2.4 VoiceEngine

音频处理引擎，包含一系列音频多媒体处理的框架，包括 Audio Codecs、NetEQ for voice、Acoustic Echo Canceller (AEC) 和 Noise Reduction (NR)

Audio Codecs 是音频编解码器，当前 WebRTC 支持 ilbc、isac、G711、G722 和 opus 等

NetEQ for voice 是自适应抖动控制算法以及语音包丢失隐藏算法，用于适应不断变化的网络环境，从而保持尽可能低的延迟，同时保持最高的语音质量

AEC 是回声消除器，用于实时消除麦克风采集到的回声

NR 是噪声抑制器，用于消除与相关 VoIP 的某些类型的背景噪音

### 2.5 VideoEngine

视频处理引擎，包含一系列视频处理的整体框架，从摄像头采集视频到视频信息网络传输再到视频显示整个完整过程的解决方案，包括 Video Codec、Video Jitter Buffer 和 Image Enhancement

Video Codec 是视频编解码器，当前 WebRTC 支持 VP8、VP9 和 H.264 编解码；

Video Jitter Buffer 是视频抖动缓冲器，用于降低由于视频抖动和视频信息包丢失带来的不良影响；

Image Enhancement 是图像质量增强模块，用于对摄像头采集回来的图像进行处理，包括明暗度检测、颜色增强、降噪处理等；

### 2.6 Transport

数据传输模块，WebRTC 对音视频进行 P2P 传输的核心模块，包括 SRTP、Multiplexing 和 P2P；

SRTP 是基于 UDP 的安全实时传输协议，为 WebRTC 中音视频数据提供安全单播和多播功能

Multiplexing 是多路复用技术，采用多路复用技术能把多个信号组合在一条物理信道上进行传输，减少对传输线路的数量消耗

P2P 是端对端传输技术，WebRTC 的 P2P 技术集成了 STUN、TURN 和 ICE，这些都是针对 UDP 的 NAT 的防火墙穿越方法，是连接有效性的保障

## 3.WebRTC 一对一通话

**一对一通话结构图示**

![img](https://pic1.zhimg.com/80/v2-42f00c6d92e3de214b7ceb310bc08610_720w.webp)

- 两个 WebRTC 终端，负责音视频采集、编解码、NAT 穿越、音视频数据传输；
- 一个 Signal（信令）服务器，负责信令处理，如加入房间、离开房间、媒体协商消息的传递等；
- 一个 STUN/TURN 服务器，负责获取 WebRTC 终端在公网的 IP 地址，以及 NAT 穿越失败后的数据中转；

### 3.1 **一对一通话流程图示**

![img](https://pic2.zhimg.com/80/v2-bc16270cf4fe167a950e3efa15bf8311_720w.webp)

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/445233381