# 【NO.542】协程的设计原理与汇编实现

**第一个问题，为什么会协程？以及协程到底解决了什么问题？**

第一个同步的方式实现异步的效率
**同步为什么效率低，而异步的为什么效率高？**
等待没错，他需要等待，那这个等待是什么意思？
第一个案例之前跟大家讲的100万并发的案例
第二个案例异步请求的案例

![在这里插入图片描述](https://img-blog.csdnimg.cn/c7510c55a62244e9903d72b54be90333.png)



现象区别：
if 1 用了 workqueue，每1000个耗时 1400
对fd直接进行写，每秒接入量5600

**那现在这个工作队列他为什么在这个过程引入工作队列，他就能够提升性能呢？**，

他是从哪些方面原因那这个要跟他分析这个要跟大家分析，
直接进行读写

![在这里插入图片描述](https://img-blog.csdnimg.cn/e8109b97b4614da18a5bb6fd1695479f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_8,color_FFFFFF,t_70,g_se,x_16)


工作队列

![在这里插入图片描述](https://img-blog.csdnimg.cn/76dd3e5e62cc44b2b6e198d4f30e9c8d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_8,color_FFFFFF,t_70,g_se,x_16)


原理类似

![在这里插入图片描述](https://img-blog.csdnimg.cn/597eef35327143c1be6bfdabbd376efa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_17,color_FFFFFF,t_70,g_se,x_16)


同步：检测io与读写io在同一个流程
异步：检测io与读写io不在一个流程



同一个流程没有切换的概念

**异步性能高，同步有什么好处？**
逻辑清晰，效率不高

**那有没有一种方式采用同步的编程方式还同时拥有异步的性能？**

那我们如何把它改进成一个同步的编程方式
**那实现的原理是什么呢？**

![在这里插入图片描述](https://img-blog.csdnimg.cn/c8a502c265d0400781abdce81ed0c025.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



king式4元组异步连接池。
有没有一种方式只管发sql过去，不去等待他的结果，这样就造成异步做法

能不能把MySQL请求改成异步的，包括mongodb,redis,rpc都是可以改的。
这里是DNS请求
异步的：

![在这里插入图片描述](https://img-blog.csdnimg.cn/af2fa6a7f994421093b99cfca6b9ec6a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_13,color_FFFFFF,t_70,g_se,x_16)


同步的

![在这里插入图片描述](https://img-blog.csdnimg.cn/273114928dbc4f028449f2ceb2cb75f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_11,color_FFFFFF,t_70,g_se,x_16)


异步结果是一起方法的，是有方法保证顺序的。



**核心：同步的编程方式，要有异步的性能
那么是怎么做的呢？**
两个关键的地方：
1.commit
2.callback

![在这里插入图片描述](https://img-blog.csdnimg.cn/a6082a8f0e984e78a2210fa8a09848e0.png)





![在这里插入图片描述](https://img-blog.csdnimg.cn/5c6124c607ea4567be8fe55094b0a11d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)



**异步提交完了之后我们怎么改成同步的方式？**
**这一点就是协程是如何实现的？**



![在这里插入图片描述](https://img-blog.csdnimg.cn/22caf11f269d4f55aa0dab4dfb7cf9f5.png)



对应的callback地方就是检测结果的返回有没有数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b274494fbfcb4df8b4101531629bea82.png)



确确实实不在一个流程里面，callback是申请一个子线程做的

![在这里插入图片描述](https://img-blog.csdnimg.cn/df15ed9045f543b8ab203d173aa402b1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/0495ea4bff7246c7aebac74458132a0a.png)


我们怎么改成一个流程里面：我们提交完就可以接受数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/27875b8c8de74f86bbcc36395c0d0689.png)



看上去他们不在一个流程里面但是我们要改起来让他们在一个流程里面并且还没有副作用。
**那怎么改？**

在fd加入到epoll里面我们在应用层引入一个yield();
**注意这个让出，让出完了之后到达哪呢？**

![在这里插入图片描述](https://img-blog.csdnimg.cn/93bc12491c4f4d48a403e0b4b02d46a4.png)



**让出完了之后到达read这个地方，解析完了之后会有一个resume恢复;**

![在这里插入图片描述](https://img-blog.csdnimg.cn/cc14e8cdcd924aacb230a13e955119c7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_9,color_FFFFFF,t_70,g_se,x_16)





![在这里插入图片描述](https://img-blog.csdnimg.cn/2662938d5ebb4a3c863a51f9078ffbcf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_9,color_FFFFFF,t_70,g_se,x_16)



给我们感觉它没有切换的，对应的引入了两个原语操作
一个是跟大家讲的yield()，另外一个就是resume()。
**让出是让出对应的流程在resume返回对应的流程
这神来之笔就是协程的实现**

**那么我们该如何实现yield()以及resume()？**

![在这里插入图片描述](https://img-blog.csdnimg.cn/da34d059d37c403a9bc4755034a00d47.png)



1.yield()让出时使用longjmp跳转到read fd；resume()恢复时在使用setjmp跳回来
2.linux下面的ucontext
3.或者汇编自己跳转

yeild() 让出这里有俩个步骤，一个是longjmp() 标签一; setjmp() 标签二回来
让出到哪呢？

![在这里插入图片描述](https://img-blog.csdnimg.cn/cdc4653ec8c342d6beed48d2285f9698.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)


在讲yield/resume()之前有没有发现他们有共同的动作：是俩个上下文环境的切换，从a往b跳，从b往a跳；

![在这里插入图片描述](https://img-blog.csdnimg.cn/2798f3c83bbe40028c1e15893f52d62e.png)


看似是汇编其实代码可读性比longjmp/setjmp还有ucontext都要强很多



把读io，io没有准备就绪的这个等待取消掉，性能会无线的接近异步方式



![在这里插入图片描述](https://img-blog.csdnimg.cn/a52de782b2414859b0e6ee0ed5b6d48b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_9,color_FFFFFF,t_70,g_se,x_16)



协程究竟解决什么问题？
注意同步与异步的效率？

reactor和协程之间区别？

![在这里插入图片描述](https://img-blog.csdnimg.cn/92685f900ae14333986eea01639a0887.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_14,color_FFFFFF,t_70,g_se,x_16)


接收完数据，解释完数据，把数据放到sendbuff里面，再把对应的fd设置epollout
recv和send是在俩个函数本质上它还是同步；
coroutin协程是在一个函数，他们性能会差不多，只是协程会跟符合人思考方式

![在这里插入图片描述](https://img-blog.csdnimg.cn/0608ff2452cf4298828ae8a8a4ea225e.png)


栈是函数调用必须要用到的东西，没有栈的话里面就根本没办法做函数调用，不推荐共享栈协程，往往共享栈协程的管理会更加复杂，
怎么确定栈大小？是自己设定的，如果你要做文件传输的话可以设置成1M或者10M，如果你只是单纯接收一些字符数据你可以开到1K或者4K；



**协程的切换到底怎么实现？**



![在这里插入图片描述](https://img-blog.csdnimg.cn/390309ee5bc4421b8b32c7e34648a82e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


**这个让出动作怎么做呢？**就是把CPU里面的寄存器组的值保存到协程A里面，**这个A怎么去保存CPU寄存器的值呢？**



## 1.协程切换

以X86为例，32位的，那这个寄存器的值也是32位的，它里面是有具体的值的，所以我们可以定义一个结构体去对应的保存值

![在这里插入图片描述](https://img-blog.csdnimg.cn/9269c751467e403b9fa0a995d2f86507.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_11,color_FFFFFF,t_70,g_se,x_16)


保存完了之后在把B加载到寄存器这样就实现了A和B的切换；

![在这里插入图片描述](https://img-blog.csdnimg.cn/83c52fba5c494af68bbd46e041e55947.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)


为什么后面是定义r1,r2,r3,r4,r5呢？为了32位与64位兼容





![在这里插入图片描述](https://img-blog.csdnimg.cn/2aac8210824943cc8fa035b560ec0124.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_19,color_FFFFFF,t_70,g_se,x_16)


前面这一段是保存的意思

![在这里插入图片描述](https://img-blog.csdnimg.cn/9a6013616331431db2dcdd753adb9e9b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_16,color_FFFFFF,t_70,g_se,x_16)



将rsi保存到rdi里面

![在这里插入图片描述](https://img-blog.csdnimg.cn/dbd142563e87423189b0eef985278a68.png)


这里偏移0就是结构体第一个值，就是说的每一个大小是8个字节，64位里面一个指针对应的是8个字节；

![在这里插入图片描述](https://img-blog.csdnimg.cn/d459ddbf6ea54906ae2b0938a4f1a578.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


movq的过程

![在这里插入图片描述](https://img-blog.csdnimg.cn/9da071522eae41e0acb4ba9d910c53f1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



下面这一段就是加载的过程，最后ret返回

![在这里插入图片描述](https://img-blog.csdnimg.cn/2990cb9075cf4db6a5b9137ce6b98b36.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)


PS：

![在这里插入图片描述](https://img-blog.csdnimg.cn/fff06a0ea4a4416dbc1a12b76be92fde.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/85e538d51de241b1ad0717dc74f5d2da.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


有一些值存的是flag还有一些存的是临时变量的值我们可以不用保存；



## 2.yield 和resume的过程



![在这里插入图片描述](https://img-blog.csdnimg.cn/8b0baed72ddb44acb693f10fc0105efd.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/28274b73660a47cc8370a110258a1a25.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_16,color_FFFFFF,t_70,g_se,x_16)


可以注意到都依赖于_switch这个函数



## 3.协程的启动

协程的启动有如下三个问题：
思考一下线程是怎么启动的，启动是什么意思？



![在这里插入图片描述](https://img-blog.csdnimg.cn/652d6b9f08994e789a86951495e4fcd9.png)



**现在有一个问题入口函数的这个函数把它加到哪里？**
刚刚比较好理解就是A和B是已经运行的比较好切换，如果说现在B是一个NEW的状态，**那么此时他的寄存器里面保存什么值的，一加入进来怎么保持运行呢？**
**就是第三点这个协程运行把它推到CPU上面这个指令怎么做？**

![在这里插入图片描述](https://img-blog.csdnimg.cn/93dfe58f39bf4923bf70d60f9f05ec22.png)



第一个把eip指针指向入口函数fun，也就是把fun的地址指向eip，然后再对应的把它的参数保存下来
还有一个东西就是关于这个栈指针这个栈esp，等下会有函数调用这个栈指针的首地址也要保存起来，

![在这里插入图片描述](https://img-blog.csdnimg.cn/27635eadd6884223b27d0e2c89d65252.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


是为_exec压栈准备好的，所以要esp栈顶指针减去那个4，就是参数



把入口函数的地址换了

![在这里插入图片描述](https://img-blog.csdnimg.cn/68c0641117c341799db3e3d644d9a0e1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_16,color_FFFFFF,t_70,g_se,x_16)


栈是从最底部开始指的：

![在这里插入图片描述](https://img-blog.csdnimg.cn/d92a127d464747318a0458469fe76224.png)


核心的就是俩个指针：esp还有eip



## 4.协程到底是怎么调度的

在说明调度之前先理清一下协程是怎么定义的？

![在这里插入图片描述](https://img-blog.csdnimg.cn/4982b73ebf344ccaa10f046a30a18828.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_8,color_FFFFFF,t_70,g_se,x_16)



假如有十个协程我们使用什么数据结构来存储？

看一看协程源码：有红黑树还有就绪队列去存储集合

![在这里插入图片描述](https://img-blog.csdnimg.cn/4c2af5ec63f340f084f8b50225037401.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_11,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/4b1352e722404d9fbf5b6c36a213e280.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_14,color_FFFFFF,t_70,g_se,x_16)


这些集合之间的关系是怎样的？





![在这里插入图片描述](https://img-blog.csdnimg.cn/789bc5f2e13d40a3971852fcfef9af19.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)


这些数据结构组织在一起，它是由多种数据结构共用的这些节点，6，7，8这几个是同时拥有的的至少在其中一个；



## 5.调度器又如何实现的

假如这里有50个协程，那么这50个coroutine我们如何组织起来，并且使用何种顺序去调度呢？
1curr_coroutine当前栈是哪个？
2.//如果是共享栈就可以放调度器里面不放协程定义2里面，
3.调度器的调度方法的具体实现可以拆出来，内核里面有cfs还有rt



![在这里插入图片描述](https://img-blog.csdnimg.cn/7fdb0123644b4e5eb2f5a6361cec1c28.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_19,color_FFFFFF,t_70,g_se,x_16)


协程调度里这三个集合是怎么组织起来的呢？



**fd如何知道是否就绪？**
在调用send还有read之前，我们将fd加入epoll管理，在引起切换yeild()，在epoll_wait()

![在这里插入图片描述](https://img-blog.csdnimg.cn/c42329b4ec9547aaaf97406de8504093.png)

原文作者：[[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605053587