# 【NO.467】神器 ffmpeg——操作视频，极度舒适

现在短视频很流行，有很多视频编辑软件，功能丰富，而我们需要的只是裁剪功能，而且需要用编程的方式调用，那么最合适的莫过于 **ffmpeg[1]** 了。

ffmpeg 是一个命令行工具，功能强大，可以编程调用。

从 ffmpeg 官网上下载对应操作系统的版本，我下的是 **Windows 版[2]**。

下载后解压到一个目录，然后将目录下的 bin，配置到环境变量里。然后打开一个命令行，输入：

```text
> ffmpeg -version
ffmpeg version 2021-10-07-git-b6aeee2d8b-full_build- ...
```

测试一下，能显示出版本信息，说明配置好了。

现在读一下文档，发现拆分视频文件的命令是：

```text
ffmpeg -i [filename] -ss [starttime] -t [length] -c copy [newfilename]
```

- i 为需要裁剪的文件
- ss 为裁剪开始时间
- t 为裁剪结束时间或者长度
- c 为裁剪好的文件存放

好了，用 Python 写一个调用：

```text
import subprocess as sp

def cut_video(filename, outfile, start, length=90):
    cmd = "ffmpeg -i %s -ss %d -t %d -c copy %s" % (filename, start, length, outfile)
    p = sp.Popen(cmd, shell=True)
    p.wait()
    return
```

- 定义了一个函数，通过参数传入 ffmpeg 需要的信息
- 将裁剪命令写成一个字符串模板，将参数替换到其中
- 用 subprocess 的 Popen 执行命令，其中参数 shell=True 表示将命令作为一个整体执行
- p.wait() 很重要，因为裁剪需要一会儿，而且是另起进程执行的，所以需要等执行完成再做后续工作，否则可能找不到裁剪好的文件

这样视频裁剪工作就完成了，然后再看看什么是最重要的。

## 1.计算分段

视频裁剪时，需要一些参数，特别是开始时间，如何确定呢？如果这件事做不好，裁剪工作就很麻烦。

所以看看如何计算裁剪分段。

我需要将视频裁剪成一分半的小段，那么将需要知道目标视频文件的时间长度。

## 2.获取视频长度

如何获得长度呢？ffmpeg 提供了另一个命令 —— ffprobe。

找了一下，可以合成一个命令来获取：

```text
> ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 -i a.flv

920.667
```

命令比较复杂哈，可以先不用管其他参数，只要将要分析的视频文件传入就好了。命令的结果是显示一行视频文件的长度。

于是可以编写一个函数：

```text
import subprocess as sp

def get_video_duration(filename):
    cmd = "ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 -i %s" % filename
    p = sp.Popen(cmd, stdout=sp.PIPE, stderr=sp.PIPE)
    p.wait()
    strout, strerr = p.communicate() # 去掉最后的回车
    ret = strout.decode("utf-8").split("\n")[0]
    return ret
```

- 函数只有一个参数，就是视频文件路径
- 合成命令语句，将视频文件路径替换进去
- 用 subprocess 来执行，注意这里需要设置一下命令执行后的输出
- 用 wait 等待命令执行完成
- 通过 communicate 提取输出结果
- 从结果中提取视频文件的长度，返回

## 3.分段

得到了视频长度，确定好每个分段的长度，就可以计算出需要多少分段了。

代码很简单：

```text
import math
duration = math.floor(float(get_video_duration(filename)))
part = math.ceil(duration / length)
```

注意，计算分段时，需要进行向上取整，即用 ceil，以包含最后的一点尾巴。

得到了需要的分段数，用一个循环就可以计算出每一段的起始时间了。

## 4.获取文件

因为处理的文件很多，所以需要自动获取需要处理的文件。

方法很简单，也很常用，一般可以用 os.walk 递归获取文件，还可以自己写，具体根据实际情况。

```text
for fname in os.listdir(dir):
    fname = os.path.join(dir, os.path.join(dir, fname))
    basenames = os.path.basename(fname).split('.')
    mainname = basenames[0].split("_")[0]
    ...
```

提供视频文件所在的目录，通过 os.listdir 获取目录中的文件，然后，合成文件的绝对路径，因为调用裁剪命令时需要绝对路径比较方便。

获取文件名，是为了在后续对裁剪好的文件进行命名。

## 5.代码集成

现在每个部分都写好了，可以将代码集成起来了：

```text
def main(dir):
    outdir = os.path.join(dir, "output")
    if not os.path.exists(outdir):
        os.mkdir(outdir)

    for fname in os.listdir(dir):
        fname = os.path.join(dir, os.path.join(dir, fname))
        if os.path.isfile(fname):
            split_video(fname, outdir)
```

- main 方法是集成后的方法
- 先创建一个裁剪好的存储目录，放在视频文件目录中的 output 目录里
- 通过 listdir 获取到文件后，对每个文件进行处理，其中判断了一下是否为文件
- 调用 split_video 方法开始对一个视频文件进行裁剪

## 6.总结

总体而言，这是个很简单的应用，核心功能就是调用了一个 ffmpeg 命令。

相对于技术，更重要的是如何对一个项目进行分析和分解，以及从什么地方开始。

这里的方式起始时，不断地找最重要地事情，以最重要的事情为线索不断地推进，最终以自下而上地方式解决整个问题。

期望这篇文章对你有所启发，比心。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/429558315