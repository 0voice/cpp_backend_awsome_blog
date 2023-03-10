# 【NO.38】一次解决Linux内核内存泄漏实战全过程

什么是内存泄漏：

程序向系统申请内存，使用完不需要之后,不释放内存还给系统回收，造成申请的内存被浪费.

发现系统中内存使用量随着时间的流逝，消耗的越来越多，例如下图所示：

![img](https://pic2.zhimg.com/80/v2-238d2a09f200f11e145b2bf263456de9_720w.webp)

接下来的排查思路是:

1.监控系统中每个用户进程消耗的PSS (使用pmap工具(pmap pid)).

PSS:按比例报告的物理内存，比如进程A占用20M物理内存，进程B和进程A共享5M物理内存，那么进程A的PSS就是(20 - 5) + 5/2 = 17.5M

2.监控/proc/meminfo输出,重点观察Slab使用量和slab对应的/proc/slabinfo信息

3.参考/proc/meminfo输出，计算系统中未被统计的内存变化，比如内核驱动代码

直接调用alloc_page()从buddy中拿走的内存不会被单独统计

以上排查思路分别对应下图中的1,2,3 :

![img](https://pic1.zhimg.com/80/v2-65fd3146cde9f3843d75b0e49ebf6c9c_720w.webp)

在排查的过程中发现系统非常空闲，都没有跑任何用户业务进程。

其中在使用slabtop监控slab的使用情况时发现size-4096 不停增长

![img](https://pic1.zhimg.com/80/v2-7d36f746609a6582b8a37a0ceb84a4d4_720w.webp)

通过监控/proc/slabinfo也发现SReclaimable 的使用量不停增长

```text
while true; 
do 
sleep 1 ; 
cat /proc/slabinfo >> /tmp/slabinfo.txt ; 
echo "===" >> /tmp/slabinfo.txt ; 
done
```

由此判断很可能是内核空间在使用size-4096 时发生了内存泄漏.

接下来使用trace event(tracepoint)功能来监控size-4096的使用和释放过程，

主要用来跟踪kmalloc()和kfree()函数对应的trace event, 因为他们的trace event被触发之后会打印kmalloc()和kfree()所申请和释放的内存地址,然后进一步只过滤申请4096字节的情况。

```text
#trace-cmd record -e kmalloc 
-f 'bytes_alloc==4096' -e kfree -T
```

(-T 打印堆栈)

等待几分钟之后…

\#ctrl ^c 中断trace-cmd

\#trace-cmd report

以上步骤相当于：

![img](https://pic1.zhimg.com/80/v2-112f1fb0265b037be2c84bce983cfaa4_720w.webp)

等待几分钟之后…

```text
#cp /sys/kernel/debug/tracing/trace_pipe  /tmp/kmalloc-trace
```

从trace-cmd report的输出结果来看，很多kmalloc 对应的ptr值都没有kfree与之对应的ptr值

![img](https://pic3.zhimg.com/80/v2-d7869c3e5f54a87033e1a79d41a7e3ea_720w.webp)

这就说明了cat进程在内核空间使用size-4096之后并没有释放，造成了内存泄漏。



为了进一步精确定位到是使用哪个内核函数造成的问题，此时手动触发vmcore

```text
#echo c > /proc/sysrq-trigger
```

然后使用crash工具分析vmcore：

```text
#crash ./vmcore ./vmlinux.debug
```

读出上面kmalloc申请的ptr内存信息

![img](https://pic3.zhimg.com/80/v2-6df11cad7524f9342bbb729e7240e2b6_720w.webp)

(读取0xffff880423744000内存开始的4096个字节，并以字符形式显示)

![img](https://pic3.zhimg.com/80/v2-bd6f9fe92a846e9c8bf66d9446ac80de_720w.webp)

发现从上面几个ptr内存中读出的内容都是非常相似，仔细看一下发现都是/proc/schedstat 的输出内容。

通过阅读相关代码发现，当读出/proc/schedstat内容之后，确实没有释放内存

![img](https://pic4.zhimg.com/80/v2-c6ea82ba477dcda95180e5f9a530e4c3_720w.webp)

然后发现kernel上游已经有patch解决了这个问题：

commit: 8e0bcc722289

fix a leak in /proc/schedstats



原文地址：https://zhuanlan.zhihu.com/p/577973486     

作者： linux