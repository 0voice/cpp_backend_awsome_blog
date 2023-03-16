# 【NO.449】后端开发【一大波干货知识】—Redis，Memcached，Nginx网络组件

## 1.reator网络编程

epoll被称为事件管理器，利用管理器去管理多个连接。

```
int clientfd=accept(listenfd,addr,sz);
clientfd ==-1 && erro==EWOLDBLOCK //表示全连接中连接为空
int connect(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
error == EINPROGRESS //正在建立连接
error == EISCONN  //连接建立成功    
```

- 关闭读端 read = 0
- 关闭写端 write = -1 && errno =EPIPE

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163416_21695.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163417_68931.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163418_80973.jpg)

- io函数像read只能检测一个fd对应的状态，可以检测具体的状态。
- io多路复用可以检测多个fd对应的状态，只能检测可读，可写，错误，断开笼统等信息。
- getsockopt 也可以检测错误。

### 2.阻塞IO 和 非阻塞IO

- 阻塞在网络线程
- 连接的fd阻塞属性决定了io函数是否阻塞
- 具体差异在：io函数在数据未到达时是否立刻返回。

```
//默认情况下，fd时阻塞的，设置非阻塞的方法如下：
int flag=fcntl(fd,F_GETFL,0);
fcntl(fd,F_SETFL,flag | O_NONBLOCK)；
```

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163418_42728.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163419_88279.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163420_39848.jpg)

- timeout == 0 是非阻塞效果，检测一下立即返回。
- timeout == -1 是永久阻塞
- timeout == 1000

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163420_99925.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163422_19459.jpg)

epoll_create 会去创建红黑树和就绪队列。

epoll_ctl 会去注册事件，会建立回调关系。当事件被触发，epoll_ctl 会将fd从红黑树中放到就绪队列。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163422_58852.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163423_87320.png)

问：代码第9行能不能监听写事件？

答：不能，因为刚开始的时候，写缓冲区是空的，会被一直触发可写。

### 3.编程细节，返回值以及错误码

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163424_77964.jpg)

读端关闭了。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163424_39067.jpg)

建议read()函数使用非阻塞io，因为出现错误会立刻返回，不会卡在这里影响别人。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163425_67893.jpg)

将数据写到缓冲区，协议栈会将数据发送到对端。

## 4.redis、nginx、memcached reactor具体使用

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163426_74357.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163427_45667.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163427_92045.jpg)

redis-6.0支持IO多线程，封装在networking.cz中。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/13/20221213163428_84292.jpg)

原文链接：https://zhuanlan.zhihu.com/p/493857761

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)