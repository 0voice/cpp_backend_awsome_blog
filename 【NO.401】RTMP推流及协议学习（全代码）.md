# 【NO.401】RTMP推流及协议学习（全代码）

每日专注分享音视频技术点，ffmpeg，webRTC技术，推流拉流，流媒体服务器，SRS，sfu模型，H264码率，等等技术栈文章视频。

------

## 1.推流工作

### 1.1 整体框架图

[Stream](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3DStream)ing from RTSP -> Get Audio/Video frame -> Convert frame -> RTMP push

使用libtrmp提供的API

lib[rtmp](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3Drtmp)提供了推流的API，可以在rtmp.h文件中查看所有API。我们只需要使用常用的几个API就可以将streaming推送到服务器。
\- RTMP_Init()//初始化结构体
\- RTMP_Free()
\- RTMP_Alloc()
\- RTMP_SetupURL()//设置rtmp server地址
\- RTMP_EnableWrite()//打开可写选项，设定为推流状态
\- RTMP_Connect()//建立NetConnection
\- RTMP_Close()//关闭连接
\- RTMP_ConnectStream()//建立NetStream
\- RTMP_DeleteStream()//删除NetStream
\- RTMP_SendPacket()//发送数据

Start -> RTMP_Init() -> RTMP_Alloc() -> RTMP_Setup() -> RTMP_EnableWrite() -> RTMP_Connect()

-> RTMP_ConnectStream() -> RTMP_SendPacket() -> End

### 1.2 将streaming封装成为RTMP格式

在发送第一帧Audio和Video的时候，需要将Audio和Video的信息封装成为RTMP header，发送给rtmp server。
Audio头有4字节，包含：头部标记0xaf 0x00、 profile、channel、bitrate 信息。
Video头有16字节，包含I[Frame](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3DFrame)、PFrame、AVC标识，除此之外，还需要将sps和pps放在header 里面。
RTMP协议定义了message Type，其中Type ID为8，9的消息分别用于传输音频和视频数据：

```text
#define RTMP_PACKET_TYPE_AUDIO 0x08
#define RTMP_PACKET_TYPE_VIDEO 0x09
```

### 1.3 Audio 格式封装的源码：

AAC header packet:

```text
body = (unsigned char *)malloc(4 + size);
memset(body, 0, 4);
body[0] = 0xaf;
body[1] = 0x00;
 
switch (profile){
 case 0:
    body[2]|=(1<<3);//main
    break;
 case 1:
    body[2]|=(1<<4);//LC
    break;
 case 2:
    body[2]|=(1<<3);//SSR
    body[2]|=(1<<4);
    break;
 default:
    ;
}
switch(this->channel){
 case 1:
    body[3]|=(1<<3);//channel1
    break;
 case 2:
    body[3]|=(1<<4);//channel2
    break;
 default:
    ;
}
switch(this->rate){
 case 48000:
    body[2]|=(1);
    body[3]|=(1<<7);
    break;
 case 44100:
    body[2]|=(1<<1);
    break;
 case 32000:
    body[2]|=(1<<1);
    body[3]|=(1<<7);
    break;
 default:
    ;
}
sendPacket(RTMP_PACKET_TYPE_AUDIO, body, 4, 0);
free(body);
```

### 1.4 Video 格式封装的源码：

H264 header packet:

```text
body = (unsigned char *)malloc(16 + sps_len + pps_len);
this->videoFist = false;
 
memset(body, 0, 16 + sps_len + pps_len);
body[i++] = 0x17;   // 1: IFrame, 7: AVC
                    // AVC Sequence Header
body[i++] = 0x00;
body[i++] = 0x00;
body[i++] = 0x00;
body[i++] = 0x00;
 
// AVCDecoderConfigurationRecord
body[i++] = 0x01;
body[i++] = sps[1];
body[i++] = sps[2];
body[i++] = sps[3];
body[i++] = 0xff;
body[i++] = 0xe1;
body[i++] = (sps_len >> 8) & 0xff;
body[i++] = sps_len & 0xff;
for (size_t j = 0; j < sps_len; j++)
{
    body[i++] = sps[j];
}
body[i++] = 0x01;
body[i++] = (pps_len >> 8) & 0xff;
body[i++] = pps_len & 0xff;
for (size_t j = 0; j < pps_len; j++)
{
    body[i++] = pps[j];
}
sendPacket(RTMP_PACKET_TYPE_VIDEO, body, i, nTimeStamp);
 
free(body);
```

只有第一帧Audio和第一帧video才需要发送header信息。之后就直接发送帧数据。

发送Audio的时候，只需要在数据帧前面加上2 byte的header信息：

```text
spec_info[0] = 0xAF;
spec_info[1] = 0x01;
```

发送Video的时候，需要在header里面标识出I P帧的信息，以及视频帧的长度信息：

```text
body = (unsigned char *)malloc(9 + size);
memset(body, 0, 9);
i = 0;
if (bIsKeyFrame== 0) {
    body[i++] = 0x17;   // 1: IFrame, 7: AVC
}
else {
    body[i++] = 0x27;   // 2: PFrame, 7: AVC
}
// AVCVIDEOPACKET
body[i++] = 0x01;
body[i++] = 0x00;
body[i++] = 0x00;
body[i++] = 0x00;
 
// NALUs
body[i++] = size >> 24 & 0xff;
body[i++] = size >> 16 & 0xff;
body[i++] = size >> 8 & 0xff;
body[i++] = size & 0xff;
memcpy(&body[i], data, size);
```



## 2.进阶

### 2.1 RTMP client与RTMP server交互流程

1 简要介绍

播放一个RTMP协议的流媒体需要经过以下几个步骤：握手，建立网络连接，建立网络流，播放。RTMP连接都是以握手作为开始的。建立连接阶段用于建立客户端与服务器之间的“网络连接”；建立流阶段用于建立客户端与服务器之间的“网络流”；播放阶段用于传输视音频数据。其中，网络连接代表服务器端应用程序和客户端之间基础的连通关系。网络流代表了发送多媒体数据的通道。服务器和客户端之间只能建立一个网络连接，但是基于该连接可以创建很多网络流。他们的关系如图所示：

![img](https://pic4.zhimg.com/80/v2-bd45a654f7b69e2ad420c201b00566a3_720w.webp)

2 握手（HandShake）

一个RTMP连接以握手开始，双方分别发送大小固定的三个数据块

a) 握手开始于客户端发送C0、C1块。服务器收到C0或C1后发送S0和S1。

b) 当客户端收齐S0和S1后，开始发送C2。当服务器收齐C0和C1后，开始发送S2。

c) 当客户端和服务器分别收到S2和C2后，握手完成。

![img](https://pic2.zhimg.com/80/v2-baf261ffdfd5222fc0026e337c6b9041_720w.webp)

3建立网络连接（NetConnection）

a) 客户端发送命令消息中的“连接”(connect)到服务器，请求与一个服务应用实例建立连接。

b) 服务器接收到连接命令消息后，发送确认窗口大小(Window Acknowledgement Size)协议消息到客户端，同时连接到连接命令中提到的应用程序。

c) 服务器发送设置带宽()协议消息到客户端。

d) 客户端处理设置带宽协议消息后，发送确认窗口大小(Window Acknowledgement Size)协议消息到服务器端。

e) 服务器发送用户控制消息中的“流开始”(Stream Begin)消息到客户端。

f) 服务器发送命令消息中的“结果”(_result)，通知客户端连接的状态。

![img](https://pic2.zhimg.com/80/v2-e187603466694e275144cfb3a0b41ffd_720w.webp)

4建立网络流（NetStream）

a) 客户端发送命令消息中的“创建流”（createStream）命令到服务器端。

b) 服务器端接收到“创建流”命令后，发送命令消息中的“结果”(_result)，通知客户端流的状态。

![img](https://pic3.zhimg.com/80/v2-62afea6f5ec5bee2f615c2fca03ab11a_720w.webp)

5 播放（Play）

a) 客户端发送命令消息中的“播放”（play）命令到服务器。

b) 接收到播放命令后，服务器发送设置块大小（ChunkSize）协议消息。

c) 服务器发送用户控制消息中的“streambegin”，告知客户端流ID。

d) 播放命令成功的话，服务器发送命令消息中的“响应状态” NetStream.Play.Start & NetStream.Play.reset，告知客户端“播放”命令执行成功。

e) 在此之后服务器发送客户端要播放的音频和视频数据。

![img](https://pic2.zhimg.com/80/v2-8a810b0cf89ade864c040fac0ce444a1_720w.webp)

## 3.RTMP协议解析

### 3.1 消息

消息是RTMP协议中基本的数据单元。不同种类的消息包含不同的Message Type ID，代表不同的功能。RTMP协议中一共规定了十多种消息类型，分别发挥着不同的作用。例如，Message Type ID在1-7的消息用于协议控制，这些消息一般是RTMP协议自身管理要使用的消息，用户一般情况下无需操作其中的数据。Message Type ID为8，9的消息分别用于传输音频和视频数据。Message Type ID为15-20的消息用于发送AMF编码的命令，负责用户与服务器之间的交互，比如播放，暂停等等。消息首部（Message Header）有四部分组成：标志消息类型的Message Type ID，标志消息长度的Payload Length，标识时间戳的Timestamp，标识消息所属媒体流的Stream ID。消息的报文结构如图3所示。

![img](https://pic2.zhimg.com/80/v2-a3fa1e48e5c7fa4cceda182b94cde491_720w.webp)

消息

### 3.2 消息块

在网络上传输数据时，消息需要被拆分成较小的数据块，才适合在相应的网络环境上传输。RTMP协议中规定，消息在网络上传输时被拆分成消息块（Chunk）。消息块首部（Chunk Header）有三部分组成：用于标识本块的Chunk Basic Header，用于标识本块负载所属消息的Chunk Message Header，以及当时间戳溢出时才出现的Extended Timestamp。消息块的报文结构如图4所示。

![img](https://pic1.zhimg.com/80/v2-c6c72b4fa6186f83d4cb87ef87b50248_720w.webp)

消息块

### 3.3 消息分块

在消息被分割成几个消息块的过程中，消息负载部分（Message Body）被分割成大小固定的数据块（默认是128字节，最后一个数据块可以小于该固定长度），并在其首部加上消息块首部（Chunk Header），就组成了相应的消息块。消息分块过程如图5所示，一个大小为307字节的消息被分割成128字节的消息块（除了最后一个）。

![img](https://pic4.zhimg.com/80/v2-54b6e7abb5c24c3706c02f0ff3305fd7_720w.webp)

RTMP分块

RTMP传输媒体数据的过程中，发送端首先把媒体数据封装成消息，然后把消息分割成消息块，最后将分割后的消息块通过TCP协议发送出去。接收端在通过TCP协议收到数据后，首先把消息块重新组合成消息，然后通过对消息进行解封装处理就可以恢复出媒体数据。

RTMPDump源码分析

握手(HandsShake)

static int HandShake(RTMP * r, int FP9HandShake);

HandShake函数在：/rtmp/rtmplib/handshack.h中。
./rtmp.c:69:#define RTMP_SIG_SIZE 1536

```text
/*client HandShake*/
 695 static int HandShake(RTMP * r, int FP9HandShake){
 709 uint8_t clientbuf[RTMP_SIG_SIZE + 4], *clientsig=clientbuf+4;
 
/*C0 字段已经写入clientsig*/
 721 if (encrypted){  
 722      clientsig[-1] = 0x06; /* 0x08 is RTMPE as well */  
 723      offalg = 1;  
 724 }else  
    //0x03代表RTMP协议的版本（客户端要求的）  
    //数组竟然能有“-1”下标,因为clientsig指向的是clientbuf+4,所以不存在非法地址
    //C0中的字段(1B)  
 725   clientsig[-1] = 0x03;
 /*准备C1字段过程略去，C1字段的数据写入clientsig中, clientsig的大小为1536个字节*/
 
/*1st part of shakehand .......*/
/*C ------- S*/
/*c0 C1-->   */
/*  <-- S0 S1*/
/*C2 -->     */
/*send clientsig C0 和 C1一起发送*/
 814   if (!WriteN(r, (char *)clientsig-1, RTMP_SIG_SIZE + 1))
 815     return FALSE;
 
/*get server response->read type, if get response type not match handshake failed*/
 817   if (ReadN(r, (char *)&type, 1) != 1)  /* 0x03 or 0x06 */
 818     return FALSE; /*encrypt type = 0x06*/
 
/*get server response->read serversig*/
 826 if (ReadN(r, (char *)serversig, RTMP_SIG_SIZE) != RTMP_SIG_SIZE)
 827     return FALSE;
 
/*如果是加密协议，则需要校验收到的serversig是否和发送的匹配，如果没有加密则直接发送收到的serversig*/
 968   if (!WriteN(r, (char *)reply, RTMP_SIG_SIZE))
 969     return FALSE;
 
/*2nd part of shakehand .....*/
/*C ----- S*/
/*   <-- S2*/
 972   if (ReadN(r, (char *)serversig, RTMP_SIG_SIZE) != RTMP_SIG_SIZE)
 973     return FALSE;
/* compare info between serversig and clientsig*/
 1060  if (memcmp(serversig, clientsig, RTMP_SIG_SIZE) != 0)
/*如果相等，则握手成功*/
}
```

建立链接（NetConnnet）

RTMP_Connect(RTMP *r, RTMPPacket *cp);

建立连接的代码位于：librtmp/rtmp.c中，定义函数：RTMP_Connect()。RTMP_Conncet()里面又分别调用了两个函数：RTMP_Connect0(), RTMP_Connect1()。RTMP_Connect0()主要进行的是socket的连接，RTMP_Connct1()进行的是RTMP相关的连接动作。

```text
1031 int RTMP_Connect(RTMP *r, RTMPPacket *cp)
1032 {
1033   struct sockaddr_in service;
1034   if (!r->Link.hostname.av_len)
1035     return FALSE;
1036 
1037   memset(&service, 0, sizeof(struct sockaddr_in));
1038   service.sin_family = AF_INET;
1039 
1040   if (r->Link.socksport)
1041     {
1042       /* Connect via SOCKS */
1043       if (!add_addr_info(&service, &r->Link.sockshost, r->Link.socksport))
1044   return FALSE;
1045     }
1046   else
1047     {
1048       /* Connect directly */
1049       if (!add_addr_info(&service, &r->Link.hostname, r->Link.port))
1050   return FALSE;
1051     }
1052 
1053   if (!RTMP_Connect0(r, (struct sockaddr *)&service))
1054     return FALSE;
1055 
1056   r->m_bSendCounter = TRUE;
1057 
1058   return RTMP_Connect1(r, cp);
1059 }
```

int RTMP_Connect0(RTMP *r, struct sockaddr* service);

RTMP_Connect0函数分析：

```text
905 int RTMP_Connect0(RTMP *r, struct sockaddr * service){
     /*创建socket*/
 913   r->m_sb.sb_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
     /*通过socket连接到服务器地址*/
 916   if (connect(r->m_sb.sb_socket, service, sizeof(struct sockaddr)) < 0)
     /*如果指定了socket端口到，则进行socks Negotiate*/
 928     if (!SocksNegotiate(r)){}
     /*连接成功之后，返回TRUE*/
 956   return TRUE;
    }
```

int RTMP_Connect1(RTMP *r, RTMPPacket *cp);

RTMP_Connect1函数分析：
根据不同的传输协议，选择传送数据的方式。之后进行HandShake，最后调用SendConnectPacket()送Connect packet

```text
int
RTMP_Connect1(RTMP *r, RTMPPacket *cp)
{
/*if crypto use tls_conncet*/
  if (r->Link.protocol & RTMP_FEATURE_SSL){
#if defined(CRYPTO) && !defined(NO_SSL)
      TLS_client(RTMP_TLS_ctx, r->m_sb.sb_ssl);
      TLS_setfd(r->m_sb.sb_ssl, r->m_sb.sb_socket);
      if (TLS_connect(r->m_sb.sb_ssl) < 0){...}
#else
      return FALSE;
#endif
    }
  /*if no crypto, use http post*/  
  if (r->Link.protocol & RTMP_FEATURE_HTTP){
      HTTP_Post(r, RTMPT_OPEN, "", 1);
      if (HTTP_read(r, 1) != 0){...}
        ...
    }
    /*进行HandShake*/
  if (!HandShake(r, TRUE)){...}
    /*握手成功之后，发送Connect Packet*/
  if (!SendConnectPacket(r, cp)){...}
  return TRUE;
}
```

SendConnectPacket() 里面主要对RTMP信息进行打包，然后调用RTMP_SendPacket函数，将内容发送出去。

```text
static int
SendConnectPacket(RTMP *r, RTMPPacket *cp)
{
  RTMPPacket packet;
  char pbuf[4096], *pend = pbuf + sizeof(pbuf);
  char *enc;
 
  if (cp)
    return RTMP_SendPacket(r, cp, TRUE);
 
  packet.m_nChannel = 0x03; /* control channel (invoke) */
  packet.m_headerType = RTMP_PACKET_SIZE_LARGE;
  packet.m_packetType = RTMP_PACKET_TYPE_INVOKE;
  packet.m_nTimeStamp = 0;
  packet.m_nInfoField2 = 0;
  packet.m_hasAbsTimestamp = 0;
  packet.m_body = pbuf + RTMP_MAX_HEADER_SIZE;
 
  enc = packet.m_body;
  enc = AMF_EncodeString(enc, pend, &av_connect);
  enc = AMF_EncodeNumber(enc, pend, ++r->m_numInvokes);
  *enc++ = AMF_OBJECT;
 
  /*encrypto 部分省略 主要就是调用AMF函数进行*/
  ...
 
  packet.m_nBodySize = enc - packet.m_body;
 
  return RTMP_SendPacket(r, &packet, TRUE);
}
```

建立流（NetStream）

RTMP_ConnectStream()函数主要用于在NetConnection基础上面建立一个NetStream。

int RTMP_ConnectStream(RTMP *r, int seekTime);

```text
int RTMP_ConnectStream(RTMP *r, int seekTime)
{
  RTMPPacket packet = { 0 };
  /* seekTime was already set by SetupStream / SetupURL.
   * This is only needed by ReconnectStream.
   */
  if (seekTime > 0)
    r->Link.seekTime = seekTime;
 
  r->m_mediaChannel = 0;
 
  // 接收到的实际上是块(Chunk)，而不是消息(Message)，因为消息在网上传输的时候要分割成块.
  while (!r->m_bPlaying && RTMP_IsConnected(r) && RTMP_ReadPacket(r, &packet)){
      // 一个消息可能被封装成多个块(Chunk)，只有当所有块读取完才处理这个消息包
      if (RTMPPacket_IsReady(&packet)){
        if (!packet.m_nBodySize)
          continue;
        // 读取到flv数据包，则继续读取下一个包
        if ((packet.m_packetType == RTMP_PACKET_TYPE_AUDIO) ||
            (packet.m_packetType == RTMP_PACKET_TYPE_VIDEO) ||
            (packet.m_packetType == RTMP_PACKET_TYPE_INFO)){
            RTMP_Log(RTMP_LOGWARNING, "Received FLV packet before play()! Ignoring.");
            RTMPPacket_Free(&packet);
            continue;
          }
        RTMP_ClientPacket(r, &packet);// 处理收到的数据包
        RTMPPacket_Free(&packet);// 处理完毕，清除数据
      }
    }
  return r->m_bPlaying;
}
```

简单的一个逻辑判断，重点在while循环里。首先，必须要满足三个条件。其次，进入循环以后只有出错或者建立流（NetStream）完成后，才能退出循环。
有两个重要的函数：

int RTMP_ReadPacket(RTMP *r, RTMPPacket *packet);

块格式：

basic header(1-3字节） chunk msg header(0/3/7/11字节) Extended Timestamp(0/4字节) chunk data
消息格式：

timestamp(3字节) msg length(3字节) msg type id(1字节，小端) msg stream id(4字节)

```text
/**
 * @brief 读取接收到的消息块(Chunk)，存放在packet中. 对接收到的消息不做任何处理。 块的格式为：
 *
 *   | basic header(1-3字节）| chunk msg header(0/3/7/11字节) | Extended Timestamp(0/4字节) | chunk data |
 *
 *   其中 basic header还可以分解为：| fmt(2位) | cs id (3 <= id <= 65599) |
 *   RTMP协议支持65597种流，ID从3-65599。ID 0、1、2作为保留。
 *      id = 0，表示ID的范围是64-319（第二个字节 + 64）；
 *      id = 1，表示ID范围是64-65599（第三个字节*256 + 第二个字节 + 64）；
 *      id = 2，表示低层协议消息。
 *   没有其他的字节来表示流ID。3 -- 63表示完整的流ID。
 *
 *    一个完整的chunk msg header 还可以分解为 ：
 *     | timestamp(3字节) | msg length(3字节) | msg type id(1字节，小端) | msg stream id(4字节) |
 */
int  
RTMP_ReadPacket(RTMP *r, RTMPPacket *packet)  
{
  uint8_t hbuf[RTMP_MAX_HEADER_SIZE] = { 0 };
  // Chunk Header长度最大值为3 + 11 + 4 = 18
  char *header = (char *)hbuf;
  // header指向从socket接收到的数据
  int   nSize, hSize, nToRead, nChunk;
  // nSize是块消息头长度，hSize是块头长度
  int   didAlloc = FALSE;
 
  RTMP_Log(RTMP_LOGDEBUG2, "%s: fd=%d", __FUNCTION__, r->m_sb.sb_socket);
 
  // 读取1个字节存入 hbuf[0]
  if (ReadN(r, (char *)hbuf, 1) == 0)
  {
    RTMP_Log(RTMP_LOGERROR, "%s, failed to read RTMP packet header", __FUNCTION__);
    return FALSE;
  }
 
  packet->m_headerType = (hbuf[0] & 0xc0) >> 6;
  // 块类型fmt
  packet->m_nChannel    = (hbuf[0] & 0x3f);
  // 块流ID（2 - 63）
  header++;
 
  // 块流ID第一个字节为0，表示块流ID占2个字节，表示ID的范围是64-319（第二个字节 + 64）
  if (packet->m_nChannel == 0)
  {
    // 读取接下来的1个字节存放在hbuf[1]中
    if (ReadN(r, (char *)&hbuf[1], 1) != 1)
    {
      RTMP_Log(RTMP_LOGERROR, "%s, failed to read RTMP packet header 2nd byte", __FUNCTION__);
      return FALSE;
    }
 
    // 块流ID = 第二个字节 + 64 = hbuf[1] + 64
    packet->m_nChannel = hbuf[1];
    packet->m_nChannel += 64;
    header++;
  }
  // 块流ID第一个字节为1，表示块流ID占3个字节，表示ID范围是64 -- 65599（第三个字节*256 + 第二个字节 + 64）
  else if (packet->m_nChannel == 1){
    int tmp;
    // 读取2个字节存放在hbuf[1]和hbuf[2]中
    if (ReadN(r, (char *)&hbuf[1], 2) != 2)
    {
      RTMP_Log(RTMP_LOGERROR, "%s, failed to read RTMP packet header 3nd byte", __FUNCTION__);
      return FALSE;
    }
 
    // 块流ID = 第三个字节*256 + 第二个字节 + 64
    tmp = (hbuf[2] << 8) + hbuf[1];
    packet->m_nChannel = tmp + 64;
    RTMP_Log(RTMP_LOGDEBUG, "%s, m_nChannel: %0x", __FUNCTION__, packet->m_nChannel);
    header += 2;
  }
 
  // 块消息头(ChunkMsgHeader)有四种类型，大小分别为11、7、3、0,每个值加1 就得到该数组的值
  // 块头 = BasicHeader(1-3字节) + ChunkMsgHeader + ExtendTimestamp(0或4字节)
  nSize = packetSize[packet->m_headerType];
 
  // 块类型fmt为0的块，在一个块流的开始和时间戳返回的时候必须有这种块
  // 块类型fmt为1、2、3的块使用与先前块相同的数据
  // 关于块类型的定义，可参考官方协议：流的分块 --- 6.1.2节
  if (nSize == RTMP_LARGE_HEADER_SIZE)
  /* if we get a full header the timestamp is absolute */
  {
    packet->m_hasAbsTimestamp = TRUE;
    // 11个字节的完整ChunkMsgHeader的TimeStamp是绝对时间戳
  }else if (nSize < RTMP_LARGE_HEADER_SIZE){
    /* using values from the last message of this channel */
    if (r->m_vecChannelsIn[packet->m_nChannel])
    memcpy(packet, r->m_vecChannelsIn[packet->m_nChannel], sizeof(RTMPPacket));
  }
 
  nSize--;
  // 真实的ChunkMsgHeader的大小，此处减1是因为前面获取包类型的时候多加了1
 
  // 读取nSize个字节存入header
  if (nSize > 0 && ReadN(r, header, nSize) != nSize){
    RTMP_Log(RTMP_LOGERROR, "%s, failed to read RTMP packet header. type: %x", 
     __FUNCTION__, (unsigned int)hbuf[0]);
    return FALSE;
  }
 
  // 目前已经读取的字节数 = chunk msg header + basic header
  hSize = nSize + (header - (char *)hbuf);
  // chunk msg header为11、7、3字节，fmt类型值为0、1、2
  if (nSize >= 3){
    // 首部前3个字节为timestamp
    packet->m_nTimeStamp = AMF_DecodeInt24(header);
 
    /* RTMP_Log(RTMP_LOGDEBUG, "%s, reading RTMP packet chunk on channel %x, 
    headersz %i, timestamp %i, abs timestamp %i", __FUNCTION__, 
    packet.m_nChannel, nSize, packet.m_nTimeStamp, packet.m_hasAbsTimestamp); */
 
    // chunk msg header为11或7字节，fmt类型值为0或1
    if (nSize >= 6)
    {
      packet->m_nBodySize = AMF_DecodeInt24(header + 3);
      packet->m_nBytesRead = 0;
      RTMPPacket_Free(packet);
 
      if (nSize > 6)
      {
        packet->m_packetType = header[6];
        // msg type id
        if (nSize == 11)
          packet->m_nInfoField2 = DecodeInt32LE(header + 7); // msg stream id，小端字节序
      }
    }
 
  // Extend Tiemstamp,占4个字节
    if (packet->m_nTimeStamp == 0xffffff){
      if (ReadN(r, header + nSize, 4) != 4)
      {
        RTMP_Log(RTMP_LOGERROR, "%s, failed to read extended timestamp", __FUNCTION__);
        return FALSE;
      }
      packet->m_nTimeStamp = AMF_DecodeInt32(header + nSize);
      hSize += 4;
    }
  }
 
  RTMP_LogHexString(RTMP_LOGDEBUG2, (uint8_t *)hbuf, hSize);
 
  // 如果消息长度非0，且消息数据缓冲区为空，则为之申请空间
  if (packet->m_nBodySize > 0 && packet->m_body == NULL){
    if (!RTMPPacket_Alloc(packet, packet->m_nBodySize)){
      RTMP_Log(RTMP_LOGDEBUG, "%s, failed to allocate packet", __FUNCTION__);
      return FALSE;
    }
    didAlloc = TRUE;
    packet->m_headerType = (hbuf[0] & 0xc0) >> 6;
  }
 
  // 剩下的消息数据长度如果比块尺寸大，则需要分块,否则块尺寸就等于剩下的消息数据长度
  nToRead = packet->m_nBodySize - packet->m_nBytesRead;
  nChunk = r->m_inChunkSize;
  if (nToRead < nChunk)
  nChunk = nToRead;
 
  /* Does the caller want the raw chunk? */
  if (packet->m_chunk){
    packet->m_chunk->c_headerSize = hSize;
    // 块头大小
    memcpy(packet->m_chunk->c_header, hbuf, hSize);
    // 填充块头数据
    packet->m_chunk->c_chunk = packet->m_body + packet->m_nBytesRead;
    // 块消息数据缓冲区指针
    packet->m_chunk->c_chunkSize = nChunk;
    // 块大小
  }
 
  // 读取一个块大小的数据存入块消息数据缓冲区
  if (ReadN(r, packet->m_body + packet->m_nBytesRead, nChunk) != nChunk){
    RTMP_Log(RTMP_LOGERROR, "%s, failed to read RTMP packet body. len: %u",
     __FUNCTION__, packet->m_nBodySize);
    return FALSE;
  }
 
  RTMP_LogHexString(RTMP_LOGDEBUG2, (uint8_t *)packet->m_body + packet->m_nBytesRead, nChunk);
 
  // 更新已读数据字节个数
  packet->m_nBytesRead += nChunk;
 
  /* keep the packet as ref for other packets on this channel */
  // 将这个包作为通道中其他包的参考
  if (!r->m_vecChannelsIn[packet->m_nChannel])
   r->m_vecChannelsIn[packet->m_nChannel] = malloc(sizeof(RTMPPacket));
  memcpy(r->m_vecChannelsIn[packet->m_nChannel], packet, sizeof(RTMPPacket));
 
  // 包读取完毕
  if (RTMPPacket_IsReady(packet)){
    /* make packet's timestamp absolute，绝对时间戳 = 上一次绝对时间戳 + 时间戳增量 */
    if (!packet->m_hasAbsTimestamp)
      /* timestamps seem to be always relative!! */
      packet->m_nTimeStamp += r->m_channelTimestamp[packet->m_nChannel];
 
    // 当前绝对时间戳保存起来，供下一个包转换时间戳使用
    r->m_channelTimestamp[packet->m_nChannel] = packet->m_nTimeStamp;
 
    /* reset the data from the stored packet.  we keep the header since we may use it later if 
                       a new packet for this channel arrives and requests to re-use some info (small packet header) */
    // 重置保存的包。保留块头数据，因为通道中新到来的包(更短的块头)可能需要使用前面块头的信息.
    r->m_vecChannelsIn[packet->m_nChannel]->m_body = NULL;
    r->m_vecChannelsIn[packet->m_nChannel]->m_nBytesRead = 0;
    r->m_vecChannelsIn[packet->m_nChannel]->m_hasAbsTimestamp = FALSE; // can only be false if we reuse header
  }
  else{
  packet->m_body = NULL;
  /* so it won't be erased on free */
  }
 
  return TRUE;
}
```

int ReadN(RTMP *r, char *buffer, int n);

```text
/**
 * @brief 从HTTP或SOCKET中读取n个数据存放在buffer中.
 */
static int ReadN(RTMP *r, char *buffer, int n)
{
    int  nOriginalSize = n;
    int  avail;
    char *ptr;
 
    r->m_sb.sb_timedout = FALSE;
    #ifdef _DEBUG
    memset(buffer, 0, n);
    #endif
 
    ptr = buffer;
    while (n > 0){
    int nBytes = 0, nRead;
    if (r->Link.protocol & RTMP_FEATURE_HTTP)
                   {
    while (!r->m_resplen)
    {
        if (r->m_sb.sb_size < 144)
        {
            if (!r->m_unackd)
            HTTP_Post(r, RTMPT_IDLE, "", 1);
            if (RTMPSockBuf_Fill(r, &r->m_sb) < 1){
                if (!r->m_sb.sb_timedout)
                RTMP_Close(r);
                return 0;
            }
        }
 
        if (HTTP_read(r, 0) == -1){
            RTMP_Log(RTMP_LOGDEBUG, "%s, No valid HTTP response found", __FUNCTION__);
            RTMP_Close(r);
            return 0;
        }
    }
 
    if (r->m_resplen && !r->m_sb.sb_size)
    RTMPSockBuf_Fill(r, &r->m_sb);
 
    avail = r->m_sb.sb_size;
    if (avail > r->m_resplen)
        avail = r->m_resplen;
    }else{
        avail = r->m_sb.sb_size;
        if (avail == 0){
            if (RTMPSockBuf_Fill(r, &r->m_sb) < 1){
                if (!r->m_sb.sb_timedout)
                    RTMP_Close(r);
                return 0;
            }
            avail = r->m_sb.sb_size;
        }
    }
 
    nRead = ((n < avail) ? n : avail);
    if (nRead > 0){
        memcpy(ptr, r->m_sb.sb_start, nRead);
        r->m_sb.sb_start += nRead;
        r->m_sb.sb_size -= nRead;
        nBytes = nRead;
        r->m_nBytesIn += nRead;
        if (r->m_bSendCounter && r->m_nBytesIn > ( r->m_nBytesInSent + r->m_nClientBW / 10))
        if (!SendBytesReceived(r))
        return FALSE;
    }
    /*RTMP_Log(RTMP_LOGDEBUG, "%s: %d bytes\n", __FUNCTION__, nBytes); */
    //#ifdef _DEBUG
    //      fwrite(ptr, 1, nBytes, netstackdump_read);
    //#endif
 
    if (nBytes == 0){
        RTMP_Log(RTMP_LOGDEBUG, "%s, RTMP socket closed by peer", __FUNCTION__);
        /*goto again; */
        RTMP_Close(r);
        break;
    }
 
    if (r->Link.protocol & RTMP_FEATURE_HTTP){
        r->m_resplen -= nBytes;
        n -= nBytes;
        ptr += nBytes;
    }
 
    return nOriginalSize - n;
}
```

int RTMPSockBuf_Fill(RTMP *r, RTMPSockBuf *sb);

```text
/**
 * @brief 调用Socket编程中的recv()函数，接收数据
 */
int RTMPSockBuf_Fill(RTMP *r, RTMPSockBuf *sb)
{
    int nBytes;
    if  (!sb->sb_size)
        sb->sb_start = sb->sb_buf;
 
    while (1)
    {
        // 缓冲区长度：总长-未处理字节-已处理字节  
        // |-----已处理--------|-----未处理--------|---------缓冲区----------|  
        // sb_buf        sb_start    sb_size  
        nBytes = sizeof(sb->sb_buf) - sb->sb_size - (sb->sb_start - sb->sb_buf);
        {
            // int recv( SOCKET s, char * buf, int len, int flags);  
            // s    ：一个标识已连接套接口的描述字。  
            // buf  ：用于接收数据的缓冲区。   
            // len  ：缓冲区长度。  
            // flags：指定调用方式。  
            // 从sb_start（待处理的下一字节） + sb_size（）还未处理的字节开始buffer为空，可以存储
            nBytes = r->m_sock.recv(sb->sb_socket, sb->sb_start + sb->sb_size, nBytes, 0);
        }
 
        if (nBytes != -1){
            // 未处理的字节又多了
            sb->sb_size += nBytes;
        }else{
            int sockerr = r->m_sock.getsockerr();
            RTMP_Log(RTMP_LOGDEBUG, "%s, recv returned %d. GetSockError(): %d (%s)", 
            __FUNCTION__, nBytes, sockerr, strerror(sockerr));
            if (sockerr == EINTR && !RTMP_ctrlC)
                continue;
 
            if (sockerr == EWOULDBLOCK || sockerr == EAGAIN){
                sb->sb_timedout = TRUE;
                nBytes = 0;
            }
        }
        break;
    }
    return nBytes;
}
```

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/456832095