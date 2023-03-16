# 【NO.464】FFMPEG 之 AVDevice

## 1.综述

在使用FFMPEG 作为编码时，可以采用FFMPEG 采集本地的音视频设备的数据，然后进行编码，封装，传输等操作。 在不同的平台上，音视频数据采集驱动并不一致。 在linux 平台上有fbdex,v4l2,x11grab ，在ios 上有avfoundation. 在window 平台上有dshow,vfwcap.gdigrab。

![img](https://pic4.zhimg.com/80/v2-3880ea1dc2ebf76fe33b0f144d4ccf83_720w.webp)

## 2.使用流程

对于FFMPEG 来说，device 也是一个文件， 在使用device 进行录屏，摄像时， 与普通播放一个片源差不多。 就初始化处稍有点区别。

播放普通片源，初始化方式：

```text
AVFormatContext *pFormatCtx = avformat_alloc_context();
avformat_open_input(&pFormatCtx, "test.h265",NULL,NULL);
```

而使用device 去录屏时：

```text
AVFormatContext *pFormatCtx = avformat_alloc_context();
AVInputFormat *ifmt=av_find_input_format("vfwcap");
avformat_open_input(&pFormatCtx, 0, ifmt,NULL);
```

可以看出录屏与播放普通视频，就多一个av_find_input_forma() 的调用。

此处仅讲关于FFMPEG 有哪些API ， 关于如何使用FFMPEG 操作device 录像，录屏请参考雷神csdn ，写的非常详细。

## 3.AVDevice 相关API

如下是AVDevice 库开出的API ， 很清晰， 比如查当前平台有哪些可用的device, 可用avdevice_list_devices()

![img](https://pic2.zhimg.com/80/v2-162eeda0c10a15856115719e8f0e5371_720w.webp)

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/451806536