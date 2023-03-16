# 【NO.477】协程的调度实现与性能测试

那我怎么在简历里面写协程，协程这个东西实在真的太好用了，你可以跟很多东西结合到一起，比如说你们对文件，做文件操作可不可以用？
好对文件操作，比如说你做日志落盘的时候，可不可以用协程来操作它也是可以的，比如说你对数据库的操作，对数据库的操作，还有包括像一些网络io的处理，这个文件的操作和网络io都是针对文件io来处理。

然后我们尽量知道他用到哪里，协程怎么用到数据库？这是我们今天等一下跟大家讲到的讲到的就是关于携程的API的封装。

**一个线程里面多个协程是怎么运行的？**
好比多个线程在一个进程中是怎么运行的，
每一个协程当它遇到io操作的时候，它就会去判断io是否就绪，如果没有就绪就会让出，**这个让出它让给谁了？**
站在应用层：让给其它协程这个说法是对的，
站在协程设计者的角度，这个协程是让给调度器了。
由调度器判断哪一个协程准备就绪了，resume该协程
大量的时间运行在哪？是在调度器上面



![在这里插入图片描述](https://img-blog.csdnimg.cn/d51d5ee535d044b390117e2d51f0ff88.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



这里一个问题我们先留在这个地方，吧就是这个协程如果一直不让出。

![在这里插入图片描述](https://img-blog.csdnimg.cn/97160158a09347808747b3aaf7752e67.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/91df0fe37f434dc3b7e9b9159f8305c0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_13,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/f05553234cc04dd0ad2a7ad728141264.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



```cpp
#include "nty_coroutine.h"



 



#include <arpa/inet.h>



 



#define MAX_CLIENT_NUM            1000000



#define TIME_SUB_MS(tv1, tv2)  ((tv1.tv_sec - tv2.tv_sec) * 1000 + (tv1.tv_usec - tv2.tv_usec) / 1000)



 



 



void server_reader(void *arg) {



    int fd = *(int *)arg;



    int ret = 0;



 



 



    struct pollfd fds;



    fds.fd = fd;



    fds.events = POLLIN;



 



    while (1) {



        



        char buf[1024] = {0};



        ret = nty_recv(fd, buf, 1024, 0);



        if (ret > 0) {



            if(fd > MAX_CLIENT_NUM) 



            printf("read from server: %.*s\n", ret, buf);



 



            ret = nty_send(fd, buf, strlen(buf), 0);



            if (ret == -1) {



                nty_close(fd);



                break;



            }



        } else if (ret == 0) {    



            nty_close(fd);



            break;



        }



 



    }



}



 



 



void server(void *arg) {



 



    unsigned short port = *(unsigned short *)arg;



    free(arg);



 



    int fd = nty_socket(AF_INET, SOCK_STREAM, 0);



    if (fd < 0) return ;



 



    struct sockaddr_in local, remote;



    local.sin_family = AF_INET;



    local.sin_port = htons(port);



    local.sin_addr.s_addr = INADDR_ANY;



    bind(fd, (struct sockaddr*)&local, sizeof(struct sockaddr_in));



 



    listen(fd, 20);



    printf("listen port : %d\n", port);



 



    



    struct timeval tv_begin;



    gettimeofday(&tv_begin, NULL);



 



    while (1) {



        socklen_t len = sizeof(struct sockaddr_in);



        int cli_fd = nty_accept(fd, (struct sockaddr*)&remote, &len);



        if (cli_fd % 1000 == 999) {



 



            struct timeval tv_cur;



            memcpy(&tv_cur, &tv_begin, sizeof(struct timeval));



            



            gettimeofday(&tv_begin, NULL);



            int time_used = TIME_SUB_MS(tv_begin, tv_cur);



            



            printf("client fd : %d, time_used: %d\n", cli_fd, time_used);



        }



        printf("new client comming\n");



 



        nty_coroutine *read_co;



        nty_coroutine_create(&read_co, server_reader, &cli_fd);



 



    }



    



}



 



 



 



int main(int argc, char *argv[]) {



    nty_coroutine *co = NULL;



 



    int i = 0;



    unsigned short base_port = 9096;



    for (i = 0;i < 100;i ++) {



        unsigned short *port = calloc(1, sizeof(unsigned short));



        *port = base_port + i;



        nty_coroutine_create(&co, server, port); ////////no run



    }



 



    nty_schedule_run(); //run



 



    return 0;



}



 



 



 
```

首先如果这个没有准备就绪，就把这个 io加入到epoll里面,这个epoll是schedule的epoll，也就是说对着这个图，
每一个协程他对io的操作，做的一件事情判端io有没有就绪，没有就绪把它加入到epoll里面。
请大家注意这epoll是一个全局的是所有协程都共用这一个，它是管理的所有的io的，把它加入进去之后，剩下的一个事情就是nty_coroutine_yield,
然后这个地方把它让出,让出请大家注意这里有1个点，就是回到了我们调度器的地方，回到调度器的地方，nty_schedule_search_wait在这个地方。
好，这里大家看到就是在前面是判断，如果这个循环里面往下面走nty_coroutine_resume，是返回到协程里面，请注意这里有个过程，
如果单独一个协调看不出来，用多个的时候，你才能够体现这个调度器的作用，
也就是说a让出之后执行到底这个点，请记住这个点就是我们要的这个点，然后在调度器往下走的时候再次返回。
这个resume的过程是走到了他最初yield的地方。
好，这样他就完成了整个时间片，

就是说这个让出的点，为什么io没有准备就绪，它能够切换出去。

就是一开始我们进行io操作之前把他加入epoll里面，加入之后，我们再让出，调度器在开始运行，调度器然后里面去判断哪些准备就绪了，然后resume到另一个协程，就这样一个过程

这就是协程在运行的时候，它底下就这样的，n多个协程，然后每一个都是从
调度器恢复到协程里面去，这就是调度器它的核心的原理就这样，很多朋友跟我聊的时候，就是这个协程的调度这让出和恢复，它怎么切换的这个点不懂。

关于现在协程一直不让出，
**就是这协程首先进行是为了解决io等待挂起的问题，**挂起的问题，啊解决io等待挂起的问题，
那如果这个协程它本身里面没有这种io操作的话，那么用协程的意义不大，
那就可以直接单独的你用线程去处理是一样的是一样的，那这个处理它就跟同步差不多，如果没有io操作，就不用东西。如果在跟这些朋友在讨论这种问题的时候，也时刻记录，如果没io操作，用协程干什么，可以用rpc底层的框架。

调度器里面在实现的时候，大家可以看到代码上面只做了三部分集合，只做了三部分集合：一部分是说的我们睡眠等待这个时间，第二部分就是就绪，第三部分就是io的等待。

这里只做了三个集合，第一个是说的我们做休眠的时候，这个时间超时的时候，那这里面有一个过程有一个情况就是关于这个时间就是sleep睡眠，
就是比如说我休眠两毫秒，现在我把当前的这个协程加入到一个sleep_tree里面，那对应说我们下次接下来再有一个休眠的时间，
又加入这个树里面，又有一个休眠我再加入树里面，那各位朋友就不太理解，**就比如说如果中间有一些时间相同的怎么办？**

是这样构建这样一个树，
构成这样一个数，这个数里面它的key就是时间，它的内容树在操作的时候是以key，value,那这个key是我们的时间，value对应的就是这个协程，那这个 key有没有冲突呢？

大家肯定会想到这个问题，就如果他说这个 key冲突的话怎么办？

就是如果当时间冲突的时候，各位朋友们在这里比如说我们一次性把它插入的时候，发现时间有一点点有冲突的话，会判断插入失败，
那插入失败了怎么办？请大家注意再次插入。
那再次插入在这边做了一个事情，就是对这个时间加上这么一丢丢，你再加上这么1纳秒加上一丢丢，

那我们可以看到
在这地方比如说我们调度nty_schedule_sched_sleepdown就是再去找到，我们这个协程里面，我们插入一个key的时候，如果时间冲突了，大家就可以看到
插入的时候如果插入失败，我们把这个sleep_usecs ++

![在这里插入图片描述](https://img-blog.csdnimg.cn/4575ca85802e4af09ef7bff2ab5ee3c8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


，请大家注意这是一个很小单位的加加就加一丢，丢那加完之后再次继续continue插入，
这样即使这里面多那么一丢丢，比如多那么一纳秒或者少了一纳秒，其实它不影响不影响这个调度，
因为对协程来说本身它的实时性它要求没那么高，
**关于时间冲突，我们就给他再加上一丢丢，就是定时器的冲突也是这么解。**



这是关于这个插入的时间。
再紧接着第二个方案，就是这是第一块判断这个时间哪些时间到期了，哪些时间到期了，我们就可以开始运行，哪些时间到期了，我们从中找到一个时间到期了，我们开始 resume恢复它的运行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/322db1d494a545dcbbd5db1c63219de1.png)



然后第二部分就是就绪，就是新建的也是加入到就绪里面，然后再去判断。这是采用一个就绪队列，队列里面拿出第一个点开始运行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/be177c1971f84531a84b032cc5d13f54.png)



第三个就给大家讲到的就是 io等待，其实在大量的io的时候，他会是在第三个步骤，是在第三个步骤里面的数量是很多的，而前面的两者第一个的它是很少的，第二个的会比第一个要多也不是很多，但是第三个是最多的，
或者你也可以想象，比如说我们现在有1000个协程现在创建完，其实你要发现每一个协程里面去加入一个sleep，这种情况是很少很少，它是主要是解决一个new，刚开始创建的时候，这个状态其实大量的是在处理io等待的时候这样一个情况。

就是一个红黑树只是把它写成宏定义了，是为了让我们多次去定义红黑树，也就是说我们在这个里面我们可能不止定义一颗红黑树，要定义多个红黑树，所以我在这里写就把它用了一个宏定义的方法，

**再跟大家讲讲就是关于这个协程多核的问题，**
解决形成多核的问题有这么几种方式，
有这么三种方式，第一种我们可以采用多进程，多进程的好处，实现起来很容易，就是我每一个进程里面，进程是单线程的，每一个进程里面都有一个调度器，那我们这个协程代码本身我们是不需要去改的。
第二个是采用的多线程的模式，
多线程会比多进程复杂很多，就是个多线程和多进程怎么联系一起的联系的，
为什么利用多核，我们是利用多CPU并行的去执行，是利用CPU的计算能力，那请注意多核来用的时候我们采用多线程和多进程。
我们可以每一个线程或者每个进程做亲缘性，
比如说每一个线程绑定一个CPU，亲和一个CPU，每一个进程亲和一个CPU，就是这样来做到多核的支持。
那有同学有没有一种不用多线程或者多进程呢？也可以采用叉86体系结构提供出来的指令实现

![在这里插入图片描述](https://img-blog.csdnimg.cn/72e2b2439d7840d79a710298cb6d94eb.png)



关于ntyco里面做的呢,是采用多进程做的，
因为这个很容易因为这个好做，然后呢这个多线程这个它不是很好写，什么意思？这个多线程中间就需要对我们的调度器进行加锁，
这个调度器加锁怎么加呢大家，你现在也想一想，

这里main函数进来之后，然后for循环监听100个端口，
0~99，然后创建协程100个，
然后里头做的事情就是监听，我们把它改成多线程也是ok，怎么改呢？

我们在这里直接把它创建多个线程，关于这个协程这个过程中间在调度的时候，
因为协程就出现一个现象，就是协程a在一个线程里面，协程b在一个线程里面，协程c在一个线程里面，协程d在一个线程里面，
他每一个协程都在不同的线程里面，**那么在调度的时候，请问你它核心加锁地方加在哪了？**

就是在调度器调度的时候加锁，
协程本身是没有关系的，是不会影响，就是对于应用层而言我们是不用管，我们压根就是代码还是这么去写，代码还是这么去写，但是请大家注意，
就是关于这个协程调度的时候，比如我们加入sleep这种树的时候，我们要对它进行加锁，
再往下面走这一层，它里面拿出一个已经超时的节点的时候，在里面
我们要对它进行一个加锁，也就是对于红黑树里面我们去找出一个节点出来，这个点要加锁，**如果我们要多线程支持这个锁我们定义在哪里？**

这个锁定义在调度器里面，全局的没错，在调度器里面，在调度器的这个结构体里面可以引入一个概念，我们在这里面我们可以加上一个，定义一个COROUTINE_MP他是否支持这个东西，这个里面就跟在这里面进行加锁，**那加这个锁的过程中间有哪些锁可以加？**。

那对于红黑树的话，就以sleep这个为例，
这个地方用红黑树的话，我们采用互斥锁是ok的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/9c8430c919584bb8873d375a9126150a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



那下面这个关于对列里面移出一个节点，我们可以采用自旋索，

![在这里插入图片描述](https://img-blog.csdnimg.cn/60185ab4eb534bfcbec99585e9a89d0f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



那还有下面这个里面，搜索一个等待的，我们也可以采用互斥锁，

也就是说这个如果对全局的对多线程的支持，我们可以加锁，可以加这两种锁，第一种是互斥锁，还有就是自旋锁。

现在再回忆一下，
这个协程对多线程的支持它是很复杂吗？它不是，但是难度有难度有。
为了使这个代码如果你是为了尽快的它以上线为主，以拿到实用为主，其实用多进程的做法，他不会去影响协程本身，但如果采用多线程的话，那我们需要对携程的代码本身进行一定的修改，

在回忆一下协程如何对多线程支持，对多线程支持，就是对协程的调度器进行加锁，它里面分为几个状态，每个状态的集合进行加锁，在调度resume这个过程中间，不会使得它出现一些副作用，

那大家可能会在想这里面会不会出现死锁的问题，会不会出现一些死锁问题，请大家注意。如果对于这锁本身，因为大家可以看到
内部分配，我们只是对这一个全局的这一块，它是一个全局的这一块它是全局的，因为这个协程的调度器是多个线程共用的一个，那这关于调度器它是一个共同那大家各位你仔细去回忆一下，其实如果你不参照业务代码进来的话，
线程死锁的概率是很低的，因为它是多个线程共用一个东西

协程的接口，如果你再去使用现成的时候，或者你自己封装这些的时候，携程应该定义哪些接口就可以？
第一个coroutine_create协程的创建都要有，好创建这个要有，
第二个scheduler_run(),就跟那个event_loop一样；
yield 和resume这个没必要提出来，lua提供出来了
**很不一样的答案很不一样的答案，就是accept()**

有朋友能够回到这个点上面，我认为是值得给他一些鲜花的。

就是关于网络io既然我们是对io进行操作，请大家注意协程所有io的操作不能用操作系统，而需要用我们自己再去封装，因为我们每一次对于IO操作，比如我们调用reed()，或者我们调用一个write()的时候，如果我们调用系统的read()的话，那这个 fd，**那这个协程，这个fd怎么能够到达协程的调度器里面，所以这里面我们需要在操作系统的read，write，这个系统的调用上面我们再封装这么一层来做。**

也就是说在这个 read和write这些做法的时候是这么做的，就是我们自己实现一个read或者write这个函数里面是什么?
就是这样判断的。
先判断这个io是否就绪，我们可以非常粗暴的一种方式，
就是只要此函数,读之前我们把所有fd全部加到epoll里面去，然后再把它让出，让出完之后我们再去调用我们系统的read这么一个流程。

![在这里插入图片描述](https://img-blog.csdnimg.cn/ff9e5e5d678d4ce0af1d524e7f7ada20.png)



那这个流程请注意它就改成了一个什么样的read，nty_read改完之后就变成了这么一个效果。

先加epoll，然后再切换,回来之后read，你就想一下这个read的这个过程中间，它现在变成变成了一个异步的read，这个read已经不在是我们最初系统调用的那个read，而是已经改为了一个异步的read。

这里面就分了一个上半部和下半部，也就是大家可能有没有接触到内核的时候，有个底半部和顶半部，这里也有一部分叫做上半部下半部，上半部就在切换之前，下半步呢就是在切换回来之后，就是以yeild为界,上面的为上半部，下面的为下半部，也就是他们两个的执行顺序，执行流程不在一起，
不是一个流程执行的，是先执行一段它，让出去再执行一个其他的read，再执行另外一个协程的read，然后执行完之后再resume回来，

**协程核心的封装关键就在这地方，这就是很多朋友聊到的这个read怎么把它封装成一个异步的read，就是这么做。**

了解清楚这个之后，我们再来把我们所有对于系统IO操作的这些，我们都需要把它剔出来，
大家可以考虑一下它有没有必要做成异步的，有这么些接口，第一个就是socket，第二个就是这个 close，这里有两个，其实不止两个，以这两个为例，那我们想一下这两个有没有必要把它做成异步？有还是没有没有？

大家看到socket他会不会引起阻塞，会不会引起不正常的关系，还有就是close的函数，它会不会引起？

**可以想一下就是这些我们在操作之前不需要介入去判断这个 fd是否就绪的时候，我们是不是就可以不用去封装？**

除了这一些之外，还有fcntl，还有包括我们的setsockopt,这些好这些请大家注意，如果单独是为了这个同步的操作变成异步的话，那这些接口它是没有必要的，

就是在这个同步的封装成的异步的这个角度上面，它是没有必要的，但是也有人为了一套我们还是封装好一点，请大家注意不是为了一套，我等一下还会跟大家讲就是关于hook，

需要封装的：

![在这里插入图片描述](https://img-blog.csdnimg.cn/771dc71de2bd41828e551bb8b9f4ca43.png)


不需要封装的：

![在这里插入图片描述](https://img-blog.csdnimg.cn/c0437aa4adb846d79e2b864c59dbf75a.png)



这些接口封装就可以统一按照这么一个方式，左边的那一排左边的一排在操作之前，是会引起这个阻塞的状态，就是在没有成功的时候它是不会返回的，它是会影响我们代码，也就是我们的accept没有获取到对应的连接的话，我们的程序就挂起，connect也是连接不成功，他就在等待，send，如果这个fd没有准备就绪的话，它也是等待；

左边的这一排我们在实现的时候，采用这么一个策略，从accept之前先加入epoll里面，然后让进去，然后交由调度器的那个大的epoll统一的去调度，
触发一次，看哪些等待就绪了，然后如果就绪了，然后在执行的下一步，然后再回到这么一个流程里面，请大家注意那所有的都是这么流程，左边的这一排所有的都是这个流程，

![在这里插入图片描述](https://img-blog.csdnimg.cn/bc3cd3bff0274064a70bc4fab55cbda9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_10,color_FFFFFF,t_70,g_se,x_16)



右边的这一盘其实我们也可以封装，我们也可以封装，比如说我们创建一个socket的时候，我们默认把这个 socket的封装变成一个非阻塞的封装，一个非阻塞，或者调用close的时候，我们自动的把这个协程关掉

怎么封装我们可以有这么两种策略，
第一种呢我们可以自己定一个，比如说我们可以这个read，就是凡是用我们这个框架都走掉这么一个read，就是自己独立成一套接口，
这是第一种方法，但这种方法呢你比如说你要去跟MySQL或者跟ridis，或者说不去修改redis源码的时候，这客户端提供的一些开发包的源码，会做不了；

**第二种策略呢就是我们直接做成跟系统一模一样的接口，**

第二个做成跟系统一模一样的，跟系统都有一样的结果，
那一样的接口我们在定义的时候这里就会引起一些冲突，请大家注意这个冲突，
介绍一个东西叫做hook，先实现一个案例，把这个钩子本身的原理给大家解释一下，有这么几个接口。

dlsym()是针对于系统的，我们提供了系统的一些调度，我们dlsym，如果说我们open就是用一些第三方的库,我们就用dllopen，我们操作mysql，用这个hook来做，大家可以看一下它什么一个效果。



![在这里插入图片描述](https://img-blog.csdnimg.cn/3cddd4b4fa564c71bc8a29abb4b41faa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_16,color_FFFFFF,t_70,g_se,x_16)



好，这里一起写了123456实现6个，当然还有2个就是resv_from和sendto这2个我们不写。
然后在初始化的时候，dlsym它是有了标准固定的写法的，



![在这里插入图片描述](https://img-blog.csdnimg.cn/2a5f6b8667664f3aafedbe4943be19dd.png)



但是在实现的时候，请大家注意这个函数是我们拿到系统调用的API这里面具体实现的把它赋值给他，然后这里面我们要加上一个，如果在检测的时候，当系统调用这个函数的时候，在我们应用程序中间的这个入口，他是哪个入口我们要重新定义一下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c0595271aa054918a00d2b13060a5e97.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_17,color_FFFFFF,t_70,g_se,x_16)



稍等一下给我们等一下看了之后，等一下我再给大家解释一下，包括内存泄露的一些方法我们怎么解的，我们怎么去判断对于线上的一些系统我们怎么去做，比如我们用到内存池的时候，我们怎么处理，也可以用这个hook把它解决。
好这里写了几个，然后初始化完了之后我们就这样，我们就实现我们对MySQL的操作，
然后就是MySQL初始化，然后再执行mysql_real_connect就可以了。



![在这里插入图片描述](https://img-blog.csdnimg.cn/02b8ffefcdd548d5a901773c231efe87.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_17,color_FFFFFF,t_70,g_se,x_16)



正常的理解，这里面我们要做的是
我们运行一下不加打印的话，我们感觉看不出什么差别。
这个打印就是区别我们现在调动的哪个函数，我们打印的函数调用哪个函数我们打印的函数。好，我们再编一下，就可以看到。

这原因大概有这个流程是这样的，现在大家记住了这一个入口，我们调用了hook之后，这里我们编译的时候，我们把动态库给链接进来了，请大家注意生成的这个点 o的文件，这个可执行程序是把这个库一起在运行的时候是把它拿来的，不管是动态库也好还是静态库也好，在我们调用connect的时候也就在这个库里面，如果有调用connect，请大家注意这个初始化这个接口这个hook就把这个contact调用的时候，就采用我们系统现在默认的调用的是这一部分，**就把它给截获了，也就现在出现了我们调用的库点，so或者动态库静态库也好，调用的是我们自己的实现接口**，这个hook的作用就是这样。

也就是说我们每调用这个，就把系统的connect,或者在我们库里面哪个调用的也好，把这个connect调用成我们自己实现这个connect,静态动态都是可以的，那大家可以看到如果有这个东西存在，如果有这个东西存在的话，各位朋友们你就想一想，我们所有的系统调用跟静态库，就是我们刚刚从一开始跟大家讲，**我们在做协程的时候，这些API我们采用方法，我们是不是就可以直接由hook来做？**
比如说我们的协程库要和我们的mysql放到一起，如果我们不去修改mysql-dev之内，一点都去改，我们只要加上hook我们就可以对应的来用。

其实这个原理啊很容易的，就是把原先的我在库里面调用的那个 connect的那个函数，名字换成了connect_f,就是我们提供的这些动态接口，我们换成了connect_f,而我们现在代码要调用的就变成我们现在的这个

比如现在我们有一个代码，就是我们现在的库已经实现的，不管是动态库也好，点a也好，还是点so也好，这里面比如说我们这个库里面我们调用到connect函数或者我们调用的read函数，我们现在呢通过dlsym()这一个我们把里面的找寻出来，找到这里面的connect我们找到他之后，然后用connect_f,也就是变成一个情况，我们系统调用的时候这样把它截获了，调用现在在我们代码里面所实现的这个connect。那就这样一个原理，就是把系统里面的connect重新截获了，那有了这个原理之后，各位朋友们那有很多东西我们可以做出很多不一样的东西。

我们把它改一下，connect_f改成connect_m，再编一下我们再跑一下，会出现什么情况？
上面这个这地方就是我们现在没有改 M之前,他是走了connect，如果我们改完m之后它是没有走connect的，对不对？也就是说这里面我就找去系统有没有定义都没有了，就不走，有的话就走好。

有这个存在我有很多东西做，所以这个刚才讲协程这个东西实现阶段，比如我们的malloc调用,free时候我们可以都是可以的。

当你发现一些内存泄露的时候，**大家知道一旦发生内存泄漏，肯定想的一个问题，你肯定能够知道就是有malloc没有free**

**我们把它用hook来做,用hook，我们在每一次malloc,我们加个打印，然后挂出来的东西哪些地方有malloc没有free，我们就能够清楚那地方，这也是一个做法。**

jemalloc还有tcmalloc也是这个原理，
这种内存池提供出来的这一部分，要去使用的时候，我们压根就不用改代码，就是我们写好代码之后可能就引入一个宏定义就ok了，什么都不用改，就直接会调用我们开发的库里面，我们只要在编译的时候把这个库给列进去，然后在我们调的时候那头文件里面定一个宏，就发现我们调的malloc不再是我们系统调的，而是jemalloc，没错，**就是把nginx运行在dpdk上面也是这样做的**。包括你后面做代码移植的时候或者是跨平台都是好的可以运用的。
再讲协程的这个API封装上面。
ntyco里面是支持MySQL的

```cpp
/*



 *  Author : WangBoJing , email : 1989wangbojing@gmail.com



 * 



 *  Copyright Statement:



 *  --------------------



 *  This software is protected by Copyright and the information contained



 *  herein is confidential. The software may not be copied and the information



 *  contained herein may not be used or disclosed except with the written



 *  permission of Author. (C) 2017



 * 



 *



 



****       *****                                      *****



  ***        *                                       **    ***



  ***        *         *                            *       **



  * **       *         *                           **        **



  * **       *         *                          **          *



  *  **      *        **                          **          *



  *  **      *       ***                          **



  *   **     *    ***********    *****    *****  **                   ****



  *   **     *        **           **      **    **                 **    **



  *    **    *        **           **      *     **                 *      **



  *    **    *        **            *      *     **                **      **



  *     **   *        **            **     *     **                *        **



  *     **   *        **             *    *      **               **        **



  *      **  *        **             **   *      **               **        **



  *      **  *        **             **   *      **               **        **



  *       ** *        **              *  *       **               **        **



  *       ** *        **              ** *        **          *   **        **



  *        ***        **               * *        **          *   **        **



  *        ***        **     *         **          *         *     **      **



  *         **        **     *         **          **       *      **      **



  *         **         **   *          *            **     *        **    **



*****        *          ****           *              *****           ****



                                       *



                                      *



                                  *****



                                  ****















 *



 */



 







#include "nty_coroutine.h"







#include <arpa/inet.h>







#define MAX_CLIENT_NUM            1000000



#define TIME_SUB_MS(tv1, tv2)  ((tv1.tv_sec - tv2.tv_sec) * 1000 + (tv1.tv_usec - tv2.tv_usec) / 1000)











void server_reader(void *arg) {



    int fd = *(int *)arg;



    int ret = 0;







 



    struct pollfd fds;



    fds.fd = fd;



    fds.events = POLLIN;







    while (1) {



        



        char buf[1024] = {0};



        ret = nty_recv(fd, buf, 1024, 0);



        if (ret > 0) {



            if(fd > MAX_CLIENT_NUM) 



            printf("read from server: %.*s\n", ret, buf);







            ret = nty_send(fd, buf, strlen(buf), 0);



            if (ret == -1) {



                nty_close(fd);



                break;



            }



        } else if (ret == 0) {    



            nty_close(fd);



            break;



        }







    }



}











void server(void *arg) {







    unsigned short port = *(unsigned short *)arg;



    free(arg);







    int fd = nty_socket(AF_INET, SOCK_STREAM, 0);



    if (fd < 0) return ;







    struct sockaddr_in local, remote;



    local.sin_family = AF_INET;



    local.sin_port = htons(port);



    local.sin_addr.s_addr = INADDR_ANY;



    bind(fd, (struct sockaddr*)&local, sizeof(struct sockaddr_in));







    listen(fd, 20);



    printf("listen port : %d\n", port);







    



    struct timeval tv_begin;



    gettimeofday(&tv_begin, NULL);







    while (1) {



        socklen_t len = sizeof(struct sockaddr_in);



        int cli_fd = nty_accept(fd, (struct sockaddr*)&remote, &len);



        if (cli_fd % 1000 == 999) {







            struct timeval tv_cur;



            memcpy(&tv_cur, &tv_begin, sizeof(struct timeval));



            



            gettimeofday(&tv_begin, NULL);



            int time_used = TIME_SUB_MS(tv_begin, tv_cur);



            



            printf("client fd : %d, time_used: %d\n", cli_fd, time_used);



        }



        printf("new client comming\n");







        nty_coroutine *read_co;



        nty_coroutine_create(&read_co, server_reader, &cli_fd);







    }



    



}















int main(int argc, char *argv[]) {



    nty_coroutine *co = NULL;







    int i = 0;



    unsigned short base_port = 9096;



    for (i = 0;i < 100;i ++) {



        unsigned short *port = calloc(1, sizeof(unsigned short));



        *port = base_port + i;



        nty_coroutine_create(&co, server, port); ////////no run



    }







    nty_schedule_run(); //run







    return 0;



}
```

**总结**

好，我们接下来再给大家捋一下，就是关于协程的过程中间，我们应该要了解到那些东西，首先来说请大家确定一点协程并不复杂，
那请大家注意第一个点协程核心的是为了去解决io读取的时候没有准备就绪的时候，这个阻塞的过程中间把这种同步的读改成异步的读，**核心用一句话用以前同步的读改成异步读。**

在这个核心的同步的读怎么改成异步的读？把对应的read，write 或者receive和send，这些把它改成里面由原来的对系统操作的read改成重定义的变成一个异步的，就是刚异步的就这么一个方式改的。
好，这是核心原本上阻塞改为非阻塞，其实阻塞和非阻塞改为非阻塞的读或者写部分，它本质上面你有没有发现它还会会出现我们在读写的时候逻辑不清晰，就是还是会发现它会出现那种逻逻没有这个同步这个效果，这个逻辑可能符合人的思考，就是你感觉非阻塞读它性能会跟协程一样，它的性能跟其实是一样的，只是说在代码读的方面，在代码的可读性上面没有形成这样符合人的逻辑

对于文件
这是服务器中间一个一个文件，对于磁盘操作这里有两个过程，这里有两个过程，第一个就是对于文件读写，
就是在协程里面我们去读写文件，还有一个就是我们读完之后能够把这个文件发送到客户端，
这个请求也就是说这里面有两个fd，一个是我们socketfd一个是我们操作文件的fd,这两个在读文件的时候这个也可以用协程，在发送完之后网络io这个地方也可以用协程来理解，读完之后从文件里面读取完之后再去把send出来。

怎么改成异步读，是这么一种方法，然后这个yeild他让到哪里去了，以及怎么回来。
好第三个就是说yeild让到哪？就让到协程的调度器，然后回来在哪？通过调度器，如果判断他IO准备就绪了，然后resume回来，
那关于这个调度器如何去判断？底层核心的，还是用的是一个epoll去管理所有的io,那这里面还有就是关于协调这个切换的事情，这里面就跟大家讲到了，比如说把寄存器替换掉这种方法

第一个解决协调到底解决什么问题，这是第一个问题。
再说同步改为异步这个怎么做的，
还有就是同步比异步的性能差异，
咱们就说这个问题，还有就是在这个过程yeild和resume关于调度这个过程中间切换怎么做的，
关于协程的API

其实
一定有一个差异的点，多线程也好，你看异步的时候异步的有的时候你自己去准备一下，你要跟你讲，比如说你这一次读出来的数据三者在哪，呢三者是在下一次才能算的数据，也就是说有时候有时候你准备好了被其他线程所关闭了，
比如说你还没遇到多个线程共有一个fd的情况，啊比如说线程 A线程a读取数据，吧然后把它关闭了，可能线程b还拿这个fd加send这种情况，这是多线程经常会出现一个,典型就是一个serve加上一个线程池，
这个很多没有写服务器就会这么写，可能也跟我最初讲服务器的时候那个版本有关系，啊前面一个一半一半管理完成把fd抛给现在只有现实进行明细，明白就是一个情况，这个fd很有可能会被多个线程同时共用，
同时共用的读那就会出现一个情况，那线程a对它比如说关闭了线程b所以它会出现很多的一些错误值
这种情况下面，所以多线程去操作fd为什么要加锁，而且等等不是很推荐这个方法去做，这还是有点考虑。

![在这里插入图片描述](https://img-blog.csdnimg.cn/87dd8ff462024dbe892fe40e04345676.png)



用它进来就是用它计量，现在相比较线程，进程有什么优势？
就是轻量级轻量级啊它的优势的话，就是
不像现在那么就是对io操作简单，如果是在线程里面一个io如果准备就绪了，如果准备就绪，那我们是线程的切换，线程的计划
就是由a线程切换到b线程，但是协程遇到io的话，它的切变清，就是它切换的不会像协程那么大，这就是协程相比的线程好处。

**切换的小在哪里？**线程切换你要想一下，包括线程的栈，包括线程的上下文，它会比协程都会要多很多的，

好，这里跟大家介绍协程的快，协程的快是针对于线程对比，针对于网络io而言,它跟reactor的差不多。
它是对比线程的切换会比快，但是对比io操作而言，其实差不多，这样两个对比跟线程对比切换快并且代码易读。

原文作者：[也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605065308