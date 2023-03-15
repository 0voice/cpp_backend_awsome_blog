# 【NO.373】IM（即时通讯）服务端

## 1.前言

首先讲讲IM（即时通讯）技术可以用来做什么：

- 聊天：qq、微信
- 直播：斗鱼直播、抖音
- 实时位置共享、游戏多人互动等等

可以说几乎所有高实时性的应用场景都需要用到IM技术。

本篇将带大家从零开始搭建一个轻量级的IM服务端，麻雀虽小，五脏俱全，我们搭建的IM服务端实现以下功能：

- 一对一的文本消息、文件消息通信
- 每个消息有“已发送”/“已送达”/“已读”回执
- 存储离线消息
- 支持用户登录，好友关系等基本功能。
- 能够方便地水平扩展

**这个项目涵盖了很多后端必备知识：**

- rpc通信
- 数据库
- 缓存
- 消息队列
- 分布式、高并发的架构设计
- docker部署

## 2.消息通信

### 2.1.文本消息

我们先从最简单的特性开始实现：一个普通消息的发送

消息格式如下：

```
message ChatMsg{
    id = 1;
    //消息id
    fromId = Alice
    //发送者userId
    destId = Bob
    //接收者userId
    msgBody = hello
    //消息体
}
```

![img](https://pic3.zhimg.com/80/v2-ed5a6448fa6a63dd0cb7a51e87ded53e_720w.webp)

如上图，我们现在有两个用户：Alice和Bob连接到了服务器，当Alice发送消息message(hello)给Bob，服务端接收到消息，根据消息的destId进行转发，转发给Bob。

### 2.2.发送回执

**那我们要怎么来实现回执的发送呢？**

我们定义一种回执数据格式ACK，MsgType有三种，分别是sent（已发送）,delivered（已送达）, read（已读）：

```
message AckMsg {
    id;
    //消息id
    fromId;
    //发送者id
    destId;
    //接收者id
    msgType;
    //消息类型
    ackMsgId;
    //确认的消息id
}
enum MsgType {
    DELIVERED;
    READ;
}
```

当服务端接受到Alice发来的消息时：

**1.向Alice发送一个sent(hello)表示消息已经被发送到服务器。**

```
message AckMsg {
    id = 2;
    fromId = Alice;
    destId = Bob;
    msgType = SENT;
    ackMsgId = 1;
}
```

![img](https://pic4.zhimg.com/80/v2-6803c7cf665cab48b17907dae9133a5f_720w.webp)

**2.服务器把hello转发给Bob后，立刻向Alice发送delivered(hello)表示消息已经发送给Bob。**

```
message AckMsg {
    id = 3;
    fromId = Bob;
    destId = Alice;
    msgType = DELIVERED;
    ackMsgId = 1;
}
```

![img](https://pic4.zhimg.com/80/v2-52e095ddf63863dfbcdb2ee05a17467b_720w.webp)

**3.Bob阅读消息后，客户端向服务器发送read(hello)表示消息已读**

```
message AckMsg {
    id = 4;
    fromId = Bob;
    destId = Alice;
    msgType = READ;
    ackMsgId = 1;
}
```

这个消息会像一个普通聊天消息一样被服务器处理，最终发送给Alice。

![img](https://pic1.zhimg.com/80/v2-aeb5b18074e37791520dfc72354cffc4_720w.webp)

**在服务器这里不区分ChatMsg和AckMsg，处理过程都是一样的：解析消息的destId并进行转发。**

## 3.水平扩展

当用户量越来越大，必然需要增加服务器的数量，用户的连接被分散在不同的机器上。此时，就需要存储用户连接在哪台机器上。

我们引入一个新的模块来管理用户的连接信息。

## 4.管理用户状态

![img](https://pic2.zhimg.com/80/v2-464d2e2722f7c282157e4aa3b3187a59_720w.webp)

模块叫做user status，共有三个接口：

```
public interface UserStatusService {
    /**
     * 用户上线，存储userId与机器id的关系
     *
     * @param userId
     * @param connectorId
     * @return 如果当前用户在线，则返回他连接的机器id，否则返回null
     */
    String online(String userId, String connectorId);
    /**
     * 用户下线
     *
     * @param userId
     */
    void offline(String userId);
    /**
     * 通过用户id查找他当前连接的机器id
     *
     * @param userId
     * @return
     */
    String getConnectorId(String userId);
}
```

这样我们就能够对用户连接状态进行管理了，具体的实现应考虑服务的用户量、期望性能等进行实现。

此处我们使用redis来实现，将userId和connectorId的关系以key-value的形式存储。

## 5.消息转发

除此之外，还需要一个模块在不同的机器上转发消息，如下结构：

![img](https://pic2.zhimg.com/80/v2-acac7d7fb6cc6740b0131192d139f279_720w.webp)

此时我们的服务被拆分成了connector和transfer两个模块，connector模块用于维持用户的长链接，而transfer的作用是将消息在多个connector之间转发。

现在Alice和Bob连接到了两台connector上，那么消息要如何传递呢？

**1.Alice上线，连接到机器[1]上时**

- 将Alice和它的连接存入内存中。
- 调用user status的online方法记录Alice上线。

**2.Alice发送了一条消息给Bob**

- 机器[1]收到消息后，解析destId，在内存中查找是否有Bob。
- 如果没有，代表Bob未连接到这台机器，则转发给transfer。

**3.transfer调用user status的getConnectorId(Bob)方法找到Bob所连接的connector，返回机器[2]，则转发给机器[2]。**

## 6.流程图：

![img](https://pic1.zhimg.com/80/v2-8f65c8d5fa01c633aa0e6d2475a48134_720w.webp)

## 7.总结：

- 引入user status模块管理用户连接，transfer模块在不同的机器之间转发，使服务可以水平扩展。
- 为了满足实时转发，transfer需要和每台connector机器都保持长链接。

## 8.离线消息

如果用户当前不在线，就必须把消息持久化下来，等待用户下次上线再推送，这里使用mysql存储离线消息。

为了方便地水平扩展，我们使用消息队列进行解耦。

- transfer接收到消息后如果发现用户不在线，就发送给消息队列入库。
- 用户登录时，服务器从库里拉取离线消息进行推送。

## 9.用户登录、好友关系

用户的注册登录、账户管理、好友关系链等功能更适合使用http协议，因此我们将这个模块做成一个restful服务，对外暴露http接口供客户端调用。

至此服务端的基本架构就完成了：

![img](https://pic2.zhimg.com/80/v2-baf1c6930b38586df4ad6a3279231061_720w.webp)

## 10.总结

以上就是这篇博客的所有内容，本篇帮大家构建了IM服务端的架构，但还有很多细节需要我们去思考，例如：

- 如何保证消息的顺序和唯一
- 多个设备在线如何保证消息一致性
- 如何处理消息发送失败
- 消息的安全性
- 如果要存储聊天记录要怎么做
- 数据库分表分库
- 服务高可用

原文链接：https://zhuanlan.zhihu.com/p/401109228

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)