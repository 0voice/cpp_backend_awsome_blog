# 【NO.404】x264码率控制

问题：在做视频编码时，当我们给定编码器一个目标码率的时候，编码器内部是怎么达到码率要求的那？

概况：关于码率控制有两个目的，第一：兼容传输，播放条件。第二：获取更高的视频质量。

码率控制分为两类：CBR：constant bit rate，固定码率。 VBR：variable bit rate 可变码率。

VBR：可变码率是一类码率控制算法的统称，他们的特点是局部的码率可变的，常用的可变码率子类包括如下：

1：abr：average bit rate，控制整个文件的平均码率。

2：crf：constant refactor ，恒定质量。总码率不可控

3：cqp：constatnt qp，恒定量化参数。关闭一切码率控制算法，与crf的区别在于，crf允许x264对每一帧，每一个宏块进行选取qp，从而产生一个恒定的质量。

对应的x264参数如下：

```text
#define X264_RC_CQP                  0
#define X264_RC_CRF                  1
#define X264_RC_ABR                  2
 
//恒定QP
int         i_qp_constant;  /* 0 to (51 + 6*(x264_bit_depth-8)). 0=lossless */
int         i_qp_min;       /* min allowed QP value */
int         i_qp_max;       /* max allowed QP value */
int         i_qp_step;      /* max QP step between frames */
 
//恒定质量
float       f_rf_constant;  /* 1pass VBR, nominal QP */
float       f_rf_constant_max;  /* In CRF mode, maximum CRF as caused by VBV */
 
//平均码率
 int         i_bitrate;
```

恒定码率CBR：

并不是每个瞬间码率都相同，也不是每一秒码率相同。固定码率指的是固定信道容量。此时就涉及到了VBV（video buffer verifier）视频缓冲区校验器。vbv模型：编码码率通过一个容量受限的信道传输到解码设备，解码设备在解码前有一个缓存，解码器实时从缓存区读取数据解码，保证即不上溢也不下溢(即拿取速度过快或过慢)。

对应参数如下：最终生成的mp4文件可以看出码率为147kbps.，buffer_size的带下取决于容忍的延迟以及播放器的硬件内存限制。

```text
int         i_vbv_max_bitrate;//缓冲区最大填充速度
int         i_vbv_buffer_size;//缓冲区大小.
 
FFmpeg.exe -i q.mp4 -crf 21 -maxrate 150k -bufsize 450k -codec：v：0 libx264 -s 320x240 -r 15 out.
```

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/454882492