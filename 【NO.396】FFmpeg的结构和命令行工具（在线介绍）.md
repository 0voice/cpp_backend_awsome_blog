# 【NO.396】FFmpeg的结构和命令行工具（在线介绍）

## 1.FFmpeg介绍以及编译

音视频开发，首先不得不提到FFmpeg。该框架为开发者们提供了非常大的帮助，它是一套可以用来采集、处理、编码、传输的开源框架。可以用在各种PC端（Linux、Windows、macOS）和移动端（iOS、Android）等平台。本节会从编译开始，介绍一下FFmpeg的框架，以及在iOS平台上的使用。

## 2.FFmpeg编译选项

我们可以到FFmpeg官网下载稳定版本的源码（一般来说4.3的版本比4.3.1的版本稳定）。然后将下载的源码解压，FFmpeg与大部分GNU软件的编译方式类似，都是通过configure脚本来实现编译前定制的，这种方式可以让用户在编译前对软件进行裁剪，同时通过对运行系统以及目标平台的指定，实现满足需求的最小包体积编译。我们可以利用help命令来查看它有哪些编译选项。

```text
./configure -help
复制代码
```

1. 标准选项：GNU软件配置项，例如安装路径、-- prefix=...等。
2. 专利使用协议选项：一些开源协议。
3. 编译、链接选项：生成静态库还是动态库这些。
4. 可执行程序选项：决定是否生成FFmpeg、ffplay、ffprobe 和ffserver等。
5. 文档选项：文档的类型。
6. 模块选项：需要开启或者关闭里面哪些模块。
7. Toolchain选项：CPU架构、交叉编译，操作系统。C、C++的一些配置等。
8. 其他：一些其他深度定制编译选项。

## 3.FFmpeg结构

默认的编译会生成4个可执行文件和8个静态库。可执行文件分别是用于转码、裁剪文件的ffmpeg、用于播放媒体文件的ffplay、用于获取文件信息的ffprobe、以及作为简单流媒体服务器的ffserver。8个静态库就是FFmpeg的8个模块，具体包括以下内容：

- AVUtil：核心工具库，该模块是最基础的模块之一，许多其他的模块都会依赖这个工具做一些操作。
- AVFormat：文件格式和协议库，该模块是用来做传输层数据的封装和解封装的，使得协议和格式转换处理更加方便。
- AVCodec：编解码库，FFmpeg有一些默认的编解码库，也可以集成如H264、FDK-AAC等库。其他第三方库以插件的形式添加在FFmpeg中，API调用符合默认接口风格，为开发者提供了统一的接口。
- AVFilter：音视频滤镜库，这个模块提供了音视频中的各种特效处理，非常方便。
- AVDevice：输入输出设备库，获取系统上的输入输出设备，从而实现采集或者播放音视频。
- SWResample：音频重采样，音频的格式转换、混音。
- SWScale：提供图像的裁剪、旋转、格式转换等功能。
- PostProc：视频后期处理。

## 4.iOS平台的编译

对于iOS平台来说，需要在configure的时候加入以下内容：

```text
./configure \
--target-os=darwin \
--arch="arm64" \
--cc="xcrun -sdk iphoneos clang" \
--extra-cflags="-arch arm64 -mios-version-min=8.0 -Ixx" \
--extra-ldflags="-arch arm64 -mios-version-min=8.0 -Lxx" \
复制代码
```

这里推荐一个编译脚本FFmpeg-iOS-build-scriptt，它可以帮我下载指定的FFmpeg版本，选择编译的平台，配置链接的第三方库，非常好用。

## 5.在macOS上安装FFmpeg命令行工具

```text
brew install ffmpeg
复制代码
```

如果安装好没有ffplay工具，那么在安装时加上--with-ffplay就有了。

```text
brew install ffmpeg --with-ffplay
```

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/429550853