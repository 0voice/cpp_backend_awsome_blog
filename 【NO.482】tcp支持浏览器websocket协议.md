# 【NO.482】tcp支持浏览器websocket协议

一个io它是怎么一种情况，一个客户端连接一个服务器，一个客户端一个连接，大家时刻在做服务器，都是时刻抓住这样一个点，
就是说一个客户端在服务端会有一个网络io，一个客户端在服务端会有一个网络io，之前用epoll来管理这些io我们写了一个版本写了一个版本是怎么做的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b360ccda8974401fb2b5b2f606c3d905.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



网络io到epoll的实现,epoll在实现的中间它有哪些问题以及如何去封装，然后再一层一层的跟大家实现的这个反应堆模式，就是以大量的网络io，然后每一个网络io它对应的事件会有对应的回调函数，
网络io对应的事件有对应的回调函数，

**那这个网络io对应的事件是说的哪些事件？**
就是所说的epollin/可读,epollout可写，
对应的回调函数怎么理解呢，就是epollin的时候我们调用recv_cb
然后epollout我们调send_cb，
当然还有与之对应的listen,我们可以调用accept_cb，封装完了之后，它的封装性更强，
它的封装性更强好吧，这是跟大家讲的reactor就这么一个模式，

在reactor的基础上面我们看看协议怎么做，基于websocket,就是因为协议很简单，它大体上的东西,核心的元素都会有

一个客户端对应一个连接，一个连接是怎么管理的呢就是通过epoll管理，一个连接我们对应的数据存储在哪呢？

服务器所生成的一个fd我们放到sockitem

![在这里插入图片描述](https://img-blog.csdnimg.cn/f146f1c9fac74abeb820d85d0cc25c83.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



如果客户端给服务器发数据，epoll检测到io可读，就会调用sockitem里面对应回调函数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/dd2fb07dcbab4f5abfd1d8b3589e7712.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_9,color_FFFFFF,t_70,g_se,x_16)


如果是收到数据那么有两种情况：一个io对应可读或者可写
读的数据发送的数据都放到buffer里面





![在这里插入图片描述](https://img-blog.csdnimg.cn/53849ead2c1e496fbbee348b8a90d5fd.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_14,color_FFFFFF,t_70,g_se,x_16)


接收

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd8edb1cdcc64d53be9f49b0f40692f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_12,color_FFFFFF,t_70,g_se,x_16)


发送

![在这里插入图片描述](https://img-blog.csdnimg.cn/c508497d888d41ceb4b2eb51c51f9ffe.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_12,color_FFFFFF,t_70,g_se,x_16)



那么分析这个代码的时候逻辑就会有很大的改变了

![在这里插入图片描述](https://img-blog.csdnimg.cn/ae3dab0845b24be3a00fc0da4c86ed7c.png)



recv_cb里面我们接收到的数据我们该怎么去发送，比如回声服务器接收数据就返回什么数据的话我们该怎么做？

我们不需要去关注在什么时候调用send，我们只要关注一点这个sendbuffer里面有没有数据就ok了，只要关注这样一点就ok了，它会自动发送，

![在这里插入图片描述](https://img-blog.csdnimg.cn/c706a1d3b2b9401aa0d3cc543f95d935.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_13,color_FFFFFF,t_70,g_se,x_16)





![在这里插入图片描述](https://img-blog.csdnimg.cn/f603b030d4574959a594519a6a1e032b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



发送是怎么发送的呢？
两步就ok
我们现在根本就不用关注他的发送，只要把send_cb这一步做好了

![在这里插入图片描述](https://img-blog.csdnimg.cn/fb5c90ab6f35443a9c96ef9d0fefbc91.png)


libevent源码也是这样设计的有一个buffer，把数据放到里面就ok了，



我们只是做到了接收与发送，要是有协议呢？
**接下来我们如何把websocket协议加进去？**



![在这里插入图片描述](https://img-blog.csdnimg.cn/6d2ebe902d414d3d978c9e72e370ea57.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



首先第一步要考虑的是websocket的握手，

关于websocket握手是怎么一回事
**websocket使用在哪？**
为什么会有一个握手？

主要它是用来浏览器跟服务器做一个长链接，什么意思？我们打开CSDN有个登录，我们看到这边我们点击登录，现在这时候出来一个微信二维码，我们现在通过微信扫描二维码登录

那么这个功能与我们的websocket有什么关系？
前端页面，二维码是前端的，现在我们通过微信的那个微信的客户端，我们扫码扫一下这个二维码，扫完之后把对应的二维码，传到微信的服务器，然后微信的服务器，进行回调到csdn的服务器，然后前端扫码完之后为什么会有一个跳转？
就是这步，在csdn的服务器会主动推送一个数据，csdn的服务器，主动发数据给网页前端。
那在这一步是服务器主动发的，主动发的过程中间，就是采用了websocket协议

服务器主动发数据给浏览器的时候，可以选择websocket，但是websocket不是唯一的解决方案，
1.网页聊天，即时通讯
2.网页弹幕
3.股票
单独使用tcp没有那么好做，websocket是基于tcp，

**了解websocket的使用场景后思考一下websocket是怎么建立的？**

weisocket协议和客户端之间是怎么一种反思？

![在这里插入图片描述](https://img-blog.csdnimg.cn/9c4b529653ac4f2596b64481b0ae19b9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



这个连接请大家注意，这个连接是在TCP建立后连接的基础上面，客户端和服务器已经有一个连接了，然后客户端给服务器发送一段应用数据，这个数据叫做握手数据，

相当于是这样一个客户端和服务器之间建立好的连接，现在这个客户端发送一段数据
首先发的第一步数据验证双方是否合法，这个数据叫做握手数据，



![在这里插入图片描述](https://img-blog.csdnimg.cn/63250e89d8934365b6f257327f0479df.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


与http协议如此相似，
客户端先握手成功之后在服务器发送消息
**握手过程**

![在这里插入图片描述](https://img-blog.csdnimg.cn/308e2a38843f4888b74d252ad7cccca9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_13,color_FFFFFF,t_70,g_se,x_16)



websocket协议由两部分组成一部分握手，另一部分通信。

**就是在recv_cb里面我们怎么接收？我们怎么去区分他是握手数据还是通信数据？**
需要引入一个状态机，

![在这里插入图片描述](https://img-blog.csdnimg.cn/69ea33c0379e497691070392ece473a0.png)



那么这个状态机的状态我们保存在哪里？每一个连接里面应该都有一个状态机

![在这里插入图片描述](https://img-blog.csdnimg.cn/35852a8788e0483ba1f77decc234c4a0.png)





![在这里插入图片描述](https://img-blog.csdnimg.cn/2b0e2650f2034e8aa0d5b559b038d93f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_11,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/22062475ad114c908f5f8e781e85cc90.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)



![在这里插入图片描述](https://img-blog.csdnimg.cn/c3213ae1069e45edaadb5e8a28ccd178.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)


区分accept_cb和recv_cb

![在这里插入图片描述](https://img-blog.csdnimg.cn/9a673125a23045d79e2bd01e51c9de92.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_14,color_FFFFFF,t_70,g_se,x_16)


握手的状态怎么进入通信的状态？



刚刚讲了状态机，横向思考一下，其实HTTP协议也需要有一个状态机
在http协议接收的时候它也有握手，同样它有它的header还有它的body
header和body里面都有自己对应的资源，就是方便理解为什么nginx里面有一个状态机的实现？

实现recv_cb的时候我们不能通过具体的数据去判断它是不是头。
第一个状态处理完了之后去处理下一个状态，状态机就是这样。

websocket通信的时候它的协议头是什么样的？

![在这里插入图片描述](https://img-blog.csdnimg.cn/53e16a3c6d334d4fa91e2046de10c71e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


如果以后自己基于TCP做协议的时候，可以看到有三个核心的点
1.操作码，fin是不是终止的包是不是数据包是不是握手包
2.包长度，分包粘包怎么解：可以选择包长度或者分隔符，这里websocket选择的就是包长度，
3.不想传输明文可以加一个mask-->key 主要与payload做一个可逆的计算得出data
4.payload data数据是纯应用层的数据，就可以采用json/xml，





![在这里插入图片描述](https://img-blog.csdnimg.cn/ea19d68a9d0e4a71855ee44eff6455f3.png)


长连接是客户端和服务器维持的一个连接，通过心跳包去维持，短连接就是一次请求不用管了，发送短信，长连接计算完需要回数据



tcp的keepalive有这么几个特点，不要去代替应用层的心跳包
1.一旦超时之后tcp会自动的去回收keepalive
2.超时之后应用层得不到可控制的反馈，没办法去判断他超时我们该做什么策略性的东西

```cpp
#include <stdio.h>



#include <stdlib.h>



#include <string.h>



 



#include <unistd.h>



#include <netinet/tcp.h>



#include <arpa/inet.h>



#include <pthread.h>



 



#include <fcntl.h>



#include <errno.h>



#include <sys/epoll.h>



 



#include <openssl/sha.h>



#include <openssl/pem.h>



#include <openssl/bio.h>



#include <openssl/evp.h>



 



#define BUFFER_LENGTH            1024



#define GUID                     "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"



 



 



enum  WEBSOCKET_STATUS {



    WS_HANDSHARK,



    WS_DATATRANSFORM,



    WS_DATAEND,



};



 



 



struct sockitem { //



    int sockfd;



    int (*callback)(int fd, int events, void *arg);



 



    char recvbuffer[BUFFER_LENGTH]; //



    char sendbuffer[BUFFER_LENGTH];



 



    int rlength;



    int slength;



 



    int status;



 



};



 



// mainloop / eventloop --> epoll -->  



struct reactor {



 



    int epfd;



    struct epoll_event events[512];



    



    



 



};



 



 



struct reactor *eventloop = NULL;



 



int recv_cb(int fd, int events, void *arg);



int send_cb(int fd, int events, void *arg);



 



#if 1  // websocket



 



char* decode_packet(char *stream, char *mask, int length, int *ret);



int encode_packet(char *buffer,char *mask, char *stream, int length);



 



struct _nty_ophdr {



 



    unsigned char opcode:4,



         rsv3:1,



         rsv2:1,



         rsv1:1,



         fin:1;



    unsigned char payload_length:7,



        mask:1;



 



} __attribute__ ((packed));



 



struct _nty_websocket_head_126 {



    unsigned short payload_length;



    char mask_key[4];



    unsigned char data[8];



} __attribute__ ((packed));



 



struct _nty_websocket_head_127 {



 



    unsigned long long payload_length;



    char mask_key[4];



 



    unsigned char data[8];



    



} __attribute__ ((packed));



 



typedef struct _nty_websocket_head_127 nty_websocket_head_127;



typedef struct _nty_websocket_head_126 nty_websocket_head_126;



typedef struct _nty_ophdr nty_ophdr;



 



 



int base64_encode(char *in_str, int in_len, char *out_str) {    



    BIO *b64, *bio;    



    BUF_MEM *bptr = NULL;    



    size_t size = 0;    



 



    if (in_str == NULL || out_str == NULL)        



        return -1;    



 



    b64 = BIO_new(BIO_f_base64());    



    bio = BIO_new(BIO_s_mem());    



    bio = BIO_push(b64, bio);



    



    BIO_write(bio, in_str, in_len);    



    BIO_flush(bio);    



 



    BIO_get_mem_ptr(bio, &bptr);    



    memcpy(out_str, bptr->data, bptr->length);    



    out_str[bptr->length-1] = '\0';    



    size = bptr->length;    



 



    BIO_free_all(bio);    



    return size;



}



 



 



int readline(char* allbuf,int level,char* linebuf) {    



    int len = strlen(allbuf);    



 



    for (;level < len; ++level)    {        



        if(allbuf[level]=='\r' && allbuf[level+1]=='\n')            



            return level+2;        



        else            



            *(linebuf++) = allbuf[level];    



    }    



 



    return -1;



}



 



int handshark(struct sockitem *si, struct reactor *mainloop) {



 



    char linebuf[256];



    char sec_accept[32]; 



    int level = 0;



    unsigned char sha1_data[SHA_DIGEST_LENGTH+1] = {0};



    char head[BUFFER_LENGTH] = {0};  



 



    do {        



        memset(linebuf, 0, sizeof(linebuf));        



        level = readline(si->recvbuffer, level, linebuf); 



 



        if (strstr(linebuf,"Sec-WebSocket-Key") != NULL)        {   



            



            strcat(linebuf, GUID);    



            



            SHA1((unsigned char*)&linebuf+19,strlen(linebuf+19),(unsigned char*)&sha1_data);  



            



            base64_encode(sha1_data,strlen(sha1_data),sec_accept);           



            sprintf(head, "HTTP/1.1 101 Switching Protocols\r\n" \



                "Upgrade: websocket\r\n" \



                "Connection: Upgrade\r\n" \



                "Sec-WebSocket-Accept: %s\r\n" \



                "\r\n", sec_accept);            



 



            printf("response\n");            



            printf("%s\n\n\n", head);            



#if 0



            if (write(cli_fd, head, strlen(head)) < 0)     //write ---> send            



                perror("write");            



#else



            memset(si->recvbuffer, 0, BUFFER_LENGTH);



 



            memcpy(si->sendbuffer, head, strlen(head)); // to send 



            si->slength = strlen(head);



 



            // to set epollout events



            struct epoll_event ev;



            ev.events = EPOLLOUT | EPOLLET;



            //ev.data.fd = clientfd;



            si->sockfd = si->sockfd;



            si->callback = send_cb;



            si->status = WS_DATATRANSFORM;



            ev.data.ptr = si;



 



            epoll_ctl(mainloop->epfd, EPOLL_CTL_MOD, si->sockfd, &ev);



 



#endif



            break;        



        }    



 



    } while((si->recvbuffer[level] != '\r' || si->recvbuffer[level+1] != '\n') && level != -1);    



 



    return 0;



}



 



int transform(struct sockitem *si, struct reactor *mainloop) {



 



    int ret = 0;



    char mask[4] = {0};



    char *data = decode_packet(si->recvbuffer, mask, si->rlength, &ret);



 



 



    printf("data : %s , length : %d\n", data, ret);



 



    ret = encode_packet(si->sendbuffer, mask, data, ret);



    si->slength = ret;



 



    memset(si->recvbuffer, 0, BUFFER_LENGTH);



 



    struct epoll_event ev;



    ev.events = EPOLLOUT | EPOLLET;



    //ev.data.fd = clientfd;



    si->sockfd = si->sockfd;



    si->callback = send_cb;



    si->status = WS_DATATRANSFORM;



    ev.data.ptr = si;



 



    epoll_ctl(mainloop->epfd, EPOLL_CTL_MOD, si->sockfd, &ev);



 



    return 0;



}



 



void umask(char *data,int len,char *mask) {    



    int i;    



    for (i = 0;i < len;i ++)        



        *(data+i) ^= *(mask+(i%4));



}



 



char* decode_packet(char *stream, char *mask, int length, int *ret) {



 



    nty_ophdr *hdr =  (nty_ophdr*)stream;



    unsigned char *data = stream + sizeof(nty_ophdr);



    int size = 0;



    int start = 0;



    //char mask[4] = {0};



    int i = 0;



 



    //if (hdr->fin == 1) return NULL;



 



    if ((hdr->mask & 0x7F) == 126) {



 



        nty_websocket_head_126 *hdr126 = (nty_websocket_head_126*)data;



        size = hdr126->payload_length;



        



        for (i = 0;i < 4;i ++) {



            mask[i] = hdr126->mask_key[i];



        }



        



        start = 8;



        



    } else if ((hdr->mask & 0x7F) == 127) {



 



        nty_websocket_head_127 *hdr127 = (nty_websocket_head_127*)data;



        size = hdr127->payload_length;



        



        for (i = 0;i < 4;i ++) {



            mask[i] = hdr127->mask_key[i];



        }



        



        start = 14;



 



    } else {



        size = hdr->payload_length;



 



        memcpy(mask, data, 4);



        start = 6;



    }



 



    *ret = size;



    umask(stream+start, size, mask);



 



    return stream + start;



    



}



 



 



int encode_packet(char *buffer,char *mask, char *stream, int length) {



 



    nty_ophdr head = {0};



    head.fin = 1;



    head.opcode = 1;



    int size = 0;



 



    if (length < 126) {



        head.payload_length = length;



        memcpy(buffer, &head, sizeof(nty_ophdr));



        size = 2;



    } else if (length < 0xffff) {



        nty_websocket_head_126 hdr = {0};



        hdr.payload_length = length;



        memcpy(hdr.mask_key, mask, 4);



 



        memcpy(buffer, &head, sizeof(nty_ophdr));



        memcpy(buffer+sizeof(nty_ophdr), &hdr, sizeof(nty_websocket_head_126));



        size = sizeof(nty_websocket_head_126);



        



    } else {



        



        nty_websocket_head_127 hdr = {0};



        hdr.payload_length = length;



        memcpy(hdr.mask_key, mask, 4);



        



        memcpy(buffer, &head, sizeof(nty_ophdr));



        memcpy(buffer+sizeof(nty_ophdr), &hdr, sizeof(nty_websocket_head_127));



 



        size = sizeof(nty_websocket_head_127);



        



    }



 



    memcpy(buffer+2, stream, length);



 



    return length + 2;



}



 



 



#endif 



 



 



 



static int set_nonblock(int fd) {



    int flags;



 



    flags = fcntl(fd, F_GETFL, 0);



    if (flags < 0) return flags;



    flags |= O_NONBLOCK;



    if (fcntl(fd, F_SETFL, flags) < 0) return -1;



    return 0;



}



 



 



int send_cb(int fd, int events, void *arg) {



 



    struct sockitem *si = (struct sockitem*)arg;



 



    send(fd, si->sendbuffer, si->slength, 0); //



 



    struct epoll_event ev;



    ev.events = EPOLLIN | EPOLLET;



    //ev.data.fd = clientfd;



    si->sockfd = fd;



    si->callback = recv_cb;



    ev.data.ptr = si;



    



    memset(si->sendbuffer, 0, BUFFER_LENGTH);



 



    epoll_ctl(eventloop->epfd, EPOLL_CTL_MOD, fd, &ev);



 



}



 



//  ./epoll 8080



 



int recv_cb(int fd, int events, void *arg) {



 



    //int clientfd = events[i].data.fd;



    struct sockitem *si = (struct sockitem*)arg;



    struct epoll_event ev;



 



    int ret = recv(fd, si->recvbuffer, BUFFER_LENGTH, 0);



    if (ret < 0) {



 



        if (errno == EAGAIN || errno == EWOULDBLOCK) { //



            return -1;



        } else {



            



        }



 



        ev.events = EPOLLIN;



        //ev.data.fd = fd;



        epoll_ctl(eventloop->epfd, EPOLL_CTL_DEL, fd, &ev);



        close(fd);



        free(si);



        



 



    } else if (ret == 0) { //



 



        // 



        printf("disconnect %d\n", fd);



        ev.events = EPOLLIN;



        //ev.data.fd = fd;



        epoll_ctl(eventloop->epfd, EPOLL_CTL_DEL, fd, &ev);



        close(fd);



        free(si);



        



    } else {



 



        //printf("Recv: %s, %d Bytes\n", si->recvbuffer, ret);



        si->rlength = 0;



 



        if (si->status == WS_HANDSHARK) {



            printf("request\n");    



            printf("%s\n", si->recvbuffer);   



 



            handshark(si, eventloop);



        } else if (si->status == WS_DATATRANSFORM) {



            transform(si, eventloop);



        } else if (si->status == WS_DATAEND) {



 



        }



 



    }



 



}



 



 



int accept_cb(int fd, int events, void *arg) {



 



    struct sockaddr_in client_addr;



    memset(&client_addr, 0, sizeof(struct sockaddr_in));



    socklen_t client_len = sizeof(client_addr);



    



    int clientfd = accept(fd, (struct sockaddr*)&client_addr, &client_len);



    if (clientfd <= 0) return -1;



 



    set_nonblock(clientfd);



 



    char str[INET_ADDRSTRLEN] = {0};



    printf("recv from %s at port %d\n", inet_ntop(AF_INET, &client_addr.sin_addr, str, sizeof(str)),



        ntohs(client_addr.sin_port));



 



    struct epoll_event ev;



    ev.events = EPOLLIN | EPOLLET;



    //ev.data.fd = clientfd;



 



    struct sockitem *si = (struct sockitem*)malloc(sizeof(struct sockitem));



    si->sockfd = clientfd;



    si->callback = recv_cb;



    si->status = WS_HANDSHARK;



    ev.data.ptr = si;



    



    epoll_ctl(eventloop->epfd, EPOLL_CTL_ADD, clientfd, &ev);



    



    return clientfd;



}



 



int main(int argc, char *argv[]) {



 



    if (argc < 2) {



        return -1;



    }



 



    int port = atoi(argv[1]);



 



    



 



    int sockfd = socket(AF_INET, SOCK_STREAM, 0);



    if (sockfd < 0) {



        return -1;



    }



 



    set_nonblock(sockfd);



 



    struct sockaddr_in addr;



    memset(&addr, 0, sizeof(struct sockaddr_in));



 



    addr.sin_family = AF_INET;



    addr.sin_port = htons(port);



    addr.sin_addr.s_addr = INADDR_ANY;



 



    if (bind(sockfd, (struct sockaddr*)&addr, sizeof(struct sockaddr_in)) < 0) {



        return -2;



    }



 



    if (listen(sockfd, 5) < 0) {



        return -3;



    }



 



    eventloop = (struct reactor*)malloc(sizeof(struct reactor));



    // epoll opera



 



    eventloop->epfd = epoll_create(1);



    



    struct epoll_event ev;



    ev.events = EPOLLIN;



    



    struct sockitem *si = (struct sockitem*)malloc(sizeof(struct sockitem));



    si->sockfd = sockfd;



    si->callback = accept_cb;



    ev.data.ptr = si;



    



    epoll_ctl(eventloop->epfd, EPOLL_CTL_ADD, sockfd, &ev);



 



    while (1) {



 



        int nready = epoll_wait(eventloop->epfd, eventloop->events, 512, -1);



        if (nready < -1) {



            break;



        }



 



        int i = 0;



        for (i = 0;i < nready;i ++) {



 



 



 



            if (eventloop->events[i].events & EPOLLIN) {



                //printf("sockitem\n");



                struct sockitem *si = (struct sockitem*)eventloop->events[i].data.ptr;



                si->callback(si->sockfd, eventloop->events[i].events, si);



            }



 



            if (eventloop->events[i].events & EPOLLOUT) {



                struct sockitem *si = (struct sockitem*)eventloop->events[i].data.ptr;



                si->callback(si->sockfd, eventloop->events[i].events, si);



            }



        }



 



    }



 



}



 



 



 
```

原文作者：[[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605160175