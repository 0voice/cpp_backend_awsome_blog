# 【NO.391】谷歌开源、高性能RPC框架：gRPC 使用体验

> 在广告系统实践中，精排服务基于 gRPC 协议调用 TF-Serving 在线推理服务。相信很多业务已经使用过 gRPC 相关语言的框架进行服务调用，尤其是基于谷歌云的出海业务的服务调用更绕不开 gRPC，所以很有必要理解 gRPC 的原理。本文通过简要介绍抓包分析一次 gRPC 的调用过程，逐步认识 gRPC。

### 1.**概述**

gRPC 是谷歌推出的一个开源、高性能的 RPC 框架。默认情况下使用 protoBuf 进行序列化和反序列化，并基于 HTTP/2 传输报文，带来诸如多请求复用一个 TCP 连接(所谓的多路复用)、双向流、流控、头部压缩等特性。gRPC 目前提供 C、Go 和 JAVA 等语言版本，对应 gRPC、gRPC-Go 和 gRPC-JAVA 等开发框架。

在 gRPC 中，开发者可以像调用本地方法一样，通过 gRPC 的客户端调用远程机器上 gRPC 服务的方法，gRPC 客户端封装了 HTTP/2 协议数据帧的打包、以及网络层的通信细节，把复杂留给框架自己，把便捷提供给用户。gRPC 基于这样的一个设计理念：定义一个服务，及其被远程调用的方法(方法名称、入参、出参)。在 gRPC 服务端实现这个方法的业务逻辑，并在 gRPC 服务端处理来着远程客户端对这个 RPC 方法的调用。在 gRPC 客户端也拥有这个 RPC 方法的存根(stub)。gRPC 的客户端和服务端都可以用任何支持 gRPC 的语言来实现，例如一个 gRPC 服务端可以是 C++语言编写的，以供 Ruby 语言的 gRPC 客户端和 JAVA 语言的 gRPC 客户端调用，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtOhjEJJB8LfMW2KvDRgvTdfYVhTf2HFaB2FemtsfsymPZdLNGt0wj7g/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

gRPC 默认使用 ProtoBuf 对请求/响应进行序列化和反序列化，这使得传输的请求体和响应体比 JSON 等序列化方式包体更小、更轻量。

gRPC 基于 HTTP/2 协议传输报文，HTTP/2 具有多路复用、头部压缩等特性，基于 HTTP/2 的帧设计，实现了多个请求复用一个 TCP 连接，基本解决了 HTTP/1.1 的队头阻塞问题，相对 HTTP/1.1 带来了巨大的性能提升。下面对 HTTP/2 进行简介。



### 2.**HTTP/2 简介**

HTTP 是一个成功的应用层协议。但是由于 HTTP 的队头阻塞等特性导致基于 HTTP 的应用程序性能有较大影响。队头阻塞是指顺序请求的一个请求必须处理完才能处理后续的其他请求，当一个请求被阻塞时会给应用程序带来延迟。虽然 HTTP/1.1 提供了流水线(request pipeline)的请求操作，但是由于受到 HTTP 自身协议的限制，无法消除 HTTP 的队头阻塞带来的延迟。为了减少延迟，需要 HTTP 的客户端与服务器建立多个连接实现并发处理请求，降低延迟。然而，在高并发情况下，大量的网络连接可能耗尽系统资源，可以使用连接池模式只维持固定的连接数可以防止服务的资源耗尽。连接池连接数的设置在对性能要求极高的应用程序也是一个挑战，需要根据实际机器配置的压测情况确定。

另外，HTTP 头字段重复且冗长，导致网络传输不必要的冗余报文，以及初始 TCP 拥塞窗口很快被填满。一个 TCP 连接处理大量请求是会导致较大的延迟。

HTTP/2 通过优化 HTTP 的报文定义，允许同一个网络连接上并发交错的处理请求和响应，并通过减少 HTTP 头字段的重复传输、压缩 HTTP 头，提高了处理性能。

HTTP 每次网络传输会携带通信的资源、浏览器属性等大量冗余头信息，为了减少这些重复传输的开销，HTTP/2 会压缩这些头部字段：

1. 基于 HTTP/2 协议的客户端和服务器使用"头部表"来跟踪与存储发送的键值对，对于相同的键值对数据，不用每次请求和响应都发送；
2. 头部表在 HTTP/2 的连接有效期内一直存在，由客户端和服务器共同维护更新；
3. 每个新的 HTTP 头键值对要么追加，要么替换头部表原来的值。

举个例子，有两个请求，在 HTTP/1.x 中，请求 1 和请求 2 都要发送全部的头数据；在 HTTP/2 中，请求 1 发送全部的头数据，请求 2 仅仅发送变更的头数据，这样就可以减少冗余的数据，降低网络开销。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtAs9kyiblXlAUlKATmEKVpA5Rd4rB6MxrojfD2mGcuba2nibpyWKIyk2g/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

这里再举个例子说明 HTTP/1.x 和 HTTP/2 处理请求的差异，浏览器打开网络要请求/index.html、styles.css 和/scripts.js 三个文件，基于 HTTP/1.x 建立的连接只能先请求/index.html,得到响应后再请求 styles.css，得到响应后再获取/scripts.js。而基于 HTTP/2 一个网络连接在获取/index.html 后,可以同时获取 styles.css 和/scripts.js 文件，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt4mia8JYicHXib9X5467J2nvR51Em6scUKeXVJPHOc6Ufriau1DrOibfDoAQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

HTTP/2 对服务资源和网络更友好，相对与 HTTP/1.x，处理同样量级的请求，HTTP/2 的需要建立的 TCP 连接数更少。这主要得益于 HTTP/2 使用二进制数据帧来传输数据，使得一个 TCP 连接可以同时处理多个请求而不用等待一个请求处理完成再处理下一个。从而充分发掘了 TCP 的并发能力。

#### 2.1 HTTP/2 帧

在 HTTP/2 中，帧是网络通信的基本单位，HTTP/2 主要定义了 10 种不同的帧类型，每种帧类型在建立和管理连接或者单个 stream 流有不同的作用。不同的帧类型都有公共字段：Length(3 字节),Type(1 字节), Flags(1 字节), Stream Identifier(4 字节) 和 Frame Payload(变长)。

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtmEGvRG5XO9PyYkvGK7vaOADM8FsMP1sXiczpx5k7dYdCzPxW8F7PkLA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

HTTP/2 帧都以固定的 9 字节大小作为帧头，后面跟着变长的包体 Paylload。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtrLiajgfZaCq5M6Mric5aGHjOs5TWiaGyBibgkCHd9vTA9zOGmE6YxWSREQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

帧头字段说明：

1. **Length** 帧的数据(Frame Payload)长度，*不包括帧头长度*，3 个字节(24bit), 帧最大长度为 1<<24 - 1(16383);
2. **Type** 帧类型，1 个字节(8bit), 目前 HTTP/2 定义了 10 中帧类型，常见的帧类型有 DATA 帧、HEADERs 帧、SETTINGS 帧等，10 种帧类型如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtQAbUOlOB3uogsQbZy8fepYc24qpHebQfiaFQowbazS9kz6q1ic8YYMuA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

1. **Flags** 帧标志，1 个字节(8bit)，没有特定帧类型的帧标志应该被忽略，在发送时帧标志需要保持未设置(0x0).常见的标志位有 END_HEADERS 表示 HTTP/2 数据头结束，相当于 HTTP 头后的空行（“\r\n”），END_STREAM 表示单方向数据发送结束（即 EOS，End of Stream），相当于 HTTP/1.x 里 Chunked 分块结束标志（“0\r\n\r\n”）；
2. **R** 保留字段 1bit,发送时保持未设置(0x0),接收时忽略；
3. **Stream Identifier** 流标识符，31bit. 一个无符号整数。由客户端发起的 Stream 数据流用奇数编号 ID 的流标识符；由服务器发起的数据流使用偶数编号 ID 的流标识符。流标识符零(0x0)用于连接控制消息；零流标识符不能用于建立新的 stream 流。

![图片](data:image/svg+xml,<%3Fxml version='1.0' encoding='UTF-8'%3F><svg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'><title></title><g stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'><g transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'><rect x='249' y='126' width='1' height='1'></rect></g></g></svg>)

#### 2.2 HTTP/2 请求模型

HTTP/2 的请求模型如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtr1VGARFDS481wj1eZEDpzqzkiacA1WAMOmnhjCstlgPWshChGJRj5ZA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**Connection 连接**：对应一个 TCP 连接，可以承载一个或者多个 Stream。

**Stream 流**：对应一个双向通信的数据流，可以承载一个或者多个 Message。每个数据流都有一个唯一的流标识符和可选的优先级信息，用于承载双向消息。

Stream 流有几个重要特性：

1. 单个 HTTP/2 连接可以承载多个并发的 stream 流，通信双方都可能交叉地收到多个 stream 流的数据帧；
2. stream 流可以单方面建立与使用，也可以由客户端和服务器双方共享消息通道;
3. 客户端或者服务器都可以关闭 stream 流;
4. 发送方在 stream 流按顺序发送数据帧，接收到按照顺序接收数据帧。特别地，HEADS 帧和 DATA 帧的顺序在语言上是较为重要的；
5. stream 流由无符号整数标识。stream 流标识符是由发起流的端点分配给 stream 流的。

**Message 消息**：对应 HTTP/1.x 的请求 Request 或响应 response.包含一个或者多个 Frame 数据帧。

**Frame 数据帧：**HTTP/2 网络通信的基本单位，承载的是压缩和编码后的二进制流，不同 Stream 数据流的帧可以交错发送，并根据帧头的流 ID(数据流标识符)进行区分和组装。

关于 HTTP/2 主要介绍这些，更多参考：https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md



### 3.**gRPC 协议**

前面对 HTTP/2 帧作了简要说明，这节开始介绍 gRPC 协议，gRPC 基于 HTTP/2/协议进行通信，使用 ProtoBuf 组织二进制数据流，gRPC 的协议格式如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtXibqTP5VmmxZ4bqToibJ5HeNPl4OtC4Ez4DuQg8Im0AeBb904tqnBaPg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从以上图可知，gRPC 协议在 HTTP 协议的基础上，对 HTTP/2 的帧的有效包体(Frame Payload)做了进一步编码：gRPC 包头(5 字节)+gRPC 变长包头，其中：

1. 5 字节的 gRPC 包头由：1 字节的压缩标志(compress flag)和 4 字节的 gRPC 包头长度组成；
2. gRPC 包体长度是变长的，其是这样的一串二进制流：使用指定序列化方式(通常是 ProtoBuf)序列化成字节流，再使用指定的压缩算法对序列化的字节流压缩而成的。如果对序列化字节流进行了压缩，gRPC 包头的压缩标志为 1。
3. 对比 tRPC 协议可知，gRPC 的帧头和包头比 tRPC 协议帧头和包头要小，当然 HTTP/2 的帧类型更复杂一些。tRPC 协议帧定义如下图：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtHpWBEicBCf7rBEg3L7VHcTIEUL4Yicia3MQoCOibkZpjMWO0g6hfUjoA4Q/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.**gRPC 调用抓包分析**

下面基于官方提供的 gRPC-Go helloword 例子，使用 Wireshark 分析通过 tcpdump 抓包 gRPC 调用的报文，加深对 gRPC 协议的理解。

#### 4.1 抓包准备

1. 下载 Wireshark 抓包工具，下载地址：https://www.wireshark.org/；
2. 安装 Go 环境；
3. 安装 protoc-gen-go: go get -u github.com/golang/protobuf/protoc-gen-go；
4. 下载 g[rpc-go/examples/helloworld](https://github.com/grpc/grpc-go/tree/master/examples/helloworld) gRPC-Go 的 helloword Go 工程。

#### 4.2 抓包

1. 运行 helloword 的服务端 greeter_server：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtK2XFnQKgOUjtF2EicsVctCLYv2MavEG0Xtt5qbssJic4zfC1oB55JNsw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

1. 使用 tcpdump 命令准备抓一次 helloword 的调用：

sudo tcpdump -iany port 50051 -w grpc.cap

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtMTnxYc54tf2ZY0KFNiaib0jrWibBSqjRcE3HxlcMib3DicGHCHPiaSXfTPGg/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

1. 运行 helloword 的客户端 greeter_client：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt7oFMzSL0cMIb0KlPb585dkSyHib1KAL8eD9COXpZG2gPWSmCnuCq0qw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

完成一次调用，tcpdump 抓到一次调用的报文，保存为 grpc.cap。

#### 4.3 Wireshark 配置

打开 Wireshark 主面板，选择 ProtoBuf 文件路径：Wireshark-->Preferences-->Protocols-->Protobuf-->Protobuf Search Paths。

选择 helloworld 的 proto 文件地址。

![图片](data:image/svg+xml,<%3Fxml version='1.0' encoding='UTF-8'%3F><svg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'><title></title><g stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'><g transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'><rect x='249' y='126' width='1' height='1'></rect></g></g></svg>)

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZthER7zqYmn85PHsftxbnvgTW97pTRrqnKCtnLtbyicjaOwiaE2P9wo47A/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

Wireshark 打开 grpc.cap 文件，选中 greeter_client 发送端口号和 greeter_server 发送端口号的报文记录，右键 Decode As...为 HTTP/2:

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtm3ibW4MR5ibqmgG2jJkHUjOic5e3greqAf6LMmdAAJHxLtNxuv64CEyIQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

Wireshark 过滤框输入 HTTP2 就可得到一次完整的 gRPC 调用细节：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtaWzhEhsD9wYyzfibm2tibLz7t2PMicy9QpfWwicb6Pdkpias3mjMde6tJmA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

#### 4.4 gRPC 调用分析

从以上抓包得到的 gRPC 调用图可知，gRPC客户端(port:62880)一次调用服务端(port:50051)的RPC方法通常会包括多次HTTP/2帧的发送，本文分析中抓包的一个帧序列例子：Magic-->SETTINGS(双向四个)-->HEADERS-->DATA(GRPC-PROTOBUF)-->WINDOW_UPDATE,PING-->PING-->HEADERS,DATA,HEADERS-->WINDOW_UPDATE,PING-->PING。

下面对调用过程中的每个帧做简要分析。

**1）客户端发送 Magic 帧****Magic 帧的为固定内容：PRI \* HTTP/2.0\r\n\r\nSM\r\n\r\n。如下图所示：客户端发送 Magic 帧后双方就会使用 HTTP/2 相关协议进行通信。**

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtiarQOiciaUOam0Dj8icdiaB5qJz93NnGUbTCM7rtRoAtAlibAxvK2rUicP8pw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**2）客户端和服务端发 SETTINGS 帧**

接着 Magic 帧后，接下来就是发送 SETTINGS 帧，[SETTINGS 帧](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#五-settings-帧)主要用于传递影响两端网络通信的配置参数，以及确认收到这些参数。

客户端和服务器首先发两个 SETTINGS 帧传递配置参数信息，接着服务端发了一个确认的 SETTINGS 帧后，客户端也发出了一个确认的 SETTINGS 帧：

a.客户端发第一个 SETTINGS， 帧类型 = 0x4，帧标志为 0x00, 流标识符为 0：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtNvlsJuxmbv7k05KTXbEOR9fdrP7ruYck0sntBib0oMvYykOQF0VibH6g/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

b.服务端向客户端回了一个 SETTINGS 帧，帧类型 = 0x4，帧标志为 0x00, 流标识符为 0，同时告诉客户端，服务端愿意接收的最大帧大小为 16384 bytes。同时我们看到，SETTINGS 帧的参数类型为 SETTINGS_MAX_FRAME_SIZE(0x5)，参数类型表示服务端愿意接受的包体大小，初始值 为 16364 个字节。此外，SETTINGS 帧长度为 6：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt3wcOqibX2BViciaaYHZL3WNs0VoOjcxE5unwCqhUw5e3m5sc9kv54zKfw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

c.随后服务端再发出一个确认的 SETTINGS 帧，帧类型 = 0x4，帧标志为 0x01, 流标识符为 0：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt3lhYbjBTdxGT2JPt52IMKcX1fwibgQnmdWgnPBaqEPqDAMoGiazMr9Ew/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

d.客户端收到服务端的确认 SETTINGS 帧后，也发出一个 SETTINGS 帧进行确认，帧类型 = 0x4，帧标志为 0x01, 流标识符为 0，双方进行确认后下面就可以开始传输头帧(HEADERS)和数据帧(DATA)了：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt2SJRnicRsK7ZUBFRs8LvV8syWxopozwexicLSIRkUaxJIxhAIQ8yRaxQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**3）客户端发送 HEADERS 帧**

客户端和服务器双方发送 SETTINGS 帧进行双方参数确认后，下一步客户端向服务端发送一个[HEADERS 帧](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#二-headers-帧)， 当前 HEADERS 帧长度为 92，帧类型 = 0x1，帧标志为 0x04(End Headers，0=End Stream:False,1=End Headers:True,0=Padded:False,0=Priority:False)，流标识符为 1，HEADERS 帧还额外带有 Head Block Fragment 头块片段(header 列表是零个或多个字段的集合。当通过网络连接传输时，使用 HTTP 头压缩[COMPRESSION] 将 header 列表序列化为 header block 块。然后将序列化的 header block 块分成字节流，称为 header 块片段)；

同时还可以看到一些 HTTP 请求头(8 个)信息，比如:method:POST，:scheme:http，:path:/helloworld.Greeter/SayHello 等等，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZthicyO0F2vpkCbZ99NDxOf8SlrUibKQKB5586kJuw86C3ZUbTGsMvTZeg/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**4）客户端发送 DATA 帧**

HEADERS 帧发送完之后，接下来客户端给服务器发送[DATA 帧](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#一-data-帧)，当前数据帧的长度为 18 字节(不包含 HTTP/2 帧头)，帧标识为 0x01:End Stream，流标识符为 1，然后是 HTTP/2 的有效包体数据信息(18 字节)，也就是经过 protobuf 序列化的字节流的 gRPC 数据；当前的 gRPC 数据由 gRPC 包头(5 字节)+gRPC 包体(13 字节)组成，gRPC 包头的压缩标志为 Not Compressed(未压缩)，gRPC 包头长度为 13 字节，gRPC 的包体内容为"who are you", 对应的 protobuf 中 message 的 Name 字段承载的信息=WireType<本身占 1 个字节>枚举值为 2[string,编码 0a]+value 长度<本身占 1 个字节>[string 需要显式的告知 value 长度]11 个字节(编码 0b)+字段 value 信息"who are you"<11 个字节>，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtf78BouaBDfP0hY4XWh8CvKMBghPqvBwANWlZv23mdD0TflbiaQ8WfnQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**5）服务端发送 WINDOW_UPDATE 帧和 PING 帧**

客户端发完 DATA 帧后，服务器先回复了两个帧，分别是 WINDOW_UPDATE 帧和 PING 帧，

[WINDOW_UPDATE 帧](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#九-window_update-帧) 主要用于流量控制。当前的 WINDOW_UPDATE 帧的长度为 4，帧类型为 WINDOW_UPDATE(8)，帧标志为 0x00，流标志符为 0，Window Size Increment（流量窗口增量）为 18(收到客户端发送的 DATA 帧长度 18)。

[PING 帧](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP:2-HTTP-Frames-Definitions.md#七-ping-帧) 用于测量最小往返时间(RTT)以及确定连接是否存活。当前 PING 帧的长度为 8，帧类型为 PING(6)，帧标志为 0x00(ACK=False)，流标志符为 0。

此次 WINDOW_UPDATE 帧和 PING 帧的发送情况如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZthP94Z6Rd16eg4I6aaq2hFdCQvIicOcxyRMabV9pB5geyv23BjO9l5eA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**6）客户端回复 PING 帧**

客户端收到服务器的 PING 帧后，会回一个 PING 帧确认(ACK=True)以及回复 Pong 信息，当前 PING 帧的长度为 8，帧类型为 PING(6)，帧标志为 0x01(ACK=True)，流标志符为 0，Pong 信息为一串 16 位的 UUID 串，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtG0OkIuaDOpfS7PVe2PwCoYpGdEJiakxyV5edc4YRMXwOJPaX7A1O86g/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**7）服务端回复 HEADERS 帧+DATA 帧(gRPC)+HEADERS 帧(终止流)**

服务端收到客户端的 PING 帧确认客户端存活状态后，

a. 首先是一个 HEADERS 帧，该帧的帧长度为 14，帧类型 Type 为 HEADS(1)，帧标志 Flags 为 End Headers(0x04)，流标志符为 1，

HEAD 长度为 54，head 数量为 2，分别为 status: 200 OK、content-type:application/grpc;

b. 然后是一个 DATA 帧，该帧的帧长度为 20，帧类型 Type 为 DATA(0)，帧标志 Flags 为 0x00，流标志符为 1，HTTP/2 的有效包体数据信息，也即是 gRPC 数据信息为 15 个字节(5 字节的 gRPC 包头+15 字节的 gRPC 包体内容(”I am datumhu“))；

c. 最后是一个终止流的 HEADERS 帧，该 HEADERS 帧的帧长度为 24，帧类型 Type 为 HEADS(1)，帧标志 Flags 为 End Headers,End Stream(0x05)，流标志符为 1，HEAD 长度为 40 字节，head 数量为 2，分别为 grpc-status: 0、grpc-message:;

如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZtjADPhj3aleJ53H3ZRictkKoyu7e0hR4yU91oTZ0cq4vfrlictPGkJHzQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**8）客户端回复 WINDOW_UPDATE 帧和 PING 帧**

客户端收到服务端的 DATA 响应后，给服务器发送一个 WINDOW_UPDATE 帧和 PING 帧，其中 WINDOW_UPDATE 的窗口大小增量为 20(收到服务端响应的 DATA 帧长度)，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt81JrQS3fg1WzQjpLEv4oVcldxOXmX5hdUG5S11ufmnN0jULPFNmsFw/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

**9）服务端回 PING 帧**

最后服务器收到客户端的 PING 帧后，回复一个 PING 帧确认(ACK=1)，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt6T4t3ZwVbSBrfAz0OMZqia80EqweQF8yu83EJnmOtPw6IBbOZKaepMQ/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

以上一次 gRPC 调用的数据流图概括为如下：

![图片](https://mmbiz.qpic.cn/mmbiz/j3gficicyOvavKkbzEXh8bBOTrlaYZEoZt3QcFWCzMdJJ7mQproOhEibFW1E3UbztzISs6xKJ3k3eX0tiauLQGU3nA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

### 5.**总结**

本文首先概述了 gRPC 的原理，由于 gRPC 是基于 HTTP/2 协议进行网络传输，随后简介了 HTTP/2 通过多路复用和头部压缩等优化措施，基本解决了 HTTP/1.x 包头阻塞的问题，相对 HTTP/1.1 带来了性能提升。HTTP/2 多路复用和头部压缩的关键在于 HTTP/2 通过帧的设计优化了 HTTP 协议语义。所以接着介绍了 HTTP/2 的帧结构和 gRPC 的协议。最后通过抓包一次完整的 gRPC 调用，分析了 GRPC-HTTP2 的数据流过程，希望能够加深对 gRPC 的理解。

原文作者：datumhu，腾讯 IEG 后开开发工程师

原文链接：https://mp.weixin.qq.com/s/6XXJfbnIaKzSFtXyDDB72g