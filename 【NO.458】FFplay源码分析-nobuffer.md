# 【NO.458】FFplay源码分析-nobuffer

在使用 FFplay 播放 RTMP 流的时候，如果 不开启 nobuffer 选项，画面延迟会高达 7 秒左右，开启了，局域网延迟可降低到100毫秒左右。

因此本文主要研究 nobuffer 的具体实现，以及播放端 缓存 7 秒的数据有何作用。

fflags 的定义在 libavformat/options_table.h，如下图代码，这是一个通用选项，所有的 解复用器都有这个选项。

![img](https://pic1.zhimg.com/80/v2-d0ecf694be9557a0f4f2f50a7a6d57fc_720w.webp)

也就是说，这个命令行参数，是在调 avformat_open_input() 函数的时候丢进去的，我为什么知道是在这个地方丢进去的？请看之前的专栏《FFplay源码分析》，所有的解复用参数，也叫格式参数，都是在 avformat_open_input() 丢进去的。可以看下图证明一下：

![img](https://pic4.zhimg.com/80/v2-6b4c1e0d67bb35945a555c72eea3b99b_720w.webp)

记得改 Clion 的调试参数，把 `-fflags nobuffer` 加上去。

------

在 `avformat_open_input()` 函数内部，会把 `fflags` 这个 AVOption 丢给 AVClass，如下图所示，AVClass 里面存储了好几个 AVOption ，fflags 这个 AVOption 的下标是 5 ，前面的是 默认的选项，自动加进去的。

![img](https://pic1.zhimg.com/80/v2-235344190b78c850b0d5fbdef4b38750_720w.webp)

注意 ，`av_opt_set_dict()` 这个函数会改变 `tmp` 的值，把能用 选项应用之后剔除。



https://link.zhihu.com/?target=https%3A//docs.qq.com/doc/DYXlQZVZoRkRBRlFk)



------

`nobuffer` 这个参数 传递过程中的函数调用有点长，推荐看之前的《[FFmpeg源码分析-参数解析篇](https://link.zhihu.com/?target=https%3A//www.xianwaizhiyin.net/%3Fcat%3D14)》，原理类似，我知道这个 参数最后会赋值到 `struct AVFormatContext` 的 `flags` 字段里面，如下图：

![img](https://pic2.zhimg.com/80/v2-fc9816793f1a3a3d0c8443347118cc99_720w.webp)

如上图所示，所以 `avformat_open_input()` 函数执行完之后 `AVFormatContext::flags` 的第7位应该会被置为1，因为 0x40 的二进制是 1000000。请看下图：

![img](https://pic1.zhimg.com/80/v2-5ea510ed6cca9e7c93220dc1b653988c_720w.webp)

从上图可以看出， ic->flags 直接就是 64 ，也就是 16 进制的 0x40。所以上面的分析没错。 avformat_open_input() 函数只是把 命令行参数解析 到这个 flags 字段，但是真正使用这个字段 是在 avformat_find_stream_info() 里面。直接搜 AVFMT_FLAG_NOBUFFER 就能找到使用的位置。

avformat_find_stream_info() 函数的内部逻辑实际上非常复杂，我直接讲重点代码，如下:

```text
if (!(ic->flags & AVFMT_FLAG_NOBUFFER)) {
    ret = avpriv_packet_list_put(&ic->internal->packet_buffer,
                                        &ic->internal->packet_buffer_end,
    pkt1, NULL, 0);
    if (ret < 0)
        goto unref_then_goto_end;

    pkt = &ic->internal->packet_buffer_end->pkt;
} else {
    pkt = pkt1;
}
```

AVFMT_FLAG_NOBUFFER 标记 如果没设置，就会导致 探测的数据包丢进去队列，我们知道 avformat_find_stream_info() 会先读一段数据包分析出流是什么编码器之类的，为了重用这个 探测的数据包，这里就会丢进去队列，播放的时候，就从这些数据包开始，但是整个探测过程，长达 5秒，也就是 ffplay 大概会读 5秒的数据，来分析输入流的情况。如果开启 nobuffer，就不会重用这些探测数据，ffpaly 探测完输入流之后，就会重新读取新的数据包来播放。不用缓存的，所以延迟就低了。

如下图，我在 ffpaly.c 的 avformat_find_stream_info() 前后输出了个时间，正好相差5秒。

```text
double start_time3 = av_gettime_relative() / 1000000.0;
if (find_stream_info) {
    ...省略代码...
    err = avformat_find_stream_info(ic, opts);
    ...省略代码...
}
double end_time3 = av_gettime_relative() / 1000000.0;
printf("start is %f , end is %f \r\n",start_time3,end_time3);
```

![img](https://pic4.zhimg.com/80/v2-aa55afd51e3011dd6d7fa7e947bd7703_720w.webp)

所以实际上 ffplay 在 实时的场景下，缓存是个鸡肋，本来这个 buffer 功能是为了分析本地文件，避免重复读取，但是影响到了实时的场景。实时场景必须把 buffer 关掉。

补充，因为我是虚拟机做服务器，所以都是本机通信，不启用 buffer 也能很流畅，但是如果我把 SRS 部署在局域网另一台机器，不开启 buffer ，视频有卡顿，估计是还没来得及解码丢进去队列，所以 ffplay 不断丢弃视频帧。因为视频比音频慢了，得丢弃。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/492176676