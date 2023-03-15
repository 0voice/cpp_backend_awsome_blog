# 【NO.322】手写线程池与性能分析

**手写线程池与性能分析**

作为一款服务器而言,很多服务器的源码，最底层最底层这些东西，是很通用的

比如我们之前讲过的网络,以及池式结构,最底层的这些组件是提供给我们应用层写业务逻辑的

在工作中接触的并不是很多，但是在面试的时候会问很多，工作中是以写业务为主，比如说工作中要你写个线程池这种可能性不大，比如说要你写一个连接池的可能性也不大，大量的时间可能是在写业务为主,那但是这个东西了，他在面试的时候,又是非常非常管用的东西。

简历里面肯定写一些通用的组件为主。

4个池式结构

![img](https://pic4.zhimg.com/80/v2-3031ddffcf5d4e7de5c50b28d0f10ab3_720w.webp)

先跟大家解释一下就是这些东西到底是用来做什么的?

对于一款服务器而言，然后n多的客户端连接上这个,然后再发每一个数据的时候，客户端给服务器发送的请求，发送的每一帧请求。

服务端这边接收完了之后进行处理，在recv()完了之后，那这里面我们正在进行解析,解析完数据之后，比如说我们写一些日志，那在这种同步的过程中间，写日志的时候

![img](https://pic4.zhimg.com/80/v2-862fdd12f75d9ddead56210fa047a7ef_720w.webp)

大概是这么一个流程，那对于parser这个过程只是把这个协议解析出来,解析出来之后，我们还有一系列的动作，比如说我们要去数据库查询，或者说我们要记录一些日志,或者说对数据库进行增删改查操作，

还有在这个过程中进行这些crud的操作的过程中，那会发现当这个日志记录的时候，它的时间周期会比较长，

就是在这一步落盘的时候，要写磁盘速度比较慢，对数据库操作的时候也会比较慢，所以很多的时候在这里大家就会引入一个东西，消息队列

抛到一个消息队列里面，然后再由消息队列进行去处理，或者说我通知给另外一个进程，由另外一个进程或者另外一个线程来对它进行操作，对这个日志进行罗盘来进行crud操作

那这个过程中间还有一种做法就是我们直接引入一个线程池，把这些日志或者这些crud当做一个一个的任务，把它抛给线程池，由线程池来处理这些任务,线程池就是解决这么一个问题，线程池的使用场景还是非常多的，

![img](https://pic2.zhimg.com/80/v2-4288cbb4bc7f5845c1d1bb4ec83d4c25_720w.webp)

比如说写日志

我们对服务器的一些业务需要计算，需要一些计算结果，计算结果计算的时间周期比较长的时候，我们也可以引入另外的线程，反正说我们在开线程的过程中间都可以想到线程池,第三个就是crud,不光是在服务器上面

**1.线程池到底是什么东西？**

我们在这里跟大家分析一下，它的使用场景先讲了，那我们再来分析一下它到底怎么做?

首先第一个它是由一个一个的任务所组成，

比如说我们写日志是一个任务，我们做计算也好，把一个计算任务把它抛进去，对数据库操作，我们也可以把它当做一次任务。

这里肯定就是有一个这个东西:任务队列

任务队列是提供给外界去用的时候，我们一个个的任务把它抛给任务队列

这时我们外面写的应用程序调用的接口,调用接口时候就直接可以写成一个push_tasks，把这任务给抛出去。

![img](https://pic3.zhimg.com/80/v2-0def69ce53fe4c39eb6de916ed173362_720w.webp)

**2.那任务在什么时候执行呢?**

这边当然还有一个执行队列,就是每1个节点，这里6个框，每1个框你就可以理解为是1个线程。

然后这一系列的线程就组成了一个集合，这个集合，就变成了一个执行队列，那这个执行队列执行什么东西呢,就从任务队列里面取任务，并且进行执行

第一个问题，

多个线程同时在一个任务队列里面，取任务,需要加锁，这是第一个问题。那这个任务队列就是这里几个线程的临界资源,取任务需要对这个任务队列进行加锁

第二个问题

举例的话就可能好理解，惊群那个案例一样，在中午的时候，你去一个食堂吃饭的时候，这个打饭的这个窗口上面，这6个阿姨相当于是去执行一个一个任务。

在这个过程中，每一次取任务的时候，他们肯定形成了某一种契约，就是每一次谁拿一个走，但是这里还有一个问题是什么？就是如果在下午三四点钟食堂没有人了，那这6个阿姨都在休息是吧？那这个状态在线程里面如何去表示这个休息的状态呢？

就是当任务不足的时候，如何去进行休息，线程如何操作?

条件变量 这是第二个。

在执行条件变量的时候是什么意思呢?这个状态进入挂起,进入一个条件等待,等待一个什么条件呢?等待这个任务队列不为空的一个条件满足，然后中间的线程取消休息，他这时候条件满足,条件等待等待任务队列不为空

这个线程池已经慢慢的整个原理性的问题，讲的差不多，可以看到线程池现在能理解透这两个问题。

那再来一个问题，线程池的工作原理?

把刚刚这个模式

一个任务队列加上一系列的线程，从这个任务队列取任务并且执行，

放在这里

就是我们每读出一个数据之后，然后把这个数据我们可以连同解析一起,把它当做一个任务,再由线程池来进行执行,就这么一个动作

那我们再来回过头来捋一下，回过头来再来思考一下。

网络这一层，整个一个服务器从他接收数据，网络数据的处理分为三个部分

就是从网卡里面接收数据到处理数据，这个过程总共有三个阶段。

![img](https://pic2.zhimg.com/80/v2-b7e1a6f923d56ef37d1a779629fcd311_720w.webp)

第一是检测io里面是否有数据，是否可读,是否可写，io事件是否就绪，这是第一个。

检测完了之后，

第二对它进行读写操作，

第三步读写完之后对数据进行解析与操作。

那这三个步骤里第一个步骤，网络数据最前面这层通过epoll

第二个是我们通过recv/send

第三步parser我们进行解析，那这个解析的流程就会比较多，

这三个过程，这是对每一帧网络数据都要经过这三个流程

那这个过程中间就引起了一个现象，写一个比如说多线程，然后多线程网络框架那这个东西怎么去理解，哪个地方用的多线程，

![img](https://pic1.zhimg.com/80/v2-4b701bbb11e0ac4c21884729c7de5c98_720w.webp)

这个流程可以通过这么一个数据的流向，

![img](https://pic1.zhimg.com/80/v2-914479c3fee678b107be5ea37c0eeca8_720w.webp)

这个线程池，在这个里面,我们怎么跟数据流程去结合?

举个例子，先假设我们已经有了线程池，现在不用去管线程池的实现，

![img](https://pic4.zhimg.com/80/v2-7088cd2cc9c7c1a0dfa4650973fe9313_720w.webp)

是一个线程，我们要引入线程池，引入线程池的话，各位那我们可以怎么做？有这么两种做法，两种做法都是可以的。

第一种我们直接把每一个任务把它抛到线程池里面，这是第一种做法。现在也就是说我们读、写、以及解析，都在这个线程

这是第一种做法。



第二种做法我们也可以是这样，

我们把recv放过来，我们再接着剩下的buffer，放线程池

![img](https://pic3.zhimg.com/80/v2-abfed434a3e91a35d37fd378365bedee_720w.webp)

讨论一下，

第一种第二种第三种都是可以实现的。

第一种做法就是一个单线程，第二种做法我们是把读写io以及解析数据这两个任务抛到线程池

第三种我们只把解析完的数据抛到线程池里面

但是每一种都有每一种优点和缺点，各位都有优点的缺点，

但是我们现在来分析一下这三种做法它的优点和缺点分别是什么？

第一种大家可以看到它是一个单线程，

如果对于任务比较轻的纯内存操作,这个是ok的，这里面不涉及到比如说写日志，它只是业务比较复杂而已，因为它在读写的时候不耗时的,可以考虑第三种

那第二种这种做法把fd抛给另外一个线程，他解决了什么问题?

如果这种情况是针对于哪一种？

针对于io的操作时间比较长，比如说这个fd是一个阻塞的，就可以采用第二种方式，

这三种做法里面速度最快的，不影响主循环第二种是最快的，但是

第二种有一个很大的问题，有一个大的问题，多个线程共用一个fd现象,

这个现象是怎么来的？

我不是一个任务，我把它抛给线程池吗?那怎么出现多个线程池共用一个fd的现象?

epoll检测完之后把这个 fd直接抛给了线程池，现在由线程池再进行处理，现在两个任务分别发出去，但是时间间隔不是很长，那也就是说这个 fd检测完之后，这个fd很有可能分给另外一线程

它的缺点就会出现一个现象，就是有可能因为这两个线程你都把它抛给了线程池来处理，所有的数据操作我们是不管的，那大家就会出现一个现象。

就是可能a线程对他进行读,对他进行写的时候，线程b很有可能把数据给closer关闭了。

就是线程a在准备数据的时候或者数据就绪的时候，

那结果线程b进行close掉了

因为这个fd大家可以看到我们这个代码上面我们是不进行任何的处理，而对应的只是把所有的fd都给有线程处理

这几个现象里面，既然这一个它的效率不影响主循环的情况下的效率是最高的，但是这个 fd共享就是比较难的。

我们该如何去解决这个问题，或者说有没有解的角度，fd不应该是在多线程共享

多线程网络框架，它的其实对于底层来说，就是把这三个组件，你哪些东西把它丢给线程池里面，

把检测io放到多个线的里面，ok的。

把读写lo放到多个线程里面也是ok的，

把协议解析以及操作放到多线程里面也是ok的

始终保持这三点

线程池的核心有俩部分组成一个任务队列,第二个是我们讲到的执行队列,第三个部分我们需要管理组件,有秩序的工作。

第一个，加锁,互斥锁

第二个,条件变量

就是通过这俩方面使它有秩序的管理

我们来分析这个任务队列由那些东西组成?

有个前提，任务队列它是任务的一个集合，首先我们先把任务描述清楚，然后再来描述这个集合,两层关系

那任务里面包含那些东西呢?

我们怎么去把它封装成一个一个任务？

第一个首先每一个人任务都不一样,存款或者有些取款，或者有些可能去打印流水等等

这里需要一个回调函数callback();这个回调函数是由任务自己本身去实现的,把它抽象出来,

第二个还有一个参数，如果你想要去办存款，你得带上你的现金，如果你去办取款，你得带上你的银行卡，所以每一个任务执行，每一个回调函数都需要带不同的参数

![img](https://pic1.zhimg.com/80/v2-36c3e344320546b5a84fe2a7e6bcfbf0_720w.webp)

这个任务我们封装完之后，我们怎么把它变成一个队列，为它加上前后指针,我们就能够把它串起来。

![img](https://pic2.zhimg.com/80/v2-7107094c350f303e70a5514ab1800ecd_720w.webp)

一个回调函数,以及一个回调函数对应的参数在加上前驱后继,我们只要记录一个头指针?

如果当我们任务很少时,我们一次性开了很多线程我们怎么去退出?

中间就用一些进行休息，这里有一个状态terminate,

关于线程的退出,pthread_exit(thid);退出线程id这个是OK的,

返回的时候，如果我们有线程加入pthread_join(thid)等待它返回的话，父线程针对子线程返回的时候它会等待这个结果,pthread_detach()分离后俩个就失去了亲缘关系,pthread_join(thid)就等待不出结果,

terminate这里线程退出我们是比较友好一点,pthread_cancel()退出比较粗暴,exit()函数我们压根就不知道这是什么状态,调用就直接关闭了,我们在执行阶段就能看到他的委婉，terminate可以走完整个流程完整的退出,会更加的优雅

![img](https://pic3.zhimg.com/80/v2-6d99e5138c4925204b838de36ffe046a_720w.webp)

这个管理组件里面大概包含这4个项目

![img](https://pic1.zhimg.com/80/v2-c57770d1bfc1dd89b80856d98d702684_720w.webp)

![img](https://pic1.zhimg.com/80/v2-98781ba2cc745c9c3cf23aaff1b1a000_720w.webp)

关于队列我们添加和删除一个节点需要使用宏定义,为什么选用宏定义去添加删除节点呢?

可以注意到NJOB和NWORKER是俩个不同的类型,不同的类型我们对她进行同样的操作,在C++里面使用模板来做是OK的但是在C代码里面没有模板这个概念,也就是模板的底层应该是宏定义,

![img](https://pic4.zhimg.com/80/v2-e6c9d9e0c45664715300a34f93594baf_720w.webp)

**这个do /while(0)是什么意思?**

宏定义的这个过程它跟内联是一样的,就是把代码copy过去,但是一个是在预编译的时候,一个是在编译的时候

由于do/while(0)这个语法就决定了它需要携带一个分号,这个地方采用头插法

![img](https://pic3.zhimg.com/80/v2-1c74ffbc8444eb179f89ec271afdeffa_720w.webp)

删除也是类似的

![img](https://pic2.zhimg.com/80/v2-4bf43ab3c4d55e7915a1e039481dab39_720w.webp)

所以后面不管是对NJOB的操作或者是NWORKER的操作都可以借助这俩个宏,

那线程池它需要具备用哪些接口？需要哪些API线程池才能够去使用?

首先第一个肯定有的一个是初始化，给出两个参数，第一个参数是线程池的对象,第二个是说我们创建多少个线程池

![img](https://pic4.zhimg.com/80/v2-cd3f5bfd2d3c5cb78a303f22a5e4f733_720w.webp)

初始化的时候我们只需要创建一个东西，就是这个线程池的数量有多少？

![img](https://pic4.zhimg.com/80/v2-2f4385e0de18b7f761052922a9cb992f_720w.webp)

线程创建我们只依赖于线程数量。

第二个函数,我们要做的就是往线程池里面抛一个任务,就是把一个task抛到对应的任务队列里面

![img](https://pic2.zhimg.com/80/v2-23bf97589d315da2d95fd611c07445c5_720w.webp)

第三个接口，既然有创建当然就有销毁，

![img](https://pic2.zhimg.com/80/v2-e5ffcfd0dd64535b8c41f4a48ef7a1ad_720w.webp)

三个接口是对外的使用，理解这三个接口就可以了。

一个线程池它由那些东西组成?

第一个一个任务队列,第二个一个执行队列,第三个一个管理这俩个队列正常运行的一个组件

那这个在创建的时候呢,我们需要把执行队列初始化完,与之对应的互斥量条件变量创建完,



那在创建这个任务队列的时候，创建这个执行队列的时候，每一个任务每一个执行每一个worker是需要跟这个线程一一对应上的，然后把这个work直接把它抛到线程里面，每一个线程拿这个worker去工作,初始化可以说是4个部分，

1.参数是否合法

2.mutex

3.cond

4.worker

创建完之后线程池就可以开始工作了。

线程的入口函数和任务的回调函数之间是什么关系?

线程回调函数是不是就等同于任务回调函数?

![img](https://pic1.zhimg.com/80/v2-8eecc7bbeeb15427f09fcb59924fc414_720w.webp)

这里面这个绿色的就相当于线程的入口函数，它是不断的去任务队列里面取任务执行,不断就是循环,取完任务之后,执行的时候就是执行回调函数

![img](https://pic4.zhimg.com/80/v2-4a90db55d53be1711c2ded13b97bdcd3_720w.webp)

这个移除就相当于是把这个头节点给移除掉。

我们开的线程比较多的时候在那个地方进行放缩?怎么释放一些线程

接下来丢一个任务进去他能做那些事情?

![img](https://pic1.zhimg.com/80/v2-0911ccd332d981b2b842a06681034b5c_720w.webp)

现在多个线程等待这个条件,通知的时候会有惊群现象吗?

![img](https://pic2.zhimg.com/80/v2-74a7f9348c63f87fa4415936796ec825_720w.webp)

**nginx线程池源码**

ngx_thread_pool_cycle

防止线程被其它的东西屏蔽这些终止,就是防止线程pthread_sigmask被其它东西影响做了一个屏蔽

![img](https://pic2.zhimg.com/80/v2-a30c6c766287bf7235f4a103d0d0104d_720w.webp)

这里的cycle就是线程的入口函数,然后对应这里面的任务就是一个永真的循环,然后判断任务队列是不是为空,

![img](https://pic4.zhimg.com/80/v2-3179692d09dcaf7e0c80eb9fd0ba9b37_720w.webp)

条件等待往下面走是封装的接口,如果不为空的话从任务队列里面取出一个任务出来,出来之后解锁,之后就是handler,就是每一个任务的回调函数,逻辑都是一样的

![img](https://pic2.zhimg.com/80/v2-a7ac94e708d542588b4d4f678fe9239d_720w.webp)

看源码就可以这样对比,先把基础主体把握住,

第二个就是往线程池里面post抛一个任务进去

![img](https://pic3.zhimg.com/80/v2-df771566dd45fdf2508d0d1a95d8e746_720w.webp)

方法逻辑都是把任务加进去然后再去通知,

这个东西是什么意思?**last它有俩个指针

![img](https://pic4.zhimg.com/80/v2-af4a2e95d89143822ca746d7ded4fb63_720w.webp)

这是往队列里面添加一个节点然后waiting++

![img](https://pic2.zhimg.com/80/v2-646bcca82aa5b525ea13de576b997df1_720w.webp)

它主要是统计任务的数量方便我们去放缩任务,但任务量很多我们可以增加线程,很少我们可以减少线程,

那这个二级指针到底做了什么?

![img](https://pic2.zhimg.com/80/v2-666814fd097273f7fa6b11454f3c1f55_720w.webp)

这个二级指针就是这个

**就是拿着它的第一个字段指向的下一个内存,**为什么这么做好处在哪?

线程池本身没有性能优化,要前后对比,不加入线程池与加入线程池的区别,大概有6倍

多个线程共用一个fd怎么解,参考协程

做线程放缩怎么做?添加一个变量waiting,每添加任务++,判断任务数量和线程数量比例,线程数量退出,任务数量和线程数量统计,比如线程放缩策略4:6

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <pthread.h>



#define LL_ADD(item, list) do { 	\
	item->prev = NULL;				\
	item->next = list;				\
	list = item;					\
} while(0)

#define LL_REMOVE(item, list) do {						\
	if (item->prev != NULL) item->prev->next = item->next;	\
	if (item->next != NULL) item->next->prev = item->prev;	\
	if (list == item) list = item->next;					\
	item->prev = item->next = NULL;							\
} while(0)


typedef struct NWORKER {
	pthread_t thread;
	int terminate;
	struct NWORKQUEUE *workqueue;
	struct NWORKER *prev;
	struct NWORKER *next;
} nWorker;

typedef struct NJOB {
	void (*job_function)(struct NJOB *job);
	void *user_data;
	struct NJOB *prev;
	struct NJOB *next;
} nJob;

typedef struct NWORKQUEUE {
	struct NWORKER *workers;
	struct NJOB *waiting_jobs;
	pthread_mutex_t jobs_mtx;
	pthread_cond_t jobs_cond;
} nWorkQueue;

typedef nWorkQueue nThreadPool;

static void *ntyWorkerThread(void *ptr) {
	nWorker *worker = (nWorker*)ptr;

	while (1) {
		pthread_mutex_lock(&worker->workqueue->jobs_mtx);

		while (worker->workqueue->waiting_jobs == NULL) {
			if (worker->terminate) break;
			pthread_cond_wait(&worker->workqueue->jobs_cond, &worker->workqueue->jobs_mtx);
		}

		if (worker->terminate) {
			pthread_mutex_unlock(&worker->workqueue->jobs_mtx);
			break;
		}
		
		nJob *job = worker->workqueue->waiting_jobs;
		if (job != NULL) {
			LL_REMOVE(job, worker->workqueue->waiting_jobs);
		}
		
		pthread_mutex_unlock(&worker->workqueue->jobs_mtx);

		if (job == NULL) continue;

		job->job_function(job);
	}

	free(worker);
	pthread_exit(NULL);
}



int ntyThreadPoolCreate(nThreadPool *workqueue, int numWorkers) {

	if (numWorkers < 1) numWorkers = 1;
	memset(workqueue, 0, sizeof(nThreadPool));
	
	pthread_cond_t blank_cond = PTHREAD_COND_INITIALIZER;
	memcpy(&workqueue->jobs_cond, &blank_cond, sizeof(workqueue->jobs_cond));
	
	pthread_mutex_t blank_mutex = PTHREAD_MUTEX_INITIALIZER;
	memcpy(&workqueue->jobs_mtx, &blank_mutex, sizeof(workqueue->jobs_mtx));

	int i = 0;
	for (i = 0;i < numWorkers;i ++) {
		nWorker *worker = (nWorker*)malloc(sizeof(nWorker));
		if (worker == NULL) {
			perror("malloc");
			return 1;
		}

		memset(worker, 0, sizeof(nWorker));
		worker->workqueue = workqueue;

		int ret = pthread_create(&worker->thread, NULL, ntyWorkerThread, (void *)worker);
		if (ret) {
			
			perror("pthread_create");
			free(worker);

			return 1;
		}

		LL_ADD(worker, worker->workqueue->workers);
	}

	return 0;
}


void ntyThreadPoolShutdown(nThreadPool *workqueue) {
	nWorker *worker = NULL;

	for (worker = workqueue->workers;worker != NULL;worker = worker->next) {
		worker->terminate = 1;
	}

	pthread_mutex_lock(&workqueue->jobs_mtx);

	workqueue->workers = NULL;
	workqueue->waiting_jobs = NULL;

	pthread_cond_broadcast(&workqueue->jobs_cond);

	pthread_mutex_unlock(&workqueue->jobs_mtx);
	
}

void ntyThreadPoolQueue(nThreadPool *workqueue, nJob *job) {

	pthread_mutex_lock(&workqueue->jobs_mtx);

	LL_ADD(job, workqueue->waiting_jobs);
	
	pthread_cond_signal(&workqueue->jobs_cond);
	pthread_mutex_unlock(&workqueue->jobs_mtx);
	
}




/************************** debug thread pool **************************/
//sdk  --> software develop kit
// 提供SDK给其他开发者使用

#if 1

#define KING_MAX_THREAD			80
#define KING_COUNTER_SIZE		1000

void king_counter(nJob *job) {

	int index = *(int*)job->user_data;

	printf("index : %d, selfid : %lu\n", index, pthread_self());
	
	free(job->user_data);
	free(job);
}



int main(int argc, char *argv[]) {

	nThreadPool pool;

	ntyThreadPoolCreate(&pool, KING_MAX_THREAD);
	
	int i = 0;
	for (i = 0;i < KING_COUNTER_SIZE;i ++) {
		nJob *job = (nJob*)malloc(sizeof(nJob));
		if (job == NULL) {
			perror("malloc");
			exit(1);
		}
		
		job->job_function = king_counter;
		job->user_data = malloc(sizeof(int));
		*(int*)job->user_data = i;

		ntyThreadPoolQueue(&pool, job);
		
	}

	getchar();
	printf("\n");

	
}

#endif
```



```text
/*
 * Copyright (C) Nginx, Inc.
 * Copyright (C) Valentin V. Bartenev
 */


#ifndef _NGX_THREAD_POOL_H_INCLUDED_
#define _NGX_THREAD_POOL_H_INCLUDED_


#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_event.h>


struct ngx_thread_task_s {
    ngx_thread_task_t   *next;
    ngx_uint_t           id;
    void                *ctx;
    void               (*handler)(void *data, ngx_log_t *log);
    ngx_event_t          event;
};


typedef struct ngx_thread_pool_s  ngx_thread_pool_t;


ngx_thread_pool_t *ngx_thread_pool_add(ngx_conf_t *cf, ngx_str_t *name);
ngx_thread_pool_t *ngx_thread_pool_get(ngx_cycle_t *cycle, ngx_str_t *name);

ngx_thread_task_t *ngx_thread_task_alloc(ngx_pool_t *pool, size_t size);
ngx_int_t ngx_thread_task_post(ngx_thread_pool_t *tp, ngx_thread_task_t *task);


#endif /* _NGX_THREAD_POOL_H_INCLUDED_ */
```



```text
/*
 * Copyright (C) Nginx, Inc.
 * Copyright (C) Valentin V. Bartenev
 * Copyright (C) Ruslan Ermilov
 */


#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_thread_pool.h>


typedef struct {
    ngx_array_t               pools;
} ngx_thread_pool_conf_t;


typedef struct {
    ngx_thread_task_t        *first;
    ngx_thread_task_t       **last;
} ngx_thread_pool_queue_t;

#define ngx_thread_pool_queue_init(q)                                         \
    (q)->first = NULL;                                                        \
    (q)->last = &(q)->first


struct ngx_thread_pool_s {
    ngx_thread_mutex_t        mtx;
    ngx_thread_pool_queue_t   queue;
    ngx_int_t                 waiting;
    ngx_thread_cond_t         cond;

    ngx_log_t                *log;

    ngx_str_t                 name;
    ngx_uint_t                threads;
    ngx_int_t                 max_queue;

    u_char                   *file;
    ngx_uint_t                line;
};


static ngx_int_t ngx_thread_pool_init(ngx_thread_pool_t *tp, ngx_log_t *log,
    ngx_pool_t *pool);
static void ngx_thread_pool_destroy(ngx_thread_pool_t *tp);
static void ngx_thread_pool_exit_handler(void *data, ngx_log_t *log);

static void *ngx_thread_pool_cycle(void *data);
static void ngx_thread_pool_handler(ngx_event_t *ev);

static char *ngx_thread_pool(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);

static void *ngx_thread_pool_create_conf(ngx_cycle_t *cycle);
static char *ngx_thread_pool_init_conf(ngx_cycle_t *cycle, void *conf);

static ngx_int_t ngx_thread_pool_init_worker(ngx_cycle_t *cycle);
static void ngx_thread_pool_exit_worker(ngx_cycle_t *cycle);


static ngx_command_t  ngx_thread_pool_commands[] = {

    { ngx_string("thread_pool"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE23,
      ngx_thread_pool,
      0,
      0,
      NULL },

      ngx_null_command
};


static ngx_core_module_t  ngx_thread_pool_module_ctx = {
    ngx_string("thread_pool"),
    ngx_thread_pool_create_conf,
    ngx_thread_pool_init_conf
};


ngx_module_t  ngx_thread_pool_module = {
    NGX_MODULE_V1,
    &ngx_thread_pool_module_ctx,           /* module context */
    ngx_thread_pool_commands,              /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    ngx_thread_pool_init_worker,           /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    ngx_thread_pool_exit_worker,           /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};


static ngx_str_t  ngx_thread_pool_default = ngx_string("default");

static ngx_uint_t               ngx_thread_pool_task_id;
static ngx_atomic_t             ngx_thread_pool_done_lock;
static ngx_thread_pool_queue_t  ngx_thread_pool_done;


static ngx_int_t
ngx_thread_pool_init(ngx_thread_pool_t *tp, ngx_log_t *log, ngx_pool_t *pool)
{
    int             err;
    pthread_t       tid;
    ngx_uint_t      n;
    pthread_attr_t  attr;

    if (ngx_notify == NULL) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
               "the configured event method cannot be used with thread pools");
        return NGX_ERROR;
    }

    ngx_thread_pool_queue_init(&tp->queue);

    if (ngx_thread_mutex_create(&tp->mtx, log) != NGX_OK) {
        return NGX_ERROR;
    }

    if (ngx_thread_cond_create(&tp->cond, log) != NGX_OK) {
        (void) ngx_thread_mutex_destroy(&tp->mtx, log);
        return NGX_ERROR;
    }

    tp->log = log;

    err = pthread_attr_init(&attr);
    if (err) {
        ngx_log_error(NGX_LOG_ALERT, log, err,
                      "pthread_attr_init() failed");
        return NGX_ERROR;
    }

    err = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    if (err) {
        ngx_log_error(NGX_LOG_ALERT, log, err,
                      "pthread_attr_setdetachstate() failed");
        return NGX_ERROR;
    }

#if 0
    err = pthread_attr_setstacksize(&attr, PTHREAD_STACK_MIN);
    if (err) {
        ngx_log_error(NGX_LOG_ALERT, log, err,
                      "pthread_attr_setstacksize() failed");
        return NGX_ERROR;
    }
#endif

    for (n = 0; n < tp->threads; n++) {
        err = pthread_create(&tid, &attr, ngx_thread_pool_cycle, tp);
        if (err) {
            ngx_log_error(NGX_LOG_ALERT, log, err,
                          "pthread_create() failed");
            return NGX_ERROR;
        }
    }

    (void) pthread_attr_destroy(&attr);

    return NGX_OK;
}


static void
ngx_thread_pool_destroy(ngx_thread_pool_t *tp)
{
    ngx_uint_t           n;
    ngx_thread_task_t    task;
    volatile ngx_uint_t  lock;

    ngx_memzero(&task, sizeof(ngx_thread_task_t));

    task.handler = ngx_thread_pool_exit_handler;
    task.ctx = (void *) &lock;

    for (n = 0; n < tp->threads; n++) {
        lock = 1;

        if (ngx_thread_task_post(tp, &task) != NGX_OK) {
            return;
        }

        while (lock) {
            ngx_sched_yield();
        }

        task.event.active = 0;
    }

    (void) ngx_thread_cond_destroy(&tp->cond, tp->log);

    (void) ngx_thread_mutex_destroy(&tp->mtx, tp->log);
}


static void
ngx_thread_pool_exit_handler(void *data, ngx_log_t *log)
{
    ngx_uint_t *lock = data;

    *lock = 0;

    pthread_exit(0);
}


ngx_thread_task_t *
ngx_thread_task_alloc(ngx_pool_t *pool, size_t size)
{
    ngx_thread_task_t  *task;

    task = ngx_pcalloc(pool, sizeof(ngx_thread_task_t) + size);
    if (task == NULL) {
        return NULL;
    }

    task->ctx = task + 1;

    return task;
}


ngx_int_t
ngx_thread_task_post(ngx_thread_pool_t *tp, ngx_thread_task_t *task)
{
    if (task->event.active) {
        ngx_log_error(NGX_LOG_ALERT, tp->log, 0,
                      "task #%ui already active", task->id);
        return NGX_ERROR;
    }

    if (ngx_thread_mutex_lock(&tp->mtx, tp->log) != NGX_OK) {
        return NGX_ERROR;
    }

    if (tp->waiting >= tp->max_queue) {
        (void) ngx_thread_mutex_unlock(&tp->mtx, tp->log);

        ngx_log_error(NGX_LOG_ERR, tp->log, 0,
                      "thread pool \"%V\" queue overflow: %i tasks waiting",
                      &tp->name, tp->waiting);
        return NGX_ERROR;
    }

    task->event.active = 1;

    task->id = ngx_thread_pool_task_id++;
    task->next = NULL;

    if (ngx_thread_cond_signal(&tp->cond, tp->log) != NGX_OK) {
        (void) ngx_thread_mutex_unlock(&tp->mtx, tp->log);
        return NGX_ERROR;
    }

    *tp->queue.last = task;
    tp->queue.last = &task->next;

    tp->waiting++;

    (void) ngx_thread_mutex_unlock(&tp->mtx, tp->log);

    ngx_log_debug2(NGX_LOG_DEBUG_CORE, tp->log, 0,
                   "task #%ui added to thread pool \"%V\"",
                   task->id, &tp->name);

    return NGX_OK;
}


static void *
ngx_thread_pool_cycle(void *data)
{
    ngx_thread_pool_t *tp = data;

    int                 err;
    sigset_t            set;
    ngx_thread_task_t  *task;

#if 0
    ngx_time_update();
#endif

    ngx_log_debug1(NGX_LOG_DEBUG_CORE, tp->log, 0,
                   "thread in pool \"%V\" started", &tp->name);

    sigfillset(&set);

    sigdelset(&set, SIGILL);
    sigdelset(&set, SIGFPE);
    sigdelset(&set, SIGSEGV);
    sigdelset(&set, SIGBUS);

    err = pthread_sigmask(SIG_BLOCK, &set, NULL);
    if (err) {
        ngx_log_error(NGX_LOG_ALERT, tp->log, err, "pthread_sigmask() failed");
        return NULL;
    }

    for ( ;; ) {
        if (ngx_thread_mutex_lock(&tp->mtx, tp->log) != NGX_OK) {
            return NULL;
        }

        /* the number may become negative */
        tp->waiting--;

        while (tp->queue.first == NULL) {
            if (ngx_thread_cond_wait(&tp->cond, &tp->mtx, tp->log)
                != NGX_OK)
            {
                (void) ngx_thread_mutex_unlock(&tp->mtx, tp->log);
                return NULL;
            }
        }

        task = tp->queue.first;
        tp->queue.first = task->next;

        if (tp->queue.first == NULL) {
            tp->queue.last = &tp->queue.first;
        }

        if (ngx_thread_mutex_unlock(&tp->mtx, tp->log) != NGX_OK) {
            return NULL;
        }

#if 0
        ngx_time_update();
#endif

        ngx_log_debug2(NGX_LOG_DEBUG_CORE, tp->log, 0,
                       "run task #%ui in thread pool \"%V\"",
                       task->id, &tp->name);

        task->handler(task->ctx, tp->log);

        ngx_log_debug2(NGX_LOG_DEBUG_CORE, tp->log, 0,
                       "complete task #%ui in thread pool \"%V\"",
                       task->id, &tp->name);

        task->next = NULL;

        ngx_spinlock(&ngx_thread_pool_done_lock, 1, 2048);

        *ngx_thread_pool_done.last = task;
        ngx_thread_pool_done.last = &task->next;

        ngx_memory_barrier();

        ngx_unlock(&ngx_thread_pool_done_lock);

        (void) ngx_notify(ngx_thread_pool_handler);
    }
}


static void
ngx_thread_pool_handler(ngx_event_t *ev)
{
    ngx_event_t        *event;
    ngx_thread_task_t  *task;

    ngx_log_debug0(NGX_LOG_DEBUG_CORE, ev->log, 0, "thread pool handler");

    ngx_spinlock(&ngx_thread_pool_done_lock, 1, 2048);

    task = ngx_thread_pool_done.first;
    ngx_thread_pool_done.first = NULL;
    ngx_thread_pool_done.last = &ngx_thread_pool_done.first;

    ngx_memory_barrier();

    ngx_unlock(&ngx_thread_pool_done_lock);

    while (task) {
        ngx_log_debug1(NGX_LOG_DEBUG_CORE, ev->log, 0,
                       "run completion handler for task #%ui", task->id);

        event = &task->event;
        task = task->next;

        event->complete = 1;
        event->active = 0;

        event->handler(event);
    }
}


static void *
ngx_thread_pool_create_conf(ngx_cycle_t *cycle)
{
    ngx_thread_pool_conf_t  *tcf;

    tcf = ngx_pcalloc(cycle->pool, sizeof(ngx_thread_pool_conf_t));
    if (tcf == NULL) {
        return NULL;
    }

    if (ngx_array_init(&tcf->pools, cycle->pool, 4,
                       sizeof(ngx_thread_pool_t *))
        != NGX_OK)
    {
        return NULL;
    }

    return tcf;
}


static char *
ngx_thread_pool_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_thread_pool_conf_t *tcf = conf;

    ngx_uint_t           i;
    ngx_thread_pool_t  **tpp;

    tpp = tcf->pools.elts;

    for (i = 0; i < tcf->pools.nelts; i++) {

        if (tpp[i]->threads) {
            continue;
        }

        if (tpp[i]->name.len == ngx_thread_pool_default.len
            && ngx_strncmp(tpp[i]->name.data, ngx_thread_pool_default.data,
                           ngx_thread_pool_default.len)
               == 0)
        {
            tpp[i]->threads = 32;
            tpp[i]->max_queue = 65536;
            continue;
        }

        ngx_log_error(NGX_LOG_EMERG, cycle->log, 0,
                      "unknown thread pool \"%V\" in %s:%ui",
                      &tpp[i]->name, tpp[i]->file, tpp[i]->line);

        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}


static char *
ngx_thread_pool(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_str_t          *value;
    ngx_uint_t          i;
    ngx_thread_pool_t  *tp;

    value = cf->args->elts;

    tp = ngx_thread_pool_add(cf, &value[1]);

    if (tp == NULL) {
        return NGX_CONF_ERROR;
    }

    if (tp->threads) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "duplicate thread pool \"%V\"", &tp->name);
        return NGX_CONF_ERROR;
    }

    tp->max_queue = 65536;

    for (i = 2; i < cf->args->nelts; i++) {

        if (ngx_strncmp(value[i].data, "threads=", 8) == 0) {

            tp->threads = ngx_atoi(value[i].data + 8, value[i].len - 8);

            if (tp->threads == (ngx_uint_t) NGX_ERROR || tp->threads == 0) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid threads value \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }

        if (ngx_strncmp(value[i].data, "max_queue=", 10) == 0) {

            tp->max_queue = ngx_atoi(value[i].data + 10, value[i].len - 10);

            if (tp->max_queue == NGX_ERROR) {
                ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                   "invalid max_queue value \"%V\"", &value[i]);
                return NGX_CONF_ERROR;
            }

            continue;
        }
    }

    if (tp->threads == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "\"%V\" must have \"threads\" parameter",
                           &cmd->name);
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}


ngx_thread_pool_t *
ngx_thread_pool_add(ngx_conf_t *cf, ngx_str_t *name)
{
    ngx_thread_pool_t       *tp, **tpp;
    ngx_thread_pool_conf_t  *tcf;

    if (name == NULL) {
        name = &ngx_thread_pool_default;
    }

    tp = ngx_thread_pool_get(cf->cycle, name);

    if (tp) {
        return tp;
    }

    tp = ngx_pcalloc(cf->pool, sizeof(ngx_thread_pool_t));
    if (tp == NULL) {
        return NULL;
    }

    tp->name = *name;
    tp->file = cf->conf_file->file.name.data;
    tp->line = cf->conf_file->line;

    tcf = (ngx_thread_pool_conf_t *) ngx_get_conf(cf->cycle->conf_ctx,
                                                  ngx_thread_pool_module);

    tpp = ngx_array_push(&tcf->pools);
    if (tpp == NULL) {
        return NULL;
    }

    *tpp = tp;

    return tp;
}


ngx_thread_pool_t *
ngx_thread_pool_get(ngx_cycle_t *cycle, ngx_str_t *name)
{
    ngx_uint_t                i;
    ngx_thread_pool_t       **tpp;
    ngx_thread_pool_conf_t   *tcf;

    tcf = (ngx_thread_pool_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                                  ngx_thread_pool_module);

    tpp = tcf->pools.elts;

    for (i = 0; i < tcf->pools.nelts; i++) {

        if (tpp[i]->name.len == name->len
            && ngx_strncmp(tpp[i]->name.data, name->data, name->len) == 0)
        {
            return tpp[i];
        }
    }

    return NULL;
}


static ngx_int_t
ngx_thread_pool_init_worker(ngx_cycle_t *cycle)
{
    ngx_uint_t                i;
    ngx_thread_pool_t       **tpp;
    ngx_thread_pool_conf_t   *tcf;

    if (ngx_process != NGX_PROCESS_WORKER
        && ngx_process != NGX_PROCESS_SINGLE)
    {
        return NGX_OK;
    }

    tcf = (ngx_thread_pool_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                                  ngx_thread_pool_module);

    if (tcf == NULL) {
        return NGX_OK;
    }

    ngx_thread_pool_queue_init(&ngx_thread_pool_done);

    tpp = tcf->pools.elts;

    for (i = 0; i < tcf->pools.nelts; i++) {
        if (ngx_thread_pool_init(tpp[i], cycle->log, cycle->pool) != NGX_OK) {
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}


static void
ngx_thread_pool_exit_worker(ngx_cycle_t *cycle)
{
    ngx_uint_t                i;
    ngx_thread_pool_t       **tpp;
    ngx_thread_pool_conf_t   *tcf;

    if (ngx_process != NGX_PROCESS_WORKER
        && ngx_process != NGX_PROCESS_SINGLE)
    {
        return;
    }

    tcf = (ngx_thread_pool_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                                  ngx_thread_pool_module);

    if (tcf == NULL) {
        return;
    }

    tpp = tcf->pools.elts;

    for (i = 0; i < tcf->pools.nelts; i++) {
        ngx_thread_pool_destroy(tpp[i]);
    }
}
```

原文地址：https://zhuanlan.zhihu.com/p/519748706

作者：linux