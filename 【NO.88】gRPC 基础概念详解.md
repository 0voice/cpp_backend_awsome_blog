# 【NO.88】gRPC 基础概念详解

**gRPC** (gRPC Remote Procedure Calls) 是 Google 发起的一个开源远程过程调用系统，该系统基于 HTTP/2 协议传输，本文介绍 gRPC 的基础概念，首先通过关系图直观展示这些基础概念之间关联，介绍异步 gRPC 的 Server 和 Client 的逻辑；然后介绍 RPC 的类型，阅读和抓包分析 gRPC 的通信过程协议，gRPC 上下文；最后分析 `grpc.pb.h` 文件的内容，包括 Stub 的能力、Service 的种类以及与核心库的关系。

之所以谓之基础，是这些内容基本不涉及 gRPC Core 的内容。

## **1.基本概念概览**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151513016353805.png)

上图中列出了 gRPC 基础概念及其关系图。其中包括：**Service(定义)、RPC、API、Client、Stub、Channel、Server、Service(实现)、ServiceBuilder** 等。

接下来，以官方提供的 `example/helloworld` 为例进行说明。

`.proto` 文件定义了**服务** `Greeter` 和 **API** `SayHello`：

```
// helloworld.proto// The greeting service definition.service Greeter {  // Sends a greeting  rpc SayHello (HelloRequest) returns (HelloReply) {}}
```

`class GreeterClient` 是 **Client**，是对 **Stub** 封装；通过 **Stub** 可以真正的调用 RPC 请求。

```
class GreeterClient { public:  GreeterClient(std::shared_ptr<Channel> channel)      : stub_(Greeter::NewStub(channel)) {}  std::string SayHello(const std::string& user) {...private:  std::unique_ptr<Greeter::Stub> stub_;};
```

**Channel** 提供一个与特定 gRPC server 的主机和端口建立的连接。

**Stub** 就是在 **Channel** 的基础上创建而成的。

```
target_str = "localhost:50051";auto channel =    grpc::CreateChannel(target_str, grpc::InsecureChannelCredentials());GreeterClient greeter(channel);std::string user("world");std::string reply = greeter.SayHello(user);
```

Server 端需要实现对应的 RPC，所有的 RPC 组成了 **Service**：

```
class GreeterServiceImpl final : public Greeter::Service {  Status SayHello(ServerContext* context, const HelloRequest* request,                  HelloReply* reply) override {    std::string prefix("Hello ");    reply->set_message(prefix + request->name());    return Status::OK;  }};
```

**Server** 的创建需要一个 **Builder**，添加上监听的地址和端口，**注册**上该端口上绑定的服务，最后构建出 Server 并启动：

```
ServerBuilder builder;builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());builder.RegisterService(&service);std::unique_ptr<Server> server(builder.BuildAndStart());
```

**RPC 和 API 的区别**：RPC (Remote Procedure Call) 是一次远程过程调用的整个动作，而 API (Application Programming Interface) 是不同语言在实现 RPC 中的具体接口。一个 RPC 可能对应多种 API，比如同步的、异步的、回调的。一次 RPC 是对某个 API 的一次调用，比如：

```
std::unique_ptr<ClientAsyncResponseReader<HelloReply> > rpc(    stub_->PrepareAsyncSayHello(&context, request, &cq));
```

不管是哪种类型 RPC，都是由 Client 发起请求。

## **2.异步相关概念**

不管是 Client 还是 Server，异步 gRPC 都是利用 [CompletionQueue](https://grpc.io/grpc/cpp/classgrpc_1_1_completion_queue.html) API 进行异步操作。基本的流程：

- 绑定一个 `CompletionQueue` 到一个 RPC 调用
- 利用唯一的 `void*` Tag 进行读写
- 调用 `CompletionQueue::Next()` 等待操作完成，完成后通过唯一的 Tag 来判断对应什么请求/返回进行后续操作

官方文档 [Asynchronous-API tutorial](https://grpc.io/docs/languages/cpp/async/) 中有上边的介绍，并介绍了异步 client 和 server 的解释，对应这 `greeter_async_client.cc` 和 `greeter_async_server.cc` 两个文件。

Client 看文档可以理解，但 Server 的代码复杂，文档和注释中的解释并不是很好理解，接下来会多做一些解释。

### 2.1 异步 Client

`greeter_async_client.cc` 中是异步 Client 的 Demo，其中只有一次请求，逻辑简单。

- 创建 CompletionQueue
- 创建 RPC (既 `ClientAsyncResponseReader<HelloReply>`)，这里有两种方式：
- - `stub_->PrepareAsyncSayHello()` + `rpc->StartCall()`
  - `stub_->AsyncSayHello()`
- 调用 `rpc->Finish()` 设置请求消息 reply 和唯一的 tag 关联，将请求发送出去
- 使用 `cq.Next()` 等待 Completion Queue 返回响应消息体，通过 tag 关联对应的请求

[TODO] **ClientAsyncResponseReader 在 `Finish()` 完之后就没有用了？**

### 2.2 异步 Server

`RequestSayHello()` 这个函数没有任何的说明。只说是：”we *request* that the system start processing SayHello requests.” 也没有说跟 `cq_->Next(&tag, &ok);` 的关系。我这里通过加上一些日志打印，来更清晰的展示 Server 的逻辑：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151513209083141.png)

上边绿色的部分为创建的第一个 CallData 对象地址，橙色的为第二个 CallData 的地址。

- 创建一个 CallData，初始构造列表中将状态设置为 **CREATE**
- 构造函数中，调用 Process()成员函数，调用 `service_->RequestSayHello()`后，状态变更为 **PROCESS**：
- - 传入 `ServerContext ctx_`
  - 传入 `HelloRequest request_`
  - 传入 `ServerAsyncResponseWriter<HelloReply> responder_`
  - 传入 `ServerCompletionQueue* cq_`
  - 将对象自身的地址作为 `tag` 传入
  - 该动作，能将**事件加入事件循环，可以在 CompletionQueue 中等待**
- 收到请求，`cq->Next()`的阻塞结束并返回，**得到 tag**，既上次传入的 CallData 对象地址
- 调用 tag 对应 CallData 对象的 `Proceed()`，此时状态为 **Process**
- - 创建新的 CallData 对象以接收新请求
  - 处理消息体并设置 `reply_`
  - 将状态设置为 **FINISH**
  - 调用 `responder_.Finish()` 将返回发送给客户端
  - 该动作，能将**事件加入到事件循环，可以在 CompletionQueue 中等待**
- 发送完毕，`cq->Next()`的阻塞结束并返回，**得到 tag**。现实中，如果发送有异常应当有其他相关的处理
- 调用 tag 对应 CallData 对象的 `Proceed()`，此时状态为 **FINISH**，`delete this` 清理自己，一条消息处理完成

### 2.3 关系图

将上边的异步 Client 和异步 Server 的逻辑通过关系图进行展示。右侧 RPC 为创建的对象中的内存容，左侧使用相同颜色的小块进行代替。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151513336865444.png)

以下 CallData 并非 gRPC 中的概念，而是异步 Server 在实现过程中为了方便进行的封装，其中的 Status 也是在异步调用过程中自定义的、用于转移的状态。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151513437399143.png)

### 2.4 异步 Client 2

在 `example/cpp/helloworld` 中还有另外一个异步 Client，对应文件名为 `greeter_async_client2.cc`。这个例子中使用了**两个线程去分别进行发送请求和处理返回**，一个线程批量发出 100 个 SayHello 的请求，另外一个不断的通过 `cq_.Next()` 来等待返回。

无论是 Client 还是 Server，**在以异步方式进行处理时，都要预先分配好一定的内存/对象，以存储异步的请求或返回。**

### 2.5 在 `example/cpp/helloworld` 中，还提供了 callback 相关的 Client 和 Server。

使用回调方式简介明了，结构上与同步方式相差不多，但是并发有本质的区别。可以通过文件对比，来查看其中的差异。

```
cd examples/cpp/helloworld/vimdiff greeter_callback_client.cc greeter_client.ccvimdiff greeter_callback_server.cc greeter_server.cc
```

其实，回调方式的异步调用属于实验性质的，不建议直接在生产环境使用，这里也只做简单的介绍：

> Notice: This API is EXPERIMENTAL and may be changed or removed at any time.

#### **2.5.1 回调 Client**

发送单个请求，在调用 `SayHello` 时，除了传入 Request、 Reply 的地址之外，还需要传入一个接收 Status 的回调函数。

例子中只有一个请求，因此在 `SayHello` 之后，就直接通过 `condition_variable` 的 wait 函数等待回调结束，然后进行后续处理。这样其实不能进行并发，跟同步请求差别不大。如果要进行大规模的并发，还是需要使用额外的对象进行封装一下。

```
stub_->async()->SayHello(&context, &request, &reply,                         [&mu, &cv, &done, &status](Status s) {                           status = std::move(s);                           std::lock_guard<std::mutex> lock(mu);                           done = true;                           cv.notify_one();                         });
```

上边函数调用函数声明如下，很明显这是实验性（experimental）的接口：

```
void Greeter::Stub::experimental_async::SayHello(    ::grpc::ClientContext* context, const ::helloworld::HelloRequest* request,    ::helloworld::HelloReply* response, std::function<void(::grpc::Status)> f);
```

#### **2.5.2 回调 Server**

与同步 Server 不同的是：

- 服务的实现是继承 `Greeter::CallbackService`
- `SayHello` 返回的不是状态，而是 `ServerUnaryReactor` 指针
- 通过 `CallbackServerContext` 获得 `reactor`
- 调用 `reactor` 的 `Finish` 函数处理返回状态

## **3.流相关概念**

可以按照 Client 和 Server 一次发送/返回的是单个消息还是多个消息，将 gRPC 分为：

- Unary RPC
- Server streaming RPC
- Client streaming RPC
- Bidirectional streaming RPC

### 3.1 Server 对 RPC 的实现

Server 需要实现 proto 中定义的 RPC，每种 RPC 的实现都需要将 ServerContext 作为参数输入。

如果是一元 (Unary) RPC 调用，则像调用普通函数一样。将 Request 和 Reply 的对象地址作为参数传入，函数中将根据 Request 的内容，在 Reply 的地址上写上对应的返回内容。

```
// rpc GetFeature(Point) returns (Feature) {}Status GetFeature(ServerContext* context, const Point* point, Feature* feature);
```

如果涉及到流，则会用 Reader 或/和 Writer 作为参数，读取流内容。如 ServerStream 模式下，只有 Server 端产生流，这时对应的 Server 返回内容，需要使用作为参数传入的 `ServerWriter`。这类似于以 `'w'` 打开一个文件，持续的往里写内容，直到没有内容可写关闭。

```
// rpc ListFeatures(Rectangle) returns (stream Feature) {}Status ListFeatures(ServerContext* context,                    const routeguide::Rectangle* rectangle,                    ServerWriter<Feature>* writer);
```

另一方面，Client 来的流，Server 需要使用一个 ServerReader 来接收。这类似于打开一个文件，读其中的内容，直到读到 `EOF` 为止类似。

```
// rpc RecordRoute(stream Point) returns (RouteSummary) {}Status RecordRoute(ServerContext* context, ServerReader<Point>* reader,                   RouteSummary* summary);
```

如果 Client 和 Server 都使用流，也就是 Bidirectional-Stream 模式下，输入参数除了 ServerContext 之外，只有一个 `ServerReaderWriter` 指针。通过该指针，既能读 Client 来的流，又能写 Server 产生的流。

例子中，Server 不断地从 stream 中读，读到了就将对应的写过写到 stream 中，直到客户端告知结束；Server 处理完所有数据之后，直接返回状态码即可。

```
// rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}Status RouteChat(ServerContext* context,                 ServerReaderWriter<RouteNote, RouteNote>* stream);
```

### 3.2 Client 对 RPC 的调用

Client 在调用一元 (Unary) RPC 时，像调用普通函数一样，除了传入 ClientContext 之外，将 Request 和 Response 的地址，返回的是 RPC 状态：

```
// rpc GetFeature(Point) returns (Feature) {}Status GetFeature(ClientContext* context, const Point& request,                  Feature* response);
```

Client 在调用 ServerStream RPC 时，不会得到状态，而是返回一个 ClientReader 的指针：

```
// rpc ListFeatures(Rectangle) returns (stream Feature) {}unique_ptr<ClientReader<Feature>> ListFeatures(ClientContext* context,                                               const Rectangle& request);
```

Reader 通过不断的 `Read()`，来不断的读取流，结束时 `Read()` 会返回 `false`；通过调用 `Finish()` 来读取返回状态。

调用 ClientStream RPC 时，则会返回一个 ClientWriter 指针：

```
// rpc RecordRoute(stream Point) returns (RouteSummary) {}unique_ptr<ClientWriter<Point>> RecordRoute(ClientContext* context,                                            Route Summary* response);
```

Writer 会不断的调用 `Write()` 函数将流中的消息发出；发送完成后调用 `WriteDone()` 来说明发送完毕；调用 `Finish()` 来等待对端发送状态。

而双向流的 RPC 时，会返回 ClientReaderWriter，：

```
// rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}unique_ptr<ClientReaderWriter<RouteNote, RouteNote>> RouteChat(    ClientContext* context);
```

前面说明了 Reader 和 Writer 读取和发送完成的函数调用。因为 RPC 都是 Client 请求而后 Server 响应，双向流也是要 Client 先发送完自己流，才有 Server 才可能结束 RPC。所以对于双向流的结束过程是：

- `stream->WriteDone()`
- `stream->Finish()`

示例中创建了单独的一个线程去发送请求流，在主线程中读返回流，实现了一定程度上的并发。

### 3.3 流是会结束的

并不似长连接，建立上之后就一直保持，有消息的时候发送。（是否有通过建立一个流 RPC 建立推送机制？）

- Client 发送流，是通过 `Writer->WritesDone()` 函数结束流
- Server 发送流，是通过结束 RPC 函数并返回状态码的方式来结束流
- 流接受者，都是通过 `Reader->Read()` 返回的 bool 型状态，来判断流是否结束

Server 并没有像 Client 一样调用 `WriteDone()`，而是在消息之后，将 status code、可选的 status message、可选的 trailing metadata 追加进行发送，这就意味着流结束了。

## **4.通信协议**

本节通过介绍 gRPC 协议文档描述和对 helloworld 的抓包，来说明 gRPC 到底是如何传输的。

官方文档《[gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)》中有描述 gRPC 基于 HTTP2 的具体实现，主要介绍的就是协议，也就是 gRPC 的请求和返回是如何基于 HTTP 协议构造的。如果不熟悉 HTTP2 可以阅读一下 [RFC 7540](https://httpwg.org/specs/rfc7540.html)。

### 4.1 ABNF 语法

ABNF 语法是一种描述协议的标准，gRPC 协议也是使用 ABNF 语法描述，几种常见的运算符在[第三节](https://datatracker.ietf.org/doc/html/rfc5234#section-3)中有介绍：

```
3.  Operators 3.1.  Concatenation:  Rule1 Rule2 3.2.  Alternatives:  Rule1 / Rule2 3.3.  Incremental Alternatives: Rule1 =/ Rule2 3.4.  Value Range Alternatives:  %c##-## 3.5.  Sequence Group:  (Rule1 Rule2) 3.6.  Variable Repetition:  *Rule 3.7.  Specific Repetition:  nRule 3.8.  Optional Sequence:  [RULE] 3.9.  Comment:  ; Comment 3.10. Operator Precedence
```

### 4.2 请求协议

`*<element>` 表示 element 会重复多次（最少 0 次）。知道这个就能理解概况里的描述了：

```
Request → Request-Headers *Length-Prefixed-Message EOSRequest-Headers → Call-Definition *Custom-Metadata
```

这表示 **Request 是由 3 部分组成**，首先是 `Request-Headers`，接下来是可能多次出现的 `Length-Prefixed-Message`，最后以一个 `EOS` 结尾（EOS 表示 End-Of-Stream）。

#### **4.2.1 Request-Headers**

根据上边的协议描述， `Request-Headers` 是由一个 `Call-Definition` 和若干 `Custom-Metadata` 组成。

`[]` 表示最多出现一次，比如 `Call-Definition` 有很多组成部分，其中 `Message-Type` 等是选填的：

```
Call-Definition → Method Scheme Path TE [Authority] [Timeout] Content-Type [Message-Type] [Message-Encoding] [Message-Accept-Encoding] [User-Agent]
```

通过 Wireshark 抓包可以看到请求的 Call-Definition 中共有所有要求的 Header，还有额外可选的，比如 user-agent：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151514217645328.png)

因为 helloworld 的示例比较简单，请求中没有填写自定义的元数据（Custom-Metadata）

#### **4.2.2 Length-Prefixed-Message**

传输的 Length-Prefixed-Message 也分为三部分：

```
Length-Prefixed-Message → Compressed-Flag Message-Length Message
```

同样的，Wireshark 抓到的请求中也有这部分信息，并且设置 `.proto` 文件的搜索路径之后可以自动解析 PB：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151514343488279.png)

其中第一个红框（Compressed-Flag）表示不进行压缩，第二个红框（Message-Length）表示消息长度为 7，蓝色反选部分则是 Protobuf 序列化的二进制内容，也就是 Message。

在 gRPC 的[核心概念](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition)介绍时提到，gRPC 默认使用 Protobuf 作为接口定义语言（IDL），也可以使用其他的 IDL 替代 Protobuf：

> By default, gRPC uses [protocol buffers](https://developers.google.com/protocol-buffers) as the Interface Definition Language (IDL) for describing both the service interface and the structure of the payload messages. It is possible to use other alternatives if desired.

这里 Length-Prefixed-Message 中传输的可以是 PB 也可以是 JSON，须通过 `Content-Type` 头中描述告知。

#### **4.2.3 EOS**

End-Of-Stream 并没有单独的数据去描述，而是通过 HTTP2 的数据帧上带一个 END_STREAM 的 flag 来标识的。比如 helloworld 中请求的数据帧，也携带了 END_STREAM 的标签：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151514519376577.png)

### 4.3 返回协议

`()` 表示括号中的内容作为单个元素对待，`/` 表示前后两个元素可选其一。Response 的定义说明，可以有两种返回形式，一种是消息头、消息体、Trailer，另外一种是只带 Trailer：

```
Response → (Response-Headers *Length-Prefixed-Message Trailers) / Trailers-Only
```

这里需要区分 gRPC 的 Status 和 HTTP 的 Status 两种状态。

```
Response-Headers → HTTP-Status [Message-Encoding] [Message-Accept-Encoding] Content-Type *Custom-MetadataTrailers-Only → HTTP-Status Content-Type TrailersTrailers → Status [Status-Message] *Custom-Metadata
```

不管是哪种形式，最后一部分都是`Trailers`，其中包含了 gRPC 的状态码、状态信息和额外的自定义元数据。

同样地，**使用 END_STREAM 的 flag 标识最后 Trailer 的结束。**

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151515032682015.png)

### 4.4 与 HTTP/2 的关系

> The libraries in this repository provide a concrete implemnetation of the gRPC protocol, layered over HTTP/2.

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151515155159188.png)

## **5.上下文**

gRPC 支持上下文的传递，其主要用途有：

- 添加自定义的 metadata，能够通过 gRPC 调用传递
- 控制调用配置，如压缩、鉴权、超时
- 从对端获取 metadata
- 用于性能测量，比如使用 opencensus 等

客户端添加自定义的 metadata key-value 对没有特别的区分，而服务端添加的，则有 inital 和 trailing 两种 metadata 的区分。这也分别对应这 ClientContext 只有一个添加 Metadata 的函数：

```
void AddMetadata (const std::string &meta_key, const std::string &meta_value)
```

而 ServerContext 则有两个：

```
void AddInitialMetadata (const std::string &key, const std::string &value)void AddTrailingMetadata (const std::string &key, const std::string &value)
```

还有一种 Callback Server 对应的上下文叫做 `CallbackServerContext`，它与 `ServerContext` 继承自同一个基类，功能基本上相同。区别在于：

- ServerContext 被 Sync Server 和基于 CQ 的 Async Server 所使用，后者需要用到 `AsyncNotifyWhenDone`
- CallbackServerContext 因为在 `CallOnDone` 的时候，需要释放 context，因此需要知道 context_allocator，因此对应设置和获取 context_allocator 的两个函数

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151515261688750.png)

## **6.Generated Code**

通过 `protoc` 生成 gRPC 相关的文件，除了用于消息体定义的 `xxx.pb.h` 和 `xxx.pb.cc` 文件之外，就是定义 RPC 过程的 `xxx.grpc.pb.h` 和 `xxx.grpc.pb.cc`。本节以 `helloworld.proto` 生成的文件为例，看看 `.grpc.pb` 相关文件具体定义了些什么。

`helloworld.grpc.pb.h` 文件中有命名空间 `helloworld`，其中就仅包含一个类 `Greeter`，所有的 RPC 相关定义都在 `Greeter` 当中，这其中又主要分为两部分：

- Client 用于调用 RPC 的媒介 `Stub` 相关类
- Server 端用于实现不同服务的 Service 相关类和类模板

### 6.1 Stub

`.proto` 中的一个 `service` 只有一个 `Stub`，该类中会提供对应每个 RPC 所有的同步、异步、回调等方式的函数都包含在该类中，而该类继承自接口类 `StubInterface`。

为什么需要一个 StubInterface 来让 Stub 继承，而不是直接产生 Stub？别的复杂的 proto 会有多个 Stub 继承同一个 StubInterface 的情况？不会，因为每个 RPC 对应的函数名是不同。

Greeter 中唯一一个函数是用于创建 Stub 的静态函数 `NewStub`：

```
static std::unique_ptr<Stub> NewStub(...)
```

Stub 中同步、异步方式的函数是直接作为 Stub 的成员函数提供，比如针对一元调用：

- SayHello
- AsyncSayHello
- PrepareAsyncSayHello

[TODO] 为什么同步函数`SayHello`的实现是放在源代码中，而异步函数`AsyncSayHello`的实现是放在头文件中（两者都是直接 `return` 的）？

```
return ::grpc::internal::BlockingUnaryCall< ::helloworld::HelloRequest, ::helloworld::HelloReply, ::grpc::protobuf::MessageLite, ::grpc::protobuf::MessageLite>(channel_.get(), rpcmethod_SayHello_, context, request, response);return std::unique_ptr< ::grpc::ClientAsyncResponseReader< ::helloworld::HelloReply>>(AsyncSayHelloRaw(context, request, cq));
```

回调方式的 RPC 调用是通过一个 `experimental_async` 的类进行了封装（有个 `async_stub_` 的成员变量），所以回调 Client 中提到，回调的调用方式用法是 `stub_->async()->SayHello(...)`。

`experimental_async` 类定义中将 `Stub` 类作为自己的友元，自己的成员可以被 `Stub` 直接访问，而在 `StubInterface` 中也对应有一个 `experimental_async_interface` 的接口类，规定了要实现哪些接口。

### 6.2 Service

有几个概念都叫 Service：proto 文件中 RPC 的集合、proto 文件中 service 产生源文件中的 `Greeter::Service` 类、gRPC 框架中的 `::grpc::Service` 类。本小节说的 Service 就是 `helloworld.grpc.pb.h` 中的 `Greeter::Service`。

#### **6.2.1 Service 是如何定义的**

`helloworld.grpc.pb.h` 文件中共定义了 **7 种 Service**，拿出最常用的 `Service` 和 `AsyncService` 两个定义来说明下 Service 的定义过程：通过类模板链式继承。

**`Service` 跟其他几种 Service 不同，直接继承自 `grpc::Service`，而其他的 Service 都是由类模板构造出来的，而且使用类模板进行嵌套，最基础的类就是这里的 `Service`。**

`Service` 有以下特点：

- 构造函数利用其父类 `grpc::Service` 的 `AddMethod()` 函数，将 `.proto` 文件中定义的 RPC API，添加到成员变量 `methods_` 中（`methods_` 是个向量）
- `AddMethod()` 时会创建 `RpcServiceMethod` 对象，而该对象有一个属性叫做 `api_type_`，构造时默认填的 `ApiType::SYNC`
- `SayHello` 函数不直接声明为纯虚函数，而是以返回 `UNIMPLEMENTED` 状态，因为这个类可能被多次、多级继承

**所以 `Service` 类中的所有 RPC API 都是同步的。**

再看 `AsyncService` 的具体定义：

```
template <class BaseClass>  class WithAsyncMethod_SayHello : public BaseClass { ... };typedef WithAsyncMethod_SayHello<Service > AsyncService;
```

所以 **`AsyncService` 的含义就是继承自 `Service`，加上了 `WithAsyncMethod_SayHello` 的新功能**：

- 构造时，将 SayHello (RPC) 对应的 `api_type_` 设置为 `ApiType::ASYNC`
- 将 `SayHello` 函数直接禁用掉， `abort()` + 返回 `UNIMPLEMENTED` 状态码
- 添加 `RequestSayHello()` 函数， 异步 Server 小节中有介绍过这个函数用法

通过 gRPC 提供的 `route_guide.proto` 例子能更明显的理解这点：

```
typedef WithAsyncMethod_GetFeature< \    WithAsyncMethod_ListFeatures< \    WithAsyncMethod_RecordRoute< \    WithAsyncMethod_RouteChat<Service> > > >    AsyncService;
```

这里 RouteGuide 服务中有 4 个 RPC，`GetFeature`、`ListFeatures`、`RecordRoute`、`RouteChat`，通过 4 个`WithAsyncMethod_{RPC_name}` 的类模板嵌套，能将 4 个 API 都设置成 `ApiType::ASYNC`、添加上对应的 `RequestXXX()` 函数、禁用同步函数。

[TODO] **通过类模板嵌套继承的方式，有什么好处？** 为什么不直接实现 `AsyncService` 这个类呢？

#### **6.2.2 Service 的种类**

`helloworld.grpc.pb.h` 文件中 7 种 Service 中，有 3 对 Service 的真正含义都相同（出于什么目的使用不同的名称？），实际只剩下 4 种 Service。前三种在前边的同步、异步、回调 Server 的介绍中都有涉及。

- Service
- AsyncService
- CallbackService
- ExperimentalCallbackService – 等价于 CallbackService
- StreamedUnaryService
- SplitStreamedService – 等价于 Service
- StreamedService – 等价于 StreamedUnaryService

其实这些不同类型的 Service 是跟前边提到的 `api_type_` 有关。使用不同的 `::grpc::Service::MarkMethodXXX` 设置**不同的 ApiType** 会产生**不同的 API 模板类**，所有 API 模板类级联起来，就得到了**不同的 Service**。这三者的关系简单列举如下：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151515452378680.png)

另外还有两种模板是通过设置其他属性产生的，这里暂时不做介绍：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151515538628129.png)

[TODO] **头文件中没有用到的类模板在什么场景中会用到？**

### 6.3 与 `::grpc` 核心库的关系

`Stub` 类中主要是用到 gRPC Channel 和不同类型 RPC 对应的方法实现:

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151516064044761.png)

`Service` 类则继承自 `::grpc::Service`，具备其父类的能力，需要自己实现一些 RPC 方法具体的处理逻辑。其它 Service 涉及到 gRPC 核心库的联系有：

- `AsyncService::RequestSayHello()` 调用 `::grpc::Service::RequestAsyncUnary`。
- `CallbackService::SayHello()` 函数返回的是 `::grpc::ServerUnaryReactor` 指针。
- `CallbackService::SetMessageAllocatorFor_SayHello()` 函数中调用 `::grpc::internal::CallbackUnaryHandler::SetMessageAllocator()` 函数设置 RPC 方法的回调的消息分配器。

[TODO] `SetMessageAllocatorFor_SayHello()` 函数并没有被调用到，默认该分配器指针初始值为空，表示用户预先自己分配好而无需回调时分配？

**参考资料**

- https://grpc.io/docs/what-is-grpc/core-concepts/
- https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md
- https://grpc.io/blog/wireshark/

原文作者：jasonzxpan，腾讯 IEG 运营开发工程师

原文链接：https://mp.weixin.qq.com/s/I2QHEBO26nGqhGwIw281Pg