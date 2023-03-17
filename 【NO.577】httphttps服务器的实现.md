# 【NO.577】http/https服务器的实现

## 1.HTTP简介

HTTP 协议（HyperText Transfer Protocol，超文本传输协议）是因特网上应用最为广泛的一种网络传输协议，所有的 WWW 文件都必须遵守这个标准。HTTP 是一个基于 TCP/IP 通信协议来传递数据（HTML 文件, 图片文件, 查询结果等）。

## 2.HTTP工作原理

HTTP 协议工作于客户端-服务端架构上。浏览器作为 HTTP 客户端通过 URL 向 HTTP 服务端即 WEB 服务器发送所有请求。
Web 服务器有：Apache 服务器，IIS 服务器（Internet Information Services）等。
Web 服务器根据接收到的请求后，向客户端发送响应信息。
HTTP 默认端口号为 80，但是你也可以改为 8080 或者其他端口。
HTTP 三点注意事项：
（1）HTTP 是无连接：无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。采用这种方式可以节省传输时间。
（2）HTTP 是媒体独立的：这意味着，只要客户端和服务器知道如何处理的数据内容，任何类型的数据都可以通过 HTTP 发送。客户端以及服务器指定使用适合的 MIME-type 内容类型。
（3）HTTP是无状态：HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快。
以下图表展示了 HTTP 协议通信流程：

## 3.HTTP消息结构

HTTP 是基于客户端/服务端（C/S）的架构模型，通过一个可靠的链接来交换信息，是一个无状态的请求/响应协议。
一个 HTTP"客户端"是一个应用程序（Web 浏览器或其他任何客户端），通过连接到服务器达到向服务器发送一个或多个 HTTP 的请求的目的。
一个 HTTP"服务器"同样也是一个应用程序（通常是一个 Web 服务，如 Apache Web 服务器或 IIS 服务器等），通过接收客户端的请求并向客户端发送 HTTP 响应数据。
HTTP 使用统一资源标识符（Uniform Resource Identifiers, URI）来传输数据和建立连接。一旦建立连接后，数据消息就通过类似 Internet 邮件所使用的格式[RFC5322]和多用途Internet 邮件扩展（MIME）[RFC2045]来传送。
1、客户端请求消息
客户端发送一个 HTTP 请求到服务器的请求消息包括以下格式：请求行（request line）、请求头部（header）、空行和请求数据四个部分组成，下图给出了请求报文的一般格式。

2、服务端请求消息
HTTP 响应也由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

下面实例是一点典型的使用 GET 来传递数据的实例：
客户端请求：

```
GET /hello.txt HTTP/1.1
User-Agent: curl/7.16.3 libcurl/7.16.3 OpenSSL/0.9.7l zlib/1.2.3
Host: www.example.com
Accept-Language: en, mi
```


服务端响应:

```
HTTP/1.1 200 OK
Date: Mon, 27 Jul 2009 12:28:53 GMT
Server: Apache
Last-Modified: Wed, 22 Jul 2009 19:15:56 GMT
ETag: "34aa387-d-1568eb00"
Accept-Ranges: bytes
Content-Length: 51
Vary: Accept-Encoding
Content-Type: text/plain
```


输出结果：

```
Hello World! My payload includes a trailing CRLF.
```



## 4.HTTP请求方法

根据 HTTP 标准，HTTP 请求可以使用多种请求方法。
HTTP1.0 定义了三种请求方法： GET, POST 和 HEAD 方法。
HTTP1.1 新增了六种请求方法：OPTIONS、PUT、PATCH、DELETE、TRACE 和 CONNECT 方法。

## 5.HTTP状态码

当浏览者访问一个网页时，浏览者的浏览器会向网页所在服务器发出请求。当浏览器接收并显示网页前，此网页所在的服务器会返回一个包含 HTTP 状态码的信息头（server header）用以响应浏览器的请求。
HTTP 状态码的英文为 HTTP Status Code。
下面是常见的 HTTP 状态码：
（1）200 - 请求成功
（2）301 - 资源（网页等）被永久转移到其它 URL
（3）404 - 请求的资源（网页等）不存在
（4）500 - 内部服务器错误
HTTP 状态码由三个十进制数字组成，第一个十进制数字定义了状态码的类型，后两个数字没有分类的作用。HTTP 状态码共分为 5 种类型：

## 6.真实示例

http的tcp连接的生命周期：
先acceptcb、再readcb、再writecb。
1、acceptcb：建立http连接；
2、readcb：接受客户端http请求；
3、writecb：发送http响应。
本实例的重点是在reactor的基础上去实现http协议。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <arpa/inet.h>

#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <time.h>

#include <sys/stat.h>
#include <sys/sendfile.h>

#define BUFFER_LENGTH		4096
#define MAX_EPOLL_EVENTS	1024
#define SERVER_PORT			8888
#define PORT_COUNT			1
#define HTTP_WEBSERVER_HTML_ROOT	"html"

#define HTTP_METHOD_GET		0
#define HTTP_METHOD_POST	1

typedef int NCALLBACK(int ,int, void*);

struct ntyevent {
	int fd;
	int events;
	void *arg;
	int (*callback)(int fd, int events, void *arg);
	

	int status;
	char buffer[BUFFER_LENGTH];
	int length;
	long last_active;
	
	// http param
	int method; // get / post
	char resource[BUFFER_LENGTH]; //资源位。url的长度
	int ret_code;

};

struct eventblock {
	struct eventblock *next;
	struct ntyevent *events;
};

struct ntyreactor {
	int epfd;
	int blkcnt;
	struct eventblock *evblk; //fd --> 100w
};


int recv_cb(int fd, int events, void *arg);
int send_cb(int fd, int events, void *arg);
struct ntyevent *ntyreactor_idx(struct ntyreactor *reactor, int sockfd);

void nty_event_set(struct ntyevent *ev, int fd, NCALLBACK callback, void *arg) {
	ev->fd = fd;
	ev->callback = callback;
	ev->events = 0;
	ev->arg = arg;
	ev->last_active = time(NULL);

	return ;

}


int nty_event_add(int epfd, int events, struct ntyevent *ev) {

	struct epoll_event ep_ev = {0, {0}};
	ep_ev.data.ptr = ev;
	ep_ev.events = ev->events = events;
	
	int op;
	if (ev->status == 1) {
		op = EPOLL_CTL_MOD;
	} else {
		op = EPOLL_CTL_ADD;
		ev->status = 1;
	}
	
	if (epoll_ctl(epfd, op, ev->fd, &ep_ev) < 0) {
		printf("event add failed [fd=%d], events[%d]\n", ev->fd, events);
		return -1;
	}
	
	return 0;

}

int nty_event_del(int epfd, struct ntyevent *ev) {

	struct epoll_event ep_ev = {0, {0}};
	
	if (ev->status != 1) {
		return -1;
	}
	
	ep_ev.data.ptr = ev;
	ev->status = 0;
	epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, &ep_ev);
	
	return 0;

}


int readline(char *allbuf, int idx, char *linebuf) {//idx是读取的位置，第几行

	int len = strlen(allbuf);
	
	for(;idx < len;idx ++) {
		if (allbuf[idx] == '\r' && allbuf[idx+1] == '\n') {
			return idx+2; //  \r\n\r\n，是idx+4
		} else {
			*(linebuf++) = allbuf[idx];
		}
	}
	
	return -1;

}

int http_request(struct ntyevent *ev) { //在http数据接收完后，才处理的

	// GET, POST 先判断是GET 还是POST，DELETE HEAD不常用
	char linebuf[1024] = {0};
	int idx = readline(ev->buffer, 0, linebuf);//读一行，以\r\n分割的
	
	if (strstr(linebuf, "GET")) {
	
	    //例子： GET /index.html HTTP/1.1
	
		ev->method = HTTP_METHOD_GET;
	
		//uri, 即 /index.html
		int i = 0;
		while (linebuf[sizeof("GET ") + i] != ' ') i++;
		linebuf[sizeof("GET ")+i] = '\0';
	
		sprintf(ev->resource, "./%s/%s", HTTP_WEBSERVER_HTML_ROOT, linebuf+sizeof("GET "));
		
	} else if (strstr(linebuf, "POST")) {
	
	}

}

int recv_cb(int fd, int events, void *arg) {

	struct ntyreactor *reactor = (struct ntyreactor*)arg;
	struct ntyevent *ev = ntyreactor_idx(reactor, fd);
	
	int len = recv(fd, ev->buffer, BUFFER_LENGTH, 0); // 
	
	if (len > 0) {
		
		ev->length = len;
		ev->buffer[len] = '\0';
	
		printf("C[%d]:%s\n", fd, ev->buffer); //http
	
		http_request(ev);//这里是保证数据全部接收完成，再http request，再send_cb
	
		//send();
		
		nty_event_del(reactor->epfd, ev);
		nty_event_set(ev, fd, send_cb, reactor);
		nty_event_add(reactor->epfd, EPOLLOUT, ev);


		

	} else if (len == 0) {
	
		nty_event_del(reactor->epfd, ev);
		close(ev->fd);
		//printf("[fd=%d] pos[%ld], closed\n", fd, ev-reactor->events);
		 
	} else {
	
		nty_event_del(reactor->epfd, ev);
		close(ev->fd);
		printf("recv[fd=%d] error[%d]:%s\n", fd, errno, strerror(errno));
		
	}
	return len;

}


int http_response(struct ntyevent *ev) {

	if (ev == NULL) return -1;
	memset(ev->buffer, 0, BUFFER_LENGTH);

#if 0
	const char *html = "<html><head><title>hello http</title></head><body><H1>King</H1></body></html>\r\n\r\n";
						  	   

	ev->length = sprintf(ev->buffer, 
		"HTTP/1.1 200 OK\r\n\
		 Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n\
		 Content-Type: text/html;charset=ISO-8859-1\r\n\
		 Content-Length: 83\r\n\r\n%s", 
		 html);

#else

	printf("resource: %s\n", ev->resource);  //ev->resource即是index.html静态文件，存在磁盘上
	
	int filefd = open(ev->resource, O_RDONLY);
	if (filefd == -1) { // return 404
	
		ev->ret_code = 404;
		ev->length = sprintf(ev->buffer, 
			"HTTP/1.1 404 Not Found\r\n"
		 	"Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
		 	"Content-Type: text/html;charset=ISO-8859-1\r\n"
			"Content-Length: 85\r\n\r\n"
		 	"<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n" );
	
	} else {
	
		struct stat stat_buf;   //stat结构体，获取文件属性，主要是为了得到文件的总长度
		fstat(filefd, &stat_buf);
		close(filefd);
	
		if (S_ISDIR(stat_buf.st_mode)) {	
			ev->ret_code = 404; //HTTP FAILED
			ev->length = sprintf(ev->buffer, 
				"HTTP/1.1 404 Not Found\r\n"
				"Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
				"Content-Type: text/html;charset=ISO-8859-1\r\n"
				"Content-Length: 85\r\n\r\n"
				"<html><head><title>404 Not Found</title></head><body><H1>404</H1></body></html>\r\n\r\n" ); // \n和\r，分别都算1个
		} else if (S_ISREG(stat_buf.st_mode)) {
			ev->ret_code = 200; //HTTP OK
			ev->length = sprintf(ev->buffer, 
				"HTTP/1.1 200 OK\r\n"
			 	"Date: Thu, 11 Nov 2021 12:28:52 GMT\r\n"
			 	"Content-Type: text/html;charset=ISO-8859-1\r\n"
				"Content-Length: %ld\r\n\r\n", 
			 		stat_buf.st_size );
		}
	}

#endif
	return ev->length;
}

int send_cb(int fd, int events, void *arg) {

	struct ntyreactor *reactor = (struct ntyreactor*)arg;
	struct ntyevent *ev = ntyreactor_idx(reactor, fd);
	
	http_response(ev);
	
	int len = send(fd, ev->buffer, ev->length, 0);
	if (len > 0) {
		printf("send[fd=%d], [%d]%s\n", fd, len, ev->buffer);
	
		if (ev->ret_code == 200) {
			int filefd = open(ev->resource, O_RDONLY);
			struct stat stat_buf;
			fstat(filefd, &stat_buf);
	
			sendfile(fd, filefd, NULL, stat_buf.st_size); // 发送文件，map 零拷贝
			close(filefd);
		}
		
		nty_event_del(reactor->epfd, ev);
		nty_event_set(ev, fd, recv_cb, reactor);
		nty_event_add(reactor->epfd, EPOLLIN, ev);
		
	} else {
	
		close(ev->fd);
	
		nty_event_del(reactor->epfd, ev);
		printf("send[fd=%d] error %s\n", fd, strerror(errno));
	
	}
	
	return len;

}

int accept_cb(int fd, int events, void *arg) {

	struct ntyreactor *reactor = (struct ntyreactor*)arg;
	if (reactor == NULL) return -1;
	
	struct sockaddr_in client_addr;
	socklen_t len = sizeof(client_addr);
	
	int clientfd;
	
	if ((clientfd = accept(fd, (struct sockaddr*)&client_addr, &len)) == -1) {
		if (errno != EAGAIN && errno != EINTR) {
			
		}
		printf("accept: %s\n", strerror(errno));
		return -1;
	}
	
	int flag = 0;
	if ((flag = fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0) {
		printf("%s: fcntl nonblocking failed, %d\n", __func__, MAX_EPOLL_EVENTS);
		return -1;
	}
	
	struct ntyevent *event = ntyreactor_idx(reactor, clientfd);
	
	nty_event_set(event, clientfd, recv_cb, reactor);
	nty_event_add(reactor->epfd, EPOLLIN, event);
	
	printf("new connect [%s:%d], pos[%d]\n", 
		inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), clientfd);
	
	return 0;

}

int init_sock(short port) {

	int fd = socket(AF_INET, SOCK_STREAM, 0);
	fcntl(fd, F_SETFL, O_NONBLOCK);
	
	struct sockaddr_in server_addr;
	memset(&server_addr, 0, sizeof(server_addr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
	server_addr.sin_port = htons(port);
	
	bind(fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
	
	if (listen(fd, 20) < 0) {
		printf("listen failed : %s\n", strerror(errno));
	}
	
	return fd;

}


int ntyreactor_alloc(struct ntyreactor *reactor) {

	if (reactor == NULL) return -1;
	if (reactor->evblk == NULL) return -1;
	
	struct eventblock *blk = reactor->evblk;
	while (blk->next != NULL) {
		blk = blk->next;
	}
	
	struct ntyevent *evs = (struct ntyevent*)malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
	if (evs == NULL) {
		printf("ntyreactor_alloc ntyevents failed\n");
		return -2;
	}
	memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
	
	struct eventblock *block = (struct eventblock *)malloc(sizeof(struct eventblock));
	if (block == NULL) {
		printf("ntyreactor_alloc eventblock failed\n");
		return -2;
	}
	memset(block, 0, sizeof(struct eventblock));
	
	block->events = evs;
	block->next = NULL;
	
	blk->next = block;
	reactor->blkcnt ++; 
	
	return 0;

}

struct ntyevent *ntyreactor_idx(struct ntyreactor *reactor, int sockfd) {

	int blkidx = sockfd / MAX_EPOLL_EVENTS;
	
	while (blkidx >= reactor->blkcnt) {
		ntyreactor_alloc(reactor);
	}
	
	int i = 0;
	struct eventblock *blk = reactor->evblk;
	while(i ++ < blkidx && blk != NULL) {
		blk = blk->next;
	}
	
	return &blk->events[sockfd % MAX_EPOLL_EVENTS];

}


int ntyreactor_init(struct ntyreactor *reactor) {

	if (reactor == NULL) return -1;
	memset(reactor, 0, sizeof(struct ntyreactor));
	
	reactor->epfd = epoll_create(1);
	if (reactor->epfd <= 0) {
		printf("create epfd in %s err %s\n", __func__, strerror(errno));
		return -2;
	}
	
	struct ntyevent *evs = (struct ntyevent*)malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
	if (evs == NULL) {
		printf("ntyreactor_alloc ntyevents failed\n");
		return -2;
	}
	memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
	
	struct eventblock *block = (struct eventblock *)malloc(sizeof(struct eventblock));
	if (block == NULL) {
		printf("ntyreactor_alloc eventblock failed\n");
		return -2;
	}
	memset(block, 0, sizeof(struct eventblock));
	
	block->events = evs;
	block->next = NULL;
	
	reactor->evblk = block;
	reactor->blkcnt = 1;
	
	return 0;

}

int ntyreactor_destory(struct ntyreactor *reactor) {

	close(reactor->epfd);
	
	struct eventblock *blk = reactor->evblk;
	struct eventblock *blk_next = NULL;
	
	while (blk != NULL) {
	
		blk_next = blk->next;
	
		free(blk->events);
		free(blk);
	
		blk = blk_next;
	
	}
	
	return 0;

}



int ntyreactor_addlistener(struct ntyreactor *reactor, int sockfd, NCALLBACK *acceptor) {

	if (reactor == NULL) return -1;
	if (reactor->evblk == NULL) return -1;
	
	struct ntyevent *event = ntyreactor_idx(reactor, sockfd);
	
	nty_event_set(event, sockfd, acceptor, reactor);
	nty_event_add(reactor->epfd, EPOLLIN, event);
	
	return 0;

}



int ntyreactor_run(struct ntyreactor *reactor) {
	if (reactor == NULL) return -1;
	if (reactor->epfd < 0) return -1;
	if (reactor->evblk == NULL) return -1;
	

	struct epoll_event events[MAX_EPOLL_EVENTS+1];
	
	int checkpos = 0, i;
	
	while (1) {
		int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);
		if (nready < 0) {
			printf("epoll_wait error, exit\n");
			continue;
		}
	
		for (i = 0;i < nready;i ++) {
	
			struct ntyevent *ev = (struct ntyevent*)events[i].data.ptr;
	
			if ((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)) {
				ev->callback(ev->fd, events[i].events, ev->arg);
			}
			if ((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)) {
				ev->callback(ev->fd, events[i].events, ev->arg);
			}
		}
	}

}

// 3, 6w, 1, 100 == 
// <remoteip, remoteport, localip, localport>
int main(int argc, char *argv[]) {

	unsigned short port = SERVER_PORT; // listen 8888
	if (argc == 2) {
		port = atoi(argv[1]);
	}
	struct ntyreactor *reactor = (struct ntyreactor*)malloc(sizeof(struct ntyreactor));
	ntyreactor_init(reactor);
	
	int i = 0;
	int sockfds[PORT_COUNT] = {0};
	for (i = 0;i < PORT_COUNT;i ++) {
		sockfds[i] = init_sock(port+i);
	    ntyreactor_addlistener(reactor, sockfds[i], accept_cb);
	}
	
	ntyreactor_run(reactor);
	
	ntyreactor_destory(reactor);
	
	for (i = 0;i < PORT_COUNT;i ++) {
		close(sockfds[i]);
	}
	
	free(reactor);
	
	return 0;

}
```







————————————————
版权声明：本文为CSDN博主「当当响」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zhpCSDN921011/article/details/124685893