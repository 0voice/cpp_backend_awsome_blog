# 【NO.459】RTSP直播延时的深度优化（干货）

每日专注分享音视频技术点，ffmpeg，webRTC技术，推流拉流，流媒体服务器，SRS，sfu模型，H264码率，等等技术栈文章视频。

------

现在ijkPlayer是许多播放器、直播平台的首选，相信很多开发者都接触过ijkPlayer，无论是Android工程师还是iOS工程师。我曾经在Github上的ijkPlayer开源项目上提问过：视频流为1080P、30fps，如何优化RTSP直播的延时为大约100ms呢？发现大家对RTSP直播延时优化非常感兴趣，纷纷提问或者给出自己的观点。本文主要是总结，也是与大家探讨RTSP直播的延时优化。

## 1.修改编译脚本支持RTSP

ijkPlayer默认是没有把RTSP协议编译进去，所以我们得修改编译脚本，原来的disable改为enable：

```text
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-protocol=rtp"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-protocol=tcp"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rtsp"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=sdp"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=rtp"
```

## 

## **2.修改播放器的option参数**

```text
//丢帧阈值
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "framedrop", 30);
//视频帧率
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "fps", 30);
//环路滤波
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_CODEC, "skip_loop_filter", 48);
//设置无packet缓存
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "packet-buffering", 0);
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "fflags", "nobuffer");
//不限制拉流缓存大小
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "infbuf", 1);
//设置最大缓存数量
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "max-buffer-size", 1024);
//设置最小解码帧数
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "min-frames", 3);
//启动预加载
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "start-on-prepared", 1);
//设置探测包数量
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "probsize", "4096");
//设置分析流时长
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "analyzeduration", "2000000");
```

值得注意的是，ijkPlayer默认使用udp拉流，因为速度比较快。如果需要可靠且减少丢包，可以改为tcp协议：

```text
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "rtsp_transport", "tcp");
```

另外，可以这样开启硬解码，如果打开硬解码失败，再自动切换到软解码：

```text
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec", 0);
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-auto-rotate", 0);
mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "mediacodec-handle-resolution-change", 0);
```

## 3.网络抖动的丢包

在拉流时，音频流、视频流是单独保存到缓冲队列的。如果发生网络抖动，就会引起缓冲抖动（JitBuffer），可以总结为网络卡顿导致音视频缓冲队列增大，从而导致解码滞后、播放滞后。此时，我们需要主动丢包来跟进当前时间戳。因为音视频同步一般以音频时钟为基准，人们对音频更加敏感，所以我们优先丢掉视频队列的包。但是，丢视频数据包时，需要丢掉整个GOP的数据包，因为B帧、P帧依赖I帧来解码，否则会引起花屏。有一位开发者叫做暴走大牙，他的一篇关于ijkPlayer直播延时的文章写得很好：ijkplay播放直播流延时控制小结

## 4.解码器设为零延时

大家应该听过编码器的零延时（zerolatency），但可能没听过解码器零延时。其实解码器内部默认会缓存几帧数据，用于后续关联帧的解码，大概是3-5帧。经过反复测试，发现解码器的缓存帧会带来100多ms延时。也就是说，假如能够去掉缓存帧，就可以减少100多ms的延时。而在avcodec.h文件的AVCodecContext结构体有一个参数（flags）用来设置解码器延时：

```text
typedef struct AVCodecContext {
......
int flags;
......
}
```

为了去掉解码器缓存帧，我们可以把flags设置为CODEC_FLAG_LOW_DELAY。在初始化解码器时进行设置：

```text
//set decoder as low deday
codec_ctx->flags |= CODEC_FLAG_LOW_DELAY;
```

## 5.减少FFmpeg拆帧等待延时

FFmpeg拆帧是根据下一帧的起始码来作为当前帧结束符，起始码一般是：0x00 0x00 0x00 0x01或者0x00 0x00 0x01。这样就会带来一帧的延时，这一帧延时能不能去掉呢？如果有帧结束符，我们以帧结束符来拆帧，这样做就能解决一帧延时。现在，问题变成找到帧结束符，然后替换成下一帧起始码来拆帧。整个调用流程是：read_frame—>read_frame_internal—>parse_packet—>av_parser_parse2—>parser_parse—>ff_combine_frame. 流程图如下：

![img](https://pic1.zhimg.com/80/v2-70f418b9043cce3d058320aa666c80f0_720w.webp)

### 5.1 找到当前帧结束符

在rtpdec.c文件的rtp_parse_packet_internal方法里，有获取帧结束符，也就是mark标志位，我们在这里设一个全局变量：

```text
static int rtp_parse_packet_internal(RTPDemuxContext *s, AVPacket *pkt,
                                     const uint8_t *buf, int len)
{
    ......
 
    if (buf[1] & 0x80)
        flags |= RTP_FLAG_MARKER;
    //the end of a frame
    mark_flag = flags;
 
    ......
}
```

### 5.2 去掉parse_packet的while循环

我们在外部调用libavformat模块的utils.c文件的read_frame读取一帧数据，而read_frame调用内部方法read_frame_internal，read_frame_internal接着调用parse_packet方法，在该方法里有一个while循环体。现在把循环体去掉，并且释放申请的内存：

```text
static int parse_packet(AVFormatContext *s, AVPacket *pkt, int stream_index)
{
    ......
 
//    while (size > 0 || (pkt == &flush_pkt && got_output)) {
        int len;
        int64_t next_pts = pkt->pts;
        int64_t next_dts = pkt->dts;
 
        av_init_packet(&out_pkt);
        len = av_parser_parse2(st->parser, st->internal->avctx,
                               &out_pkt.data, &out_pkt.size, data, size,
                               pkt->pts, pkt->dts, pkt->pos);
        pkt->pts = pkt->dts = AV_NOPTS_VALUE;
        pkt->pos = -1;
        /* increment read pointer */
        data += len;
        size -= len;
 
        got_output = !!out_pkt.size;
 
        if (!out_pkt.size){
            av_packet_unref(&out_pkt);//release current packet
            av_packet_unref(pkt);//release current packet
            return 0;
//            continue;
        }
    ......        
   
        ret = add_to_pktbuf(&s->internal->parse_queue, &out_pkt,
                            &s->internal->parse_queue_end, 1);
        av_packet_unref(&out_pkt);
        if (ret < 0)
            goto fail;
//    }
 
    /* end of the stream => close and free the parser */
    if (pkt == &flush_pkt) {
        av_parser_close(st->parser);
        st->parser = NULL;
    }
 
fail:
    av_packet_unref(pkt);
    return ret;
}
```

### 5.3 修改av_parser_parse2的帧偏移量

在libavcodec模块的parser.c文件中，parse_packet调用到av_parser_parse2来解释数据包，该方法内部有记录帧偏移量。原先是等待下一帧的起始码，现在改为当前帧结束符，所以要把下一帧起始码这个偏移量长度去掉：

```text
int av_parser_parse2(AVCodecParserContext *s, AVCodecContext *avctx,
                     uint8_t **poutbuf, int *poutbuf_size,
                     const uint8_t *buf, int buf_size,
                     int64_t pts, int64_t dts, int64_t pos)
{
    ......
 
    /* WARNING: the returned index can be negative */
    index = s->parser->parser_parse(s, avctx, (const uint8_t **) poutbuf,
                                    poutbuf_size, buf, buf_size);
    av_assert0(index > -0x20000000); // The API does not allow returning AVERROR codes
#define FILL(name) if(s->name > 0 && avctx->name <= 0) avctx->name = s->name
    if (avctx->codec_type == AVMEDIA_TYPE_VIDEO) {
        FILL(field_order);
    }
 
    /* update the file pointer */
    if (*poutbuf_size) {
        /* fill the data for the current frame */
        s->frame_offset = s->next_frame_offset;
 
        /* offset of the next frame */
//        s->next_frame_offset = s->cur_offset + index;
        //video frame don't plus index
        if (avctx->codec_type == AVMEDIA_TYPE_VIDEO) {
            s->next_frame_offset = s->cur_offset;
        }else{
            s->next_frame_offset = s->cur_offset + index;
        }
        s->fetch_timestamp   = 1;
    }
    if (index < 0)
        index = 0;
    s->cur_offset += index;
    return index;
}
```

### 5.4 去掉parser_parse的寻找帧起始码

av_parser_parse2调用到parser_parse方法，而我们这里使用的是h264解码，所以在libavcodec模块的h264_parser.c有一个结构体ff_h264_parser，把h264_parse赋值给parser_parse：

```text
AVCodecParser ff_h264_parser = {
    .codec_ids      = { AV_CODEC_ID_H264 },
    .priv_data_size = sizeof(H264ParseContext),
    .parser_init    = init,
    .parser_parse   = h264_parse,
    .parser_close   = h264_close,
    .split          = h264_split,
};
```

现在我们需要h264_parser.c文件的h264_parse方法，去掉寻找下一帧起始码作为当前帧结束符的过程：

```text
static int h264_parse(AVCodecParserContext *s,
                      AVCodecContext *avctx,
                      const uint8_t **poutbuf, int *poutbuf_size,
                      const uint8_t *buf, int buf_size)
{
    ......
 
    if (s->flags & PARSER_FLAG_COMPLETE_FRAMES) {
        next = buf_size;
    } else {
//TODO:don't use next frame start code, modify by xufulong
//        next = h264_find_frame_end(p, buf, buf_size, avctx);
 
        if (ff_combine_frame(pc, next, &buf, &buf_size) < 0) {
            *poutbuf      = NULL;
            *poutbuf_size = 0;
            return buf_size;
        }
 
/*        if (next < 0 && next != END_NOT_FOUND) {
            av_assert1(pc->last_index + next >= 0);
            h264_find_frame_end(p, &pc->buffer[pc->last_index + next], -next, avctx); // update state
        }*/
    }
 
    ......
}
```

### 5.5 修改parser.c的组帧方法

h264_parse又调用parser.c的ff_combine_frame组帧方法，我们在这里把mark替换起始码作为帧结束符：

```text
external int mark_flag;//引用全局变量
 
int ff_combine_frame(ParseContext *pc, int next,const uint8_t **buf, int *buf_size)
{
    ......
 
    /* copy into buffer end return */
//    if (next == END_NOT_FOUND) {
        void *new_buffer = av_fast_realloc(pc->buffer, &pc->buffer_size,
                                           *buf_size + pc->index +
                                           AV_INPUT_BUFFER_PADDING_SIZE);
 
        if (!new_buffer) {
          
            pc->index = 0;
            return AVERROR(ENOMEM);
        }
        pc->buffer = new_buffer;
        memcpy(&pc->buffer[pc->index], *buf, *buf_size);
        pc->index += *buf_size;
//        return -1;
          if(!mark_flag)
            return -1;
        next = 0;
//    }
 
    ......
 
}
```

经过以上修改，局域网用电脑推送1080P、30fps的视频流，Android设备拉流解码播放，整体延时可优化至130ms左右。而手机推流，延时可达到86ms。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/457790444