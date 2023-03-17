# 【NO.526】Linux服务器开发,fastdfs架构分析和配置

## 1.前言

了解fastdfs存储，对了解ceph、hdfs都有很大的帮助。对于数据存储，我们最关心的问题是：

- 文件丢失，比如某一个盘崩了会不会导致我们所有的盘丢失。
- 上传速度
- 下载速度
- 水平扩展-group多个分组

fastdfs可以应对单点故障，是弱一致性的存储方案，一个storage是一个服务器，3个storage就够了否则会影响同步的效率。

## 2.框架



![在这里插入图片描述](https://img-blog.csdnimg.cn/49c0a312577f4e71941d438646f08276.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



Tracker Server：只是作为代理，并非实际存储文件的。
Storage Server:是文件存储服务，同一个group可以有多喝storage，每个同group的storage文件是一样的。一个storage可以挂载多个磁盘。

- /组名/磁盘/目录/文件名

比如当我们上传一个文件到group1时，存储到了storage1，那就要询问storage2有没有被同步，否则不能到storage2中去下载，后面会继续深入探讨。

![在这里插入图片描述](https://img-blog.csdnimg.cn/997e685c93ab4c91a7c3407083800bfd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



在少写多读的场景，可以一个group多个storage。小视频播放，1000人观看，3个和10个storage对比，肯定10个storage更加滋润。

一般的，Tracker Server和Storage Server不会部署在同一台服务器上，要分开部署。

```bash
lsof -i:23200
```

## 3.总结

Darren老师建议备份一下nginx服务，因为担心不熟悉。我觉得从这件事可以明白一个道理，就是对于不熟悉的东西，应该留雨余地谨慎操作。通过今天老师的讲解，对fastfds项目有了一个大致的了解，对于一个新手来说，配置文件和通讯流程还是比较复杂的，相信下一节课我会涨更多的见识。

原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

原文链接：https://bbs.csdn.net/topics/605063191