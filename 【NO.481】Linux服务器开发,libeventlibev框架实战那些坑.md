# 【NO.481】Linux服务器开发,libevent/libev框架实战那些坑

## 1.前言

libevent、libev和libuv都是c语言实现的异步事件库。注册异步事件，检测异步事件，根据事件的触发先后顺序调用相对应的函数处理事件。
处理的时间包括：**网络IO事件，定时事件以及信号事件。**这三个是驱动服务器逻辑的三个重要事件。
libevent和libev解决了跨平台的问题，封装了异步事件库与操作系统的交互。
libevent使用了大量的全局变量，很难安全得在多线程环境中运行；event的数据结构太大，包含了io、时间以及信号 处理全封装在一个结构体中，额外的组件如http、dns、openssl等实现质量差，计时器采用最小二叉堆(libuv改为最小四叉堆)不能很好的处理时间事件。



![在这里插入图片描述](https://img-blog.csdnimg.cn/c4fa6a6ac20b4fcbb337ac0cf4e35a7f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)



## 2.如何阅读网络库

记住两个线索：

### 2.1 网络封装

- IO检测
- IO操作

### 2.2 事件操作

- 连接建立的问题 （限制最大连接数，设置黑白名单，创建用户的对象）
- 连接断开
- 数据到达 （具体消息的分发，解密）
- 数据发送

不注重效率的都会使用getdateoftime()，会对我们的系统有较大的考验，因为它会读一个文件，把时间取出来。如果我们要经常获取时间，我可以运用缓存时间。
systime_mono() 机器启动到现在的时间。
systime_wall() 现在的时间。
wall+mono2-mono1为什么要这样计算时间？担心有的人会更好系统时间，导致我们服务器出现错乱。
更新时间缓存的目的是避免过多的调用系统调用，影响性能。
最小堆的顶端是最近要触发的任务。
收集网络事件，收集定时事件，将事件进行包装，根据事件优先级不同放入不同队列中，并没有急着处理。
event_asign()相当于epoll_mod，是个增强版，没有还会增加。

### 2.3 巧妙设计

linux协议栈中每一个fd在内核态中都有一个读写缓冲区，而在内核态也同样有一个读写缓冲区，为什么用户态需要实现一个对写缓冲区？
能不能确保一次int n=read(fd,buf,size)读出内核态所有数据？答案肯定是不能，因为数据包的界定是有固定长度和特殊字符两种，size参数的填写是盲猜的。有了用户态缓冲区就更容易读出完整的数据。
而写的时候int n=write(fd,buf,size)，size是打算写的值，而n表示实际写入的值，如果n<size说明数据没有全部写出去，剩下的数据要放到缓冲区。我们要缓存数据，然后注册写事件。
常用的解决方案：

- fix buffer char buf[16*1024*1024] char buf[16*1024*1024] 容纳的最大值，还要进行错误判断，数据移动频繁，存在空间浪费。

- ringbuf 多线程、网卡，逻辑上连续但是物理上上是数组不连续。

- chainbuf 最大的缺点是内存不连续，会带来多次IO

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/1ffdbb87f6a54eff9386f1a9cce20ff9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)

用不同的回调函数对不同的事件进行解耦。

## 3.总结

今天通过Mark老师细心的讲解，我又被深深的上了一课，脱离libevent这个框架之外的习惯我觉得更值得我去学习。Mark老师反复的强调，写代码之前要思路清晰，只有思路清晰代码写起来才能更加流畅。将事件进行归类，读起代码来也会变得得心应手。虽然小生目前依然才疏学浅，相信在不远的将来一定会成为栋梁之才。

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/605018548