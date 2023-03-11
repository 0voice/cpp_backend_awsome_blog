# 【NO.137】腾讯T9/T3.1级别的后台服务器开发技术大佬是怎样炼成的？

今天给大家分享一下腾讯T9/T3.1级别的技术大佬的学习路线，希望对在自学提升的朋友有一些帮助，学习途径总结在下面这张思维导图里面了，觉得还不错的请点赞收藏支持一下

思维导图：

![img](https://pic2.zhimg.com/80/v2-6c7e07b9bed3742db0c0fb2981440cd5_720w.webp)

## 1.精进基石

![img](https://pic4.zhimg.com/80/v2-bd0019385cfcfd1db5b6ff08702029af_720w.webp)

### **1.1.数据算法与结构**

1.1 排序（11种）与KMP
1.2 红黑树证明
1.3 B树与B+树
1.4 Hash与布隆过滤器

### **1.2.设计模式（23种）**

2.1 责任链模式
2.2 过滤器墨海
2.3 发布订阅模式
2.4 工厂模式

### **1.3.工程管理**

3.1 Makefile/cmake/configure
3.2 git /svn与持续集成
3.3 Linux系统运行时命令

## 2.高性能网络设计

![img](https://pic3.zhimg.com/80/v2-6a25fadbc85acbd4be4631b447f5dcf6_720w.webp)

### **2.1.代码实现**

1.1 网络io与select/poll/epoll
1.2 reactor的原理与实现
1.3 http/https web服务器的实现
1.4 websocket协议与服务器实现

### **2.2.方案分析**

2.1 服务器百万并发的实现（c10K，c1000k， C10M）
2.2 redis/memcached/Nginx网络组件
2.3 Posix API与网络协议栈
2.4 UDP可靠协议 QUIC/KCP

## 3.基础组件实现

![img](https://pic3.zhimg.com/80/v2-d97cb962371caf08e136514bb2ceba52_720w.webp)

### **3.1.池式结构**

1.1 手写线程池
1.2 内存池 ringbuffer
1.3 异步请求池 性能优化，异步mysql 异步dns 异步redis
1.4 mysql连接池
1.5 redis连接池

### **3.2.高性能组件**

2.1 原子操作 CAS
2.2 消息队列与无锁队列
2.3 定时器的方案 红黑树 时间轮 最小堆
2.4 锁的实现原理 互斥锁，自旋锁 ，乐观锁，悲观锁，分布式锁
2.5 服务器连接保活 keepalived
2.6 try/catch的实现

### **3.3.开源组件**

3.1 libevent/libev框架
3.2 异步日志方案 log4cpp
3.3 应用层协议 protobuf/thrift
3.4 openssl加密
3.5 json与xml解析器
3.6 字符编码unicode/gbk/utf-8

## 4.自主自研框架

![img](https://pic2.zhimg.com/80/v2-75f29501141f73fc63913b35f4dced25_720w.webp)

### **4.1.协程框架的实现 NtyCo**

1.1 协程的原理与工程案例
1.2 协程的调度器实现

### **4.2.用户态协议栈NtyTcp(tcp/ip)**

2.1 滑动窗口 拥塞控制 满启动
2.2 tcp定时器的实现
2.3 epoll的源码实现

## 5.基础开源框架

![img](https://pic1.zhimg.com/80/v2-9d94f6a8b5363ee429efa2cda7c12914_720w.webp)

### **5.1.skynet**

1.1 skynet高性能网关
1.2 actor实现与cluster/负载均衡
1.3 skynet网络与热更新 数据共享**

### 5.2.ZeroMQ

2.1 ZeroMQ Router-Dealter模式
2.2 源码分析：消息模型与工程案例
2.3 源码分析：网络机制**

### **5.3.DPDK**

3.1 dpdk PCI原理与 testpmd/l3fwd/skeletion
3.2 kni数据流程
3.3 dpdk实现dns
3.4 dpdk的高性能网关的实现
3.5 半虚拟化 virtio/vhost的加速**

## 6.中间件开发

![img](https://pic1.zhimg.com/80/v2-6b633e9c908268602c2446707a0ff5b4_720w.webp)

### **6.1.MySQL**

1.1 SQL语句 索引 存储过程 触发器
1.2 数据库连接池与sql解析剖析
1.3 存储引擎原理 MyISAM与Innodb 事务隔离
1.4 自己实现一个存储引擎 MySQL源码
1.5 MySQL集群与分布式 高可用高并发

### **6.2.Redis**

2.1 Redis相关命令与持久化
2.2 Redis连接池与异步操作
2.3 源码分析：存储原理与数据模型
2.4 源码分析：主从 原子模型
2.5 redis的集群方案

### **6.3.NGINX**

3.1 Nginx使用conf配置
3.2 nginx模块开发 过滤器模块
3.3 Nginx模块开发 handler模块
3.4 源码分析： Nginx Http状态机
3.5 源码分析：进程间通信与Slab共享机制

### **6.4.mongodb**

4.1 Mongo接口编程与MongoDB命令使用
4.2 MongoDB的集群方案**

### **6.5.dfs**

5.1 ceph
5.2 fastdfs

## 7.Linux内核

![img](https://pic1.zhimg.com/80/v2-f932bd93349874895089a871f98f8594_720w.webp)

### **7.1.进程管理**

1.1 进程管理与调度
1.2 锁与进程间通信
1.3 系统调用 如何自己实现一个syscall

### **7.2.内存管理**

2.1 物理内存 伙伴算法
2.2 进程虚拟内存 mm_struct
2.3 页的回收与页交换

### **7.3.文件系统**

3.1 虚拟文件系统
3.2 Ext2/3/4 文件系统
3.3 无持久的存储

### **7.4.设备驱动**

4.1 内核编译与升级
4.2 进程通信组件的实现
4.3 网卡的实现吧

## 8.性能分析

![img](https://pic4.zhimg.com/80/v2-c59a00c4ae87cff81c61613cd9dd885b_720w.webp)

1、工具 wrk/ webbench/ loadbalance/valgrind

2、Google gTest/Memtrack

3、火焰图/热图

## 9.分布式架构

![img](https://pic2.zhimg.com/80/v2-7c6d0935583d01c34a1a1ac48218c709_720w.webp)

1、腾讯的Tars

2、虚拟化的docker

3、分布式注册中心etcd

4、P2P 网络穿透 打洞 去中心化的网络

原文链接：https://linuxcpp.0voice.com/?id=245

作者：[HG](https://linuxcpp.0voice.com/?auth=10)