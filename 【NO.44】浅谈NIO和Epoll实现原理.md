# 【NO.44】浅谈NIO和Epoll实现原理

## **1 前置知识**

### **1.1 socket是什么？**

就是传输控制层tcp连接通信。

### **1.2 fd是什么？**

fd既file descriptor-文件描述符。Java中用对象代表输入输出流等...在Linux系统中不是面向对象的，是一切皆文件的，拿文件来代表输入输出流。其实是一个数值，例如1,2,3,4...0是标准输入，1是标准输出，2是错误输出...

任何进程在操作系统中都有自己IO对应的文件描述符，参考如下：

```text
[root@iZj6c1dib207c054itjn30Z ~]# ps -ef | grep redis
root     20688     1  0 Nov23 ?        00:01:01 /opt/redis/bin/redis-server 127.0.0.1:6379
root     20786     1  0 Nov23 ?        00:01:00 /opt/redis/bin/redis-server 127.0.0.1:6380
root     30741 30719  0 12:26 pts/1    00:00:00 grep --color=auto redis
[root@iZj6c1dib207c054itjn30Z ~]# cd /proc/20688/fd
[root@iZj6c1dib207c054itjn30Z fd]# ll
total 0
lrwx------ 1 root root 64 Nov 23 21:50 0 -> /dev/null
lrwx------ 1 root root 64 Nov 23 21:50 1 -> /dev/null
lrwx------ 1 root root 64 Nov 23 21:50 2 -> /dev/null
lr-x------ 1 root root 64 Nov 23 21:50 3 -> pipe:[27847377]
l-wx------ 1 root root 64 Nov 23 21:50 4 -> pipe:[27847377]
lrwx------ 1 root root 64 Nov 23 21:50 5 -> anon_inode:[eventpoll]
lrwx------ 1 root root 64 Nov 23 21:50 6 -> socket:[27847384]
```

## **2 从BIO到epoll**

### **2.1 BIO**

计算机有内核，客户端和内核连接产生fd，计算机的进程或线程会读取相应的fd。
因为socket在这个时期是blocking的，线程读取socket产生的文件描述符时，如果数据包没到，读取命令不能返回，会被阻塞。导致会有更多的线程被抛出，单位CPU在某一时间片只能处理一个线程，出现数据包到了的线程等着数据包没到的线程的情况，造成CPU资源浪费，CPU切换线程成本巨大。
例如早期Tomcat7.0版默认就是BIO。

![img](https://pic1.zhimg.com/80/v2-dbb0aa89b63d7608ed3e6ce233b2dd60_720w.webp)

### **2.2 早期NIO**

socket对应的fd是非阻塞的
单位CPU只用一个线程，就一颗CPU只跑一个线程，没有CPU切换损耗，在用户空间轮询遍历所有的文件描述符。从遍历到取数据都是自己完成，所以是同步非阻塞IO。
问题是如果有1000个fd，代表用户进程轮询调用1000次kernel，
用户空间查询一次文件描述符就得调用一次系统调用，让后内核态用户态来回切换成本很大。

![img](https://pic1.zhimg.com/80/v2-71a08825a2f4b3b9056b8e77d0389efc_720w.webp)

### **2.3 多路复用NIO**

内核向前发展，轮询放到内核里，内核增加一个系统调用select函数，统一把一千个fd传给select函数，内核操作select函数，fd准备好后返回给线程，拿着返回的文件描述符找出ready的再去调read。如果1000个fd只有1个有数据，以前要调1000次，现在只调1次，相对前面在系统调用上更加精准。还是同步非阻塞的，只是减少了用户态和内核态的切换。


注意Linux只能实现NIO，不能实现AIO。
问题：用户态和内核态沟通的fd相关数据要来回拷贝。

![img](https://pic3.zhimg.com/80/v2-75a2e0d3d06804b35b3bc49b1cd73ea6_720w.webp)

### **2.4 epoll**

内核通过mmap实现共享空间，用户态和内核态有一个空间是共享的，文件描述符fd存在共享空间实现用户态和内核态共享。epoll里面有三个调用，用户空间先epoll_create准备一个共享空间mmap，里面维护一个红黑树，内核态将连接注册进红黑树，epoll_ctl写入。当有数据准备好了，调用epoll_wait中断阻塞，取链表fd，再单独调用read。
mmap应用：kafka实现数据通过socket存到服务器文件上的过程也是mmap。


epoll应用：redis和nginx的worker进程等等。

![img](https://pic1.zhimg.com/80/v2-4939e2fc0d43fbdf3a3df57e7b36f068_720w.webp)

原文链接：https://zhuanlan.zhihu.com/p/482549633

原文作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)