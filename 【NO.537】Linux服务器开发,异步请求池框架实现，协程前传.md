# 【NO.537】Linux服务器开发,异步请求池框架实现，协程前传

## 1.前言

服务器发送完请求不用等待对端回数据，而是用另一个线程进行等待接收收据。
\#
1、发送请教，1个io还是多个io 是多个io
2、接收结果的线程，如何拿到fd
解决方法是每次发送出的fd，放到epoll中，接收线程有io可读就读。

## 2.King式四元组

1、init

- epoll_create
- pthread_create
  2、commit
- 创建socket
- 连接server
- 准备协议encode
- send
- 加入epoll
  3、thread_callback
- while(){
  recv();
  parser();
  fd->epoll_delete
  }
  4、destory
- close(fd)
- phthred_cancel

服务器向第三方服务提供请求，拿redis举例：set，get等命令，每一个命令都有一个回调函数处理请教，通过epoll_ctl()增加节点，epoll_wait()函数去等待检测的数据。只是传入一个指针，不会影响红黑树的大小。

c和c++最大的特点就是内存泄漏。
发送信号提交commit后，让出协程call_back，检测是否有数据到达。

线程创建确实会失败，程序运行时间较短确实不容易捕捉这一现象，但是长时间就不一定了，所以要判断返回值。

## 3.DNS 工作原理举例



![在这里插入图片描述](https://img-blog.csdnimg.cn/6c28b303ff584d9aa5dadd8758cadc91.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



## 4.补充

内存池最不好理解，需要重点去关注的方面：

- 内存块的组织
- 内存分配
- 内存回收

## 5.总结

通过今天的异步请求池框架的实现学习，我对异步请求池的实现思路已经是非常清晰了。异步请求池被King老师成为协程的前传，和后面要学的skynet以及协程的调度有着紧密的关系，后续的课程始终在一种懵懂的状态，看完这节课有一种如梦方醒的感觉，这个感觉美妙极了。
最后，感谢一路上帮助小生的朋友们，感谢零声学院的King老师~

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/604990131