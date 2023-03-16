# 【NO.462】【音视频技术】播放器架构设计

## 1.概述

首先，我们了解一下播放器的定义是什么 ？

> “播放器，是指能播放以数字信号形式存储的视频或音频文件的软件，也指具有播放视频或音频文件功能的电子器件产品。” —— 《百度百科》

我的解读如下：“播放器，是指能读取、解析、渲染存储在本地或者服务器上的音视频文件的软件，或者电子产品。”

归纳起来，它主要有如下 3 个方面的功能特性：

读取（IO）：“获取” 内容 -> 从 “本地” or “服务器” 上获取

解析（Parser）：“理解” 内容 -> 参考 “格式&协议” 来 “理解” 内容

渲染（Render）：“展示” 内容 -> 通过扬声器/屏幕来 “展示” 内容

把这 3 个方面的功能串起来，就构成了整个播放器的数据流，如图所示：

![img](https://pic4.zhimg.com/80/v2-a8538ca68d150e575bbf825458c7412b_720w.webp)

IO：负责数据的读取。从数据源读取数据有多种标准协议，比如常见的有：File，HTTP(s)，RTMP，RTSP 等

Parser & Demuxer：负责数据的解析。音视频数据的封装格式，都有着各种业界标准，只需要参考这些行业标准文档，即可解析各种封装格式，比如常见的格式：mp4，flv，m3u8，avi 等

Decoder：其实也属于数据解析的一种，只不过更多的是负责对压缩的音视频数据进行解码，拿到原始的 YUV 和 PCM 数据，常见的视频压缩格式如：H.264、MPEG4、VP8/VP9，音频压缩格式如 G.711、AAC、Speex 等

Render：负责视频数据的绘制和渲染，是一个平台相关的特性，不同的平台有不同的渲染 API 和方法，比如：Windows 的 DDraw/DirectSound，Android 的 SurfaceView/AudioTrack，跨平台的如：OpenGL 和 ALSA 等

下面我们逐一剖析一下播放器整个数据流的每一个模块的输入和输出，并一起设计一下每一个模块的接口 API。



## 2.模块设计

### 2.1 IO 模块

IO 模块的输入：数据源的地址（URL），这个 URL 可以是一个本地的文件路径，也可以是一个网络的流地址。

IO 模块的输出：二进制的数据，即通过 IO 协议读取的音视频二进制数据。

视频数据源的 URL 示例如下：

file:///c:/WINDOWS/clock.avi rtmp://[http://live.hkstv.hk.lxdns.com/live/hks](https://link.zhihu.com/?target=http%3A//live.hkstv.hk.lxdns.com/live/hks) [http://www.w3school.com.cn/i/movie.mp4](https://link.zhihu.com/?target=http%3A//www.w3school.com.cn/i/movie.mp4) [http://devimages.apple.com/iphone/samples/bipbop/bipbopall.m3u8](https://link.zhihu.com/?target=http%3A//devimages.apple.com/iphone/samples/bipbop/bipbopall.m3u8)

综上，播放器 IO 模块的接口设计如下所示：

![img](https://pic1.zhimg.com/80/v2-865d7ae6fa7017dad9445c2b6913fb20_720w.webp)

Open/Close 方法主要是用于打开/关闭视频流，播放器内核可以通过 URL 的头（Schemes）知道需要采用哪一种 IO 协议来拉流（如：FILE/RTMP/HTTP），然后通过继承本接口的子类去完成实际的协议解析和数据读取。

IO 模块读取数据，则定义了 2 个方法，Read 方法用于顺序读取数据，ReadAt 用于从指定的 Offset 偏移的位置读取数据，后者主要用于文件或者视频点播，为播放器提供 Seek 能力。

对于网络流，可能出现断线的情况，因此独立出一个 Reconnect 接口，用于提供重连的能力。

### 2.2 解析模块

![img](https://pic4.zhimg.com/80/v2-220a675c0f8bdafa56e4647c09fe0adf_720w.webp)

从 IO 模块读到的音视频二进制数据，其实都是用如 mp4、flv、avi 等格式封装起来的，如果想分离出音频包和视频包，则需要通过一个 Parser & Demuxer 模块进行解析。

解析模块的输入：由 IO 模块读取出来的 bytes 二进制数据

解析模块的输出：音视频的媒体信息，未解码的音频数据包，未解码的视频数据包

音视频的媒体信息主要包括如下内容：

视频时长、码率、帧率等 

音频的格式：编码算法，采样率，通道数等 

视频的格式：编码算法，宽高，长宽比等

综上，解析模块的接口设计如下图所示：

![img](https://pic2.zhimg.com/80/v2-2f7542a333d90f18c2113971b22dcac5_720w.webp)

创建好解析对象后，通过 Parse 函数输入音视频数据解析出基本的音视频媒体信息，通过 Read 函数读取分离的音视频数据包，然后分别送入音频和视频×××，通过 Get 方法获取各种音视频参数信息。

### 2.3 解码模块

![img](https://pic4.zhimg.com/80/v2-b02300d4dbe4a3fa179ad3bb32cd9197_720w.webp)

解析模块分离好音频和视频包以后，就可以分配送入到音频×××和视频×××了

解码模块的输入：未解压的音频/视频包

解码模块的输出：解压好的音频/图像的原始数据，即 PCM 和 YUV

由于音视频的解码，往往不是每送入×××一帧数据就一定能输出一帧数据，而是经常需要缓存几帧参考帧才能拿到输出，所以编码器的接口设计常常采用一种 “生产者-消费者” 模型，通过一个公共的 buffer 队列来串联 “生产者-消费者”，如下图所述（截取自 Android MediaCodec 编解码库的设计）：

![img](https://pic2.zhimg.com/80/v2-8ef9428b526b22a8e56a2e63be58a291_720w.webp)

综上，解码模块的接口设计如下所示：

![img](https://pic3.zhimg.com/80/v2-3e5e75853576950582ffd569af3546c2_720w.webp)

解析模块输出的媒体信息，包含有该使用什么类型的音频/视频×××，可利用该信息完成×××的初始化。剩下的过程，就是通过 Queue 和 Dequeue 不断跟×××交互，送入未解码的数据，拿到解码后的数据了。

### 2.4 渲染模块

![img](https://pic3.zhimg.com/80/v2-3e025b35d11de92fddd9f115fe5f9b36_720w.webp)

×××输出原始的图像和音频数据后，下一步就是送入到渲染模块进行图像的渲染和音频的播放了。

一般视频数据渲染是输出到显卡展示在窗口上，音频数据则是送入声卡利用扬声器播放出来。虽然不同平台的窗口绘制和扬声器播放的系统层 API 都不太一样，但是接口层面的流程也都差不多，如图所示：

![img](https://pic2.zhimg.com/80/v2-5af3cf28412e6c8394b11b21915a57a9_720w.webp)

对于视频渲染而言，流程则是：Init 初始化 -> SetView 设置窗口对象 -> SetParam 设置渲染参数 -> Render 执行渲染/绘制

![img](https://pic4.zhimg.com/80/v2-e77544143a2fea3a0de929c093c815bf_720w.webp)

对于音频播放而言，流程则是：Init 初始化 -> SetParam 设置播放参数 -> Render 执行播放操作

### 2.5 把模块串起来

![img](https://pic3.zhimg.com/80/v2-738da61d14e2ec35976a7aa5648933de_720w.webp)

如图所示，把各个模块这样串起来后，就是播放器的整个数据流走向了，但这是一个单线程的结构，从 IO 读到数据后，立马送入解析 -> 解码 -> 渲染，这样的单线程结构的播放器设计，会存在如下几个问题：

1. 音视频分离后 -> 解码 -> 播放，中间无法插入逻辑进行音画同步
2. 无数据缓冲区，一旦网络/解码抖动 -> 导致频繁的卡顿
3. 单线程运行，没有充分利用 CPU 多核

要想解决单线程结构的问题，可以以数据的 “生产者 - 消费者” 为边界，添加数据缓冲区，将单线程模型，改造为多线程模型（IO 线程、解码线程、渲染线程），如图所示：

![img](https://pic1.zhimg.com/80/v2-291f1897527db0182911ff7e7ccfbcb4_720w.webp)

改造为多线程模型后，其优势如下：

帧队列（Packet Queue）：可抵抗网络抖动

显示队列（Frame Queue）：可抵抗解码/渲染的抖动

渲染线程：添加 AV Sync 逻辑，可支持音画同步的处理

并行工作，高效，充分利用多核 CPU

注：我们将在下一篇文章专门来聊一聊这 2 个新增的缓冲区该如何设计和管理。

## 3.播放器 SDK 接口设计

前面详细介绍了播放器内涵的关键架构设计和数据流，如果期望以该播放器内核作为 SDK 给 APP 提供底层能力的话，还需要设计一套易用的 API 接口，这套 API 接口，其实可抽象为如下 5 大部分：

创建/销毁播放器

配置参数（如：窗口句柄、视频 URL、循环播放等）

发送命令（如：初始化，开始播放，暂停播放，拖动，停止等）

音视频数据回调（如：解码后的音视频数据回调）

消息/状态消息回调（如：缓冲开始/结束、播放完成等）

综上，播放器常见接口列表如下：

Create/Release/Reset

SetDataSource/SetOptions/SetView/SetVolume

Prepare/Start/Pause/Stop/SeekTo

SetXXXListener/OnXXXCallback

## 4.播放器的状态模型

总体来说，播放器其实是一个状态机，被创建出来了以后，会根据应用层发送给它的命令以及自身产生的事件在各个状态之间切换，可以用如下这张图来展示：

![img](https://pic4.zhimg.com/80/v2-be98c67242f06757613dd883b5513cc3_720w.webp)

播放器一共有 9 种状态，其中，Idle 是创建后/重置后的到达的初始状态，End 和 Error 分别是主动销毁播放器和发生错误后进入的最终状态（通过 reset 重置后可恢复 Idle 状态）

其他的状态切换和达到方式，图中已经标注得比较清楚了，这里就不再赘述了。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/454178003