# 【NO.571】websocket协议介绍与基于reactor模型的websocket服务器实现

## 0.前言

  本文对websocket协议与参数进行详细的介绍，并基于reactor模型实现websocket服务器

  本专栏知识点是通过零声教育的线上课学习，进行梳理总结写下文章，对c/c++linux课程感兴趣的读者，可以点击链接 C/C++后台高级服务器课程介绍 详细查看课程的服务。

## 1.websocket介绍

### 1.1 websocket是什么

  websocket是基于tcp协议的应用层协议，也就是建立在tcp协议之上的自定义协议。这个协议比http协议更加的简单，因为websocket只对协议的格式做要求，只要符合数据格式就可以使用。

  websocket一般用来服务器主动推送消息给客户端，反观HTTP，HTTP是请求响应的模式，客户端来一个请求，服务器响应一个请求，服务器无法主动发送数据给客户端；并且使用websocket，客户端和服务器只需要一次“握手”，两者之间就成功建立了长连接，可以双向传输数据。

  现在有很多网站都有推送功能，比如现在有个人关注了我的CSDN号，或者给我点了赞，只要我这个浏览器在CSDN界面，就能立刻收到提醒，这就是推送功能，一般都是按照时间间隔轮询；如果我们使用HTTP去做的话，浏览器需要不断的向服务器发请求，而HTTP请求头又有很多无用数据，显而易见的是浪费带宽等资源。

  而websocket不一样，websocket的开销很小，并且主要是由服务器主动推送消息给客户端，不再需要轮询了，所以实时性很高。

### 1.2 websocket的优点

  总结一下websocket的优点：

1.websocket协议简单
2.可以基于websocket自定义协议
3.websocket一般用来服务器主动推送消息给客户端（实时性很高）
4.客户端和服务器建立连接只需要一次“握手”就可以保持长连接（开销很小）

### 1.3 websocket应用场景

举个登陆CSDN的例子：

1.用户选择微信扫码登陆，浏览器发送HTTP请求给CSDN服务器
2.服务器返回一个二维码给浏览器
3.用户通过微信扫码登陆
4.微信扫码成功，将消息传给微信服务器进行处理
5.微信服务器触发回调给CSDN服务器发一个通知
6.CSDN服务器给浏览器发送一个websocket通知浏览器登陆成功

![在这里插入图片描述](https://img-blog.csdnimg.cn/6fb7388a063f4907aa610fc04a78ee88.png)
  访客给我的文章点了个赞，我这里立刻收到提醒，可以看到这就是服务器主动推送消息给客户端，websocket的应用。

![![在这里插入图片描述](https://img-blog.csdnimg.cn/65debf3564f54db8bcd26fc98b5bb97f.png](https://img-blog.csdnimg.cn/68b9dd5aeb6343a69edf491d9764c785.png)

### 1.4 websocket协议剖析

**握手协议**
  websocket握手是HTTP的GET请求的升级版，现在假设客户端连接服务器，服务器返回一次握手信息，连接即可建立，具体步骤如下：

1.客户端使用ws://192.168.109.100:8081连接服务器，实际上是采用HTTP的GET请求来进行握手

```
GET / HTTP/1.1

# 对端主机

Host: 192.168.109.100:8081

# 协议书升级

Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36

# 升级的websocket

Upgrade: websocket
Origin: http://www.websocket-test.com

# websocket版本号

Sec-WebSocket-Version: 13
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9

# 客户端随机生成的一个16字节随机数，作为简单的认证标识

Sec-WebSocket-Key: 9jEz8msH1BBH9H43adEMZQ==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```


2.服务器接收到对应的GET请求后，发现协议需要升级，返回响应，其中需要注意的是Sec-WebSocket-Accept

服务器接收到对应的GET请求后，发现协议需要升级，返回响应，其中需要注意的是Sec-WebSocket-Accept

```
# 101 表示切换协议

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 9oFmMWgFISY8DBlo5xq1L1rc0+0=
```

3.Sec-WebSocket-Accept
  Sec-WebSocket-Accept需要做3次计算得出，主要是为了验证客户端合法性。客户端收到服务器发来的响应后，同样会进行下面的操作，然后将收到的结果与自身算的结果进行对比，如果一样则说明合法，握手成功（WebSocket建立成功）。后续数据传输不再使用HTTP协议，而是使用websocket自定义的一套协议规范。



```
# GUID是websocket规定好的全局唯一的不变的字符串

#define    GUID    "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

# 将客户端发来的GET请求中的Sec-WebSocket字段的字符串与全局唯一的GUID拼接

1. key=Sec-WebSocket-Key+GUID

# 对这个新字符串做SHA1运算，得到20B长的字符串sha_key

2. sha_key=SHA1(key)

# 对sha_key做base64_encode编码，就能得到Sec-WebSocket-Accept了

3. sec_key=base64_encode(sha_key)
```



### 1.5 传输协议



![在这里插入图片描述](https://img-blog.csdnimg.cn/7c8c794b758442729ed29915a6fc9e0f.png)


  写代码的时候只需要根据图中的协议来解析数据包即可，所以要将协议的情况分清楚。

### 1.6 参数介绍

```
FIN：1bit，当FIN值为0的时候代表，消息还没完整，只是其中的一个数据包；当值为1的时候代表这段消息已经完全发送了。
RSV 1 / 2 / 3：1bit，这个默认为 0 ，如果一定要启用，就得和服务端协商好，才具有意义。
opcode：4bit，该数据包类型。
0 代表数据不完整，这只是其中的一个，不是最后的那个数据包。（Continuation Frame）

1 代表数据包内容的类型为 文本类型（Text Frame）

2 代表数据内容类型为 二进制类型 （Binary Frame）

8 代表连接断开 （Connection Close Frame）

9 和 10 是心跳检测，如果服务端发出 Ping Frame 那么客户端就得发回 Pong Frame ，如果服务端接受不到 Pong Frame 就代表客户端可能已经下线了。

0 - 15 中现在除了这 6 个，都为保留帧。
```

MASK： 1bit，是否开启掩码。1开0关。如果开启了，下面的Masking-key就有意义了，一般是发送消息的数据包会开启，返回响应的数据包不开启，并且也没由Masking-key这4个bit。

```
// 如果Mask为1，则Payload Data 就需要通过 Mask 掩码解密（这里指接收消息）
void umask(char *payload, int length, char *mask_key) {
    int i = 0;
    for (i = 0; i < length; i++) {
        payload[i] ^= mask_key[i % 4];
    }
}
```


Payload len：7bit，内容数据（Payload Data）的长度

```
如果Payload len<126,则data len=Payload len

如果Payload len=126，则启用Extended payload length，多加了16bit，并且data len=Extended payload length

如果Payload len=127，则启用Extended payload length和Extended payload length continued，多加了16+48bit，并且data len=Extended payload length continued
```

Masking-key：32bit，掩码数据，如果 Mask 为 1 就启用，否则不启用

Payload Data：数据 0-127Byte，由上面3个如果决定大小

### 1.7 大白话

  协议中前面2Byte固定存在，后面的 Extended payload length 这2Byte和 Extended payload length continued 这6Byte存不存在由 Payload len决定。

```
如果Payload len<126,则data len=Payload len

如果Payload len=126，则启用Extended payload length，多加了16bit，并且data len=Extended payload length

如果Payload len=127，则启用Extended payload length和Extended payload length continued，多加了16+48bit，并且data len=Extended payload length continued
```

  如果MASK=1，则后面有4Byte的Masking-key，则MASK=0则没有这4Byte，再后面就是Payload Data了，多长就是前面Payload len的三种情况了。

## 2.websocket四问

### 2.1 websocket协议格式

websocket协议格式一共有两种：

一种是握手的GET请求

![在这里插入图片描述](https://img-blog.csdnimg.cn/170d0aedcce14126877c809feaeb065c.png)

一种是数据传输协议格式

![在这里插入图片描述](https://img-blog.csdnimg.cn/7c8c794b758442729ed29915a6fc9e0f.png)

### 2.2 websocket如何验证客户端合法

  Sec-WebSocket-Accept需要做3次计算得出，主要是为了验证客户端合法性。客户端收到服务器发来的响应后，同样会进行下面的操作，然后将收到的结果与自身算的结果进行对比，如果一样则说明合法，握手成功（WebSocket建立成功）。

```
# GUID是websocket规定好的全局唯一的不变的字符串

#define    GUID    "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

# 将客户端发来的GET请求中的Sec-WebSocket字段的字符串与全局唯一的GUID拼接

1. key=Sec-WebSocket-Key+GUID

# 对这个新字符串做SHA1运算，得到20B长的字符串sha_key

2. sha_key=SHA1(key)

# 对sha_key做base64_encode编码，就能得到Sec-WebSocket-Accept了

3. sec_key=base64_encode(sha_key)
```

### 2.3 明文与密文如何传输

使用参数 MASK： 1bit，是否开启掩码。1开0关。

如果是要发送密文，首先将MASK置1，然后将明文数据与Masking-key进行异或操作
如果是要解码成明文，将密文数据与Masking-key进行异或操作
如果是要发送明文，MASK置0即可

```
for (i = 0; i < length; i++) {
    payload[i] ^= mask_key[i % 4];
}
```



### 2.4 websocket如何断开

  我们知道websocket是建立在TCP之上的，直接close不就好了吗，为什么还要规定opcode=8的时候代表断开连接呢？

  客户端在调用close之前，先发送一个断开连接的包给服务器；服务器接收到这个包后，把对应的fd连接数据（相关联的用户数据，业务数据）做清空，然后再调用close断开TCP，这样就是优雅的断开连接，close流畅，不会出现大量的close_wait的情况

## 3.基于reactor模型的websocket服务器

### 3.1 握手代码介绍

  websocket有3个状态，握手，传输与关闭。所以我们定义一个状态机。
  在读完数据后，将数据交由 websocket_request(ev);管理；其会判断当前连接处于哪个状态，第一次就是握手状态；
  handshake(ev);对GET请求解析出Sec-WebSocket-Key，然后计算Sec-WebSocket-Accept，组装响应。

  这里就对着上面介绍的握手协议写代码即可

```
struct ntyevent {
    //...略
    int state_machine;
};
//state_machine
enum {
    WS_HANDSHAKE = 0,
    WS_TRANSMISSION = 1,
    WS_END = 2
};
```



```
int websocket_request(struct ntyevent *ev) {
    if (ev->state_machine == WS_HANDSHAKE) {
        handshake(ev);
        ev->state_machine = WS_TRANSMISSION;
    }
    else if (ev->state_machine == WS_TRANSMISSION) {
        transmission(ev);
    }
    else {

    }

}

int recv_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    struct ntyevent *ev = ntyreactor_find_event_idx(reactor, fd);
    memset(ev->buffer, 0, BUFFER_LENGTH);
#if 0
    long len = recv(fd, ev->buffer, BUFFER_LENGTH, 0); //
#elif 1
    int len = 0;
    int n = 0;
    while (1) {
        n = recv(fd, ev->buffer + len, BUFFER_LENGTH - len, 0);
        printf("[recv data len = %d]\n", n);
        if (n != -1) {
            len += n;
        }
        else {
            break;
        }
    }
#endif
    nty_event_del(reactor->epfd, ev);
    printf("[recv buffer total len=%d]\n", len);
    printf("buffer:[%s]\n", ev->buffer);
    if (len > 0) {
        ev->length = len;
        ev->buffer[len] = '\0';

        websocket_request(ev);
    
        nty_event_set(ev, fd, send_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLOUT, ev);
    }
    else if (len == 0) {
        close(ev->fd);
    }
    else {
        close(ev->fd);
    }
    return len;

}
```



```
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
    out_str[bptr->length - 1] = '\0';
    size = bptr->length;
    
    BIO_free_all(bio);
    return size;

}

int readline(char *all_buffer, int idx, char *line_buffer) {
    int len = strlen(all_buffer);
    for (; idx < len; idx++) {
        if (all_buffer[idx] == '\r' && all_buffer[idx + 1] == '\n') {
            return idx + 2;
        }
        else {
            *(line_buffer++) = all_buffer[idx];
        }
    }
    return -1;
}

int handshake(struct ntyevent *ev) {
    char line_buffer[1024] = {0};
    char sha_key[32] = {0};//实际只需20B
    char sec_key[32] = {0};//实际只需28B
    int idx = 0;
    //找到Sec-WebSocket-Key这一行
    while (!strstr(line_buffer, "Sec-WebSocket-Key")) {
        memset(line_buffer, 0, 1024);
        idx = readline(ev->buffer, idx, line_buffer);
        if (idx == -1)return -1;
    }
    //1. key=KEY+GUID
    //2. sha_key=SHA1(key)
    //3. sec_key=base64_encode(sha_key)

    strcpy(line_buffer, line_buffer + strlen("Sec-WebSocket-Key: "));
    //1. key=KEY+GUID
    strcat(line_buffer, GUID);
    //2.sha_key = SHA1(key)
    SHA1(line_buffer, strlen(line_buffer), sha_key);
    //3. sec_key=base64_encode(sha_key)
    base64_encode(sha_key, strlen(sha_key), sec_key);
    //set head
    memset(ev->buffer, 0, BUFFER_LENGTH);
    ev->length = sprintf(ev->buffer, "HTTP/1.1 101 Switching Protocols\r\n"
                                     "Upgrade: websocket\r\n"
                                     "Connection: Upgrade\r\n"
                                     "Sec-WebSocket-Accept: %s\r\n\r\n", sec_key);
    
    printf("[handshake response]\n%s\n", ev->buffer);
    return 0;

}
```

### 3.2 传输代码介绍

  transmission函数会调用decode_packet对数据包进行解析，将有效数据长度和数据读取出来。
  之后再调用encode_packet对数据包进行封装回发回去（这里做的是echo）。

  这里就对着上面介绍的传输协议写代码即可

```
//大端
typedef struct _ws_ophdr {
    unsigned char opcode: 4,
            rsv3: 1,
            rsv2: 1,
            rsv1: 1,
            fin: 1;
    unsigned char payload_len: 7,
            mask: 1;
} ws_ophdr;

typedef struct _ws_ophdr126 {
    unsigned short payload_len;
    char mask_key[4];
} ws_ophdr126;

typedef struct _ws_ophdr127 {
    long long payload_len;
    char mask_key[4];
} ws_ophdr127;

void umask(char *payload, int length, char *mask_key) {
    int i = 0;
    for (i = 0; i < length; i++) {
        payload[i] ^= mask_key[i % 4];
    }
}

char *decode_packet(struct ntyevent *ev, int *real_len, int *virtual_len) {
    ws_ophdr *hdr = (ws_ophdr *) ev->buffer;
    printf("decode_packet fin:%d rsv1:%d rsv2:%d rsv3:%d opcode:%d mark:%d\n",
           hdr->fin,
           hdr->rsv1,
           hdr->rsv2,
           hdr->rsv3,
           hdr->opcode,
           hdr->mask);
    char *payload = NULL;
    *virtual_len = hdr->payload_len;
    if (hdr->opcode == 8) {
        ev->state_machine = WS_END;
        close(ev->fd);
        return NULL;
    }

    if (hdr->payload_len < 126) {
        payload = ev->buffer + sizeof(ws_ophdr) + 4; // 6  payload length < 126
        if (hdr->mask) {
            umask(payload, hdr->payload_len, ev->buffer + 2);
        }
        *real_len = hdr->payload_len;
    }
    else if (hdr->payload_len == 126) {
        payload = ev->buffer + sizeof(ws_ophdr) + sizeof(ws_ophdr126);
        ws_ophdr126 *hdr126 = (ws_ophdr126 *) (ev->buffer + sizeof(ws_ophdr));
        hdr126->payload_len = ntohs(hdr126->payload_len);
        if (hdr->mask) {
            umask(payload, hdr126->payload_len, hdr126->mask_key);
        }
        *real_len = hdr126->payload_len;
    }
    else if (hdr->payload_len == 127) {
        payload = ev->buffer + sizeof(ws_ophdr) + sizeof(ws_ophdr127);
        ws_ophdr127 *hdr127 = (ws_ophdr127 *) (ev->buffer + sizeof(ws_ophdr));
        if (hdr->mask) {
            umask(payload, hdr127->payload_len, hdr127->mask_key);
        }
        *real_len = hdr127->payload_len;
    }
    printf("virtual len=%d  real_len=%d\n", hdr->payload_len, *real_len);
    return payload;

}

int encode_packet(struct ntyevent *ev, int real_len, int virtual_len, char *buf) {
    ws_ophdr head = {0};
    head.fin = 1;
    head.opcode = 1;
    head.payload_len = virtual_len;
    memcpy(ev->buffer, &head, sizeof(ws_ophdr));

    int head_offset = 0;
    if (virtual_len < 126) {
        head.payload_len = real_len;
        head_offset = sizeof(ws_ophdr);
    }
    else if (virtual_len == 126) {
        ws_ophdr126 hdr126 = {0};
        hdr126.payload_len = htons(real_len);
        memcpy(ev->buffer + sizeof(ws_ophdr), &hdr126, sizeof(unsigned short));//返回不需要mask，中间去掉4B
        head_offset = sizeof(ws_ophdr) + sizeof(unsigned short);
    }
    else if (virtual_len == 127) {
        ws_ophdr127 hdr127 = {0};
        hdr127.payload_len = real_len;
        memcpy(ev->buffer + sizeof(ws_ophdr), &hdr127, sizeof(long long));//返回不需要mask，中间去掉4B
        head_offset = sizeof(ws_ophdr) + sizeof(long long);
    }
    printf("encode_packet fin:%d rsv1:%d rsv2:%d rsv3:%d opcode:%d mark:%d \n",
           head.fin,
           head.rsv1,
           head.rsv2,
           head.rsv3,
           head.opcode,
           head.mask);
    memcpy(ev->buffer + head_offset, buf, real_len);
    return head_offset + real_len;//头+payload

}

int transmission(struct ntyevent *ev) {
    char *payload_buffer = NULL;
    int real_len = 0, virtual_len;
    payload_buffer = decode_packet(ev, &real_len, &virtual_len);

    printf("real_len=[%d] , buf=[%s]\n", real_len, payload_buffer);
    
    ev->length = encode_packet(ev, real_len, virtual_len, payload_buffer);

}
```

### 3.3 程序运行测试结果

![在这里插入图片描述](https://img-blog.csdnimg.cn/c68459d94a7f4387a91a0bc4f5d7a36b.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/8d7cc635d7c94961bb03f071a16b91a1.png)

### 3.4 完整代码

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
#include <openssl/sha.h>
#include <openssl/pem.h>
#include <openssl/bio.h>
#include <openssl/evp.h>

#define BUFFER_LENGTH           4096
#define MAX_EPOLL_EVENTS        1024
#define SERVER_PORT             8081
#define PORT_COUNT              100
#define    GUID    "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

typedef int (*NCALLBACK)(int, int, void *);

struct ntyevent {
    int fd;
    int events;
    void *arg;

    NCALLBACK callback;
    
    int status;
    char buffer[BUFFER_LENGTH];
    int length;
    
    int state_machine;

};
//state_machine
enum {
    WS_HANDSHAKE = 0,
    WS_TRANSMISSION = 1,
    WS_END = 2
};
struct eventblock {
    struct eventblock *next;
    struct ntyevent *events;
};

struct ntyreactor {
    int epfd;
    int blkcnt;
    struct eventblock *evblk;
};


int recv_cb(int fd, int events, void *arg);

int send_cb(int fd, int events, void *arg);

struct ntyevent *ntyreactor_find_event_idx(struct ntyreactor *reactor, int sockfd);

void nty_event_set(struct ntyevent *ev, int fd, NCALLBACK callback, void *arg) {
    ev->fd = fd;
    ev->callback = callback;
    ev->events = 0;
    ev->arg = arg;
}

int nty_event_add(int epfd, int events, struct ntyevent *ev) {
    struct epoll_event ep_ev = {0, {0}};
    ep_ev.data.ptr = ev;
    ep_ev.events = ev->events = events;
    int op;
    if (ev->status == 1) {
        op = EPOLL_CTL_MOD;
    }
    else {
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
    out_str[bptr->length - 1] = '\0';
    size = bptr->length;
    
    BIO_free_all(bio);
    return size;

}

int readline(char *all_buffer, int idx, char *line_buffer) {
    int len = strlen(all_buffer);
    for (; idx < len; idx++) {
        if (all_buffer[idx] == '\r' && all_buffer[idx + 1] == '\n') {
            return idx + 2;
        }
        else {
            *(line_buffer++) = all_buffer[idx];
        }
    }
    return -1;
}

int handshake(struct ntyevent *ev) {
    char line_buffer[1024] = {0};
    char sha_key[32] = {0};//实际只需20B
    char sec_key[32] = {0};//实际只需28B
    int idx = 0;
    //找到Sec-WebSocket-Key这一行
    while (!strstr(line_buffer, "Sec-WebSocket-Key")) {
        memset(line_buffer, 0, 1024);
        idx = readline(ev->buffer, idx, line_buffer);
        if (idx == -1)return -1;
    }
    //1. key=KEY+GUID
    //2. sha_key=SHA1(key)
    //3. sec_key=base64_encode(sha_key)

    strcpy(line_buffer, line_buffer + strlen("Sec-WebSocket-Key: "));
    //1. key=KEY+GUID
    strcat(line_buffer, GUID);
    //2.sha_key = SHA1(key)
    SHA1(line_buffer, strlen(line_buffer), sha_key);
    //3. sec_key=base64_encode(sha_key)
    base64_encode(sha_key, strlen(sha_key), sec_key);
    //set head
    memset(ev->buffer, 0, BUFFER_LENGTH);
    ev->length = sprintf(ev->buffer, "HTTP/1.1 101 Switching Protocols\r\n"
                                     "Upgrade: websocket\r\n"
                                     "Connection: Upgrade\r\n"
                                     "Sec-WebSocket-Accept: %s\r\n\r\n", sec_key);
    
    printf("[handshake response]\n%s\n", ev->buffer);
    return 0;

}

//暂时的小端
typedef struct _ws_ophdr {
    unsigned char opcode: 4,
            rsv3: 1,
            rsv2: 1,
            rsv1: 1,
            fin: 1;
    unsigned char payload_len: 7,
            mask: 1;
} ws_ophdr;

typedef struct _ws_ophdr126 {
    unsigned short payload_len;
    char mask_key[4];
} ws_ophdr126;

typedef struct _ws_ophdr127 {
    long long payload_len;
    char mask_key[4];
} ws_ophdr127;

void umask(char *payload, int length, char *mask_key) {
    int i = 0;
    for (i = 0; i < length; i++) {
        payload[i] ^= mask_key[i % 4];
    }
}

char *decode_packet(struct ntyevent *ev, int *real_len, int *virtual_len) {
    ws_ophdr *hdr = (ws_ophdr *) ev->buffer;
    printf("decode_packet fin:%d rsv1:%d rsv2:%d rsv3:%d opcode:%d mark:%d\n",
           hdr->fin,
           hdr->rsv1,
           hdr->rsv2,
           hdr->rsv3,
           hdr->opcode,
           hdr->mask);
    char *payload = NULL;
    *virtual_len = hdr->payload_len;
    if (hdr->opcode == 8) {
        ev->state_machine = WS_END;
        close(ev->fd);
        return NULL;
    }

    if (hdr->payload_len < 126) {
        payload = ev->buffer + sizeof(ws_ophdr) + 4; // 6  payload length < 126
        if (hdr->mask) {
            umask(payload, hdr->payload_len, ev->buffer + 2);
        }
        *real_len = hdr->payload_len;
    }
    else if (hdr->payload_len == 126) {
        payload = ev->buffer + sizeof(ws_ophdr) + sizeof(ws_ophdr126);
        ws_ophdr126 *hdr126 = (ws_ophdr126 *) (ev->buffer + sizeof(ws_ophdr));
        hdr126->payload_len = ntohs(hdr126->payload_len);
        if (hdr->mask) {
            umask(payload, hdr126->payload_len, hdr126->mask_key);
        }
        *real_len = hdr126->payload_len;
    }
    else if (hdr->payload_len == 127) {
        payload = ev->buffer + sizeof(ws_ophdr) + sizeof(ws_ophdr127);
        ws_ophdr127 *hdr127 = (ws_ophdr127 *) (ev->buffer + sizeof(ws_ophdr));
        if (hdr->mask) {
            umask(payload, hdr127->payload_len, hdr127->mask_key);
        }
        *real_len = hdr127->payload_len;
    }
    printf("virtual len=%d  real_len=%d\n", hdr->payload_len, *real_len);
    return payload;

}

int encode_packet(struct ntyevent *ev, int real_len, int virtual_len, char *buf) {
    ws_ophdr head = {0};
    head.fin = 1;
    head.opcode = 1;
    head.payload_len = virtual_len;
    memcpy(ev->buffer, &head, sizeof(ws_ophdr));

    int head_offset = 0;
    if (virtual_len < 126) {
        head.payload_len = real_len;
        head_offset = sizeof(ws_ophdr);
    }
    else if (virtual_len == 126) {
        ws_ophdr126 hdr126 = {0};
        hdr126.payload_len = htons(real_len);
        memcpy(ev->buffer + sizeof(ws_ophdr), &hdr126, sizeof(unsigned short));//返回不需要mask，中间去掉4B
        head_offset = sizeof(ws_ophdr) + sizeof(unsigned short);
    }
    else if (virtual_len == 127) {
        ws_ophdr127 hdr127 = {0};
        hdr127.payload_len = real_len;
        memcpy(ev->buffer + sizeof(ws_ophdr), &hdr127, sizeof(long long));//返回不需要mask，中间去掉4B
        head_offset = sizeof(ws_ophdr) + sizeof(long long);
    }
    printf("encode_packet fin:%d rsv1:%d rsv2:%d rsv3:%d opcode:%d mark:%d \n",
           head.fin,
           head.rsv1,
           head.rsv2,
           head.rsv3,
           head.opcode,
           head.mask);
    memcpy(ev->buffer + head_offset, buf, real_len);
    return head_offset + real_len;//头+payload

}

int transmission(struct ntyevent *ev) {
    char *payload_buffer = NULL;
    int real_len = 0, virtual_len;
    payload_buffer = decode_packet(ev, &real_len, &virtual_len);

    printf("real_len=[%d] , buf=[%s]\n", real_len, payload_buffer);
    
    ev->length = encode_packet(ev, real_len, virtual_len, payload_buffer);

}

int websocket_request(struct ntyevent *ev) {
    if (ev->state_machine == WS_HANDSHAKE) {
        handshake(ev);
        ev->state_machine = WS_TRANSMISSION;
    }
    else if (ev->state_machine == WS_TRANSMISSION) {
        transmission(ev);
    }
    else {

    }

}

int recv_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    struct ntyevent *ev = ntyreactor_find_event_idx(reactor, fd);
    memset(ev->buffer, 0, BUFFER_LENGTH);
#if 0
    long len = recv(fd, ev->buffer, BUFFER_LENGTH, 0); //
#elif 1
    int len = 0;
    int n = 0;
    while (1) {
        n = recv(fd, ev->buffer + len, BUFFER_LENGTH - len, 0);
        printf("[recv data len = %d]\n", n);
        if (n != -1) {
            len += n;
        }
        else {
            break;
        }
    }
#endif
    nty_event_del(reactor->epfd, ev);
    printf("[recv buffer total len=%d]\n", len);
    printf("buffer:[%s]\n", ev->buffer);
    if (len > 0) {
        ev->length = len;
        ev->buffer[len] = '\0';

        websocket_request(ev);
    
        nty_event_set(ev, fd, send_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLOUT, ev);
    }
    else if (len == 0) {
        close(ev->fd);
    }
    else {
        close(ev->fd);
    }
    return len;

}


int send_cb(int fd, int events, void *arg) {
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    struct ntyevent *ev = ntyreactor_find_event_idx(reactor, fd);
    printf("[send buffer]\n%s\n", ev->buffer);

    int len = send(fd, ev->buffer, ev->length, 0);
    if (len > 0) {
        nty_event_del(reactor->epfd, ev);
        nty_event_set(ev, fd, recv_cb, reactor);
        nty_event_add(reactor->epfd, EPOLLIN, ev);
    }
    else {
        nty_event_del(reactor->epfd, ev);
        close(ev->fd);
    }
    return len;

}

int accept_cb(int fd, int events, void *arg) {//非阻塞
    struct ntyreactor *reactor = (struct ntyreactor *) arg;
    if (reactor == NULL) return -1;

    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);
    
    int clientfd;
    if ((clientfd = accept(fd, (struct sockaddr *) &client_addr, &len)) == -1) {
        printf("accept: %s\n", strerror(errno));
        return -1;
    }
    if ((fcntl(clientfd, F_SETFL, O_NONBLOCK)) < 0) {
        printf("%s: fcntl nonblocking failed, %d\n", __func__, MAX_EPOLL_EVENTS);
        return -1;
    }
    struct ntyevent *event = ntyreactor_find_event_idx(reactor, clientfd);
    
    nty_event_set(event, clientfd, recv_cb, reactor);
    event->status = WS_HANDSHAKE;
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

    bind(fd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    
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
    
    struct ntyevent *evs = (struct ntyevent *) malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    if (evs == NULL) {
        printf("ntyreactor_alloc ntyevents failed\n");
        return -2;
    }
    memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    
    struct eventblock *block = (struct eventblock *) malloc(sizeof(struct eventblock));
    if (block == NULL) {
        printf("ntyreactor_alloc eventblock failed\n");
        return -2;
    }
    memset(block, 0, sizeof(struct eventblock));
    
    block->events = evs;
    block->next = NULL;
    
    blk->next = block;
    reactor->blkcnt++; //
    return 0;

}

struct ntyevent *ntyreactor_find_event_idx(struct ntyreactor *reactor, int sockfd) {
    int blkidx = sockfd / MAX_EPOLL_EVENTS;

    while (blkidx >= reactor->blkcnt) {
        ntyreactor_alloc(reactor);
    }
    int i = 0;
    struct eventblock *blk = reactor->evblk;
    while (i++ < blkidx && blk != NULL) {
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
    
    struct ntyevent *evs = (struct ntyevent *) malloc((MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    if (evs == NULL) {
        printf("ntyreactor_alloc ntyevents failed\n");
        return -2;
    }
    memset(evs, 0, (MAX_EPOLL_EVENTS) * sizeof(struct ntyevent));
    
    struct eventblock *block = (struct eventblock *) malloc(sizeof(struct eventblock));
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
    //free(reactor->events);

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

    struct ntyevent *event = ntyreactor_find_event_idx(reactor, sockfd);
    
    nty_event_set(event, sockfd, acceptor, reactor);
    nty_event_add(reactor->epfd, EPOLLIN, event);
    return 0;

}


_Noreturn int ntyreactor_run(struct ntyreactor *reactor) {
    if (reactor == NULL) return -1;
    if (reactor->epfd < 0) return -1;
    if (reactor->evblk == NULL) return -1;

    struct epoll_event events[MAX_EPOLL_EVENTS + 1];
    
    int i;
    
    while (1) {
        int nready = epoll_wait(reactor->epfd, events, MAX_EPOLL_EVENTS, 1000);
        if (nready < 0) {
            printf("epoll_wait error, exit\n");
            continue;
        }
        for (i = 0; i < nready; i++) {
            struct ntyevent *ev = (struct ntyevent *) events[i].data.ptr;
            if ((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
            if ((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }

}


// <remoteip, remoteport, localip, localport,protocol>
int main(int argc, char *argv[]) {
    unsigned short port = SERVER_PORT; // listen 8081
    if (argc == 2) {
        port = atoi(argv[1]);
    }
    struct ntyreactor *reactor = (struct ntyreactor *) malloc(sizeof(struct ntyreactor));
    ntyreactor_init(reactor);
    int i = 0;
    int sockfds[PORT_COUNT] = {0};
    for (i = 0; i < PORT_COUNT; i++) {
        sockfds[i] = init_sock(port + i);
        ntyreactor_addlistener(reactor, sockfds[i], accept_cb);
    }
    ntyreactor_run(reactor);
    ntyreactor_destory(reactor);
    for (i = 0; i < PORT_COUNT; i++) {
        close(sockfds[i]);
    }
    free(reactor);
    return 0;
}
```



————————————————
版权声明：本文为CSDN博主「cheems~」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_42956653/article/details/125701357