# 【NO.470】FFmpeg命令行格式和转码过程

## 1.命令行基本格式为：

```text
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...
```

格式分解如下：

```text
ffmpeg
    global_options
        input1_options -i input1
        input2_options -i input2
        ...
            output1_options output1
            output2_options output2
            ...
```

“ffmpeg” 读取任意数量的输入 “文件”(可以是常规文件、管道、网络流、录制设备等，由 “-i” 选项指定)，写入任意数量的输出 “文件”。命令行中无法被解释为选项(option)的任何元素都会被当作输出文件。

每个输入或输出文件，原则上都可以包含任意数量的流。FFmpeg 中流的类型有五种：视频(video)、音频(audio)、字幕(subtitle)、附加数据(attachment)、普通数据(data)。文件中流的数量和(或)流类型种数的极限值由文件封装格式决定。选择哪一路输入文件的哪一路流输出到哪一路输出，这个选择过程既可以由 FFmpeg 自动完成，也可以通过 “-map” 选项手动指定(后面第 6 节 “流选择” 章节会深入描述)。

注：关于附加数据(attachment)和普通数据(data)的说明如下：

> Attachments could be liner notes, related images, metadata files, fonts, etc. Data tracks would be for things like timecode, navigation items, cmml, streaming tracks. 参考资料[3] “What are the the data and attachment stream type?”

命令行中的输入文件及输入文件中的流都可以通过对应的索引引用，文件、流的索引都是从 0 开始。例如，2:4 表示第 3 个输入文件中的第 5 个流。(后面 6.3 节 “stream specifier” 章节会详细介绍)。

一个通用规则是：**输入/输出选项(options)作用于跟随此选项后的第一个文件。因此，顺序很重要，并且可以在命令行中多次指定同一选项。每个选项仅作用于离此选项最近的下一输入或输出文件。全局选项不受此规则限制。**

不要把输入文件和输出文件混在一起———应该先将输入文件写完，再写输出文件。也不要把不同文件的选项混在一起，各选项仅对其下一输入或输出文件有效，一个选项不能跨越一个文件传递到后续文件。

举几个命令行例子：

■ 设置输出文件码率为 64 kbit/s：

```text
ffmpeg -i input.avi -b:v 64k -bufsize 64k output.avi
```

其中 “-b:v 64k” 和 “-bufsize 64k” 是输出选项。

■ 强制输入文件帧率(仅对 raw 格式有效)是 1 fps，输出文件帧率为 24 fps：

```text
ffmpeg -r 1 -i input.m2v -r 24 output.avi
```

其中 “-r 1” 是输入选项，“-r 24” 是输出选项。

■ 转封装：将 avi 格式转为 mp4 格式，并将视频缩放为 vga 分辨率：

```text
ffmpeg -y -i video.avi -s vga video.mp4
```

其中 “-y” 是全局选项，“-s vga” 是输出选项。

## 2. 转码过程

```text
_______              ______________
|       |            |              |
| input |  demuxer   | encoded data |   decoder
| file  | ---------> | packets      | -----+
|_______|            |______________|      |
                                           v
                                       _________
                                      |         |
                                      | decoded |
                                      | frames  |
                                      |_________|
 ________             ______________       |
|        |           |              |      |
| output | <-------- | encoded data | <----+
| file   |   muxer   | packets      |   encoder
|________|           |______________|
```

“ffmpeg” 调用 libavformat 库(包含解复用器 demuxer)，从输入文件中读取到包含编码数据的包(packet)。如果有多个输入文件，“ffmpeg” 尝试追踪多个有效输入流的最小时间戳(timestamp)，用这种方式实现多个输入文件的同步。

然后编码包(packet)被传递到解码器(decoder)，解码器解码后生成原始帧(frame)，原始帧可以被滤镜(filter)处理(图中未画滤镜)，经滤镜处理后的帧送给编码器，编码器将之编码后输出编码包。最终，由复用器(muxex)将编码包写入特定封装格式的输出文件。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/427312103