# 【NO.540】Linux服务器开发,应用层协议设计ProtoBuf/Thrift

## 0.前言

协议对于前后端通讯是非常重要的，作为开发新手往往不能理解和掌握协议的设计，导致增加了很多不必要的工作。

## 1.为什么需要协议设计

让发送端和接收端的通道能理解之间发送的数据。

## 2.消息帧的完整性判断

四种判断消息的完整性

- 固定长度
- 特别标记/r/n
- 固定消息头+消息体结构
- 序列化后的buffer前面增加一个字符流的头部，其中有一个字段村村消息总长度。

## 3.协议设计范例



![在这里插入图片描述](https://img-blog.csdnimg.cn/f85bc2f79e8a4c59b1ec54b2e33bcc15.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



## 4.jason/xml/protobuf不同序列化对比

- xm本地配置，ui配置qt，Androidl
- json，websocket http协议，注册账号，web里面登录
- protobuf，业务内部使用，各个服务之间rpc调用，即时通讯项目，游戏项目，带宽占用少的。

IDL(Interface description language)接口描述语言：将文件通过工具生成.c文件。
完整的protobuf库支持C++反射。

项目.模块.proto



![在这里插入图片描述](https://img-blog.csdnimg.cn/08a22ca87cdf492082acf5c88a78f4c7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



## 5.protobuf工程实践和原理

base128 Bariants 表示值 ： 每个字节的最高位不能直接用来表示我们的value，它是用来判断自己是否结束。0代表解释，1表示没有结束。小端形式base128。
Zigzag：针对负数进行优化，内部将int32类型负数转化为uint64来处理。都是整数用int32，有负数用sint32。

## 6.总结

Darren老师建议把person对象进行手写一下，最起码要把protobuf文件进行修改，我觉得也是非常有必要的。从目前自己对知识掌握的程度上来看，二次学习是必须的，既然已经花了大力气就应该把知识完全掌握，那就既来之则安之吧！

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/605060882