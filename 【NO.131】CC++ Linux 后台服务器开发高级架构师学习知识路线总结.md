# 【NO.131】C/C++ Linux 后台服务器开发高级架构师学习知识路线总结

前言：

小编也是从事c方面10多年的工作经验、今天跟大家分享一下我总结出来的一系列 C/C Linux后台服务器开发的学习路线。从Linux开发工程师-Linux后台开发工程师-Linux高级互联网架构师。

想必大家都知道从事后台开发首先就是要选择一种语言，小编今天跟大家分享是用C/C++ 做的后台开发。所以想从事这方面的朋友得有C/C++的基础。

![img](https://pic2.zhimg.com/80/v2-6586beb7e3fe0b7335f722ee264504c5_720w.jpg)

首先跟大家说的是从学习步骤：（Linux入门到精通篇）

## 1.Linux开发环境

1.了解Linux环境搭建，了解LinuxC编程

2.了解Linux安装，命令使用，shell编程

3.shell脚本实现检测局域网内哪些ip地址机器宕机

## 2.Linux C编程

1.Linux C编程 统计文件单词数量

包括：文件操作、文件指针

2.Linux C编程 实现通讯录

包括：结构体

## 3.Linux环境编程

1.并发下的计数方案

包括：互斥锁、自旋锁、原子操作

2.实现线程池

包括：线程队列，任务队列，条件变量

3.CPU与进程的关系

包括：进程操作，进程与CPU粘合，进程通信

4.数据库操作

包括：数据库封装，sql语句封装，网络连接封装

## 4.网络编程

1.DNS请求器

包括：UDP通信，DNS协议，协议解析

2.实现http请求器 TCP客户端

包括：TCP编程，HTTP请求协议

3.百万级并发服务器 TCP服务器

包括：tcp，网络io，Linux系统

**总结：把以上知识点内容掌握之后你的Linux就已经比较成熟了，达到了一个Linux开发工程师的水平了。**

![img](https://pic1.zhimg.com/80/v2-bf8e7ff84a673171a747eaf82d44afc0_720w.webp)

熟练掌握上面的知识点后就可以来了解一下后面的知识点了：（Linux后台开发篇）

## 5.算法于设计

千里之行，始于足下。不积跬步，无以致千里。既能仰望星空又能脚踏实地

1.排序与查找

包括：插入排序、快速排序、希尔排序、桶排序、基数排序、归并排序

2.常用算法

包括：布隆过滤器、字符串匹配KMP算法、回溯算法、贪心算法、推荐算法、深度 广度优先

3.常用的数据结构

包括：平衡二叉树、红黑树、B-树、KMP算法、栈/队列

4.常用设计模式

包括：单列模式、责任链模式、过滤器模式、发布订阅模式、代理模式、工厂模式

## 6.后台组件编程

工欲善其事，必先利其器。后台组件是开发的入门石

\1. 持久化 MySQL

包括：MySQL安装配置与远程连接、数据操作源于SQL语句、存储过程与事务处理、SQL函数，运算，临时表、防数据丢失 备份与恢复、MySQL建库建表建索引

2.消息队列 ZeroMQ

包括：ZMQ编译安装与开发环境搭建、publisher-subscriber模式实现、request-response模式实现、Router-Dealer模式实现、消息队列—性能分析

3.缓存 Redis

包括： Redis编译安装配置、客户端全局唯一ID保存机制、Redis消息队列机制 发布订阅、Redis事务实战、Redis安全性能，数据备份与恢复、Redis分布式锁详解

\4. 反向代理 Nginx

包括： Nginx开发介绍、反向代理负载均衡配置详解、自定义协议upstream开发、子域名映射、服务器后台攻击预防、nginx双虚拟主机

\5. Restful Http

包括：Http第三方接口实现、异步Http请求、ngrok与Restlet、长连接与短链接

\6. 协调服务 ZooKeeper

包括：ZK编译安装与C API开发环境、集群管理与服务注册、节点创建与监控、分布式锁的实现、ZK伪集群部署与服务管理

7.NoSQL MongoDB

包括：MongDB安装与开发介绍、MongoDB备份与恢复、MongoDB文档操作、全文检索与正则表达式、MongoDB建库建集合

## 7.代码工程化

优秀的工程师有优秀的代码组织能力与代码迭代能力。

1.架构工程

包括：工程参数配置与编译 cmake、代码规范与命名规则、文件命名与变量命名规则、脚本配置工具 autoconf、代码工程组织架构 Makefile

\2. 管理代码

包括： 分布式版本控制系统 git、远程仓库，标签管理、 github与码云、创建仓库，导入，checkout、svn环境搭建与原理、 分支管理 冲突解决、产品代码版本管理 SVN

## 8.网络服务

网络IO是网络通信的血管，数据是血液。血液的流动是不能离开血管的。

1.源码实现

包括：服务器IO核心— epoll编程实战、客户端多网络连接机制poll、文件IO管理select

2.框架

包括：高性能的时间循环 libev、跨平台异步I/O libuv、跨平台的C++库 Boost.Asio、事件通知库 libevent

3.理论

包括：阻塞型 BIO、异步IO AIO、非阻塞型IO NIO

## 9.开源框架

欲穷千里目，更上一层楼。站在巨人的肩膀上，看到窗外的景色。

1.TCP协议栈

包括：基于DPDK的高性能用户态协议栈 f-stack、基于Netmap单线程协议栈 NtyTcp、精简版tcp协议栈 LWIP

2.并发性

包括：用OpenCL的C++ GPU计算库 Boost.Compute、Intel线程构件块 Intel TBB、并行编程的异构系统的开放标准 OpenCL、C++11的反应性编程库 C++ React

\3. 数据库

包括：Redis数据库的C客户端库 hiredis、Facebook的嵌入键值的快速存储 RocksDB、用于Sqlite3的C++对象关系映射 hiberlite

\4. 国际化

包括：Unicode 和全球化支持的C、C++ 和Java库 IBM ICU、不同字符编码之间的编码转换库 libiconv、GNU gettext

5.压缩

包括：非常紧凑的数据流压缩库 Zlib、快速压缩和解压缩 Snappy、非常快速的压缩算法 LZ4、单一的C源文件，紧缩/膨胀压缩库 Miniz

6.日志

包括：设计非常模块化，并且具有扩展性 Boost.Log、灵活添加日志到文件，系统日志 Log4cpp、添加日志到你的C++应用程序 templog、C++日志库，只包含单一的头文件 easyloggingpp

7.多媒体库

包括：开源音频库—跨平台的音频API OpenAL、网络实时流媒体通信 WebRTC、音频和音乐数字信号处理库 Maximilian、C++易用和高效的音频合成 Tonic

\8. 序列化

包括：快速数据交换格式和RPC系统 Cap’n Proto、协议缓冲，谷歌的数据交换格式 ProtoBuf、高效的跨语言IPC/RPC Thrift、内存高效的序列化库 FlatBuffers

9.XML库

包括：Gnome的xml C解析器和工具包 LibXml2、单快速的C++CML解析器 TinyXML2、简单快速的XML解析器 PugiXML、C++的xml解析器 LibXml++

10.脚本

包括：小型快速脚本引擎 Lua、谷歌的快速JavaScript引擎 V8、嵌入式脚本语言 ChaiScript、

11.Json库

包括：进行编解码和处理Jason数据的C语言库 Jansson、C语言中的JSON解析和打印库 ibjson、轻量级的JSON库 libjson、C/C++的Jason解析生成器 Frozen

12.数学库

包括：高质量的C++线性代数库 Armadillo、数学图形模板库 GMTL、用于个高精度计算的C/C++库 GMP、高级C++模板头文件库 Eigen

13.安全

包括：SSL，TLS和DTLS协议的安全通信库 GnuTLS、功能齐全的，开源加密库 Openssl、有关加密方案的免费的C++库 Cryto++

14.Web应用框架

包括：安全快速开源Web服务器 Lighttpd、于Qt库的web框架 QDjango、高性能的HTTP和反向代理web服务器 Nginx

15.网络库

包括：C异步网络开发库 Dyad.c、多协议文件传输库 Curl、高速模块化的异步通信库 ZeroMQ、C++面向对象网络工具包 ACE

16.异步事件

包括：事件通知库 libevent、 跨平台异步I/O libuv、功能齐全，高性能的时间循环 libev、网络和底层I/O编程的跨平台的C++库 Boost.Asio

17.协程

包括：纯c版的协程框架 ntyco、C++11实现协程库, golang风格 libgo、微信支持8亿用户同时在线的底层IO库 libco

## 10.性能测试

学而不思则罔，思而不学则殆。从技术反馈中理解知识的原理。

1.调试库

包括：Boost测试库 Boost.Test、内存调试性能分析工具 Valgrind、谷歌C++测试框架 GoogleTest、内存分配跟踪库 MemTrack

2.测试库

包括：单元测试框架 minUnit、测试用例编写 libtap、轻量级的C++单元测试框架 UnitTest++、自动化测试用例 gtest和luatest

3.性能工具

包括：高性能代码构建系统 tundra、Http压测工具 WRK、 网站压测工具 webbench、高性能构建系统 FASTBuild

## 11.Linux系统

上帝关闭一扇门，就会打开一扇窗，Linux是程序员世界的另一扇窗。

1.系统命令工具

包括：进程间通信设施状态 ipcs、Linux系统运行时长 uptime、CPU平均负载和磁盘活动 iostat、监控，收集和汇报系统活动 sar、监控多处理器使用情况 mpstat、监控进程的内存使用情况 pmap、系统管理员调优和基准测量工具 nmon、密切关注Linux系统 glances、查看系统调用 strace

\2. 基础命令工具

包括：系统进程状态 ps、虚拟内存统计工具 vmstat、控制台的流量监控工具 vnstat、 进程监控工具 atop，htop、内存使用状态 free

3.网络参数工具

包括：Linux网络统计监控工具 netstat、显示和修改网络接口控制器 ethtool、网络数据包分析利刃 tcpdump、远程登陆服务的标准协议 telnet、获取实时网络统计信息 iptraf、显示主机上网络接口带宽使用情况 iftop

4.磁盘参数工具

包括：磁盘卸载 umount、读取、转换并输出数据 dd、文件系统系统 df、磁盘挂载 mount

5.日志监控工具

包括：实时网络日志分析器 GoAccess、多窗口之下日志监控 MultiTail、日志分析系统 LogWatch/Swatch

6.参数监控工具

包括：监控apache网络服务器整体性能 apachetop、ftp 服务器基本信息 ftptop、IO监控 iotop、电量消耗和电源管理 powertop、监控 mysql 的线程和性能 mytop、系统运行参数分析 htop/top/atop

**总结：以上知识点比较多、但是想要真正了解后台开发就必需要了解跟熟悉的掌握这些知识点内容。在你以后的工作中看的是会要用到。熟练掌握以上知识点内容你的水平就达到了后台开发工程师了。**

![img](https://pic3.zhimg.com/80/v2-3421a8b795b5a401287c7a8e250dc5a6_720w.webp)

原文链接：https://zhuanlan.zhihu.com/p/96792003

作者：Hu先生的Linux