# 【NO.402】【网络通信 -- 直播】流媒体协议中的时间戳理解与音视频同步,RTP/RTCP/RTMP推流拉流/音视频同步

------

## 1.音视频同步的概念示例

音视频同步即视频同步到音频，视频在渲染的时候，每一帧视频与音频的时间戳对比，以判定是立即渲染还是延迟渲染；

比如有一个音频序列，其时间戳是 A(0，20，40，60，80，100，120 ...)，视频序列 V(0，40，80，120 ... )

音画同步的步骤如下 :

\1) 取一帧音频 A(0) 播放；取一帧视频 V(0)，视频帧时间戳与音频相等，视频立即渲染

\2) 取一帧音频 A(20) 播放；取一帧视频 V(40)，视频帧时间戳大于音频，视频延迟渲染

\3) 取一帧音频 A(40) 播放；取出视频，还是上面的 V(40)，视频帧时间戳与音频相等，视频立即渲染

说明

真实场景中音视频时间戳不一定完全相等，只要音视频时间戳差值的绝对值在一个帧间隔时间内，即可认为是相同的时间戳

------

## 2.RTP 协议中的时间戳与音视频同步

### 2.1 RTP 协议中时间戳相关的基本概念

时间戳单位，时间戳的单位是由采样频率所代替的单位，目的是为了使时间戳单位更为精准；

比如一个音频的采样频率为 8000 Hz，则时间戳单位为 1/8000

时间戳增量，相邻两个 RTP 包之间的时间差 (以时间戳单位为基准)

### 2.2 RTP 包的时间戳

净荷中第一个采样数据的采样时间 (在一次会话开始时的时间戳初值是随机选择的)，每一个采样数据的采样时间通过一个单调且线性递增的时钟获得，时钟频率取决于净荷数据的编码格式，相邻两个 RTP 分组的时间戳差值，取决于前一个 RTP 分组净荷中包含的采样数据数目；

\1. 时间戳就是一个值，用来反映某个数据块的产生(采集)时间点，后采集数据块的时间戳肯定大于先采集数据块的时间戳，从而可以标记数据块的先后顺序

\2. 在实时流传输中，数据采集后立刻传递到 RTP 模块进行发送，那么，数据块的采集时间戳就直接作为 RTP 包的时间戳

\3. 时间戳的单位为采样频率的倒数，例如采样频率为 8000 Hz 时，时间戳的单位为 1 / 8000

\4. 时间戳增量是指两个 RTP 包之间的时间间隔，即发送第二个 RTP 包相距发送第一个 RTP 包时的时间间隔(单位是时间戳单位)

总结

在采用 RTP 协议进行媒体控制传输时，音频和视频作为不同的会话传输，音视频在 RTP 层没有直接的关联，RTP 包头携带的时间戳信息的基点(起始时间)，因此，时间戳的信息只能在同一个音频或视频会话内，确定收到的 RTP 包的时间先后的相对顺序，但该时间戳信息无法确定不同媒体会话之间的时间关系

### 2.3 音视频流间同步实现

发送端在发送音视频数据的同时会发送 SR 包，从而使接收方能够正确使音视频同步播放，在 RTCP 的 SR 包中，包含有 <NTP 时间，RTP 时间戳> 对，音频帧 RTP 时间戳和视频帧 RTP 时间戳通过 <NTP 时间，RTP 时间戳> 对，可以准确定位到绝对时间轴 NTP 上，即可确定音频帧和视频帧的相对时间关系；

计算方式

变量描述

![img](https://pic4.zhimg.com/80/v2-ec1bec101e89bdd58ff2f0487a8b4617_720w.webp)

RTP 时间戳频率计算公式

```text
Audio_Fre = (AudioSRRTPtime2 - AudioSRRTPtime1)/(AudioSRNTPtime2 - AudioSRNTPtime1)
Video_Fre = (VideoSRRTPtime2 - VideoSRRTPtime1)/(VideoSRNTPtime2 - VideoSRNTPtime1)
```

以 AUDIO 为基准，在某个 Audio_SRNTP 时刻，对应视频 RTP 时间计算方式

```text
Video_RTPTime = Video_SRRTPTime + (Audio_SRNTP - Video_SRNTP) × Video_Fre
```

对于任何播放时刻，以 Audio 作为主轴，假设 AUDIO 的播放时间为 PLAY_AUDIO_TIME，视频的播放时间为 PLAY_VIDEO_TIME，计算方式

```text
(PLAY_VIDEO_TIME - Video_RTPTime) / Video_Fre = PLAY_AUDIO_TIME - AudioSRRTPtime) / Audio_Fre
```

------

## 3.RTMP 协议中的时间戳与音视频同步

### 3.1 RTMP Chunk 数据包中的时间戳

Chunk Type(fmt) = 0，timestamp 字段表示绝对时间，(最多能表示到 16777215 = 0xFFFFFF = 2^24 - 1)，当超过最大值时，使用 Extended Timestamp 字段表示绝对时间

Chunk Type(fmt) = 1，timestamp 字段表示与上一个 Chunk 数据的时间差值，(最多能表示到 16777215 = 0xFFFFFF = 2^24 - 1)，当超过最大值时，使用 Extended Timestamp 字段表示与上一个 Chunk 数据的时间差值

Chunk Type(fmt) = 2，timestamp 字段表示与上一个 Chunk 数据的时间差值，(最多能表示到 16777215 = 0xFFFFFF = 2^24 - 1)，当超过最大值时，使用 Extended Timestamp 字段表示与上一个 Chunk 数据的时间差值

Chunk Type(fmt) = 3，其消息头与上一个 Chunk 数据包的消息头完全相同

### 3.2 RTMP 时间戳与 PTS、DTS 的关系

RTMP 数据包时间值与 DTS 的关系

\1. 若当前 RTMP Chunk 数据包 Chunk Type(fmt) = 0，则可以直接解析 timestamp 字段获取当前 RTMP 数据包时间值，该值即为 DTS；

\2. 若当前 RTMP Chunk 数据包 Chunk Type(fmt) = 1 或 2，则根据解析出的 timestamp 字段值加上上一个 RTMP 数据包的时间值，可以获取当前 RTMP 数据包的时间值，该值即为 DTS；

\3. 若 RTMP Chunk 数据包 Chunk Type(fmt) = 3

上一个 RTMP Chunk 数据包 Chunk Type(fmt) = 0，则当前 RTMP 数据包的时间值为上一个 RTMP 数据包时间值，该值即为 DTS

上一个 RTMP Chunk 数据包 Chunk Type(fmt) = 1 或 2，则根据解析出的 timestamp 字段值加上上一个 RTMP 数据包的时间值，可以获取当前 RTMP 数据包的时间值，该值即为 DTS

基于 DTS 确定 PTS

对于 H264 视频数据，FLV 的 Video Tag Data 存在 CTS 字段，该字段表示 DTS 与 PTS 的差值，并有 PTS = DTS + CTS，从而可以获取 PTS 的值

对于 AAC 音频数据，其 DTS 与 PTS 的值一致

------

## 4.RTP 与 RTMP 协议之间的时间戳转换

\1. 根据收到的 sr rtcp 数据包，将音视频的时间戳转成绝对时间，然后取音视频绝对时间的最小值作为基准

后续的时间戳 - 基准时间 -> 得到 rtmp 的时间戳

该同步方式比较准确

\2. 不需要 sr rtcp 数据包，直接取当前时间点的音频时间戳、视频时间戳，分别作为各自的音频、视频时间戳基准

后续的音频时间戳 - 音频基准 -> rtmp 音频时间戳

后续的视频时间戳 - 视频基准 -> rtmp 视频时间戳

该同步方式由于音频和视频的基准不同，当偏差较大的时候则音视频没有那么同步。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/455991811