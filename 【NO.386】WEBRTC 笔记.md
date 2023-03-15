# 【NO.386】WEBRTC 笔记

## 1.WEBRTC 是什么

它的全称是WEB Real-time communication。一开始我还觉得是一种通讯技术。这里的communication主要是人与人之间的，因此它解决了在网页视频、音频的播放和获取的问题。它的目标是但愿用户之间直接通讯，而不是经过服务器来进行交互。简单地说就是在浏览器上实现视频通话，并且最好不须要中央服务器。html

你们应该仔细看看这个[教程](http://www.noobyard.com/link?url=https://www.html5rocks.com/en/tutorials/webrtc/basics/) ，我但愿这篇笔记能够更快地帮助你们理解，说明一下比较容易困惑的点，少走一些弯路，而不是取代这篇教程。html5

## 2.核心技术

1. [getUserMedia()](http://www.noobyard.com/link?url=https://webrtc.github.io/samples/src/content/getusermedia/gum/) : 获取视频和音频。git
2. [MediaRecorder](http://www.noobyard.com/link?url=https://webrtc.github.io/samples/src/content/getusermedia/record/) : 记录视频和音频。github
3. [RTCPeerConnection](http://www.noobyard.com/link?url=https://webrtc.github.io/samples/src/content/peerconnection/pc1/) : 创建视频流。web
4. [RTCDataChannel](http://www.noobyard.com/link?url=https://webrtc.github.io/samples/src/content/datachannel/basic/) : 创建数据流。浏览器

## 3.实际问题

然而在现实中网络是不通畅的，2个浏览器之间没法直接创建链接，甚至都没法发现对方。为此须要额外的技术来完成链接。服务器

1. [ICE](http://www.noobyard.com/link?url=https://www.html5rocks.com/en/tutorials/webrtc/basics/#ice) 这个框架应该是嵌入浏览器内部的，咱们并不须要了解太多的细节。网络
2. [signaling](http://www.noobyard.com/link?url=https://www.html5rocks.com/en/tutorials/webrtc/basics/#toc-signaling) 就个人理解，这个至关于媒人，来帮助2个浏览器来创建链接。框架

## 4.创建链接

[JSEP](http://www.noobyard.com/link?url=http://tools.ietf.org/html/draft-ietf-rtcweb-jsep-00)：
![JavaScript Session Establishment Protocol](https://ewr1.vultrobjects.com/imgur2/000/004/556/742_592_7b5.jpg)socket

1. 首先建立 RTCPeerConnection 对象，仅仅是初始化。
2. 使用 createOffer/createAnswer 来交换[SDP](http://www.noobyard.com/link?url=http://en.wikipedia.org/wiki/Session_Description_Protocol)，sdp中包含网络信息，RTCPeerConnection 对象得以创建链接。
3. 激活onicecandidate完成链接。

WEBRTC没有规定createOffer/createAnswer时使用的协议，所以signaling server 只要能够与浏览器交换SDP便可。能够用socket.io/wensocket等通讯技术把createOffer/createAnswer中的SDP给送到对方手里就行了。

------

下面我将用一个简单的例子来讲明链接是如何创建的。
为了更好地说明信号服务器的做用，我把它直接给拿掉了。取而代之的是一块公告牌。
在`sendMessage`和`receiveMsg`中，将要发送的信息写在页面的msg下方。没错，人工复制便可。

1. 首先打开2个页面，一个主动方点击call，另外一个被动方点击recv
2. 将caller的消息复制到receiver的answer按钮边上的文本框内，再点击answer。
3. 将receiver的消息复制到caller的answer按钮边上的文本框内，再点击answer。
4. 点击send将send左边的文本发送到对方send右侧的文本框内。

[demo code，人工信号服务器](http://www.noobyard.com/link?url=https://github.com/erow/webrtc_demo.git)

## 5.概述

1. 建立对象 。

2. 绑定回调函数。

   ```
   peerConn = new RTCPeerConnection(pcConfig);
   peerConn.onicecandidate = handleIceCandidate;
   
   dataChannel = peerConn.createDataChannel('1');
   channel.onopen = function() {
   console.log('CHANNEL opened!!!');
     };
   
     channel.onmessage = function(event){
     remoteText.innerText = event.data;
   };
   ```

3. 提供服务：createOffer。

   ```
   这期间要发送offer,candidate消息。
   `peerConn.createOffer(onLocalSessionCreated, logError);`
   在`onLocalSessionCreated`中调用`sendMessage`。
   随后会触发`handleIceCandidate`调用`sendMessage`。
   ```

4. 建立应答： createAnswer。

   ```
   peerConn.setRemoteDescription(new RTCSessionDescription(message), function() {},
                                 logError);
   peerConn.createAnswer(onLocalSessionCreated, logError);
   ```

   ```
   注意，这一步是在receiver端进行的。
   跟createOffer相似，createAnswer会发送一个answer消息，随后发送candidate消息。
   ```

5. 添加candidate

   ```
   peerConn.addIceCandidate(new RTCIceCandidate({
     candidate: message.candidate
   }));
   ```

6. 链接创建

原文作者：[菜鸟学院](http://www.noobyard.com/)

原文链接：http://www.noobyard.com/article/p-bvixwrsr-mn.html