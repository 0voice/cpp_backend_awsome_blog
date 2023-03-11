# 【NO.190】skynet源码结构、启动流程以及多线程工作原理

本文主要介绍skynet源码目录结构、启动流程以及其多线程工作原理。

## **1、skynet目录结构**

![img](https://pic3.zhimg.com/80/v2-c98cba27955e00194aa1180e0f3203a2_720w.webp)

- skynet-src: c语言写的skynet核心代码
- service-src: c语言编写的封装给Lua使用的服务，编译后生成的so文件在cservice中（如gate.so, harbor.so, logger.so, snlua.so）
- lualib-src: c语言编写的封装给lua使用的库，编译后生成的so文件在luaclib中（如bson.so, skynet.so, sproto.so等），提供C层级的api调用，如调用socket模块的api，调用skynet消息发送，注册回调函数的api，甚至是对C服务的调用等，并导出lua接口，供lua层使用，可以视为lua调C的媒介。
- lualib: lua编写的库
- service: lua写服务
- 3rd ：第三方库如jemalloc, lua等
- test：lua写的一些测试代码
- examples: 框架示例

只允许上层调用下层，而下层不能直接调用上层的api，这样做层次清晰

## **2、skynet启动流程**

1、启动skynet方式：终端输入./skynet exmaple/config

2、启动入口函数为skynet_main.c/main, config作为args[1]参数传入

![img](https://pic1.zhimg.com/80/v2-cdfe3c7bce21de12ff6a0f5ee66c590c_720w.webp)

3、调用skynet_start.c/skynet_start函数

![img](https://pic2.zhimg.com/80/v2-39086a348147da96f15da2854c47aa4d_720w.webp)



## 3、**skynet多线程工作原理**

skynet线程创建工作由上述skynet_start.c/start完成，主要有以下四类线程：

- monitor线程：监测所有的线程是否有卡死在某服务对某条消息的处理
- timer线程：运行定时器
- socket线程: 进行网络数据的收发
- worker线程：负责对消息队列进行调度

![img](https://pic2.zhimg.com/80/v2-4a892948c76cf96d19b977f6ab611ed1_720w.webp)

1、moniter线程

初始化该线程的key对应的私有数据块

每5s对所有工作线程进行一次检测

调用skynet_monitor_check函数检测线程是否有卡住在某条消息处理

![img](https://pic3.zhimg.com/80/v2-ed056e859d5ce9b9b2ef9c56190dd3c6_720w.webp)

2、timer定时器线程

每隔2500微秒刷新计时、唤醒等待条件触发的工作线程并检查是否有终端关闭的信号，如果有则打开log文件，将log输出至文件中，在刷新计时中会对每个时刻的链表进行相应的处理.

![img](https://pic4.zhimg.com/80/v2-45d146ebe0655b1339ffd74d5bfcb66f_720w.webp)

3、socket套接字线程

处理所有的套接字上的事件，该线程确保所有的工作线程中至少有一条工作线程是处于运行状态的，以便可以处理套接字上的事件。

![img](https://pic4.zhimg.com/80/v2-82aad05d895f4f104a53928c88204c27_720w.webp)

4、worker工作线程

从全局队列中取出服务队列对其消息进行处理，其运行函数thread_worker的工作原理：首先初始化该线程的key对应的私有数据块，然后从全局队列中取出服务队列对其消息进行处理，最后当全局队列中没有服务队列信息时进入等待状态，等待定时器线程或套接字线程触发条件；

![img](https://pic2.zhimg.com/80/v2-55008c229c2c83cf16b32d5c4d55c2dd_720w.webp)

## **4、skynet消息处理如何保证线程安全？**

最后探讨一个问题，消息处理为什么线程安全？解释如下：

- 每个worker线程，从global_mq取出的次级消息队列都是唯一的，有且只有一个服务与之对应，取出之后，在该worker线程完成所有callback调用之前，不会push回global_mq中，也就是说，在这段时间内，其他worker线程不能拿到这个次级消息队列所对应的服务，并调用callback函数，也就是说一个服务不可能同时在多条worker线程内执行callback函数，从而保证了线程安全。
- 不论是global_mq还是次级消息队列，在入队和出队操作时，都需加上spinlock，这样多个线程同时访问mq的时候，第一个访问者会进入临界区并锁住，其他线程会阻塞等待，直至该锁解除，这样也保证了线程安全。
- 我们在通过handle从skynet_context list里获取skynet_context的过程中（比如派发消息时，要要先获取skynet_context指针，再调用该服务的callback函数），需要加上一个读写锁，因为在skynet运作的过程中，获取skynet_context，比创建skynet_context的情况要多得多，因此这里用了读写锁

以上介绍了skynet源码中的目录结构以及各部分功能，接着介绍了skynet的启动流程，最后介绍了skynet的多个线程是如何进行协同工作的。

原文地址：https://zhuanlan.zhihu.com/p/549861580

作者：linux