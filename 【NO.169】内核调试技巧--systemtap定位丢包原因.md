# 【NO.169】内核调试技巧--systemtap定位丢包原因



内核收发包，可能会由于backlog队列满、内存不足、包校验失败、特性开关如rpf、路由不可达、端口未监听等等因素将包丢弃。

在内核里面，数据包对应一个叫做skb(sk_buff结构)。当发生如上等原因丢包时，内核会调用***kfree_skb***把这个包释放（丢掉）。kfree_skb函数中已经埋下了trace点，并且通过__builtin_return_address(0)记录下了调用kfree_skb的函数地址并传给location参数，因此可以利用systemtap kernel.trace来跟踪kfree_skb获取丢包函数。考虑到该丢包函数可能调用了子函数，子函数继续调用子子函数，如此递归。为了揪出最深层的函数，本文通过举例几个丢包场景，来概述一种通用方法，来定位丢包原因及精确行号。

## 1.**用例1：ospf hello组播报文被drop，导致ospf 邻居不能建立**

### 1.1 方法：saddr是10.10.2.4的skb是我们关心的数据。

1、 drop_watch跟踪下kfree_skb，定位函数位置：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWt7b2VIqjUyrLCdbr1nkx6AuUKz5gmic3CHAZ0CaYibMAveS4jxD4kaBA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

2、 查看ip_rcv_finish内核源码，编写stap脚本，通过pp()行号来跟踪执行流：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWR5z053Sj8uztIBiaMFg8vfEovM8xs6w887g60X3uicR7q2xGA573V4sQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWEB7BUQKcgL2JlUDaKjic2ZHlWeiamgQvdkjMw4QRuINY6E0lslsNWOJg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

3、 从行号可知ip_route_input_noref返回错误，编写stap脚本，查看ip_route_input_noref的返回值：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWPBkwQCH6d9xSsiasicPE7SpCIiag7c0CNqk0VK4AVy4jeQ7mKlg95wBBg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

返回-22，即-EINVAL。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWb3q5pYd2AFkWR161x4K0lAB5hWhZDDicMialCibb7P4aUopE9iaFLtdkHQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

4、 即ip_route_input_rcu返回错误，同样方法，通过pp()行号来跟踪执行流：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWR6FNIjH5vrgiaQDicYic34lHz0V9cbwl4B6PZ3cc4yWdjtbvicuf3ibnkrg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWjC3gz5mRkA6iavia70BTicpAvibMKsLia99LgB4umDicQic95ToRWWUYkVK6w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWOmXD64Z5PGSOvYXf8CxyskI61qbM6SrfJpYmXsQzG7mIlRovng3iaVQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

此路不通，看下原因：原来有些行号$saddr不能访问。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW5xdhWjAGWJDqYicxjPVZScgP6v6ZmVBNMxDGO9Yyuhpq3uMwzIDpCpw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

5、 此时需要码一码代码了，由于ospf的hello报文是组播,所以下图中红色方框的ipv4_is_multicast为真：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWaJgHns98IDDjiadl7qS2w5K0y9vH2iatodJK0yKrhOnSg5kias2NJW2Qw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

先看下__in_dev_get_rcu(dev)的返回值是否为null，同时也把dev->priv_flags的值打印出来：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWtAxE3pKOEqwUp4xzZBs3rxZnxW1ibfrBLLL7emSXGUOvhtcQuHa41hA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

__in_dev_get_rcu(dev)返回不为null，再看ip_check_mc_rcu函数的返回值：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW7E4M1VDicmVnWSzeHRzRgbp6IWQFRoggHoCuHOEu1je7y0SfArqzsiaw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

返回值为0，同时通过打印的dev->priv_flags 为0x00029820，可判断netif_is_l3_slave为假：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWXoicDF05Dp8ribicfr6ZnzPWBMpt4eXL68NfRLOdpSwhz2icwh67b6catA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

另外ospf使用的组播ip为224.0.0.5或224.0.0.6，本例中使用的是224.0.0.5。因此ipv4_is_local_multicast为真：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW4KE16ia9ufic6rfiaazjjCBGug4t8NxQCS3xKyHLYOeSxk8rYW2gLvDDA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

最终精确定位到了ip_check_mc_rcu：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWUt6OVlrFwjIK0XRVZic4jtFUYvLRsQnafTrOhAtpr0CeqjfrfsftcDg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

先看下in_dev->mc_hash是否为null，通过cast强转来访问结构体成员mc_hash：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWEdHnuBF7ITfSXZjdQvIRdUhsZeRdeTcPqyUTESSiaibCG1yJsRt6UVRA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

则需要走如下分支：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWc5jC2NYRJsux16U9q98Zf0iclkibo6BSvH2s3OKqicF0BMCqFwnrTDrPg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

需要遍历in_dev->mc_list，有两种方法，一种是通过crash工具查看，遍历in_dev->mc_list，输出im->multiaddr,如下：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWleEWV3o2rcJ5JnicnlFBTDeeSMAkicVZMjSX2YwNmoLz63SIqo2YGLEA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

既然我们的主题是stap，那么继续stap，这里通过嵌入c代码来遍历mc_list,输出multiaddr到dmesg：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWbotsGluNnsicibh1HnnwMHcSVRFn6eB0WibKIe8Wia7EsXZkcLaBS14VXA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWB9hofbnvb8AkHHzbBegwjmgfWboNKPPPSDBClHVlysQDNnKbBPYOVg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

存在接口加入了224.0.0.5组播组，革命尚未成功，继续跟踪下面的代码：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWJ0kq9byAA67I1CDLXibfs8E7GibJgE5D9iad3diaicCfbyf4WTVmgaNAiaYw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

同样的方式，编写stap脚本或者crash通过struct展开im->sources来查看。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW8CyKyhbIVTH9LrOJiaHbibGd8HZNfpp6nIKRkFgZib4zZJmqwBiath6fQA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWGrzQRcLoGkj2hJzXKpGVPKAjwCvQibJyB2ozGCXUg9ezvHJbVrTglsw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

至此，一目了然，原来报文接收的接口没有加入组播组224.0.0.5。google一下：

https://www.cnblogs.com/my_life/articles/6077569.html

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWVYwtP5kxugJ7OserXwABmydL3LgbBbvwogAvtmIeuKTvUs78cYpCjg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

综述：那为什么ens5没有加入组播组呢，这要从ospf的原理来说起，ospf建立邻居的时候，是不需要指定接口的，那用于建立邻居的接口是如何选择的呢：实际上是根据指定的area network配置来选择的。当配置area network的时候，会查看系统当前路由，选择合适的接口加入组播组，进而创建邻居。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWV7oYrKUdg1LBnJFx1DLxdyEmm5oLcgdqzyZxd4Ra9Xf6iadpPtGt7mQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW4x9aSucxBWmrdyaUpETGicYeBB1RdRoHYOPb8DgQIibqCzV92Zkt7MrQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWI20bvgLVRZBI9aZBQWcvJwwzpY8oykHz51LfDu9YgMwzmbZhiapHX9Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWfjSCjraL9mk9lVs3LTPhUQ0ibzFXIDviaMwXlS1mD9wF0OHz6eBhy1icA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW4fbgWnQdSXuAALslrPmc056icsdKkD9ib9C5vSIBGzUN8RuoIkOibYUsg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.**用例2：gre报文的version字段被置位，导致skb被drop。**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWjnFZYbH3bH9jKFztx0QGJWkJoxoI9DKaqr0fFWL4pNK8KlW496JSWw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWib1hf37UtgK0qM8ZNZyaiatErEH1btk5XfFx76icV67ldpILqyS5A0w2A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.1 方法：saddr是10.10.2.2的skb是我们关心的数据。

1、 依然是drop_watch跟踪下kfree_skb，定位函数位置：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW9qa0b5jeh5E1icFuvDhEcf9b1SMuJ6dD8mrB2cmiclNRIDcKg0LruoGA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

2、 查看gre_rcv的源码，有两个内核模块都存在gre_rcv,由于上面的drop_watch已经定位为ffffffffc099411a，通过crash查看模块的加载地址，来确定调用的是哪一个模块的gre_rcv:

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWm16uF1Yarz7nhwRmmzlEWhwQhB10LzP1T8s8LicEgymcUOzZn2QELew/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

3、 依然是pp()行号来跟踪执行流,和上述不同的是，gre是模块的形式，使用stap的probe module的方式：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWCmjwNRfFRnAC1MUgS78ojSUC558qEnW9sawYyCd2Tah1ib1PD86VRTQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWowBJWBYKJ2ziakFwWM45wibbOZu8ibrztFOc13p8LicxBt5StSmo70KIMg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

综述：这里大家要小心，这个存在”写”skb相关成员，不建议嵌入c代码的方式，去修改skb，如果真的要怎么做，记得恢复回来。

## 3.**用例3: overlay报文根据路由找到出的vxlan隧道接口后，没有从underlay接口出,导致网络不通：**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWCesCQLu7uicDHUicssUdEyya3aW6zqNQNxrs0VWrwuY7TLOCwW3oXOkg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 3.1 方法：saddr是172.16.14.2的skb是我们关心的数据。

1、 依然是drop_watch跟踪下kfree_skb，定位函数位置：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWXP6khzFBdMp48sgDCWyRmeZiaOM0W16poF65gpopDV9LfWErhiaLpezg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

2、 查看vxlan_xmit_one，依然是pp()行号来跟踪执行流：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWWXGpfbge7tibRKW6g0r0YNPm9p4dPSvjuWcevSRmqcia5Jom02TqE2Tw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLWqmP9WxMWNyEEepRLe0vO3WibJZJXKFIZJoVshOz5VfNTxpRSxjGmvNw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvat460JtIJeNfiam0HJOcUOLW3vydBdLGDoujNG9MD0f9UjNEAcGuZZmibp596aZic2L1uNCUEKQJbhSQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

综述：从接口中取到underlay信息后，再去查找路由，由于underlay路由不存在，导致skb被drop。这个问题较简单，也可直接通过查看ip route定位。但是细心的你一定会发现一个有趣的问题，关键字overlay arp，欢迎读者来撩。

## 4.总结，丢包精确定位行的方法：

1、 drop_watch先定位函数。

2、 使用pp()定位行。必要的时候，编写一些脚本，直接抄写内核代码或者调用stap库就可以了。

3、 递归重复步骤1和2。

是不是跃跃欲试的感觉。

**最后：**

这里”rpf检查”，”accept_local检查”留给读者来尝试了。实际上systemtap可以做的更多，如内存泄露，系统调用失败，统计流量等等，github上也有很多实用的脚本。

参考链接：

https://cloud.tencent.com/developer/article/1631874

https://www.cnblogs.com/wanpengcoder/p/11768483.html

原文作者：wqiangwang，腾讯 TEG 后台开发工程师

原文链接：https://mp.weixin.qq.com/s/xO2dkyZqKnUcaEXn_B39rw