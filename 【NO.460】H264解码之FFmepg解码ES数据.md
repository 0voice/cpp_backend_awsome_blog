# 【NO.460】H264解码之FFmepg解码ES数据

## 1.**前言**

在项目开发中，我们收到的数据流，一般是RTP数据流，所以在解码过程中，我们分为以下几个步骤：

解析RTP数据包，去RTP头，得到PS数据

解析PS数据，去掉PS头，得到PES数据

解析PES数据，去掉PES头，得到ES数据

解析ES数据，如解析出：PPS、SPS、IDR、P等

本文，我们就来探究怎么用FFmepg解析ES数据。

## 2.解析ES数据

在解析ES数据之前，我们会解析PES，在解析PES过程中，会解析出PTS、DTS、ES数据包、ES数据包长度。

接下来就开始正式解码了。

现在很多人或者说网上很多讲的还是用的FFmepg3.X.X以前的接口，如avcodec_decode_video2()，甚至更早的。

我们这里两种方法都说一说。

首先使用avcodec_decode_video2()方式：

**代码片段：**

```text
void decode(unsigned char * stream, long len, unsigned __int64 pts, long pts_ref_distance, unsigned __int64 dts)
{
	int ret;
 	m_avpacket.data = stream;
 	m_avpacket.size = len;
	
	//这里需要前面pes解析出的pts、dts进行显示控制，要不然，视频帧会时快时慢 
 	m_avpacket.pts = pts;
 	m_avpacket.dts = dts;
 	int got_picture = 0;
 	long diff = 0;
 
 	int re = avcodec_decode_video2(m_avcodeccontext, m_avframe, &got_picture, &m_avpacket);
	if (got_picture > 0 && re > 0)
	{
		__int64 the_pts = 0;
		the_pts = av_frame_get_best_effort_timestamp(m_avframe);
		if (0 == m_last_pts)
		{
			m_last_pts = the_pts;
		}
		diff = (long)(abs(the_pts - m_last_pts) *1000.0 / 90000);
		m_last_pts = the_pts;
		long lStoreLen = m_avframe->width * m_avframe->height * 3 >> 1;

		if (m_avframe->width > 0 && m_avframe->height > 0 && lStoreLen > YUV_BUF)
		{
			if (lStoreLen > YUV_BUF_MAX )
			{
				OutputDebugStringA("yuv buf overflow 20M!");
				return;
			}
			delete[] m_dest;
			m_dest = new unsigned char[lStoreLen];
		}
		//这里解析出来的数据通过av_image_copy_to_buffer拷贝为YUV数据
		AVPixelFormat fmt = (AVPixelFormat)m_avframe->format;
        av_image_copy_to_buffer(m_dest, lStoreLen, 
            m_avframe->data, m_avframe->linesize, 
            fmt, m_avframe->width, m_avframe->height, 1);
		/*................................
		//这里就可以将YUV数据行进操作了
		................................*/
	}
	else
	{
		OutputDebugStringA("****decode failed\n");
	}
}
```

接下来我们说说新方式，就是代替官方推荐的代替avcodec_decode_video2方式：

![img](https://pic2.zhimg.com/80/v2-5ad76c28f7ff19c6125766607e830d5d_720w.webp)

**代码片段：**

```text
void decode(unsigned char * stream, long len, unsigned __int64 pts, long pts_ref_distance, unsigned __int64 dts)
{
	int ret;
	long diff = 0;
	while (len > 0)
	{
		ret = av_parser_parse2(m_avCodePCparser, m_avcodeccontext, &m_avPpkt->data, &m_avPpkt->size,stream, len,pts, dts, 0);
		if (ret < 0)
		{
			OutputDebugStringA("Error while parsing\n");
		}

		stream += ret;
		len -= ret;

		if (m_avPpkt->size)
		{
			m_avPpkt->pts = pts;
			m_avPpkt->dts = dts;

			ret = avcodec_send_packet(m_avcodeccontext, m_avPpkt);
			if (ret < 0)
			{
				OutputDebugStringA("Error submitting the packet to the decoder\n");
			}

			while (ret == 0) {
				ret = avcodec_receive_frame(m_avcodeccontext, m_avframe);
				if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
					continue;
				else if (ret < 0)
				{
					OutputDebugStringA("Error during decoding\n");
				}

				if (ret >= 0)
				{
					__int64 the_pts = 0;
					the_pts = av_frame_get_best_effort_timestamp(m_avframe);
					if (0 == m_last_pts)
					{
						m_last_pts = the_pts;
					}
					diff = (long)(abs(the_pts - m_last_pts) *1000.0 / 90000);
					m_last_pts = the_pts;

					int data_size = av_image_get_buffer_size((AVPixelFormat)m_avframe->format, m_avframe->width, m_avframe->height, 1);

					if (m_avframe->width > 0 && m_avframe->height > 0 && data_size > YUV_BUF)
					{
						delete[] m_dest;
						m_dest = new unsigned char[data_size];
					}

					AVPixelFormat fmt = (AVPixelFormat)m_avframe->format;
					av_image_copy_to_buffer(m_dest, data_size,
						m_avframe->data, m_avframe->linesize,
						fmt, m_avframe->width, m_avframe->height, 1);
					/*................................
					//这里就可以将YUV数据行进操作了
					................................*/
				}
			}
		}
	}
	
	return;
}
```

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/457349444