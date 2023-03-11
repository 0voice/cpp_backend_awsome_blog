# 【NO.199】DNS异步请求池原理与实现

## 1.同步与异步的区别

要设计异步请求池，首先要明白什么是异步、同步。

![img](https://pic3.zhimg.com/80/v2-915fc4ca05461891e8a2bbb0c16ccdea_720w.webp)

异步：就是发送完消息，不用等待结果的返回。发送消息的线程 和 处理消息的线程 是并行的。

同步：就是在发送消息后要等待返回结果，返回结果没有回来的时候这个线程是等待（阻塞）的状态，发送消息的线程 和 处理消息的线程 是串行的。

从上面的概念可以得知做成异步的好处是：

1、不需要等待（阻塞）返回结果，可以尽快的处理其他任务。

2、发送消息的线程 和 处理消息的线程 是并行的，这样可以减少一次发送到接收结果的程序运行时间。

## 2.为什么要做异步请求池

当业务服务器访问需要等待的服务时，业务访问线程需要等待挂起直到该服务给出反馈，例如对mysql,redis,http,dns等服务的请求，这些请求的返回需要等待一段时间，大大加深了业务服务器的承载压力，也会影响其总体性能，异步请求池就是为解决这个问题应运而生的。

## 3.如何做异步请求池

![img](https://pic2.zhimg.com/80/v2-bf49dabc0ab65c1c9dbb3a879dc9ec85_720w.webp)

如上图，**请求池是属于请求端（客户端）的**，请求端 发送连接 给 被请求端，被请求端处理完成后发送结果给请求端。异步请求的话，在发送完连接后，请求端的这个发送线程就可以处理其他任务，无需等待结果。

![img](https://pic2.zhimg.com/80/v2-b8fad2723bbe2bf4cda12f6c30181f41_720w.webp)

现在是异步请求池，说明是要建立多个连接。然后都**不需要等待结果的返回。**当结果有返回的时候，请求端在找到对应的发连接的信息，把结果转发给它。

也就是说，当建立多个连接，就会有多个fd，然后，把这些发送完消息的fd，存到epoll中进行管理。当被请求端发送结果过来的时候，epoll可以找到对应的fd，进行处理。 （ 需要注意的是调用 建立连接函数 的是一个线程，处理接收到结果消息的是一个线程。 ）

异步请求池的实现组成4元组：1.提交commit，2.线程回调函数，3.init 4.释放

工作流程：

**1、init**

epoll创建用来监听fd变化；

线程创建用来接受返回数据；

**2、提交commit**

创建socket，建立连接

封装协议

调用发送数据到对应的第三方服务器

发送完成后，将发送的fd设置为可读事件添加到epoll管理

**3、线程回调函数**

epoll_wait()检查哪些fd可读

遍历可读fd，recv接收数据

读出的数据按照对应的协议进行解析并进行操作

**4、释放销毁**

对应fd关闭close

线程退出释放

## 4.代码实现DNS异步请求池

**初始化init**

初始化函数dns_async_client_init，不需要接收参数。其中主要是**创建 epoll的fd，使用此fd来管理 commit (连接)的 fd，还有要创建处理结果的线程，线程函数是dns_async_client_proc**。

```text
struct async_context {
	int epfd;
};
//dns_async_client_init()
//epoll init
//thread init
/*struct async_context *ctx = dns_async_client_init();*/
struct async_context *dns_async_client_init(void) {

	int epfd = epoll_create(1); // 
	if (epfd < 0) return NULL;

	struct async_context *ctx = calloc(1, sizeof(struct async_context));
	if (ctx == NULL) {
		close(epfd);
		return NULL;
	}
	ctx->epfd = epfd;

	pthread_t thread_id;
	int ret = pthread_create(&thread_id, NULL, dns_async_client_proc, ctx);
	if (ret) {
		perror("pthread_create");
		return NULL;
	}
	usleep(1); //child go first

	return ctx;
}
```

**提交commit**

建立连接函数是dns_async_client_commit，接收参数：struct async_context* ctx、 const char* domain、async_result_cb cb。

首先说一下domain，这个是请求的url。cb是针对此函数中要创建的fd的接收到的结果处理的回调函数。ctx表示async_context结构体指针，此结构中存的是epoll的fd。在此函数还有很多dns的代码，这里就不进行介绍了，百度上应该有很多的。

```text
//dns_async_client_commit(ctx, domain)
//socket init
//dns_request
//sendto dns send
/*dns_async_client_commit(ctx, domain[i], dns_async_client_result_callback);*/
int dns_async_client_commit(struct async_context* ctx, const char *domain, async_result_cb cb) {

	int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
	if (sockfd < 0) {
		perror("create socket failed\n");
		exit(-1);
	}

	printf("url:%s\n", domain);

	set_block(sockfd, 0); //nonblock

	struct sockaddr_in dest;
	bzero(&dest, sizeof(dest));
	dest.sin_family = AF_INET;
	dest.sin_port = htons(53);
	dest.sin_addr.s_addr = inet_addr(DNS_SVR);
	
	int ret = connect(sockfd, (struct sockaddr*)&dest, sizeof(dest));
	//printf("connect :%d\n", ret);

	struct dns_header header = {0};
	dns_create_header(&header);

	struct dns_question question = {0};
	dns_create_question(&question, domain);


	char request[1024] = {0};
	int req_len = dns_build_request(&header, &question, request);
	int slen = sendto(sockfd, request, req_len, 0, (struct sockaddr*)&dest, sizeof(struct sockaddr));

	struct ep_arg *eparg = (struct ep_arg*)calloc(1, sizeof(struct ep_arg));
	if (eparg == NULL) return -1;
	eparg->sockfd = sockfd;
	eparg->cb = cb;

	struct epoll_event ev;
	ev.data.ptr = eparg;
	ev.events = EPOLLIN;

	ret = epoll_ctl(ctx->epfd, EPOLL_CTL_ADD, sockfd, &ev); 
	//printf(" epoll_ctl ADD: sockfd->%d, ret:%d\n", sockfd, ret);

	return ret;	
}
```

**线程回调函数**

dns_async_client_proc函数是处理被请求端返回结果的，是处理所有fd的是否接收到消息的函数，此函数中会一直循环处理epoll_wait，判断是不是有fd来消息了。

epoll_wait循环的判断epoll的fd对应的红黑树中是不是有client的fd来消息了，epoll_wait函数中最后的参数-1表示的是阻塞等待 如果有fd来消息了会返回有多少个fd来消息了，并且把这个些fd从红黑树中移动到一个链表中。nready表示有多少个fd来了消息。

```text
typedef void (*async_result_cb)(struct dns_item *list, int count);
/*typedef void (*async_result_cb)(struct dns_item *list, int count);*/
/*dns_async_client_commit(ctx, domain[i], dns_async_client_result_callback);*/
static void dns_async_client_result_callback(struct dns_item *list, int count) {
	int i = 0;

	for (i = 0;i < count;i ++) {
		printf("name:%s, ip:%s\n", list[i].domain, list[i].ip);
	}
}
//dns_async_client_proc()
//epoll_wait
//result callback
/*int ret = pthread_create(&thread_id, NULL, dns_async_client_proc, ctx);*/
/*接受请求的监听*/
static void* dns_async_client_proc(void *arg) {
	struct async_context *ctx = (struct async_context*)arg;

	int epfd = ctx->epfd;

	while (1) {

		struct epoll_event events[ASYNC_CLIENT_NUM] = {0};

		int nready = epoll_wait(epfd, events, ASYNC_CLIENT_NUM, -1);
		if (nready < 0) {
			if (errno == EINTR || errno == EAGAIN) {
				continue;
			} else {
				break;
			}
		} else if (nready == 0) {
			continue;
		}

		printf("nready:%d\n", nready);
		int i = 0;
		for (i = 0;i < nready;i ++) {

			struct ep_arg *data = (struct ep_arg*)events[i].data.ptr;
			int sockfd = data->sockfd;

			char buffer[1024] = {0};
			struct sockaddr_in addr;
			size_t addr_len = sizeof(struct sockaddr_in);
			int n = recvfrom(sockfd, buffer, sizeof(buffer), 0, (struct sockaddr*)&addr, (socklen_t*)&addr_len);

			struct dns_item *domain_list = NULL;
			int count = dns_parse_response(buffer, &domain_list);

			data->cb(domain_list, count); //call cb
			
			int ret = epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);
			//printf("epoll_ctl DEL --> sockfd:%d\n", sockfd);

			close(sockfd); /

			dns_async_client_free_domains(domain_list, count);
			free(data);

		}
		
	}
	
}
```

**销毁释放**

```text
int dns_async_clinet_destroy( struct async_context * ctx )
{
close(ctx->epfd);
pthread_cancel(ctx->threadId);
return 0;
}
```

原文地址：https://zhuanlan.zhihu.com/p/541548631