# 【NO.395】FFmpeg入门宝典，音视频流媒体开发学习，一篇看到就要收藏的文章（附20个视频资料）

2G打开了了移动互联网天下，3G带来了即时通信，诞生了QQ 微信等巨头，4G 带来了短视频兴起。字节跳动等公司崛起。2 3 4G的出现促成了移动互联网10年繁荣。而5G的出现，也会促成至少10年音视频行业的繁荣。所以，做音视频研发的前景是广阔的，对于很早看出音视频前景的伙伴来说，已经开始学习，想要尽早的投入音视频研发的队伍，掌握这个技术，跳槽涨薪，未来发展提升是必然。

**音视频行业行业现状-**核心竞争力：定义音视频是程序届的皇冠，掌握音视频意味着拿到通往未来的船票，不用担心会被其他人替代。音视频是有门槛的。是与其他人拉开差距的分水岭

高端人才相关缺乏：Boss直聘中，北上广深很多年限上50w-70w的音视频岗位，常年还招不到人，月薪2-3万大多是刚从事音视频入门级开发者

技术迭代慢：就H264编码从95年成为标准至今，都在使用。比较偏底层技术，底层技术几十年不会有太大的改变。

想要入门音视频开发必须要从FFmpeg学起，本篇介绍了一个基础的学习资料，附带视频讲解，建议码住收藏，关注我，持续输出音视频开发技术文章！

## 1.FFmpeg入门宝典

## **2.播放器框架**

![img](https://pic2.zhimg.com/80/v2-acb7e5e4a6dddc6aa0211c13616049a5_720w.webp)

------

## 3.**常见音视频概念**

![img](https://pic4.zhimg.com/80/v2-616d2509ce9eecf2de7938b52512015b_720w.webp)

**容器／文件（Conainer/File）：**即特定格式的多媒体文件， 比如mp4、flv、mkv等。

• **媒体流（Stream）：**表示时间轴上的一段连续数据，如一 段声音数据、一段视频数据或一段字幕数据，可以是压缩的，也可以是非压缩的，压缩的数据需要关联特定的编解码器。

• **数据帧／数据包（Frame/Packet）：**通常，一个媒体流是由大量的数据帧组成的，对于压缩数据，帧对应着编解码器的最小处理单元，分属于不同媒体流的数据帧交错存储于容器之中。

• **编解码器：**编解码器是以帧为单位实现压缩数据和原始数据之间的相互转换的

------

## 4.**常用概念-复用器**

![img](https://pic2.zhimg.com/80/v2-e7686000de2ede9f3d7c3f50aec09ca1_720w.webp)

## 5.**常用概念-编解码器**

![img](https://pic2.zhimg.com/80/v2-74e6fb3f1789dda608f6f4220776711d_720w.webp)



## 6.**FFMPEG入门**

## **7.FFmpeg8个常用库简介**

![img](https://pic3.zhimg.com/80/v2-2957b34da10d9b7902f2117a8e1f790e_720w.webp)

**FFMPEG有8个常用库**：

• AVUtil：核心工具库，下面的许多其他模块都会依赖该库做一些基本的音视频处理操作。

• **AVFormat**：文件格式和协议库，该模块是最重要的模块之一，封装了Protocol层和Demuxer、Muxer层，使得协议和格式对于开发者来说是透明的。

• **AVCodec**：编解码库，封装了Codec层，但是有一些Codec是具备自己的License的，FFmpeg是不会默认添加像libx264、FDK-AAC等库的，但是FFmpeg就像一个平台一样，可以将其他的第三方的Codec以插件的方式添加进来，然后为开发者提供统一的接口。

• AVFilter：音视频滤镜库，该模块提供了包括音频特效和视频特效的处理，在使用FFmpeg的API进行编解码的过程中，直接使用该模块为音视频数据做特效处理是非常方便同时也非常高效的一种方式。

**AVDevice**：输入输出设备库，比如，需要编译出播放声音或者视频的工具ffplay，就需要确保该模块是打开的，同时也需要SDL的预先编译，因为该设备模块播放声音与播放视频使用的都是SDL库。

• **SwrRessample**：该模块可用于**音频重采样**，可以对数字音频进行声道数、数据格式、采样率等多种基本信息的转换。

• **SWScale**：该模块是将图像进行格式转换的模块，比如，可以将YUV的数据转换为RGB的数据，缩放尺寸由1280*720变为800*480。

• **PostProc**：该模块可用于进行后期处理，当我们使用AVFilter的时候需要打开该模块的开关，因为Filter中会使用到该模块的一些基础函数。

◼ av_register_all()：注册所有组件,4.0已经弃用

◼ avdevice_register_all()对设备进行注册，比如V4L2等。

◼ avformat_network_init();初始化网络库以及网络加密协议相关的库（比如openssl）

------

**FFmpeg函数简介**

◼ avformat_alloc_context();负责申请一个AVFormatContext结构的内存,并进行简单初始化

◼ avformat_free_context();释放该结构里的所有东西以及该结构本身

◼ avformat_close_input();关闭解复用器。关闭后就不再需要使用avformat_free_context 进行释放。

◼ avformat_open_input();打开输入视频文件

◼ avformat_find_stream_info()：获取视频文件信息

◼ av_read_frame(); 读取音视频包

◼ avformat_seek_file(); 定位文件

◼ av_seek_frame():定位文件

![img](https://pic1.zhimg.com/80/v2-2b667e31292012695860503595554184_720w.webp)

------

**编解码相关**

• avcodec_alloc_context3():分配解码器上下文

• avcodec_find_decoder()：根据ID查找解码器

• avcodec_find_decoder_by_name():根据解码器名字

• avcodec_open2()：打开编解码器

• avcodec_decode_video2()：解码一帧视频数据

• avcodec_decode_audio4()：解码一帧音频数据

• avcodec_send_packet():发送编码数据包

• avcodec_receive_frame():接收解码后数据

• avcodec_free_context():释放解码器上下文，包含了avcodec_close()

• avcodec_close():关闭解码器

![img](https://pic1.zhimg.com/80/v2-585fdf9938d7a51ca02da6e013b1c8bc_720w.webp)

**FFmpeg3.3 组件注册方式**

我们使用ffmpeg，首先要执行av_register_all，把全局的解码器、编码器等结构体注册到**各自全局的对象链表里**，以便后面查找调用。

![img](https://pic3.zhimg.com/80/v2-6f776c9920e353aa66d1f91c0d620c12_720w.webp)

**FFmpeg 4.0.2 组件注册方式**

FFMPEG内部去做，不需要用户调用API去注册。

**以codec编解码器为例**：

\1. 在configure的时候生成要注册的组件./configure:7203:print_enabled_components libavcodec/codec_list.c AVCodec codec_list $CODEC_LIST

这里会生成一个**codec_list.c** 文件，里面只有static const AVCodec * const **codec_list[]**数组。

\2. 在**libavcodec/allcodecs.c**将static const AVCodec * const codec_list[]的编解码器用链表的方式组织起来。

FFMPEG内部去做，不需要用户调用API去注册。 对于demuxer/muxer（解复用器，也称容器）则对应

\1. libavformat/muxer_list.c libavformat/demuxer_list.c 这两个文件也是在configure的时 候生成，也就是说直接下载源码是没有这两个文件的。

\2. 在libavformat/allformats.c将demuxer_list[]和muexr_list[] 以链表的方式组织。

**其他组件也是类似的方式**

**FFmpeg数据结构简介**

AVFormat**Context**

封装格式上下文结构体，也是统领全局的结构体，保存了视频文件封装格式相关信息。

AVInputFormat demuxer

每种封装格式（例如FLV, MKV, MP4, AVI）对应一个该结构体。

AVOutputFormat muxer AVStream

视频文件中每个视频（音频）流对应一个该结构体。

AVCodec**Context**

编解码器上下文结构体，保存了视频（音频）编解码相关信息。

AVCodec

每种视频（音频）编解码器(例如H.264解码器)对应一个该结构体。

AVPacket

存储一帧压缩编码数据。

AVFrame

存储一帧解码后像素（采样）数据。

------

## **8.FFmpeg数据结构之间的关系**

**AVFormatContext和AVInputFormat之间的关系**

**AVFormatContext API调用**

**AVInputFormat 主要是FFMPEG内部调用**

数据：AVFormatContext 封装格式上下文结构体

struct **AVInputFormat** *iformat;

所有方法可重入的

方法：AVInputFormat 每种封装格式（例如FLV, MKV, MP4）

int (*read_header)(struct **AVFormatContext** * );

int (*read_packet)(struct **AVFormatContext** *, AVPacket *pkt);

面向对象的封装？

int avformat_open_input(AVFormatContext **ps, const char *filename, AVInputFormat *fmt, AVDictionary **options)

**AVCodecContext和AVCodec之间的关系**

**数据：**AVCodecContext 编码器上下文结构体

struct **AVCodec** *codec;

**方法**：**AVCodec** 每种视频（音频）编解码器

int (*decode)(**AVCodecContext** *, void *outdata, int *outdata_size, AVPacket *avpkt);

int (*encode2)(**AVCodecContext** *avctx, AVPacket *avpkt, const AVFrame *frame, int *got_packet_ptr);

**AVFormatContext, AVStream和AVCodecContext之间的关系**

![img](https://pic4.zhimg.com/80/v2-5f546884eb74641d28dbbb47eeb496c7_720w.webp)

**区分不同的码流**

AVMEDIA_TYPE_VIDEO视频流

video_index = av_find_best_stream(ic, AVMEDIA_TYPE_VIDEO,

-1,-1, NULL, 0)

AVMEDIA_TYPE_AUDIO音频流

audio_index = av_find_best_stream(ic, AVMEDIA_TYPE_AUDIO,

-1,-1, NULL, 0)

**AVPacket和AVFrame之间的关系**

![img](https://pic1.zhimg.com/80/v2-ebf69498ebb7745807e2df6cc09b6b38_720w.webp)

**AVFormatContext**

• iformat：输入媒体的AVInputFormat，比如指向AVInputFormat ff_flv_demuxer

• nb_streams：输入媒体的AVStream 个数

• streams：输入媒体的AVStream []数组

• duration：输入媒体的时长（以微秒为单位），计算方式可以参考**av_dump_format()**函数。

• bit_rate：输入媒体的码率

**AVInputFormat**

• name：封装格式名称

• extensions：封装格式的扩展名

• id：封装格式ID

• 一些封装格式处理的接口函数,比如read_packet()

**AVStream**

• index：标识该视频/音频流

• time_base：该流的时基，PTS*time_base=真正的时间（秒）

• avg_frame_rate： 该流的帧率

• duration：该视频/音频流长度

• codecpar：编解码器参数属性

**AVCodecParameters**

• codec_type：媒体类型AVMEDIA_TYPE_VIDEO/AVMEDIA_TYPE_AUDIO等

• codec_id：编解码器类型， AV_CODEC_ID_H264/AV_CODEC_ID_AAC等。

**AVCodecContext**

• codec：编解码器的AVCodec，比如指向AVCodec ff_aac_latm_decoder

• width, height：图像的宽高（只针对视频）

• pix_fmt：像素格式（只针对视频）

• sample_rate：采样率（只针对音频）

• channels：声道数（只针对音频）

• sample_fmt：采样格式（只针对音频）

**AVCodec**

• name：编解码器名称

• type：编解码器类型

• id：编解码器ID

• 一些编解码的接口函数，比如int (*decode)()

**AVPacket**

• pts：显示时间戳

• dts：解码时间戳

• data：压缩编码数据

• size：压缩编码数据大小

• pos:数据的偏移地址

• stream_index：所属的AVStream

**AVFrame**

• data：解码后的图像像素数据（音频采样数据）

• linesize：对视频来说是图像中一行像素的大小；对音频来说是整个音频帧的大小

• width, height：图像的宽高（只针对视频）

• key_frame：是否为关键帧（只针对视频） 。

• pict_type：帧类型（只针对视频） 。例如I， P， B

• sample_rate：音频采样率（只针对音频）

• nb_samples：音频每通道采样数（只针对音频）

• pts：显示时间戳

------

## 9.**SDL简介作用**

SDL(Simple DirectMedia Layer)库的作用主要是封装了复杂的视音频底层交互工作， 简化了视音频处理的难度。

本次课程我们重点不在SDL，只因为涉及到一些函数的调用，所以稍微做一个简介。

SDL结构如下所示。可以看出它实际上还是调用了DirectX等底层的API完成了和硬件的交互

![img](https://pic1.zhimg.com/80/v2-179ec6305981767a4ae01d5c1db843b4_720w.webp)

**SDL视频显示函数简介**

• SDL_Init()：初始化SDL系统

• SDL_CreateWindow()：创建窗口SDL_Window

• SDL_CreateRenderer()：创建渲染器SDL_Renderer

• SDL_CreateTexture()：创建纹理SDL_Texture

• SDL_UpdateTexture()：设置纹理的数据

• SDL_RenderCopy()：将纹理的数据拷贝给渲染器

• SDL_RenderPresent()：显示

• SDL_Delay()：工具函数，用于延时。

• SDL_Quit()：退出SDL系统

**SDL中事件和多线程**

**SDL多线程**

• 函数

• SDL_CreateThread()：创建一个线程

• SDL_LockMutex(), SDL_UnlockMutex()：互斥量操作

• 数据结构

• SDL_Thread：线程的句柄

**SDL事件**

• 函数

• SDL_WaitEvent()：等待一个事件

• SDL_PushEvent()：发送一个事件

• SDL_PumpEvents()：将硬件设备产生的事件放入事件队列

• SDL_PeepEvents()：从事件队列提取一个事件

• 数据结构

• SDL_Event：代表一个事件



原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/432586853