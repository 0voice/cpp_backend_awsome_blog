# 【NO.397】音视频编解码常用知识点

## 1. 前言

ffplay是ffmpeg的一个子工具，它具有强大的音视频解码播放能力，目前它广泛被各种流行播放器（QQ影音、暴风影音……）集成应用。作为一款开源软件，ffplay囊括Linux、Windows、Ios、Android等众多主流系统平台，十分适合进行二次开发。这里有必要介绍一下它常用的技巧。首先下载ffmpeg代码包，里面有免编译版、源代码百、静态库版、动态库版，具体怎么下载安装请参考我的博文《FFmpeg简介、功能入门、源码下载安装、常规应用》。接下来以Windows平台为例子讲述一下具体用法。

## 2. 使用技巧

Win+r组合键运行cmd进入Windows命令行控制界面，使用cd命令进入ffplay.exe的可执行目录（当然也可以使用环境变量等手段使ffplay.exe命令全局可用），其他平台如linux的操作也类似。ffplay的基本用法很简单，其一般形式如下：

```text
ffplay [option] file
ffplay [option] URL
```

总结起来ffplay的用法就是option项加上资源路径，option项是用来指定播放时的一些参数的，如指定连接的协议、视频画面的大小，音视频解码器选用、传输码率设定等，一般这些参数我们很少会设置，使用默认就OK，此时option项可以直接忽略，ffplay会帮我们选择，这也是它功能强大的体现，option的更多具体选项可以参考其官方文档；资源路径则包括文件资源路径和网络资源路径，文件资源路径是指定需要播放的音视频文件，如*.mp3、*.mp4、*,avi、*.mkv、*.rmvb等等类型的文件，网络资源路径根据协议可以分为RTSP、RTMP、HTTP流资源，心情好，来个直播，如：

```text
ffplay rtmp://live.hkstv.hk.lxdns.com/live/hks
```

再或者，用http浏览一下视频，如：

```text
ffplay http://live.hkstv.hk.lxdns.com/live/hks/playlist.m3u8
```

RTSP播放也了解一下（对于RTSP播放有个坑，请参考《ffplay播放rtsp网络串流失败问题》），如：

```text
rtsp://184.72.239.149/vod/mp4://BigBuckBunny_175k.mov
```

音视频文件指定分辨率播放

```text
ffplay -vfscale=1920:1080 xxxx.avi
```



![img](https://pic3.zhimg.com/80/v2-4c1fb9a29803126870077cf5cd729406_720w.webp)

## 3. 番外篇

在安防等视频流媒体数据处理领域，我们可能更关注的是用ffplay播放RTSP音视频流，其实国内各大厂商的VMS（video management system）平台也是基于此设计的。每每使用它们的IPC、NVR时都需要下载它们，但是有了ffplay神器，一个就够了，它可以播放诸如海康、大华、长视等厂商IPC、NVR的RTSP流，视频监控就变得如此简单。这里很有必要介绍一下RTSP链接的格式。

RTSP链接格式与HTTP链接格式类似，也是由URL（Uniform Resource Locator）发展继承而来。URL由三部分组成：资源类型、存放资源的主机域名、资源文件名，一般语法格式为(带方括号[]的为可选项)：

```text
protocol :// hostname[:port] / path / [;parameters][?query]#fragment
```

这里就不一一解释其各项的含义了，我们重点关注RTSP链接的格式，相比于URL，RTSP由于参数表列是嵌入RTSP报文中的，格式上会少了parameters等参数选项，其一般格式如下：

```text
rtsp://[username]:[password]@[ip]:[port]/path
```

## 4. 总结

运用ffplay播放小技巧可以轻松应对各种文件资源和网络资源的播放，特别是在安防监控领域，使用它播放个监控资源那简直太方便了，而且还可以用它来检查验证音视频格式封包是否异常，在调试优化过程ffplay总能带给你惊喜。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/429125576