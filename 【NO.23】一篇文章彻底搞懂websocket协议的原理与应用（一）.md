# 【NO.23】一篇文章彻底搞懂websocket协议的原理与应用（一）

## 0.前言

![img](https://pic4.zhimg.com/80/v2-47c47bee014ade2ff0f82e457bfb98c7_720w.webp)

WebSocket协议是基于TCP的一种新的网络协议。它实现了浏览器与服务器全双工(full-duplex)通信——允许服务器主动发送信息给客户端。

WebSocket通信协议于2011年被IETF定为标准RFC 6455，并被RFC7936所补充规范。

## 1.WebSocket简介

webSocket是什么:

1、WebSocket是一种在单个TCP连接上进行全双工通信的协议

2、WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据

3、在WebSocket API中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输

4、需要安装第三方包：cmd中：go get -u -v [github.com/gorilla/websocket](https://link.zhihu.com/?target=http%3A//github.com/gorilla/websocket)

WebSocket 是一种标准协议，用于在客户端和服务端之间进行双向数据传输。但它跟 HTTP 没什么关系，它是一种基于 TCP 的一种独立实现。

以前客户端想知道服务端的处理进度，要不停地使用 Ajax 进行轮询，让浏览器隔个几秒就向服务器发一次请求，这对服务器压力较高。另外一种轮询就是采用 long poll 的方式，这就跟打电话差不多，没收到消息就一直不挂电话，也就是说，客户端发起连接后，如果没消息，就一直不返回 Response 给客户端，连接阶段一直是阻塞的。

而 WebSocket 解决了 HTTP 的这几个难题。首先，当服务器完成协议升级后（ HTTP -> WebSocket ），服务端可以主动推送信息给客户端，解决了轮询造成的同步延迟问题。由于 WebSocket 只需要一次 HTTP 握手，服务端就能一直与客户端保持通讯，直到关闭连接，这样就解决了服务器需要反复解析 HTTP 协议，减少了资源的开销。

WebSocket协议支持（在受控环境中运行不受信任的代码的）客户端与（选择加入该代码的通信的）远程主机之间进行全双工通信。用于此的安全模型是Web浏览器常用的基于原始的安全模式。 协议包括一个开放的握手以及随后的TCP层上的消息帧。 该技术的目标是为基于浏览器的、需要和服务器进行双向通信的（服务器不能依赖于打开多个HTTP连接（例如，使用XMLHttpRequest或和长轮询））应用程序提供一种通信机制。

![img](https://pic2.zhimg.com/80/v2-e84d51f6e63c249e9464cfed1cf57519_720w.webp)

websocket 是一个基于应用层的网络协议，建立在tcp 协议之上，和 http 协议可以说是兄弟的关系，但是这个兄弟有点依赖 http ，为什么这么说呢？我们都知道 HTTP 实现了三次握手来建立通信连接，实际上 websocket 的创始人很聪明，他不想重复的去造轮子，反正我兄弟已经实现了握手了，我干嘛还要重写一套呢？先让它去冲锋陷阵呢，我坐收渔翁之利不是更香 吗，所以一般来说，我们会先用 HTTP 先进行三次握手，再向服务器请求升级为websocket 协议，这就好比说，嘿兄弟你先去给我排个队占个坑位建个小房子，到时候我在把这房子改造成摩天大楼。而且一般来说 80 和 443 端口一般 web 服务端都会外放出去，这样可以有效的避免防火墙的限制。当然，你创建的 websocket 服务端进程的端口也需要外放出去。

很多人会想问，web开发 使用 HTTP 协议不是已经差不多够用了吗？为什么还要我再多学一种呢？这不是搞事情嘛，仔细想想，一门新技术的产生必然有原因的，如果没有需求，我们干嘛那么蛋疼去写那么多东西，就是因为 HTTP 这个协议有些业务需求支持太过于鸡肋了，从 HTTP 0.9 到现在的 HTTP3.0 ，HTTP协议可以说说是在普通的web开发领域已经是十分完善且高效的了，说这个协议养活了全球半数的公司也不为过吧，像 2.0 服务器推送技术，3.0 采用了 UDP 而放弃了原来的 TCP ,这些改动都是为了进一步提升协议的性能，然而大家现在还是基本使用的 HTTP 1.1 这个最为经典的协议, 也是让开发者挺尴尬的。

绝大多数的web开发都是应用层开发者,大多数都是基于已有的应用层去开发应用，可以说我们最熟悉、日常打交道最多的就是应用层协议了，底下 TCP/IP 协议我们基本很少会去处理，当然大厂可能就不一样了，自己弄一套协议也是正常的，这大概也是程序员和码农的区别吧，搬砖还是创新，差别还是很大的。网络这种分层协议的好处我在之前的文章也说过了，这种隔离性很方便就可以让我们基于原来的基础去拓展，具有较好的兼容性。

总的来说，它就是一种依赖HTTP协议的，支持全双工通信的一种应用层网络协议。

## 2.WebSocket产生背景

简单的说，WebSocket协议之前，双工通信是通过多个http链接来实现，这导致了效率低下。WebSocket解决了这个问题。下面是标准RFC6455中的产生背景概述。

长久以来, 创建实现客户端和用户端之间双工通讯的web app都会造成HTTP轮询的滥用: 客户端向主机不断发送不同的HTTP呼叫来进行询问。

**这会导致一系列的问题：**

- 1.服务器被迫为每个客户端使用许多不同的底层TCP连接：一个用于向客户端发送信息，其它用于接收每个传入消息。
- 2.有些协议有很高的开销，每一个客户端和服务器之间都有HTTP头。
- 3.客户端脚本被迫维护从传出连接到传入连接的映射来追踪回复。

一个更简单的解决方案是使用单个TCP连接双向通信。 这就是WebSocket协议所提供的功能。 结合WebSocket API ，WebSocket协议提供了一个用来替代HTTP轮询实现网页到远程主机的双向通信的方法。

WebSocket协议被设计来取代用HTTP作为传输层的双向通讯技术,这些技术只能牺牲效率和可依赖性其中一方来提高另一方，因为HTTP最初的目的不是为了双向通讯。

## 3.WebSocket实现原理

![img](https://pic3.zhimg.com/80/v2-cc90a0344d7e70f6215bb5d1a14d340e_720w.webp)

在实现websocket连线过程中，需要通过浏览器发出websocket连线请求，然后服务器发出回应，这个过程通常称为“握手” 。在 WebSocket API，浏览器和服务器只需要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。两者之间就直接可以数据互相传送。在此WebSocket 协议中，为我们实现即时服务带来了两大好处：

\1. Header：互相沟通的Header是很小的-大概只有 2 Bytes。

\2. Server Push：服务器的推送，服务器不再被动的接收到浏览器的请求之后才返回数据，而是在有新数据时就主动推送给浏览器。

## 4.WebSocket协议举例

![img](https://pic1.zhimg.com/80/v2-7a2acc6bc32134ee998b50ff9f331b60_720w.webp)

浏览器请求:

- GET /webfin/websocket/ HTTP/1.1。
- Host: localhost。
- Upgrade: websocket。
- Connection: Upgrade。
- Sec-WebSocket-Key: xqBt3ImNzJbYqRINxEFlkg==。
- Origin: [http://服务器地址](https://link.zhihu.com/?target=http%3A//xn--zfru1gfr6bz63i/)。
- Sec-WebSocket-Version: 13。

服务器回应:

- HTTP/1.1 101 Switching Protocols。
- Upgrade: websocket。
- Connection: Upgrade。
- Sec-WebSocket-Accept: K7DJLdLooIwIG/MOpvWFB3y3FE8=。
- WebSocket借用http请求进行握手，相比正常的http请求，多了一些内容。其中：
- Upgrade: websocket。
- Connection: Upgrade。
- 表示希望将http协议升级到Websocket协议。Sec-WebSocket-Key是浏览器随机生成的base64 encode的值，用来询问服务器是否是支持WebSocket。

服务器返回:

- Upgrade: websocket。
- Connection: Upgrade。
- 告诉浏览器即将升级的是Websocket协议

Sec-WebSocket-Accept是将请求包“Sec-WebSocket-Key”的值，与”258EAFA5-E914-47DA-95CA-C5AB0DC85B11″这个字符串进行拼接，然后对拼接后的字符串进行sha-1运算，再进行base64编码得到的。用来说明自己是WebSocket助理服务器。

Sec-WebSocket-Version是WebSocket协议版本号。RFC6455要求使用的版本是13，之前草案的版本均应当被弃用。

## 5.WebSocket使用

### 5.1 WebSocket 介绍

WebSocket 发起单个请求，服务端不需要等待客服端，客户端在任何时候也能发消息到服务端，减少了轮询时候的延迟.经历一次连接后，服务器能给客户端发多次。下图是轮询与WebSocket的区别。

![img](https://pic1.zhimg.com/80/v2-460cb0b19bf881cc5fdd4d18b8e531c8_720w.webp)

基于http的实时消息是相当的复杂，在无状态的请求中维持回话的状态增加了复杂度，跨域也很麻烦，使用ajax处理请求有序请求需要考虑更多。通过ajax进行交流也不简单。每一个延伸http功能的目的不是增加他的复杂度。websocket 可以大大简化实时通信应用中的链接。

Websocket是一种底层网络协议，可以让你在这个基础上建立别的标准协议。比如在WebSocket的客户端的基础上使用XMPP登录不同的聊天服务器，因为所有的XMPP服务理解相同的标准协议。WebSocket是web应用的一种创新。

为了与其他平台竞争，WebSocket是H5应用提供的一部分先进功能。每个操作系统都需要网络功能，能够让应用使用Sockets与别的主机进行通信，是每个大平台的核心功能。在很多方面，让Web应用表现的像操作系统平台是html5的趋势。像socket这样底层的网络协议APIs不会符合原始的安全模型，也不会有web api那样的设计风格。WebSocket给H5应用提供TCP的方式不会消弱网络安全且有现代的Api。

WebSocket是Html5平台的一个重要组件也是开发者强有力的工具。简单的说，你需要WebSocket创建世界级的web应用。它弥补了http不适合实时通信的重大缺陷。异步、双向通信模式，通过传输层协议使WebSocket具有普遍灵活性。想象一下你能用WebSocket创建正真实实时应用的所有方式。比如聊天、协作文档编辑、大规模多人在线游戏（MMO）,股票交易应用等等。

WebSocket是一个协议，但也有一个WebSocket API，这让你的应用去控制WebSocket的协议去响应被服务端触发的事件。API是W3C开发，协议是IETE制定。现代浏览器支持WebSocket API，这包括使用全双工和双向链接的方法和特性。让你执行像打开关闭链接、发送接收消息、监听服务端事件等必要操作。

### 5.2 WebSocket API

WebSocket API其实就是一个使用WebSocket协议的接口，通过它来建立全双工通道来收发消息，简单易学，要连接远程服务器，只需要创建一个WebSocket对象实体，并传入一个服务端的URL。在客户端和服务端一开始握手的期间，http协议升级到WebSocket协议就建立了连接，底层都是TCP协议。一旦建立连接，通过WebSocket接口可以反复的发送消息。在你的代码里面，你可以使用异步事件监听连接生命周期的每个阶段。

WebSocket API是纯事件驱动，一旦建立全双工连接，当服务端给客户端发送数据或者资源，它能自动发送状态改变的数据和通知。所以你不需要为了状态的更新而去轮训Server，在客户端监听即可。

首先，我们需要通过调用WebSocket构造函数来创建一个WebSocket连接，构造函数会返回一个WebSocket实例，可以用来监听事件。这些事件会告诉你什么时候连接建立，什么时候消息到达，什么时候连接关闭了，以及什么时候发生了错误。WebSocket协议定义了两种URL方案，WS和WSS分别代表了客户端和服务端之间未加密和加密的通信。WS(WebSocket)类似于Http URL，而WSS（WebSocket Security）URL 表示连接是基于安全传输层（TLS/SSL）和https的连接是同样的安全机制。

WebSocket的构造函数需要一个URL参数和一个可选的协议参数（一个或者多个协议的名字），协议的参数例如XMPP（Extensible Messaging and Presence Protocol）、SOAP（Simple Object Access Protocol）或者自定义协议。而URL参数需要以WS://或者WSS://开头，例如：ws://[http://www.websocket.org](https://link.zhihu.com/?target=http%3A//www.websocket.org)，如果URL有语法错误，构造函数会抛出异常。

```
// Create new WebSocket connection
var ws = new WebSocket("ws://www.websocket.org");
//测试了下链接不上。
```

![img](https://pic2.zhimg.com/80/v2-52654c7090642d0302d7b06eb5d61a11_720w.webp)

第二个参数是协议名称，是可选的，服务端和客服端使用的协议必须一致，这样收发消息彼此才能理解，你可以定义一个或多个客户端使用的协议，服务端会选择一个来使用，一个客服端和一个服务端之间只能有一个协议。当然都得基于WebSocket，WebSocket的重大好处之一就是基于WebSocket协议的广泛使用，让你的Web能够拥有传统桌面程序那样的能力。

言归正传，我们回到构造函数，在第一次握手之后，和协议的名称一起，客户端会发送一个Sec-WebSocket-Protocol 头，服务端会选择0个或一个协议，响应会带上同样的Sec-WebSocket-Protocol 头，否则会关闭连接。通过协议协商（Protocol negotiation ），我们可以知道给定的WebSocket服务器所支持的协议和版本，然后应用选择协议使用。

```
// Connecting to the server with one protocol called myProtocol
var ws = new WebSocket("ws://echo.websocket.org", "myProtocol");
//myProtocol 是假设的一个定义好的且符合标准的协议。
```

你可以传递一个协议的数组。

```
var echoSocket = new WebSocket("ws://echo.websocket.org", ["com.kaazing.echo","example.imaginary.protocol"])
//服务端会选择其中一个使用
echoSocket.onopen = function(e) {
// Check the protocol chosen by the server
console.log(echoSocket.protocol);
}
```

输出：com.kaazing.ech

协议这个参数有三种。

1.注册协议：根据RFC6455（WebSocket 协议）和IANA被官方注册的标准协议。例如 微软的SOAP。

![img](https://pic4.zhimg.com/80/v2-d799e900f3493fee0b65c893ae1e4b83_720w.webp)

看到两个华为的：

2.开放协议：被广泛使用的标注协议，例如XMPP和STOMP。但没有被正式注册。

3.自定义协议：自己编写和使用的WebSocket的协议。 协议会再后续章节给出详细介绍，下面先看事件、对象和方法以及实例。

### 5.3 WebSocket事件

WebSocket API是纯事件驱动，通过监听事件可以处理到来的数据和改变的链接状态。客户端不需要为了更新数据而轮训服务器。服务端发送数据后，消息和事件会异步到达。WebSocket编程遵循一个异步编程模型，只需要对WebSocket对象增加回调函数就可以监听事件。你也可以使用addEventListener()方法来监听。而一个WebSocket对象分四类不同事件。

#### 5.3.1 open

一旦服务端响应WebSocket连接请求，就会触发open事件。响应的回调函数称为onopen。

```
// Event handler for the WebSocket connection opening
ws.onopen = function(e) {
console.log("Connection open...");
};
```

open事件触发的时候，意味着协议握手结束，WebSocket已经准备好收发数据。如果你的应用收到open事件，就可以确定服务端已经处理了建立连接的请求，且同意和你的应用通信。

#### **5.3.2 Message**

当消息被接受会触发消息事件，响应的回调函数叫做onmessage。如下：

```
// 接受文本消息的事件处理实例：
ws.onmessage = function(e) {
if(typeof e.data === "string"){
console.log("String message received", e, e.data);
} else {
console.log("Other message received", e, e.data);
}
};
```

除了文本消息，WebSocket消息机制还能处理二进制数据，有Blob和ArrayBuffer两种类型，在读取到数据之前需要决定好数据的类型。

```
// 设置二进制数据类型为blob（默认类型）
ws.binaryType = "blob";
// Event handler for receiving Blob messages
ws.onmessage = function(e) {
if(e.data instanceof Blob){
console.log("Blob message received", e.data);
var blob = new Blob(e.data);
}
};
```



```
//ArrayBuffer
ws.binaryType = "arraybuffer";
ws.onmessage = function(e) {
if(e.data instanceof ArrayBuffer){
console.log("ArrayBuffer Message Received", + e.data);
// e.data即ArrayBuffer类型
var a = new Uint8Array(e.data);
}
};
```

#### 5.3.**3 Error**

如果发生意外的失败会触发error事件，相应的函数称为onerror,错误会导致连接关闭。如果你收到一个错误事件，那么你很快会收到一个关闭事件，在关闭事件中也许会告诉你错误的原因。而对错误事件的处理比较适合做重连的逻辑。

```
//异常处理
ws.onerror = function(e) {
console.log("WebSocket Error: " , e);
//Custom function for handling errors
handleErrors(e);
};
```

#### **5.3.4 Close**

不言而喻，当连接关闭的时候回触发这个事件，对应onclose方法，连接关闭之后，服务端和客户端就不能再收发消息。

WebSocket的规范其实还定义了ping和pong 架构（frames），可以用来做keep-alive，心跳，网络状态查询，latency instrumentation（延迟仪表？），但是目前 WebSocket API还没有公布这些特性，尽管浏览器支持了ping，但不会触发ping事件，相反，浏览器会自动响应pong，第八章会将更多关于ping和pong的细节。

当然你可以调用close方法断开与服务端的链接来触发onclose事件：

```
ws.onclose = function(e) {
console.log("Connection closed", e);
};
```

连接失败和成功的关闭握手都会触发关闭事件，WebSocket的对象的readyState属性就代表连接的状态（2代表正在关闭，3代表已经关闭）。关闭事件有三个属性可以用来做异常处理和重获： wasClean,code和reason。wasClean是一个bool值，代表连接是否干净的关闭。 如果是响应服务端的close事件，这个值为true，如果是别的原因，比如因为是底层TCP连接关闭，wasClean为false。code和reason代表关闭连接时服务端发送的状态，这两个属性和给入close方法的code和reason参数是对应的，稍后会描述细节。

### 5.4 WebSocket 方法

**WebSocket 对象有两个方法：send()和close()。**

#### 5.4.1 send()

一旦在服务端和客户端建立了全双工的双向连接，可以使用send方法去发送消息。

```
//发送一个文本消息
ws.send("Hello WebSocket!");
```

当连接是open的时候send()方法传送数据，当连接关闭或获取不到的时候回抛出异常。一个通常的错误是人们喜欢在连接open之前发送消息。如下所示：

```
// 这将不会工作
var ws = new WebSocket("ws://echo.websocket.org")
ws.send("Initial data");
```

正确的姿势如下，应该等待open事件触发后再发送消息。

```
var ws = new WebSocket("ws://echo.websocket.org")
ws.onopen = function(e) {
ws.s
```

如果想通过响应别的事件去发送消息，可以检查readyState属性的值为open的时候来实现。

```
function myEventHandler(data) {
if (ws.readyState === WebSocket.OPEN) {
//open的时候即可发送
ws.send(data);
} else {
// Do something else in this case.
//Possibly ignore the data or enqueue it.
}
}
```

发送二进制数据：

```
// Send a Blob
var blob = new Blob("blob contents");
ws.send(blob);
// Send an ArrayBuffer
var a = new Uint8Array([8,6,7,5,3,0,9]);
ws.send(a.buffer);
```

Blob对象和JavaScript File API一起使用的时候相当有用，可以发送或接受文件，大部分的多媒体文件，图像，视频和音频文件。这一章末尾会结合File API提供读取文件内容来发送WebSocket消息的实例代码。

#### **5.4.2 close()**

使用close方法来关闭连接，如果连接以及关闭，这方法将什么也不做。调用close方法只后，将不能发送数据。

```
ws.close();
```

close方法可以传入两个可选的参数，code（numerical）和reason（string）,以告诉服务端为什么终止连接。第三章讲到关闭握手的时候再详细讨论这两个参数。

```
// 成功结束会话
ws.close(1000, "Closing normally");
//1000是状态码，代表正常结束。
```

### 5.5 WebSocket 属性

WebSocket对象有三个属性，readyState，bufferedAmount和Protocol。

#### **5.5.1 readyState**

WebSocket对象通过只读属性readyState来传达连接状态，它会更加连接状态自动改变。下表展示了readyState属性的四个不同的值。

![img](https://pic1.zhimg.com/80/v2-213323a5a774d488ab5b1df395344728_720w.webp)

了解当前连接的状态有助于我们调试。

#### **5.5.2 bufferedAmount**

有时候需要检查传输数据的大小，尤其是客户端传输大量数据的时候。虽然send()方法会马上执行，但数据并不是马上传输。浏览器会缓存应用流出的数据，你可以使用bufferedAmount属性检查已经进入队列但还未被传输的数据大小。这个值不包含协议框架、操作系统缓存和网络软件的开销。

下面这个例子展示了如何使用bufferedAmount属性每秒更新发送。如果网络不能处理这个频率，它会自适应。

```
// 10k
var THRESHOLD = 10240;
//建立连接
var ws = new WebSocket("ws://echo.websocket.org");
// Listen for the opening event
ws.onopen = function () {
setInterval( function() {
//缓存未满的时候发送
if (ws.bufferedAmount < THRESHOLD) {
ws.send(getApplicationState());
}
}, 1000);
};
//使用bufferedAmount属性发送数据可以避免网络饱和。
```

#### **5.5.3 protocol**

在构造函数中，protocol参数让服务端知道客户端使用的WebSocket协议。而WebSocket对象的这个属性就是指的最终服务端确定下来的协议名称，当服务端没有选择客户端提供的协议或者在连接握手结束之前，这个属性都是空的。

完整实例：

现在我们已经过了一遍WebSocket的构造函数、事件、属性和方法，接下来通过一个完整的实例来学习WebSocket API。实例使用“Echo”服务器：ws://[http://echo.websocket.org](https://link.zhihu.com/?target=http%3A//echo.websocket.org)，它能够接受和返回发过去的数据。这样有助于理解WebSocket API是如何和服务器交互的。

首先，我们先建立连接，让页面展示客户端连接服务端的信息，然后发送、接受消息，最后关闭连接。

```
<h2>Websocket Echo Client</h2>
<div id="output"></div>
```



```
// 初始化连接和事件
        function setup() {
            output = document.getElementById("output");
            ws = new WebSocket("ws://echo.websocket.org/echo");
            // 监听open
            ws.onopen = function (e) {
                log("Connected");
                sendMessage("Hello WebSocket!");
            }
            // 监听close
            ws.onclose = function (e) {
                log("Disconnected: " + e.reason);
            }
            //监听errors
            ws.onerror = function (e) {
                log("Error ");
            }
            // 监听 messages 
            ws.onmessage = function (e) {
                log("Message received: " + e.data);
                //收到消息后关闭
                ws.close();
            }
        }
        // 发送消息
        function sendMessage(msg) {
            ws.send(msg);
            log("Message sent");
        }
        // logging
        function log(s) {
            var p = document.createElement("p");
            p.style.wordWrap = "break-word";
            p.textContent = s;
            output.appendChild(p);
            // Also log information on the javascript console
            console.log(s);
        }
        // Start 
        setup();
```

![img](https://pic1.zhimg.com/80/v2-0e30d090faef031ed71d0302037639f4_720w.webp)

判断浏览器是否支持：

```
if (window.WebSocket){
console.log("This browser supports WebSocket!");
} else {
console.log("This browser does not support WebSocket.");
}
```

![img](https://pic1.zhimg.com/80/v2-cf0c9668301238ff351e3af9d9096400_720w.webp)

## 6.WebSocket语言支持

![img](https://pic3.zhimg.com/80/v2-a8625dcb12e58c913d33b3dda011328e_720w.webp)

- 所有主流浏览器都支持RFC6455。但是具体的WebSocket版本有区别。
- php jetty netty ruby Kaazing nginx python Tomcat Django erlang
- WebSocket浏览器支持
- WebSocket浏览器支持
- netty .net等语言均可以用来实现支持WebSocket的服务器。
- websocket api在浏览器端的广泛实现似乎只是一个时间问题了, 值得注意的是服务器端没有标准的api, 各个实现都有自己的一套api, 并且tcp也没有类似的提案, 所以使用websocket开发服务器端有一定的风险.可能会被锁定在某个平台上或者将来被迫升级。

WebSocket是HTML5出的东西（协议），也就是说HTTP协议没有变化，或者说没关系，但HTTP是不支持持久连接的（长连接，循环连接的不算）。

- 首先HTTP有1.1和1.0之说，也就是所谓的keep-alive，把多个HTTP请求合并为一个，但是Websocket其实是一个新协议，跟HTTP协议基本没有关系，只是为了兼容现有浏览器的握手规范而已，也就是说它是HTTP协议上的一种补充可以通过这样一张图理解：

![img](https://pic1.zhimg.com/80/v2-b7cd97b09822b27c8faf0381ca1aaf40_720w.webp)

- 有交集，但是并不是全部。另外Html5是指的一系列新的API，或- 者说新规范，新技术。Http协议本身只有1.0和1.1，而且跟Html本身没有直接关系。
- 通俗来说，你可以用HTTP 协议 传输非Html 数据 ，就是这样=。=
- 再简单来说， 层级不一样 。

## 7.WebSocket通信

### 7.1 连接握手

连接握手分为两个步骤：请求和应答。WebSocket利用了HTTP协议来建立连接，使用的是HTTP的协议升级机制。

#### 7.1.1 请求

一个标准的HTTP请求，格式如下：

![img](https://pic4.zhimg.com/80/v2-cbf2967bda07d89726555b1e8988f307_720w.webp)

请求头的具体格式定义参见**[Request-Line格式](https://link.zhihu.com/?target=https%3A//link.csdn.net/%3Ftarget%3Dhttps%3A//www.w3.org/Protocols/rfc2616/rfc2616-sec5.html)**。

**请求header中的字段解析:**

![img](https://pic2.zhimg.com/80/v2-f1b5808b3593ad7ec8767fac2bf32fb1_720w.webp)

协议升级机制

Origin

所有浏览器将会发送一个 Origin请求头。 你可以将这个请求头用于安全方面（检查是否是同一个域，白名单/ 黑名单等），如果你不喜欢这个请求发起源，你可以发送一个403 Forbidden。需要注意的是非浏览器只能发送一个模拟的 Origin。大多数应用会拒绝不含这个请求头的请求。

Sec-WebSocket-Key

由客户端随机生成的，提供基本的防护，防止恶意或者无意的连接。

#### 7.1.2 应答

**返回字段解析：**

![img](https://pic4.zhimg.com/80/v2-8b4c6f01a74a6f13da2b3e6b5e0c84b3_720w.webp)

Connection

可以参见 rfc7230 6.1 和 rfc7230 6.7。

Sec-WebSocket-Accept

它通过客户端发送的Sec-WebSocket-Key 计算出来。计算方法：

将 Sec-WebSocket-Key 跟 258EAFA5-E914-47DA-95CA-C5AB0DC85B11 拼接；

通过 SHA1 计算出摘要，并转成 base64 字符串。

Sec-WebSocket-Key / Sec-WebSocket-Accept 的主要作用还是为了避免一些网络通信过程中，一些非期待的数据包，”乱入“进来，导致一些错误的响应，并不能用于实现登录认证和数据安全，这些功能还需要应用层自己实现。

### 7.2 数据传输（双工）

WebSocket 以 frame 为单位传输数据, frame 是客户端和服务端数据传输的最小单元。当一条消息过长时, 通信方可以将该消息拆分成多个 frame 发送, 接收方收到以后重新拼接、解码从而还原出完整的消息, 在 WebSocket 中, frame 有多种类型, frame 的类型由 frame 头部的 Opcode 字段指示, WebSocket frame 的结构如下所示:

![img](https://pic1.zhimg.com/80/v2-b1c04497d138689f6b9e4e62a29c07d0_720w.webp)

该结构的字段语义如下:

FIN, 长度为 1 比特, 该标志位用于指示当前的 frame 是消息的最后一个分段, 因为 WebSocket 支持将长消息切分为若干个 frame 发送, 切分以后, 除了最后一个 frame, 前面的 frame 的 FIN 字段都为 0, 最后一个 frame 的 FIN 字段为 1, 当然, 若消息没有分段, 那么一个 frame 便包含了完成的消息, 此时其 FIN 字段值为 1。

RSV 1 ~ 3, 这三个字段为保留字段, 只有在 WebSocket 扩展时用, 若不启用扩展, 则该三个字段应置为 1, 若接收方收到 RSV 1 ~ 3 不全为 0 的 frame, 并且双方没有协商使用 WebSocket 协议扩展, 则接收方应立即终止 WebSocket 连接。

Opcode, 长度为 4 比特, 该字段将指示 frame 的类型, RFC 6455 定义的 Opcode 共有如下几种:

- 0x0, 代表当前是一个 continuation frame，既被切分的长消息的每个分片frame
- 0x1, 代表当前是一个 text frame
- 0x2, 代表当前是一个 binary frame
- 0x3 ~ 7, 目前保留, 以后将用作更多的非控制类 frame
- 0x8, 代表当前是一个 connection close, 用于关闭 WebSocket 连接
- 0x9, 代表当前是一个 ping frame
- 0xA, 代表当前是一个 pong frame
- 0xB ~ F, 目前保留, 以后将用作更多的控制类 frame

1. Mask, 长度为 1 比特, 该字段是一个标志位, 用于指示 frame 的数据 (Payload) 是否使用掩码掩盖, RFC 6455 规定当且仅当由客户端向服务端发送的 frame, 需要使用掩码覆盖, 掩码覆盖主要为了解决代理缓存污染攻击 (更多细节见 RFC 6455 Section 10.3)。
2. Payload Len, 以字节为单位指示 frame Payload 的长度, 该字段的长度可变, 可能为 7 比特, 也可能为 7 + 16 比特, 也可能为 7 + 64 比特. 具体来说, 当 Payload 的实际长度在 [0, 125] 时, 则 Payload Len 字段的长度为 7 比特, 它的值直接代表了 Payload 的实际长度; 当 Payload 的实际长度为 126 时, 则 Payload Len 后跟随的 16 位将被解释为 16-bit 的无符号整数, 该整数的值指示 Payload 的实际长度; 当 Payload 的实际长度为 127 时, 其后的 64 比特将被解释为 64-bit 的无符号整数, 该整数的值指示 Payload 的实际长度。
3. Masking-key, 该字段为可选字段, 当 Mask 标志位为 1 时, 代表这是一个掩码覆盖的 frame, 此时 Masking-key 字段存在, 其长度为 32 位, RFC 6455 规定所有由客户端发往服务端的 frame 都必须使用掩码覆盖, 即对于所有由客户端发往服务端的 frame, 该字段都必须存在, 该字段的值是由客户端使用熵值足够大的随机数发生器生成, 关于掩码覆盖, 将下面讨论, 若 Mask 标识位 0, 则 frame 中将设置该字段 (注意是不设置该字段, 而不仅仅是不给该字段赋值)。
4. Payload, 该字段的长度是任意的, 该字段即为 frame 的数据部分, 若通信双方协商使用了 WebSocket 扩展, 则该扩展数据 (Extension data) 也将存放在此处, 扩展数据 + 应用数据, 它们的长度和便为 Payload Len 字段指示的值。

以下是一个客户端和服务端相互传递文本消息的示例

![img](https://pic3.zhimg.com/80/v2-e82dd7828a0c15f5d3217768ad289f1a_720w.webp)

**其中模拟了长消息被切分为多个帧（continuation frame）的例子。**

### 7.3 关闭请求

**关闭相对简单，由客户端或服务端发送关闭帧，即可完成关闭。**

![img](https://pic4.zhimg.com/80/v2-845db9777fd53b43e20ad9a1b68860e7_720w.webp)



## 8.WebSocket协议进一步理解

WebSocket是一种在单个TCP连接上进行全双工通信的协议。WebSocket通信协议于2011年被IETF定为标准RFC 6455，并由RFC7936补充规范。WebSocket API也被W3C定为标准。

### **8.1 WebSocket是什么**

WebSocket 协议在2008年诞生，2011年成为国际标准。主流浏览器都已经支持。

WebSocket 是一种全新的协议。它将 TCP 的 Socket（套接字）应用在了网页上，从而使通信双方建立起一个保持在活动状态连接通道，并且属于全双工通信。

WebSocket 协议在2008年诞生，2011年成为国际标准。主流浏览器都已经支持。WebSocket 协议借用 HTTP协议 的 101 switch protocol 来达到协议转换，从HTTP协议切换WebSocket通信协议。它的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话。

### 8.2 WebSocket出现之前的实时技术

轮询：最早的一种实现实时 Web 应用的方案。客户端以一定的时间间隔向服务端发出请求，以频繁请求的方式来保持客户端和服务器端的通信。

长轮询：长轮询也采用轮询的方式，不过采取的是阻塞模型，客户端发起连接后，如果没消息，就一直不返回Response给客户端。直到有消息才返回，返回完之后，客户端再次建立连接，周而复始。

其他方式：如xhr-streaming、隐藏iframe、ActiveX控件、SSE。

![img](https://pic2.zhimg.com/80/v2-3bfabd9b252c79a407a2d2ae48bea36d_720w.webp)

轮询技术非真正实时技术。使用 Ajax 方式模拟实时效果，每次客户端和服务器端交互，都是一次 HTTP 的请求和应答过程，且每次的 HTTP 请求和应答都带有完整 HTTP 头信息，增加传输的数据量。需构建两个http连接。客户端和服务器端编程实现比较复杂，为模拟真实的实时效果，需构造两个 HTTP 连接来模拟客户端和服务器的双向通信，一个连接用来处理客户端到服务器端的数据传输，一个连接用来处理服务器端到客户端的数据传输，增加了编程实现的复杂度、服务器端的负载，制约了应用系统的扩展性。

### 8.3 WebSocket应用场景

BS架构下的即时通讯、游戏等应用需要客户端与服务端间的双向通信，而HTTP的请求/响应模型并不适合这种场景。会存在一定的问题：

- 服务器端被迫提供两类接口，一类提供给客户端轮询新消息，一类提供给客户端推送消息给服务器端。
- HTTP协议有较多的额外开销，每次发送消息都会有一个HTTP header信息，而且如果不用Keep-Alive每次还都要握手。
- 客户端的脚本比如JS可能还需要跟踪整个过程，发送一个消息后，我可能需要跟踪这个消息的返回。

Websocket出现使得浏览器提供socket的支持成为可能，从而在浏览器和服务器之间建立一条基于tcp的双向连接通道，web开发人员可以很方便的利用websocket构建实时web应用。**WebSocket适用于以下场景:**

- 在线聊天场景：例如qq聊天、淘宝与客服聊天、在线客服等等。这种场景都是需要实时的接收服务器推送的内容。
- 协同办公：例如腾讯在线文档，腾讯的在线文档是支持多人编辑的，在excel中，一旦有人修改就要立即同步给所有人。
- 直播弹幕：例如虎牙、斗鱼等各大直播平台，在直播时都是有弹幕的，遇到一些精彩片段时，往往会有弹幕刷屏。在这种情况下使用WebSocket会有一个更好的用户体验。
- 位置共享：例如微信里位置共享，这种场景需要用户实时的共享自己的位置给服务器，服务器收到位置信息后，要实时的推送给其它共享者的，实时性要求较高；百度地图导航系统，在自己位置到达某个地方之后，语音会立即播报前面道路情况，比如上高架、下地道、拐弯、直行、学校慢行等等。这种场景实时性特别高，汽车速度很快，延迟1秒钟，可能就错过了最佳提醒时机。
- 其他通过定义WebSocket子协议的扩展支持：例如sip、mqtt、xmpp、stomp等。

### 8.4 WebSocket协议栈

![img](https://pic4.zhimg.com/80/v2-aaf587560b040bb6e76c9dd2718fddeb_720w.webp)

**WebSocket是基于TCP的应用层协议。**需要特别注意的是：虽然WebSocket协议在建立连接时会使用HTTP协议，但这并意味着WebSocket协议是基于HTTP协议实现的。

### 8.5 WebSocket与HTTP的区别

![img](https://pic3.zhimg.com/80/v2-00ee437654f960e507d6809adfce415e_720w.webp)

- 通信方式不同。WebSocket是双向通信模式，客户端与服务器之间只有在握手阶段是使用HTTP协议的“请求-响应”模式交互，而一旦连接建立之后的通信则使用双向模式交互，不论是客户端还是服务端都可以随时将数据发送给对方；而HTTP协议则至始至终都采用“请求-响应”模式进行通信。也正因为如此，HTTP协议的通信效率没有WebSocket高。
- 协议格式不同。HTTP协议的一个数据包就是一条完整的消息；而WebSocket客户端与服务端通信的最小单位是帧，由1个或多个帧组成一条完整的消息。即：发送端将消息切割成多个帧，并发送给服务端；服务端接收消息帧，并将关联的帧重新组装成完整的消息。

### 8.6 WebSocket握手过程

**客户端到服务端:**

![img](https://pic4.zhimg.com/80/v2-bf7136dc1931af0645a6a70b6de9856b_720w.webp)

- GET ws://localhost…… HTTP/1.1 ：打开阶段握手，使用http1.1协议。
- Upgrade：websocket，表示请求为特殊http请求，请求的目的是要将客户端和服务端的通信协议从http升级为websocket。
- Sec-websocket-key：Base64 encode 的值，是浏览器随机生成的。客户端向服务端提供的握手信息。

**服务端到客户端:**

![img](https://pic2.zhimg.com/80/v2-326704f41a81df809c07fade43fc3e21_720w.webp)

- 101状态码：表示切换协议。服务器根据客户端的请求切换到Websocket协议。
- Sec-websocket-accept: 将请求头中的Set-websocket-key添加字符串并做SHA-1加密后做Base64编码，告知客户端服务器能够发起websocket连接。

**客户端发起连接的约定:**

- 如果请求为wss,则在TCP建立后，进行TLS连接建立。
- 请求的方式必须为GET，HTTP版本至少为HTTP1.1。
- 请求头中必须有Host。
- 请求头中必须有Upgrade，取值必须为websocket。
- 请求头中必须有Connection，取值必须为Upgrade。
- 请求头中必须有Sec-WebSocket-Key，取值为16字节随机数的Base64编码。
- 请求头中必须有Sec-WebSocket-Version，取值为13。
- 请求头中可选Sec-WebSocket-Protocol，取值为客户端期望的一个或多个子协议(多个以逗号分割)。
- 请求头中可选Sec-WebSocket-Extensitons，取值为子协议支持的扩展集(一般是压缩方式)。
- 可以包含cookie、Authorization等HTTP规范内合法的请求头。

**客户端检查服务端的响应:**

- 服务端返回状态码为101代表升级成功，否则判定连接失败。
- 响应头中缺少Upgrade或取值不是websocket，判定连接失败。
- 响应头中缺少Connection或取值不是Upgrade，判定连接失败。
- 响应头中缺少Sec-WebSocket-Accept或取值非法（其值为请求头中的Set-websocket-key添加字符串并做SHA-1加密后做Base64编码），判定连接失败。
- 响应头中有Sec-WebSocket-Extensions,但取值不是请求头中的子集，判定连接失败。
- 响应头中有Sec-WebSocket-Protocol,但取值不是请求头中的子集，判定连接失败。

**服务端处理客户端连接:**

- 服务端根据请求中的Sec-WebSocket-Protocol 字段，选择一个子协议返回，如果不返回，表示不同意请求的任何子协议。如果请求中未携带，也不返回。
- 如果建立连接成功，返回状态码为101。
- 响应头Connection设置为Upgrade。
- 响应头Upgrade设置为websocket。
- Sec-WebSocket-Accpet根据请求头Set-websocket-key计算得到，计算方式为：Set-websocket-key的值添加字符串： 258EAFA5-E914-47DA-95CA-C5AB0DC85B11并做SHA-1加密后得到16进制表示的字符串，将每两位当作一个字节进行分隔，得到字节数组，对字节数组做Base64编码。

### 8.7 WebSocket帧格式

**WebSocket通信流程如下:**

![img](https://pic4.zhimg.com/80/v2-7f140f2953205aae25737fdb5ad88f53_720w.webp)

**Websocket帧格式如下:**

![img](https://pic4.zhimg.com/80/v2-4faf01bacfaa0cf69e19f07e4507df4f_720w.webp)

**第一部分:**

![img](https://pic1.zhimg.com/80/v2-9f9dc9fedf00d66c215b027282fe0758_720w.webp)

FIN：1位，用于描述消息是否结束，如果为1则该消息为消息尾部，如果为零则还有后续数据包。

RSV1,RSV2,RSV3：各1位，用于扩展定义，如果没有扩展约定的情况则必须为0。

OPCODE：4位，用于表示消息接收类型，如果接收到未知的opcode，接收端必须关闭连接。

OPCODE说明:

- 0x0表示附加数据帧，当前数据帧为分片的数据帧。
- 0x1表示文本数据帧，采用UTF-8编码。
- 0x2表示二进制数据帧。
- 0x3-7暂时无定义，为以后的非控制帧保留。
- 0x8表示连接关闭。
- 0x9表示ping。
- 0xA表示pong。
- 0xB-F暂时无定义，为以后的控制帧保留。

**第二部分:**

![img](https://pic1.zhimg.com/80/v2-14d9365cdb2da722fec007d231022b30_720w.webp)

- MASK：1位，用于标识PayloadData是否经过掩码处理。服务端发送给客户端的数据帧不能使用掩码，客户端发送给服务端的数据帧必须使用掩码。如果一个帧的数据使用了掩码，那么在Maksing-key部分必须是一个32个bit位的掩码，用来给服务端解码数据。
- Payload len：数据的长度：默认位7个bit位。如果数据的长度小于125个字节（注意：是字节）则用默认的7个bit来标示数据的长度。如果数据的长度为126个字节，则用后面相邻的2个字节来保存一个16bit位的无符号整数作为数据的长度。如果数据的长度大于126个字节，则用后面相邻的8个字节来保存一个64bit位的无符号整数作为数据的长度。
- payload len本来是只能用7bit来表达的，也就是最多一个frame的payload只能有127个字节，为了表示更大的长度，给出的解决方案是添加扩展payload len字段。当payload实际长度超过126（包括），但在2^16-1长度内，则将payload len置为126，payload的实际长度由长为16bit的extended payload length来表达。当payload实际长度超过216（包括），但在264-1长度内，则将payload置为127，payload的实际长度由长为64bit的extended payload length来表达。

**第三部分:**

![img](https://pic2.zhimg.com/80/v2-fa5ad43529f61a048f8a782f4fefca6d_720w.webp)

数据掩码：如果MASK设置位0，则该部分可以省略，如果MASK设置位1，则Masking-key是一个32位的掩码。用来解码客户端发送给服务端的数据帧。

**第四部分:**

![img](https://pic2.zhimg.com/80/v2-8c6a2c34c41df996508fa31c7c0c2089_720w.webp)

数据：该部分，也是最后一部分，是帧真正要发送的数据，可以是任意长度。

### 8.8 WebSocket分片传输

![img](https://pic4.zhimg.com/80/v2-6ba4dd10e8d88f7edeb3d5c74a02a907_720w.webp)

控制帧可能插在一个Message的多个分片之间，但一个Message的分片不能交替传输(除非有扩展特别定义)。

控制帧不可分片。

分片需要按照分送方提交顺序传递给接收方，但由于IP路由特性，实际并不能保证顺序到达。

控制帧包括:

- Close：用于关闭连接，可以携带数据，表示关闭原因。
- Ping：可以携带数据。
- Pong：用于Keep-alive，返回最近一次Ping中的数据，可以只发送Pong帧，做单向心跳。

连接关闭时状态码说明:

![img](https://pic1.zhimg.com/80/v2-29bd022bc176f35566ade86b15e3d858_720w.webp)

### 8.9 WebSocket相关扩展

Stomp：

STOMP是基于帧的协议，它的前身是TTMP协议（一个简单的基于文本的协议），专为消息中间件设计。是属于消息队列的一种协议, 和AMQP, JMS平级。它的简单性恰巧可以用于定义websocket的消息体格式. STOMP协议很多MQ都已支持, 比如RabbitMq, ActiveMq。生产者（发送消息）、消息代理、消费者（订阅然后收到消息）。

![img](https://pic1.zhimg.com/80/v2-3f439d6a7256805b37c6ba5b8fb3a7cc_720w.webp)

\2. SockJs：

SockJS是一个浏览器JavaScript库，它提供了一个类似于网络的对象。SockJS提供了一个连贯的、跨浏览器的Javascript API，它在浏览器和web服务器之间创建了一个低延迟、全双工、跨域通信通道。

SockJS的一大好处在于提供了浏览器兼容性。优先使用原生WebSocket，如果在不支持websocket的浏览器中，会自动降为轮询的方式。 除此之外，spring也对socketJS提供了支持。

\3. [Socket.io](https://link.zhihu.com/?target=http%3A//Socket.io)：

[http://Socket.io](https://link.zhihu.com/?target=http%3A//Socket.io)实际上是WebSocket的父集，[http://Socket.io](https://link.zhihu.com/?target=http%3A//Socket.io)封装了WebSocket和轮询等方法，会根据情况选择方法来进行通讯。

[http://Sockei.io](https://link.zhihu.com/?target=http%3A//Sockei.io)最早由Node.js实现，Node.js提供了高效的服务端运行环境，但由于Browser对HTML5的支持不一，为了兼容所有浏览器，提供实时的用户体验，并为开发者提供客户端与服务端一致的编程体验，于是[http://Socket.io](https://link.zhihu.com/?target=http%3A//Socket.io)诞生了。Java模仿Node.js实现了Java版的[http://Netty-socket.io](https://link.zhihu.com/?target=http%3A//Netty-socket.io)库。

[http://Socket.io](https://link.zhihu.com/?target=http%3A//Socket.io)将WebSocket和Polling机制以及其它的实时通信方式封装成通用的接口，并在服务端实现了这些实时机制相应代码，包括：AJAX Long Polling、Adobe Flash Socket、AJAX multipart streaming、Forever Iframem、JSONP Polling。

## 9.WebSocket 能解决什么问题

工程师应该是以解决问题为主的，如果不会解决问题，只会伸手，必然不会长远，有思考，才会有突破，才能高效的处理事情，所以 websocket 到底解决了什么问题呢？它存在的价值是什么？

这还是得从HTTP说起，大家应该都很熟悉这门协议，我们简单说一下它的特点：

•三次握手、四次挥手 的方式建立连接和关闭连接

•支持长连接和短连接两种连接方式

•有同源策略的限制（端口，协议，域名）

•单次 请求-响应 机制，只支持单向通信

其中最鸡肋的就是最后一个特点，单向通信，什么意思呐？ 就是说只能由一方发起请求（客户端），另一方响应请求（服务端），而且每一次的请求都是一个单独的事件，请求之间还无法具有关联性，也就是说我上个请求和下个请求完全是隔离的，无法具有连续性。

也许你觉得这样的说法比较难懂，我们来举一个栗子：

每个人都打过电话吧，电话打通后可以一直聊天是不是觉得很舒服啊，这是一种全双工的通信方式，双方都可以主动传递信息。彼此的聊天也具有连续性。我们简单把这种方式理解为 websocket 协议支持的方式。

如果打电话变成了 HTTP 那种方式呢？ 那就不叫打电话了，而是联通爸爸的智能语音助手了，我们知道客户端和服务端本身的身份并不是固定的，只要你可以发起通信，就可以充当客户端，能响应请求，就可以当做服务端，但是在HTTP的世界里一般来说，客户端（大多数情况下是浏览器）和服务器一般是固定的，我们打电话 去查话费，会询问要人工服务还是智能助手，如果选了助手，你只要问她问题，她就会找对应的答案来回答你（响应你），一般都是简单的业务，你不问她也不会跟你闲聊，主动才有故事啊！

但是实际上有很多的业务是需要双方都有主动性的，半双工的模式肯定是不够用的，例如聊天室，跟机器人聊天没意思啊，又例如主动推送，我无聊的时候手都不想点屏幕，你能不能主动一点给我推一些好玩的信息过来。

只要做过前后端分离的同学应该都被跨域的问题折磨过。浏览器的这种同源策略，会导致 不同端口/不同域名/不同协议 的请求会有限制，当然这问题前后端都能处理，然而 websocket 就没有这种要求，他支持任何域名或者端口的访问（协议固定了只能是 ws/wss) ,所以它让人用的更加舒服

所以，上面 HTTP 存在的这些问题，websocket 都能解决！！！

## 10.WebSocket工作原理

主动是 websocket 的一大特点，像之前如果客户端想知道服务端对某个事件的处理进度，就只能通过轮训( Poll )的方式去询问，十分的耗费资源，会存在十分多的无效请求。下面我简单说推送技术的三种模型区别：

- •pull (主动获取) 即客户端主动发起请求，获取消息
- •poll (周期性主动获取) 即周期性的主动发起请求，获取消息
- •push (主动推送) 服务端主动推送消息给客户端

pull 和 poll 的唯一区别只在于周期性，但是很明显周期性的去询问，对业务来说清晰度很高，这也是为什么很多小公司都是基于轮训的方式去处理业务，因为简单嘛，能力不够机器来撑。这也是很多公司都会面临的问题，如果业务达到了瓶颈，使劲的堆机器，如果用新技术或者更高级的作法，开发成本和维护成本也会变高，还不如简单一点去增加机器配置。

如果两个人需要通话，首先需要建立一个连接，而且必须是一个长链接，大家都不希望讲几句话就得重新打吧，根据上面说的，websocket 会复用之前 HTTP 建立好的长链接，然后再进行升级，所以他和轮训的区别大致如下所示：

![img](https://pic1.zhimg.com/80/v2-306c1e8fb9159bed67e11d249bd9cf74_720w.webp)

图片省去了建立连接的过程，我们可以发现，基于轮训的方式，必须由客户端手动去请求，才会有响应，而基于 websocket 协议，不再需要你主动约妹子了，妹子也可以主动去约你，这才是公平的世界。

为了更好的阐述这个连接的原理，可以使用swoole 自带的 创建websocket 的功能进行测试，服务端代码如下，如果连接不上，可以看看是不是检查一下端口开放情况（iptables/filewall)和网络的连通性，代码如下：

```
//创建websocket服务器对象，监听0.0.0.0:9501端口
$ws = new Swoole\WebSocket\Server("0.0.0.0", 9501);

//监听WebSocket连接打开事件
$ws->on('open', function ($ws, $request) {
    var_dump($request->fd, $request->get, $request->server); //request 对象包含请求的相关信息
    //$ws->push($request->fd, "hello, welcome\n");
});

//监听WebSocket消息事件
$ws->on('message', function ($ws, $frame) {  // frame 是存储信息的变量，也就是传输帧
    echo "Message: {$frame->data}\n";
    $ws->push($frame->fd, "server: {$frame->data}");
});

//监听WebSocket连接关闭事件
$ws->on('close', function ($ws, $fd) { // fd 是客户端的标志
    echo "client-{$fd} is closed\n";
});

$ws->start(); // 启动这个进程
```

我们可以发现，相比于 HTTP 的头部，websocket 的数据结构十分的简单小巧，没有像 HTTP 协议一样老是带着笨重的头部，这一设计让websocket的报文可以在体量上更具优势，所以传输效率来说更高 。

当然，我们传输的文本也不能在大街上裸跑啊，既然 HTTP 都有衣服穿了（HTTPS），websocket（ws) 当然也有 （wss）。

在以前的文章我们也简单聊过 HTTPS 是个什么东西，大家不了解可以去翻一下之前的文章，总的来说就是使用了非对称加密算法进行了对称加密密钥的传输，后续采用对称加密解密的方式进行数据安全处理。

如果你的业务需要支撑双全工的通信，那么 websocket 便是一个很不错的选择。网上大多数关于 websocket 的文章，大多是基于前端学习者的角度，他们使用 Chrome 的console 的调试实验，本篇文章更多是基于后端开发者的角度。希望对你有所帮助。

## 11.进一步解析什么是WebSocket协议

### 11.1websocket特点

1.websocket优点

- 保持连接状态：websocket需要先创建连接，使其成为有状态的协议。
- 更好支持二进制：定义了二进制帧，增加安全性。
- 支持扩展：定义了扩展，可以自己实现部分部分自定义。
- 压缩效果好：可以沿用上下文的内容，有更好的压缩效果。

2.websocket缺点

- 开发要求高： 前端后端都增加了一定的难度。
- 推送消息相对复杂。
- HTTP协议已经很成熟，现今websocket则太新了一点。

### 11.2websocket协议通信过程

**协议有两个部分：handshake（握手）和 data transfer（数据传输）。**

1.handshake

#### 11.2.1 客户端

客户端握手报文是在HTTP的基础上发送一次HTTP协议升级请求。

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

Sec-WebSocket-Key 是由浏览器随机生成的，提供基本的防护，防止恶意或者无意的连接。

Sec-WebSocket-Version 表示 WebSocket 的版本，最初 WebSocket 协议太多，不同厂商都有自己的协议版本，不过现在已经定下来了。如果服务端不支持该版本，需要返回一个 Sec-WebSocket-Versionheader，里面包含服务端支持的版本号。

#### 11.2.2 服务端

服务端响应握手也是在HTTP协议基础上回应一个Switching Protocols。

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

Linux下对应实现代码，注释在代码中。

```
int websocket_handshake(struct qsevent *ev)
{
    char linebuf[128];
    int index = 0;
    char sec_data[128] = {0};
    char sec_accept[32] = {0};
    do
    {
        memset(linebuf, 0, sizeof(linebuf));//清空以暂存一行报文
        index = readline(ev->buffer, index, linebuf);//获取一行报文
        if(strstr(linebuf, "Sec-WebSocket-Key"))//如果一行报文里面包括了Sec-WebSocket-Key
        {
            strcat(linebuf, GUID);//和GUID连接起来
            SHA1(linebuf+WEBSOCK_KEY_LENGTH, strlen(linebuf+WEBSOCK_KEY_LENGTH), sec_data);//SHA1
            base64_encode(sec_data, strlen(sec_data), sec_accept);//base64编码
            memset(ev->buffer, 0, MAX_BUFLEN);//清空服务端数据缓冲区

            ev->length = sprintf(ev->buffer,//组装握手响应报文到数据缓冲区，下一步有进行下发
                                 "HTTP/1.1 101 Switching Protocols\r\n"
                                 "Upgrade: websocket\r\n"
                                 "Connection: Upgrade\r\n"
                                 "Sec-websocket-Accept: %s\r\n\r\n", sec_accept);
            break;   
        }
    }while(index != -1 && (ev->buffer[index] != '\r') || (ev->buffer[index] != '\n'));//遇到空行之前
    return 0;
}
```

#### 11.2.3 data transfer

先看数据包格式。

![img](https://pic1.zhimg.com/80/v2-a5012dc34aa781ae2222dac5e080b34c_720w.webp)

FIN：指示这是消息中的最后一个片段。第一个片段也可能是最后的片段。

RSV1, RSV2, RSV3：一般情况下全为 0。当客户端、服务端协商采用 WebSocket 扩展时，这三个标志位可以非0，且值的含义由扩展进行定义。如果出现非零的值，且并没有采用 WebSocket 扩展，连接出错。

opcode：操作代码。

```
%x0：表示一个延续帧。当 Opcode 为 0 时，表示本次数据传输采用了数据分片，当前收到的数据帧为其中一个数据分片；
%x1：表示这是一个文本帧（frame）；
%x2：表示这是一个二进制帧（frame）；
%x3-7：保留的操作代码，用于后续定义的非控制帧；
%x8：表示连接断开；
%x9：表示这是一个 ping 操作；
%xA：表示这是一个 pong 操作；
%xB-F：保留的操作代码，用于后续定义的控制帧。
```

- mask：是否需要掩码。
- Payload length: 7bit or 7 + 16bit or 7 + 64bit

```
 表示数据载荷的长度
x 为 0~126：数据的长度为 x 字节；
x 为 126：后续 2 个字节代表一个 16 位的无符号整数，该无符号整数的值为数据的长度；
x 为 127：后续 8 个字节代表一个 64 位的无符号整数（最高位为 0），该无符号整数的值为数据的长度。
```

payload data：消息体。 下面是服务端的代码实现：

```
#define GUID            "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
enum			
{ 
    WS_HANDSHAKE = 0,		//握手
    WS_TANSMISSION = 1,		//通信
    WS_END = 2,				//end
};

typedef struct _ws_ophdr{
    unsigned char opcode:4,
    rsv3:1,
    rsv2:1,
    rsv1:1,
    fin:1;
    unsigned char pl_len:7,
    mask:1;
}ws_ophdr;//协议前两个字节

typedef struct _ws_head_126{
    unsigned short payload_lenght;
    char mask_key[4];
}ws_head_126;//协议mask和消息体长度


/*解码*/
void websocket_umask(char *payload, int length, char *mask_key)
{
    int i = 0;
    for( ; i<length; i++)
    payload[i] ^= mask_key[i%4];//异或
}

int websocket_transmission(struct qsevent *ev)
{
    ws_ophdr *ophdr = (ws_ophdr*)ev->buffer;//协议前两个自己
    printf("ws_recv_data length=%d\n", ophdr->pl_len);
    if(ophdr->pl_len <126)//如果消息体长度小于126
    {
        char * payload = ev->buffer + sizeof(ws_ophdr) + 4;//获取消息地址
        if(ophdr->mask)//如果消息是掩码
        {
            websocket_umask(payload, ophdr->pl_len, ev->buffer+2);//解码，异或
            printf("payload:%s\n", payload);
        }
        printf("payload : %s\n", payload);//消息回显
    }
    else if (hdr->pl_len == 126) {
		ws_head_126 *hdr126 = ev->buffer + sizeof(ws_ophdr);
	} else {
		ws_head_127 *hdr127 = ev->buffer + sizeof(ws_ophdr);
	}
    return 0;
}

int websocket_request(struct qsevent *ev)
{
    if(ev->status_machine == WS_HANDSHAKE)
    {
        websocket_handshake(ev);//握手
        ev->status_machine = WS_TANSMISSION;//设置标志位
    }else if(ev->status_machine == WS_TANSMISSION){
        websocket_transmission(ev);//通信
    }
    return 0;
}
```

**代码是基于reactor百万并发服务器框架实现的。**

### 11.3.epoll反应堆模型下实现http协议

![img](https://pic2.zhimg.com/80/v2-a39426bc462aa425c544f091ef5ad3f5_720w.webp)

1.客户端结构体

```
struct qsevent{
    int fd;				//clientfd
    int events;			//事件：读、写或异常
    int status;			//是否位于epfd红黑监听树上
    void *arg;			//参数
    long last_active;	//上次数据收发的事件

    int (*callback)(int fd, int event, void *arg);	//回调函数，单回调，后面修改成多回调
    unsigned char buffer[MAX_BUFLEN];				//数据缓冲区
    int length;										//数据长度

    /*http param*/
    int method;						//http协议请求头部
    char resource[MAX_BUFLEN];		//请求的资源
    int ret_code;					//响应状态码
};
```

[2.int](https://link.zhihu.com/?target=http%3A//2.int) http_response(struct qsevent *ev)

当客户端发送tcp连接时，服务端的listenfd会触发输入事件会调用ev->callback即accept_cb回调函数响应连接并获得clientfd，连接之后，http数据报文发送上来，服务端的clientfd触发输入事件会调用ev->callback即recv_cb回调函数进行数据接收，并解析http报文。

```
int http_request(struct qsevent *ev)
{
    char linebuf[1024] = {0};//用于从buffer中获取每一行的请求报文
    int idx = readline(ev->buffer, 0, linebuf);//读取第一行请求方法，readline函数，后面介绍
    if(strstr(linebuf, "GET"))//strstr判断是否存在GET请求方法
    {
        ev->method = HTTP_METHOD_GET;//GET方法表示客户端需要获取资源

        int i = 0;
        while(linebuf[sizeof("GET ") + i] != ' ')i++;//跳过空格
        linebuf[sizeof("GET ") + i] = '\0';
        sprintf(ev->resource, "./%s/%s", HTTP_METHOD_ROOT, linebuf+sizeof("GET "));//将资源的名字以文件路径形式存储在ev->resource中
        printf("resource:%s\n", ev->resource);//回显
    }
    else if(strstr(linebuf, "POST"))//POST的请求方法，暂时没写，方法差不多
    {}
    return 0;
}
```

[http://3.int](https://link.zhihu.com/?target=http%3A//3.int) http_response(struct qsevent *ev)

服务器对客户端的响应报文数据进行http封装储存在buffer中，事件触发时在send_cb回调函数发送给客户端。详细解释请看代码注释。

```
int http_response(struct qsevent *ev)
{
    if(ev == NULL)return -1;
    memset(ev->buffer, 0, MAX_BUFLEN);//清空缓冲区准备储存报文

    printf("resource:%s\n", ev->resource);//resource：客户端请求的资源文件，通过http_reques函数获取
    int filefd = open(ev->resource, O_RDONLY);//只读方式打开获得文件句柄
    if(filefd == -1)//获取失败则发送404 NOT FOUND
    {
        ev->ret_code = 404;//404状态码
        ev->length = sprintf(ev->buffer,//将下面数据传入ev->buffer
        					 /***状态行***/
        					 /*版本号 状态码 状态码描述 */
                             "HTTP/1.1 404 NOT FOUND\r\n"
                             /***消息报头***/
                             /*获取当前时间*/
                             "date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
                             /*响应正文类型；              编码方式*/
                             "Content-Type: text/html;charset=ISO-8859-1\r\n"
                             /*响应正文长度          空行*/
                             "Content-Length: 85\r\n\r\n"
                             /***响应正文***/
                             "<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n");
    }
    else 
    {
        struct stat stat_buf;			//文件信息
        fstat(filefd, &stat_buf);		//fstat通过文件句柄获取文件信息
        if(S_ISDIR(stat_buf.st_mode))	//如果文件是一个目录
        {

            printf(ev->buffer, //同上，将404放入buffer中
                   "HTTP/1.1 404 Not Found\r\n"
                   "Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
                   "Content-Type: text/html;charset=ISO-8859-1\r\n"
                   "Content-Length: 85\r\n\r\n"
                   "<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n" );

        } 
        else if (S_ISREG(stat_buf.st_mode)) //如果文件是存在
        {

            ev->ret_code = 200;		//200状态码

            ev->length = sprintf(ev->buffer, //length记录长度，buffer储存响应报文
                                 "HTTP/1.1 200 OK\r\n"
                                 "Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
                                 "Content-Type: text/html;charset=ISO-8859-1\r\n"
                                 "Content-Length: %ld\r\n\r\n", 
                                 stat_buf.st_size );//文件长度储存在stat_buf.st_size中

        }
        return ev->length;//返回报文长度
    }
}
```

4.总代码

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>

#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <time.h>

#include <sys/stat.h>
#include <sys/sendfile.h>

#define HTTP_METHOD_ROOT    "html"

#define MAX_BUFLEN          4096
#define MAX_EPOLLSIZE       1024
#define MAX_EPOLL_EVENTS    1024

#define HTTP_METHOD_GET     0
#define HTTP_METHOD_POST    1

typedef int (*NCALLBACK)(int, int, void*);

struct qsevent{
    int fd;
    int events;
    int status;
    void *arg;
    long last_active;

    int (*callback)(int fd, int event, void *arg);
    unsigned char buffer[MAX_BUFLEN];
    int length;

    /*http param*/
    int method;
    char resource[MAX_BUFLEN];
    int ret_code;
};

struct qseventblock{
    struct qsevent *eventsarrry;
    struct qseventblock *next;
};

struct qsreactor{
    int epfd;
    int blkcnt;
    struct qseventblock *evblk;
};

int recv_cb(int fd, int events, void *arg);
int send_cb(int fd, int events, void *arg);
struct qsevent *qsreactor_idx(struct qsreactor *reactor, int sockfd);

int readline(char *allbuf, int idx, char *linebuf)
{
    int len = strlen(allbuf);    
    for( ; idx<len; idx++ )
    {
        if(allbuf[idx] == '\r' && allbuf[idx+1] == '\n')
        return idx+2;
        else
        *(linebuf++) = allbuf[idx];
    }
    return -1;
}

void qs_event_set(struct qsevent *ev, int fd, NCALLBACK callback, void *arg)
{
    ev->events = 0;
    ev->fd = fd;
    ev->arg = arg;
    ev->callback = callback;
    ev->last_active = time(NULL);
    return;
}

int qs_event_add(int epfd, int events, struct qsevent *ev)
{
    struct epoll_event epv = {0, {0}};;
    epv.events = ev->events = events;
    epv.data.ptr = ev;

    if(ev->status == 1)
    { 
        if(epoll_ctl(epfd, EPOLL_CTL_MOD, ev->fd, &epv) < 0)
        {
            perror("EPOLL_CTL_MOD error\n");
            return -1;
        }
    }
    else if(ev->status == 0)
    {
        if(epoll_ctl(epfd, EPOLL_CTL_ADD, ev->fd, &epv) < 0)
        {
            perror("EPOLL_CTL_ADD error\n");    
            return -2;
        }
        ev->status = 1;
    }
    return 0;
}

int qs_event_del(int epfd, struct qsevent *ev)
{
    struct epoll_event epv = {0, {0}};
    if(ev->status != 1)
    return -1;
    ev->status = 0;
    epv.data.ptr = ev;
    if((epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, &epv)))
    {
        perror("EPOLL_CTL_DEL error\n");
        return -1;
    }
    return 0;
}

int sock(short port)
{
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(fd, F_SETFL, O_NONBLOCK);

    struct sockaddr_in ser_addr;
    memset(&ser_addr, 0, sizeof(ser_addr));
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_port = htons(port);

    bind(fd, (struct sockaddr*)&ser_addr, sizeof(ser_addr));

    if(listen(fd, 20) < 0)
    perror("listen error\n");

    printf("listener[%d] lstening..\n", fd);
    return fd;
}

int http_request(struct qsevent *ev)
{
    char linebuf[1024] = {0};
    int idx = readline(ev->buffer, 0, linebuf);
    if(strstr(linebuf, "GET"))
    {
        ev->method = HTTP_METHOD_GET;

        int i = 0;
        while(linebuf[sizeof("GET ") + i] != ' ')i++;
        linebuf[sizeof("GET ") + i] = '\0';
        sprintf(ev->resource, "./%s/%s", HTTP_METHOD_ROOT, linebuf+sizeof("GET "));
        printf("resource:%s\n", ev->resource);
    }
    else if(strstr(linebuf, "POST"))
    {}
    return 0;
}

int http_response(struct qsevent *ev)
{
    if(ev == NULL)return -1;
    memset(ev->buffer, 0, MAX_BUFLEN);

    printf("resource:%s\n", ev->resource);

    int filefd = open(ev->resource, O_RDONLY);
    if(filefd == -1)
    {
        ev->ret_code = 404;
        ev->length = sprintf(ev->buffer,
                             "HTTP/1.1 404 NOT FOUND\r\n"
                             "date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
                             "Content-Type: text/html;charset=ISO-8859-1\r\n"
                             "Content-Length: 85\r\n\r\n"
                             "<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n");
    }
    else 
    {
        struct stat stat_buf;
        fstat(filefd, &stat_buf);
        if(S_ISDIR(stat_buf.st_mode))
        {

            printf(ev->buffer, 
                   "HTTP/1.1 404 Not Found\r\n"
                   "Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
                   "Content-Type: text/html;charset=ISO-8859-1\r\n"
                   "Content-Length: 85\r\n\r\n"
                   "<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n" );

        } 
        else if (S_ISREG(stat_buf.st_mode)) 
        {

            ev->ret_code = 200;

            ev->length = sprintf(ev->buffer, 
                                 "HTTP/1.1 200 OK\r\n"
                                 "Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
                                 "Content-Type: text/html;charset=ISO-8859-1\r\n"
                                 "Content-Length: %ld\r\n\r\n", 
                                 stat_buf.st_size );

        }
        return ev->length;
    }
}

int qsreactor_init(struct qsreactor *reactor)
{
    if(reactor == NULL)
    return -1;
    memset(reactor, 0, sizeof(struct qsreactor));
    reactor->epfd = epoll_create(1);
    if(reactor->epfd <= 0)
    {
        perror("epoll_create error\n");
        return -1;
    }
    struct qseventblock *block = (struct qseventblock*)malloc(sizeof(struct qseventblock));
    if(block == NULL)
    {
        printf("blockinit malloc error\n");
        close(reactor->epfd);
        return -2;
    }
    memset(block, 0, sizeof(block));

    struct qsevent *evs = (struct qsevent*)malloc(MAX_EPOLLSIZE * sizeof(struct qsevent));
    if(evs == NULL)
    {
        printf("evsnit malloc error\n");
        close(reactor->epfd);
        return -3;
    }
    memset(evs, 0, sizeof(evs));

    block->next = NULL;
    block->eventsarrry = evs;

    reactor->blkcnt = 1;
    reactor->evblk = block;
    return 0;
}

int qsreactor_alloc(struct qsreactor *reactor)
{
    if(reactor == NULL)return -1;
    if(reactor->evblk == NULL)return -1;
    struct qseventblock *tailblock = reactor->evblk;
    while(tailblock->next != NULL)
    tailblock = tailblock->next;
    struct qseventblock *newblock = (struct qseventblock*)malloc(sizeof(struct qseventblock));
    if(newblock == NULL)
    {
        printf("newblock alloc error\n");
        return -1;
    }
    memset(newblock, 0, sizeof(newblock));

    struct qsevent *neweventarray = (struct qsevent*)malloc(sizeof(struct qsevent) * MAX_EPOLLSIZE);
    if(neweventarray == NULL)
    {
        printf("neweventarray malloc error\n");
        return -1;
    }
    memset(neweventarray, 0, sizeof(neweventarray));

    newblock->eventsarrry = neweventarray;
    newblock->next = NULL;

    tailblock->next = newblock;
    reactor->blkcnt++;

    return 0;
}

struct qsevent *qsreactor_idx(struct qsreactor *reactor, int sockfd)
{
    int index = sockfd / MAX_EPOLLSIZE;
    while(index >= reactor->blkcnt)qsreactor_alloc(reactor);
    int i=0;
    struct qseventblock *idxblock = reactor->evblk;
    while(i++<index && idxblock != NULL)
    idxblock = idxblock->next;

    return &idxblock->eventsarrry[sockfd%MAX_EPOLLSIZE];
}

int qsreactor_destory(struct qsreactor *reactor)
{
    close(reactor->epfd);
    free(reactor->evblk);
    reactor = NULL;
    return 0;
}

int qsreactor_addlistener(struct qsreactor *reactor, int sockfd, NCALLBACK acceptor)
{
    if(reactor == NULL)return -1;
    if(reactor->evblk == NULL)return -1;

    struct qsevent *event = qsreactor_idx(reactor, sockfd);
    qs_event_set(event, sockfd, acceptor, reactor);
    qs_event_add(reactor->epfd, EPOLLIN, event);

    return 0;
}

int send_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    struct qsevent   *ev = qsreactor_idx(reactor, fd);

    http_response(ev);
    int ret = send(fd, ev->buffer, ev->length, 0);
    if(ret < 0)
    {
        qs_event_del(reactor->epfd, ev);
        printf("clent[%d] ", fd);
        perror("send error\n");
        close(fd);
    }
    else if(ret > 0)
    {
        if(ev->ret_code == 200) 
        {
            int filefd = open(ev->resource, O_RDONLY);
            struct stat stat_buf;
            fstat(filefd, &stat_buf);

            sendfile(fd, filefd, NULL, stat_buf.st_size);
            close(filefd);   
        }
        printf("send to client[%d]:%s", fd, ev->buffer);
        qs_event_del(reactor->epfd, ev);
        qs_event_set(ev, fd, recv_cb, reactor);
        qs_event_add(reactor->epfd, EPOLLIN, ev);
    }
    return ret;
}

int recv_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    struct qsevent *ev = qsreactor_idx(reactor, fd);

    int len = recv(fd, ev->buffer, MAX_BUFLEN, 0);
    qs_event_del(reactor->epfd, ev);
    if(len > 0)
    {
        ev->length = len;
        ev->buffer[len] = '\0';

        printf("client[%d]:%s", fd, ev->buffer);

        http_request(ev);

        qs_event_del(reactor->epfd, ev);
        qs_event_set(ev, fd, send_cb, reactor);
        qs_event_add(reactor->epfd, EPOLLOUT, ev);
    }
    else if(len == 0)
    {
        qs_event_del(reactor->epfd, ev);
        close(fd);
        printf("client[%d] close\n", fd);
    }
    else
    {
        qs_event_del(reactor->epfd, ev);
        printf("client[%d]", fd);
        perror("reacv error,\n");
        close(fd);
    }
    return 0;
}

int accept_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    if(reactor == NULL)return -1;

    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);

    int clientfd;


    if((clientfd = accept(fd, (struct sockaddr*)&client_addr, &len)) == -1)
    {
        if(errno != EAGAIN && errno != EINTR)
        {}
        perror("accept error\n");
        return -1;
    }

    int flag = 0;
    if((flag = fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0)
    {
        printf("fcntl noblock error, %d\n",MAX_BUFLEN);
        return -1;
    }
    struct qsevent *event = qsreactor_idx(reactor, clientfd);

    qs_event_set(event, clientfd, recv_cb, reactor);
    qs_event_add(reactor->epfd, EPOLLIN, event);

    printf("new connect [%s:%d], pos[%d]\n",
           inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), clientfd);

    return 0;
}

int qsreactor_run(struct qsreactor *reactor)
{
    if(reactor == NULL)
    return -1;
    if(reactor->evblk == NULL)
    return -1;
    if(reactor->epfd < 0)
    return -1;

    struct epoll_event events[MAX_EPOLL_EVENTS + 1];
    while(1)
    {
        int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);

        if(nready < 0)
        {
            printf("epoll_wait error\n");
            continue;
        }
        for(int i=0; i<nready; i++)
        {
            struct qsevent *ev = (struct qsevent*)events[i].data.ptr;
            if((events[i].events & EPOLLIN) && (ev->events & EPOLLIN))
            {
                ev->callback(ev->fd, events[i].events, ev->arg);    
            }
            if((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT))
            {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }
}

int main(int argc, char **argv)
{
    unsigned short port = atoi(argv[1]);

    int sockfd = sock(port);


    struct qsreactor *reactor = (struct qsreactor*)malloc(sizeof(struct qsreactor));
    qsreactor_init(reactor);

    qsreactor_addlistener(reactor, sockfd, accept_cb);
    qsreactor_run(reactor);

    qsreactor_destory(reactor);
    close(sockfd);
}
```

5.epoll反应堆模型下实现websocket协议

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>

#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <time.h>

#include <sys/stat.h>
#include <sys/sendfile.h>

#include <openssl/sha.h>
#include <openssl/pem.h>
#include <openssl/bio.h>
#include <openssl/evp.h>

#define MAX_BUFLEN          4096
#define MAX_EPOLLSIZE       1024
#define MAX_EPOLL_EVENTS    1024

#define GUID            "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

enum
{ 
    WS_HANDSHAKE = 0,
    WS_TANSMISSION = 1,
    WS_END = 2,
};

typedef struct _ws_ophdr{
    unsigned char opcode:4,
    rsv3:1,
    rsv2:1,
    rsv1:1,
    fin:1;
    unsigned char pl_len:7,
    mask:1;
}ws_ophdr;

typedef struct _ws_head_126{
    unsigned short payload_lenght;
    char mask_key[4];
}ws_head_126;

typedef struct _ws_head_127{
    long long payload_lenght;
    char mask_key[4];
}ws_head_127;

typedef int (*NCALLBACK)(int, int, void*);

struct qsevent{
    int fd;
    int events;
    int status;
    void *arg;
    long last_active;

    int (*callback)(int fd, int event, void *arg);
    unsigned char buffer[MAX_BUFLEN];
    int length;

    /*websocket param*/
    int status_machine;
};

struct qseventblock{
    struct qsevent *eventsarrry;
    struct qseventblock *next;
};

struct qsreactor{
    int epfd;
    int blkcnt;
    struct qseventblock *evblk;
};

int recv_cb(int fd, int events, void *arg);
int send_cb(int fd, int events, void *arg);
struct qsevent *qsreactor_idx(struct qsreactor *reactor, int sockfd);

int readline(char *allbuf, int idx, char *linebuf)
{
    int len = strlen(allbuf);

    for(;idx < len;idx ++) {
        if (allbuf[idx] == '\r' && allbuf[idx+1] == '\n') {
            return idx+2;
        } else {
            *(linebuf++) = allbuf[idx];
        }
    }
    return -1;
}

int base64_encode(char *in_str, int in_len, char *out_str)
{    
    BIO *b64, *bio;    
    BUF_MEM *bptr = NULL;    
    size_t size = 0;    

    if (in_str == NULL || out_str == NULL)        
    return -1;    

    b64 = BIO_new(BIO_f_base64());    
    bio = BIO_new(BIO_s_mem());    
    bio = BIO_push(b64, bio);

    BIO_write(bio, in_str, in_len);    
    BIO_flush(bio);    

    BIO_get_mem_ptr(bio, &bptr);    
    memcpy(out_str, bptr->data, bptr->length);    
    out_str[bptr->length-1] = '\0';    
    size = bptr->length;    

    BIO_free_all(bio);    
    return size;
}

#define WEBSOCK_KEY_LENGTH  19

int websocket_handshake(struct qsevent *ev)
{
    char linebuf[128];
    int index = 0;
    char sec_data[128] = {0};
    char sec_accept[32] = {0};
    do
    {
        memset(linebuf, 0, sizeof(linebuf));
        index = readline(ev->buffer, index, linebuf);
        if(strstr(linebuf, "Sec-WebSocket-Key"))
        {
            strcat(linebuf, GUID);
            SHA1(linebuf+WEBSOCK_KEY_LENGTH, strlen(linebuf+WEBSOCK_KEY_LENGTH), sec_data);
            base64_encode(sec_data, strlen(sec_data), sec_accept);
            memset(ev->buffer, 0, MAX_BUFLEN);

            ev->length = sprintf(ev->buffer,
                                 "HTTP/1.1 101 Switching Protocols\r\n"
                                 "Upgrade: websocket\r\n"
                                 "Connection: Upgrade\r\n"
                                 "Sec-websocket-Accept: %s\r\n\r\n", sec_accept);
            break;   
        }
    }while(index != -1 && (ev->buffer[index] != '\r') || (ev->buffer[index] != '\n'));
    return 0;
}

void websocket_umask(char *payload, int length, char *mask_key)
{
    int i = 0;
    for( ; i<length; i++)
    payload[i] ^= mask_key[i%4];
}

int websocket_transmission(struct qsevent *ev)
{
    ws_ophdr *ophdr = (ws_ophdr*)ev->buffer;
    printf("ws_recv_data length=%d\n", ophdr->pl_len);
    if(ophdr->pl_len <126)
    {
        char * payload = ev->buffer + sizeof(ws_ophdr) + 4;
        if(ophdr->mask)
        {
            websocket_umask(payload, ophdr->pl_len, ev->buffer+2);
            printf("payload:%s\n", payload);
        }
        memset(ev->buffer, 0, ev->length);
        strcpy(ev->buffer, "00ok");
    }
    return 0;
}

int websocket_request(struct qsevent *ev)
{
    if(ev->status_machine == WS_HANDSHAKE)
    {
        websocket_handshake(ev);
        ev->status_machine = WS_TANSMISSION;
    }else if(ev->status_machine == WS_TANSMISSION){
        websocket_transmission(ev);
    }
    return 0;
}
void qs_event_set(struct qsevent *ev, int fd, NCALLBACK callback, void *arg)
{
    ev->events = 0;
    ev->fd = fd;
    ev->arg = arg;
    ev->callback = callback;
    ev->last_active = time(NULL);
    return;
}

int qs_event_add(int epfd, int events, struct qsevent *ev)
{
    struct epoll_event epv = {0, {0}};;
    epv.events = ev->events = events;
    epv.data.ptr = ev;

    if(ev->status == 1)
    { 
        if(epoll_ctl(epfd, EPOLL_CTL_MOD, ev->fd, &epv) < 0)
        {
            perror("EPOLL_CTL_MOD error\n");
            return -1;
        }
    }
    else if(ev->status == 0)
    {
        if(epoll_ctl(epfd, EPOLL_CTL_ADD, ev->fd, &epv) < 0)
        {
            perror("EPOLL_CTL_ADD error\n");    
            return -2;
        }
        ev->status = 1;
    }
    return 0;
}

int qs_event_del(int epfd, struct qsevent *ev)
{
    struct epoll_event epv = {0, {0}};
    if(ev->status != 1)
    return -1;
    ev->status = 0;
    epv.data.ptr = ev;
    if((epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, &epv)))
    {
        perror("EPOLL_CTL_DEL error\n");
        return -1;
    }
    return 0;
}

int sock(short port)
{
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(fd, F_SETFL, O_NONBLOCK);

    struct sockaddr_in ser_addr;
    memset(&ser_addr, 0, sizeof(ser_addr));
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_port = htons(port);

    bind(fd, (struct sockaddr*)&ser_addr, sizeof(ser_addr));

    if(listen(fd, 20) < 0)
    perror("listen error\n");

    printf("listener[%d] lstening..\n", fd);
    return fd;
}

int qsreactor_init(struct qsreactor *reactor)
{
    if(reactor == NULL)
    return -1;
    memset(reactor, 0, sizeof(struct qsreactor));
    reactor->epfd = epoll_create(1);
    if(reactor->epfd <= 0)
    {
        perror("epoll_create error\n");
        return -1;
    }
    struct qseventblock *block = (struct qseventblock*)malloc(sizeof(struct qseventblock));
    if(block == NULL)
    {
        printf("blockinit malloc error\n");
        close(reactor->epfd);
        return -2;
    }
    memset(block, 0, sizeof(block));

    struct qsevent *evs = (struct qsevent*)malloc(MAX_EPOLLSIZE * sizeof(struct qsevent));
    if(evs == NULL)
    {
        printf("evsnit malloc error\n");
        close(reactor->epfd);
        return -3;
    }
    memset(evs, 0, sizeof(evs));

    block->next = NULL;
    block->eventsarrry = evs;

    reactor->blkcnt = 1;
    reactor->evblk = block;
    return 0;
}

int qsreactor_alloc(struct qsreactor *reactor)
{
    if(reactor == NULL)return -1;
    if(reactor->evblk == NULL)return -1;
    struct qseventblock *tailblock = reactor->evblk;
    while(tailblock->next != NULL)
    tailblock = tailblock->next;
    struct qseventblock *newblock = (struct qseventblock*)malloc(sizeof(struct qseventblock));
    if(newblock == NULL)
    {
        printf("newblock alloc error\n");
        return -1;
    }
    memset(newblock, 0, sizeof(newblock));

    struct qsevent *neweventarray = (struct qsevent*)malloc(sizeof(struct qsevent) * MAX_EPOLLSIZE);
    if(neweventarray == NULL)
    {
        printf("neweventarray malloc error\n");
        return -1;
    }
    memset(neweventarray, 0, sizeof(neweventarray));

    newblock->eventsarrry = neweventarray;
    newblock->next = NULL;

    tailblock->next = newblock;
    reactor->blkcnt++;

    return 0;
}

struct qsevent *qsreactor_idx(struct qsreactor *reactor, int sockfd)
{
    int index = sockfd / MAX_EPOLLSIZE;
    while(index >= reactor->blkcnt)qsreactor_alloc(reactor);
    int i=0;
    struct qseventblock *idxblock = reactor->evblk;
    while(i++<index && idxblock != NULL)
    idxblock = idxblock->next;

    return &idxblock->eventsarrry[sockfd%MAX_EPOLLSIZE];
}

int qsreactor_destory(struct qsreactor *reactor)
{
    close(reactor->epfd);
    free(reactor->evblk);
    reactor = NULL;
    return 0;
}

int qsreactor_addlistener(struct qsreactor *reactor, int sockfd, NCALLBACK acceptor)
{
    if(reactor == NULL)return -1;
    if(reactor->evblk == NULL)return -1;

    struct qsevent *event = qsreactor_idx(reactor, sockfd);
    qs_event_set(event, sockfd, acceptor, reactor);
    qs_event_add(reactor->epfd, EPOLLIN, event);

    return 0;
}

int send_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    struct qsevent   *ev = qsreactor_idx(reactor, fd);

    int ret = send(fd, ev->buffer, ev->length, 0);
    if(ret < 0)
    {
        qs_event_del(reactor->epfd, ev);
        printf("clent[%d] ", fd);
        perror("send error\n");
        close(fd);
    }
    else if(ret > 0)
    {
        printf("send to client[%d]:\n%s\n", fd, ev->buffer);
        qs_event_del(reactor->epfd, ev);
        qs_event_set(ev, fd, recv_cb, reactor);
        qs_event_add(reactor->epfd, EPOLLIN, ev);
    }
    return ret;
}

int recv_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    struct qsevent *ev = qsreactor_idx(reactor, fd);

    int len = recv(fd, ev->buffer, MAX_BUFLEN, 0);
    qs_event_del(reactor->epfd, ev);
    if(len > 0)
    {
        ev->length = len;
        ev->buffer[len] = '\0';

        printf("client[%d]:\n%s\n", fd, ev->buffer);

        websocket_request(ev);

        qs_event_del(reactor->epfd, ev);
        qs_event_set(ev, fd, send_cb, reactor);
        qs_event_add(reactor->epfd, EPOLLOUT, ev);
    }
    else if(len == 0)
    {
        qs_event_del(reactor->epfd, ev);
        close(fd);
        printf("client[%d] close\n", fd);
    }
    else
    {
        qs_event_del(reactor->epfd, ev);
        printf("client[%d]", fd);
        perror("reacv error,\n");
        close(fd);
    }
    return 0;
}

int accept_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    if(reactor == NULL)return -1;

    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);

    int clientfd;


    if((clientfd = accept(fd, (struct sockaddr*)&client_addr, &len)) == -1)
    {
        if(errno != EAGAIN && errno != EINTR)
        {}
        perror("accept error\n");
        return -1;
    }

    int flag = 0;
    if((flag = fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0)
    {
        printf("fcntl noblock error, %d\n",MAX_BUFLEN);
        return -1;
    }
    struct qsevent *event = qsreactor_idx(reactor, clientfd);

    event->status_machine = WS_HANDSHAKE;

    qs_event_set(event, clientfd, recv_cb, reactor);
    qs_event_add(reactor->epfd, EPOLLIN, event);

    printf("new connect [%s:%d], pos[%d]\n",
           inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), clientfd);

    return 0;
}

int qsreactor_run(struct qsreactor *reactor)
{
    if(reactor == NULL)
    return -1;
    if(reactor->evblk == NULL)
    return -1;
    if(reactor->epfd < 0)
    return -1;

    struct epoll_event events[MAX_EPOLL_EVENTS + 1];
    while(1)
    {
        int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);

        if(nready < 0)
        {
            printf("epoll_wait error\n");
            continue;
        }
        for(int i=0; i<nready; i++)
        {
            struct qsevent *ev = (struct qsevent*)events[i].data.ptr;
            if((events[i].events & EPOLLIN) && (ev->events & EPOLLIN))
            {
                ev->callback(ev->fd, events[i].events, ev->arg);    
            }
            if((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT))
            {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }
}

int main(int argc, char **argv)
{
    unsigned short port = atoi(argv[1]);

    int sockfd = sock(port);


    struct qsreactor *reactor = (struct qsreactor*)malloc(sizeof(struct qsreactor));
    qsreactor_init(reactor);

    qsreactor_addlistener(reactor, sockfd, accept_cb);
    qsreactor_run(reactor);

    qsreactor_destory(reactor);
    close(sockfd);
}
```

6.C1000K reactor模型，epoll实现，连接并回发一段数据，测试正常

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>

#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <time.h>

#include <sys/stat.h>
#include <sys/sendfile.h>

#define MAX_BUFLEN          4096
#define MAX_EPOLLSIZE       1024
#define MAX_EPOLL_EVENTS    1024

typedef int (*NCALLBACK)(int, int, void*);

struct qsevent{
    int fd;
    int events;
    int status;
    void *arg;
    long last_active;

    int (*callback)(int fd, int event, void *arg);
    unsigned char buffer[MAX_BUFLEN];
    int length;
};

struct qseventblock{
    struct qsevent *eventsarrry;
    struct qseventblock *next;
};

struct qsreactor{
    int epfd;
    int blkcnt;
    struct qseventblock *evblk;
};

int recv_cb(int fd, int events, void *arg);
int send_cb(int fd, int events, void *arg);
struct qsevent *qsreactor_idx(struct qsreactor *reactor, int sockfd);

void qs_event_set(struct qsevent *ev, int fd, NCALLBACK callback, void *arg)
{
    ev->events = 0;
    ev->fd = fd;
    ev->arg = arg;
    ev->callback = callback;
    ev->last_active = time(NULL);
    return;
}

int qs_event_add(int epfd, int events, struct qsevent *ev)
{
    struct epoll_event epv = {0, {0}};;
    epv.events = ev->events = events;
    epv.data.ptr = ev;

    if(ev->status == 1)
    { 
        if(epoll_ctl(epfd, EPOLL_CTL_MOD, ev->fd, &epv) < 0)
        {
            perror("EPOLL_CTL_MOD error\n");
            return -1;
        }
    }
    else if(ev->status == 0)
    {
        if(epoll_ctl(epfd, EPOLL_CTL_ADD, ev->fd, &epv) < 0)
        {
            perror("EPOLL_CTL_ADD error\n");    
            return -2;
        }
        ev->status = 1;
    }
    return 0;
}

int qs_event_del(int epfd, struct qsevent *ev)
{
    struct epoll_event epv = {0, {0}};
    if(ev->status != 1)
        return -1;
    ev->status = 0;
    epv.data.ptr = ev;
    if((epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, &epv)))
    {
        perror("EPOLL_CTL_DEL error\n");
        return -1;
    }
    return 0;
}

int sock(short port)
{
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(fd, F_SETFL, O_NONBLOCK);

    struct sockaddr_in ser_addr;
    memset(&ser_addr, 0, sizeof(ser_addr));
    ser_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    ser_addr.sin_family = AF_INET;
    ser_addr.sin_port = htons(port);

    bind(fd, (struct sockaddr*)&ser_addr, sizeof(ser_addr));

    if(listen(fd, 20) < 0)
        perror("listen error\n");

    printf("listener[%d] lstening..\n", fd);
    return fd;
}

int qsreactor_init(struct qsreactor *reactor)
{
    if(reactor == NULL)
    return -1;
    memset(reactor, 0, sizeof(struct qsreactor));
    reactor->epfd = epoll_create(1);
    if(reactor->epfd <= 0)
    {
        perror("epoll_create error\n");
       return -1;
    }
    struct qseventblock *block = (struct qseventblock*)malloc(sizeof(struct qseventblock));
    if(block == NULL)
    {
        printf("blockinit malloc error\n");
        close(reactor->epfd);
        return -2;
    }
    memset(block, 0, sizeof(block));

    struct qsevent *evs = (struct qsevent*)malloc(MAX_EPOLLSIZE * sizeof(struct qsevent));
    if(evs == NULL)
    {
        printf("evsnit malloc error\n");
        close(reactor->epfd);
        return -3;
    }
    memset(evs, 0, sizeof(evs));

    block->next = NULL;
    block->eventsarrry = evs;

    reactor->blkcnt = 1;
    reactor->evblk = block;
    return 0;
}

int qsreactor_alloc(struct qsreactor *reactor)
{
    if(reactor == NULL)return -1;
    if(reactor->evblk == NULL)return -1;
    struct qseventblock *tailblock = reactor->evblk;
    while(tailblock->next != NULL)
    tailblock = tailblock->next;
    struct qseventblock *newblock = (struct qseventblock*)malloc(sizeof(struct qseventblock));
    if(newblock == NULL)
    {
        printf("newblock alloc error\n");
        return -1;
    }
    memset(newblock, 0, sizeof(newblock));

    struct qsevent *neweventarray = (struct qsevent*)malloc(sizeof(struct qsevent) * MAX_EPOLLSIZE);
    if(neweventarray == NULL)
    {
        printf("neweventarray malloc error\n");
        return -1;
    }
    memset(neweventarray, 0, sizeof(neweventarray));

    newblock->eventsarrry = neweventarray;
    newblock->next = NULL;

    tailblock->next = newblock;
    reactor->blkcnt++;

    return 0;
}

struct qsevent *qsreactor_idx(struct qsreactor *reactor, int sockfd)
{
    int index = sockfd / MAX_EPOLLSIZE;
    while(index >= reactor->blkcnt)qsreactor_alloc(reactor);
    int i=0;
    struct qseventblock *idxblock = reactor->evblk;
    while(i++<index && idxblock != NULL)
        idxblock = idxblock->next;
    
    return &idxblock->eventsarrry[sockfd%MAX_EPOLLSIZE];
}

int qsreactor_destory(struct qsreactor *reactor)
{
    close(reactor->epfd);
    free(reactor->evblk);
    reactor = NULL;
    return 0;
}

int qsreactor_addlistener(struct qsreactor *reactor, int sockfd, NCALLBACK acceptor)
{
    if(reactor == NULL)return -1;
    if(reactor->evblk == NULL)return -1;

    struct qsevent *event = qsreactor_idx(reactor, sockfd);
    qs_event_set(event, sockfd, acceptor, reactor);
    qs_event_add(reactor->epfd, EPOLLIN, event);

    return 0;
}

int send_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    struct qsevent   *ev = qsreactor_idx(reactor, fd);

    int ret = send(fd, ev->buffer, ev->length, 0);
    if(ret < 0)
    {
        qs_event_del(reactor->epfd, ev);
        printf("clent[%d] ", fd);
        perror("send error\n");
        close(fd);
    }
    else if(ret > 0)
    {
        printf("send to client[%d]:%s", fd, ev->buffer);
        qs_event_del(reactor->epfd, ev);
        qs_event_set(ev, fd, recv_cb, reactor);
        qs_event_add(reactor->epfd, EPOLLIN, ev);
    }
    return ret;
}

int recv_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    struct qsevent *ev = qsreactor_idx(reactor, fd);

    int len = recv(fd, ev->buffer, MAX_BUFLEN, 0);
    qs_event_del(reactor->epfd, ev);
    if(len > 0)
    {
        ev->length = len;
        ev->buffer[len] = '\0';

        printf("client[%d]:%s", fd, ev->buffer);
   
        qs_event_del(reactor->epfd, ev);
        qs_event_set(ev, fd, send_cb, reactor);
        qs_event_add(reactor->epfd, EPOLLOUT, ev);
    }
    else if(len == 0)
    {
        qs_event_del(reactor->epfd, ev);
        close(fd);
        printf("client[%d] close\n", fd);
    }
    else
    {
        qs_event_del(reactor->epfd, ev);
        printf("client[%d]", fd);
        perror("reacv error,\n");
        close(fd);
    }
    return 0;
}

int accept_cb(int fd, int events, void *arg)
{
    struct qsreactor *reactor = (struct qsreactor*)arg;
    if(reactor == NULL)return -1;

    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);

    int clientfd;


    if((clientfd = accept(fd, (struct sockaddr*)&client_addr, &len)) == -1)
    {
        if(errno != EAGAIN && errno != EINTR)
        {}
        perror("accept error\n");
        return -1;
    }

    int flag = 0;
    if((flag = fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0)
    {
        printf("fcntl noblock error, %d\n",MAX_BUFLEN);
        return -1;
    }
    struct qsevent *event = qsreactor_idx(reactor, clientfd);

    qs_event_set(event, clientfd, recv_cb, reactor);
    qs_event_add(reactor->epfd, EPOLLIN, event);

    printf("new connect [%s:%d], pos[%d]\n",
           inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), clientfd);

    return 0;
}

int qsreactor_run(struct qsreactor *reactor)
{
    if(reactor == NULL)
       return -1;
    if(reactor->evblk == NULL)
       return -1;
    if(reactor->epfd < 0)
       return -1;

    struct epoll_event events[MAX_EPOLL_EVENTS + 1];
    while(1)
    {
        int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);

        if(nready < 0)
        {
            printf("epoll_wait error\n");
            continue;
        }
        for(int i=0; i<nready; i++)
        {
            struct qsevent *ev = (struct qsevent*)events[i].data.ptr;
            if((events[i].events & EPOLLIN) && (ev->events & EPOLLIN))
            {
                ev->callback(ev->fd, events[i].events, ev->arg);    
            }
            if((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT))
            {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }
}

int main(int argc, char **argv)
{
    unsigned short port = atoi(argv[1]);

    int sockfd = sock(port);


    struct qsreactor *reactor = (struct qsreactor*)malloc(sizeof(struct qsreactor));
    qsreactor_init(reactor);
       
    qsreactor_addlistener(reactor, sockfd, accept_cb);
    qsreactor_run(reactor);

    qsreactor_destory(reactor);
    close(sockfd);
}
```

原文链接：https://zhuanlan.zhihu.com/p/581974844