# 【NO.535】服务器开发,无锁消息队列实现(初步认识)

## 1.多种锁效率对比



![在这里插入图片描述](https://img-blog.csdnimg.cn/76463554390143c4958019abe5b00fb1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)


mutex
blockqueue:condition+mutex
lockfreequeue:指针方式的无锁队列
arraylockfreeque:数组方式实现无锁队列
ypipe:是接下来的额重点
无锁队列应用在元素非常多，每秒处理几十万的数据，几百个数据的话就是杀鸡用牛刀，没有任何帮助。为什么几百个数据不合适？因为几百个数据经常会造成condition_wait处于阻塞的状态，所以建议数据量要足够的大。



## 2.ypipe的特性：

- 一写一读，不支持多读多写，多读多写会消耗一定的内容。

- 链表方式去分配节点，采用chunk机制，减少节点的分配时间。chunk就是一次分配多个节点。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/f44336eb2e294b9883d671723e48c671.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)

  个人理解Darren老师讲解的意思是使用chunk一次分配多个节点，可以节省分配资源的耗时。从20-30-20数据量有这个变动的过程中，chunk元素也会有所波动。为了提上性能，中间30会回到20个元素的状态，不过多出来的10给元素位置并不会立刻释放，而是保留最新的数据，等到时机成熟便可以轻松使用。

  ## 批量写入

  批量写入在kafka，tcp都有所应用，通过多次写入统一调用flush函数读端才能显示，目的是提高吞吐量。

## 3.无锁队列原理

如果使用condition+mutex这种方式，如何知道读端是在休眠我正好去唤醒？以前也从来没有考虑过这个问题，不可能发一个消息就去notify，效率会大打折扣。notify这个东西导致用户态和内核态相互切换，肯定是会影响效率的。

### 3.1 当读端没有数据，这时候应该怎么办？

- sleep睡一两毫秒算是一个办法，不过这样会导致吞吐量上不去。只能用condition_wait+mutx 这钟方式。

  #### 写端如何去唤醒数据呢？

  condition_wait+mutex这种方式了。



![在这里插入图片描述](https://img-blog.csdnimg.cn/5b16795d2e104438a0fd214c6bf5c71f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)



- 大部分情况下，end_chunk和back_chunk指向的都是相同的一个chunk。

- back_pos表示当前chunk要插入的位置，而end_pos表示结束的位置。

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2fcdf8ce0cf54fdc9662763b8414005c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/bfbef2f5b3164deb994b1438475c20a1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bGv6Zeo5bGx6bih5Y-r5oiR5bCP6bih,size_20,color_FFFFFF,t_70,g_se,x_16)

  需要注意的是，使用无锁队列也要解决阻塞和唤醒的问题，如果性能差别不大那使用无锁队列就值得商榷了，性能提示10%都没必要考虑使用无锁队列。

  ## 4.总结

  源码剖析这部分笔者的理解确实还不到位，所以不想妄加评论，留在下一期继续更新。通过Darren老师本节课程的讲解，我只是初步认识到了无锁队列的概念，大概明白了原理。更重要的是，认识到了无锁队列的重要性，如果性能提升不大就可以不优先考虑无锁队列。如果项目上有需要，最优的策略也是先完成工作，应付好任务，到项目性能需要提升时，再仔细的研究无锁队列，毕竟无锁队列想要商业上应用，还是一件比较困难的事。确实是有必要学习上一期的无锁队列，一天能把无锁队列研究清楚就已经是一件令人开心的事了！

  原文作者：[屯门山鸡叫我小鸡](https://blog.csdn.net/sinat_28294665)

  原文链接：https://bbs.csdn.net/topics/604954011