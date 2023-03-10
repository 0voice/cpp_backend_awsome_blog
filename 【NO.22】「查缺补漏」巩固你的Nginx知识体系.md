# 【NO.22】「查缺补漏」巩固你的Nginx知识体系

## 1.**基本介绍**

Nginx是一款轻量级的 Web服务器 / 反向代理服务器 / 电子邮件（IMAP/POP3）代理服务器，主要的优点是：

1. 支持高并发连接，尤其是静态界面，官方测试Nginx能够支撑5万并发连接
2. 内存占用极低
3. 配置简单，使用灵活，可以基于自身需要增强其功能，同时支持自定义模块的开发

使用灵活：可以根据需要，配置不同的负载均衡模式，URL地址重写等功能

1. 稳定性高，在进行反向代理时，宕机的概率很低
2. 支持热部署，应用启动重载非常迅速

## 2.**基础使用**

### **2.1 安装**

文件下载地址：[http://nginx.org/en/docs/windows.html](https://link.zhihu.com/?target=http%3A//nginx.org/en/docs/windows.html)

### 2.2 **基本命令**

```
# 启动
# 建议使用第一种，第二种会使窗口一直处于执行中，不能进行其他命令操作
C:\server\nginx-1.19.2> start nginx
C:\server\nginx-1.19.2> nginx.exe

# 停止
# stop是快速停止nginx，可能并不保存相关信息；quit是完整有序的停止nginx，并保存相关信息
C:\server\nginx-1.19.2> nginx.exe -s stop
C:\server\nginx-1.19.2> nginx.exe -s quit

# 重载Nginx
# 当配置信息修改，需要重新载入这些配置时使用此命令
C:\server\nginx-1.19.2> nginx.exe -s reload

# 重新打开日志文件
C:\server\nginx-1.19.2> nginx.exe -s reopen

# 查看Nginx版本
C:\server\nginx-1.19.2> nginx -v

# 查看配置文件是否正确
C:\server\nginx-1.19.2> nginx -t
```

### **2.3 简单Demo**

1. 利用`SwitchHost`软件编辑域名和IP的映射关系，或到目录`C:\Windows\System32\drivers\etc`下，编辑`hosts`文件，增加配置如下（Mac 同理）

```
   127.0.0.1  kerwin.demo.com
```

PS：推荐使用软件`SwitchHost`，工作时几乎是必用的

1. 修改配置，如图所示：

![img](https://pic1.zhimg.com/80/v2-b4b9567750fc4770edc6a0f91130b240_720w.webp)

效果如图所示：

![img](https://pic1.zhimg.com/80/v2-a3ce920a77c6f7ca65771b24bb5a2538_720w.webp)

## **3.Nginx在架构体系中的作用**

- 网关 （面向客户的总入口）
- 虚拟主机（为不同域名 / ip / 端口提供服务。如：VPS虚拟服务器）
- 路由（正向代理 / 反向代理）
- 静态服务器
- 负载集群（提供负载均衡）

### **3.1 网关**

网关：可以简单的理解为用户请求和服务器响应的关口，即面向用户的总入口

网关可以拦截客户端所有请求，对该请求进行权限控制、负载均衡、日志管理、接口调用监控等，因此无论使用什么架构体系，都可以使用`Nginx`作为最外层的网关

### 3.2**虚拟主机**

**虚拟主机的定义**：虚拟主机是一种特殊的软硬件技术，它可以将网络上的每一台计算机分成多个虚拟主机，每个虚拟主机可以独立对外提供 www 服务，这样就可以实现一台主机对外提供多个 web 服务，每个虚拟主机之间是独立的，互不影响的。

通过 Nginx 可以实现虚拟主机的配置，Nginx 支持三种类型的虚拟主机配置

- 基于 IP 的虚拟主机
- 基于域名的虚拟主机
- 基于端口的虚拟主机

表现形式其实大家多见过，即：

```
# 每个 server 就是一个虚拟主机
http {
    # ...
    server{
        # ...
    }
    
    # ...
    server{
        # ...
    }
}
```

### 3.3**路由**

在`Nginx`的配置文件中，我们经常可以看到这样的配置：

```
location / {
	#....
}
```

`location`在此处就起到了路由的作用，比如我们在同一个虚拟主机内定义两个不同的路由，如下：

```
location / {
	proxy_pass https://www.baidu.com/;
}
		
location /api {
	proxy_pass https://apinew.juejin.im/user_api/v1/user/get?aid=2608&user_id=1275089220013336&not_self=1;
 }
```

效果如下：

![img](https://pic2.zhimg.com/80/v2-301ac72979c8f9ca972a6944ff8019c9_720w.webp)

因为路由的存在，为我们后续解决`跨域问题`提供了一定的思路，同时配置内容和API接口等更加方便

PS：路由的功能非常强大，`支持正则匹配`

### 3.4**正向与反向代理**

此处额外解释一下`proxy_pass`的含义

在`Nginx`中配置`proxy_pass`代理转发时，如果在`proxy_pass`后面的url加 `/`，表示绝对根路径；

如果没有`/`，表示相对路径

**正向代理**

1. 代理客户;
2. 隐藏真实的客户，为客户端收发请求，使真实客户端对服务器不可见;
3. 一个局域网内的所有用户可能被一台服务器做了正向代理，由该台服务器负责 HTTP 请求;
4. 意味着同服务器做通信的是正向代理服务器;

**反向代理**

1. 代理服务器;
2. 隐藏了真实的服务器，为服务器收发请求，使真实服务器对客户端不可见;
3. 负载均衡服务器，将用户的请求分发到空闲的服务器上;
4. 意味着用户和负载均衡服务器直接通信，即用户解析服务器域名时得到的是负载均衡服务器的 IP ;

**共同点**

1. 都是做为服务器和客户端的中间层
2. 都可以加强内网的安全性，阻止 web 攻击
3. 都可以做缓存机制，提高访问速度

**区别**

1. 正向代理其实是客户端的代理,反向代理则是服务器的代理。
2. 正向代理中，服务器并不知道真正的客户端到底是谁；而在反向代理中，客户端也不知道真正的服务器是谁。
3. 作用不同。正向代理主要是用来解决访问限制问题；而反向代理则是提供负载均衡、安全防护等作用。



### 3.5**静态服务器**

静态服务器是`Nginx`的强项，使用非常容易，在默认配置下本身就是指向了静态的HTML界面，如：

```
location / {
	root   html;
	index  index.html index.htm;
}
```

所以前端同学们，如果构建好了界面，可以进行相应的配置，把界面指向目标文件夹中即可，`root`指的是`html`文件夹

### 3.6**负载均衡**

负载均衡功能是`Nginx`另一大杀手锏，一共有5种方式，着重介绍一下。

### 3.7**轮询**

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除，配置如下：

```
upstream tomcatserver {
	server 192.168.0.1;
	server 192.168.0.2;
}
```

轮询策略是默认的负载均衡策略

### 3.7**指定权重**

即在轮询的基础之上，增加权重的概念，`weight`和访问比率成正比，用于后端服务器性能不均的情况，配置如下：

```
upstream tomcatserver {
	server 192.168.0.1 weight=1;
	server 192.168.0.2 weight=10;
}
```

### **3.8IP Hash**

每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题，配置如下：

```
upstream tomcatserver {
	ip_hash;
	server 192.168.0.14:88;
	server 192.168.0.15:80;
}
```

### 3.9**fair**

第三方提供的负载均衡策略，按后端服务器的响应时间来分配请求，响应时间短的优先分配，生产环境中有各种情况可能导致响应时间波动，需要慎用

```
upstream tomcatserver {
	server server1;
	server server2;
	fair;
}
```

### 3.10**url_hash**

第三方提供的负载均衡策略，按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器

```
upstream tomcatserver {
	server squid1:3128;
	server squid2:3128;
	hash $request_uri;
	hash_method crc32;
}
```

## 4.**Nginx的模块化设计**

先来看看`Nginx`模块架构图：

![img](https://pic2.zhimg.com/80/v2-f3bf04a705602a05f660ab7f6fac0149_720w.webp)

这5个模块由上到下重要性一次递减。

（1）核心模块；

核心模块是Nginx服务器正常运行必不可少的模块，如同操作系统的内核。它提供了Nginx最基本的核心服务。像进程管理、权限控制、错误日志记录等；

（2）标准HTTP模块；

标准HTTP模块支持标准的HTTP的功能，如：端口配置，网页编码设置，HTTP响应头设置等；

（3）可选HTTP模块；

可选HTTP模块主要用于扩展标准的HTTP功能，让Nginx能处理一些特殊的服务，如：解析GeoIP请求，SSL支持等；

（4）邮件服务模块；

邮件服务模块主要用于支持Nginx的邮件服务；

（5）第三方模块；

第三方模块是为了扩展Nginx服务器应用，完成开发者想要的功能，如：Lua支持，JSON支持等；

> 模块化设计使得Nginx方便开发和扩展，功能很强大

## **5.Nginx的请求处理流程**

基于上文中的`Nginx`模块化结构，我们很容易想到，在请求的处理阶段也会经历诸多的过程，`Nginx`将各功能模块组织成一条链，当有请求到达的时候，请求依次经过这条链上的部分或者全部模块，进行处理，每个模块实现特定的功能。

一个 HTTP Request 的处理过程：

- 初始化 HTTP Request
- 处理请求头、处理请求体
- 如果有的话，调用与此请求（URL 或者 Location）关联的 handler
- 依次调用各 phase handler 进行处理
- 输出内容依次经过 filter 模块处理

![img](https://pic1.zhimg.com/80/v2-bc3cd90e30dc7c1dadf5548c8c32fd7c_720w.webp)

## **6.Nginx的多进程模型**

Nginx 在启动后，会有一个 `master`进程和多个 `worker`进程。

`master`进程主要用来管理`worker`进程，包括接收来自外界的信号，向各 worker 进程发送信号，监控 worker 进程的运行状态以及启动 worker 进程。

`worker`进程是用来处理来自客户端的请求事件。多个 worker 进程之间是对等的，它们同等竞争来自客户端的请求，各进程互相独立，一个请求只能在一个 worker 进程中处理。worker 进程的个数是可以设置的，一般会设置与机器 CPU 核数一致，这里面的原因与事件处理模型有关

Nginx 的进程模型，可由下图来表示：

![img](https://pic4.zhimg.com/80/v2-1fe4b562a2abae1177cc82e4b543eb63_720w.webp)

这种设计带来以下优点：

**1） 利用多核系统的并发处理能力**

现代操作系统已经支持多核 CPU 架构，这使得多个进程可以分别占用不同的 CPU 核心来工作。Nginx 中所有的 worker 工作进程都是完全平等的。这提高了网络性能、降低了请求的时延。

**2） 负载均衡**

多个 worker 工作进程通过进程间通信来实现负载均衡，即一个请求到来时更容易被分配到负载较轻的 worker 工作进程中处理。这也在一定程度上提高了网络性能、降低了请求的时延。

**3） 管理进程会负责监控工作进程的状态，并负责管理其行为**

管理进程不会占用多少系统资源，它只是用来启动、停止、监控或使用其他行为来控制工作进程。首先，这提高了系统的可靠性，当 worker 进程出现问题时，管理进程可以启动新的工作进程来避免系统性能的下降。其次，管理进程支持 Nginx 服务运行中的程序升级、配置项修改等操作，这种设计使得动态可扩展性、动态定制性较容易实现。

## **7.Nginx如何解决惊群现象**

### **7.1什么是惊群现象？**

惊群效应（thundering herd）是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。

上文中介绍了Nginx的多进程模型，而典型的多进程模型正如文中所说，多个worker进程之间是对等的，因此当一个请求到来的时候，所有进程会同时开始竞争，最终执行的又只有一个，这样势必会造成资源的浪费。

Nginx解决该问题的思路是：**不让多个进程在同一时间监听接受连接的socket，而是让每个进程轮流监听**，这样当有连接过来的时候，就只有一个进程在监听那肯定就没有惊群的问题。

具体做法是：利用一把进程间锁，每个进程中都**尝试**获得这把锁，如果获取成功将监听socket加入wait集合中，并设置超时等待连接到来，没有获得锁的进程则将监听socket从wait集合去除。

## **8.事件驱动模型和异步非阻塞IO**

承接上文，我们知道了Nginx的多进程模型后了解到，其工作进程实际上只有几个，但为什么依然能获得如此高的并发性能，当然与其采用的事件驱动模型和异步非阻塞IO的方式来处理请求有关。

Nginx服务器响应和处理Web请求的过程，是基于事件驱动模型的，它包含事件收集器、事件发送器和事件处理器等三部分基本单元，着重关注`事件处理器`，而一般情况下事件处理器有这么几种办法：

- 事件发送器每传递过来一个请求，目标对象就创建一个新的进程
- 事件发送器每传递过来一个请求，目标对象就创建一个新的线程，来进行处理
- 事件发送器每传递过来一个请求，目标对象就将其放入一个待处理事件的列表，使用非阻塞I/O方式调用

第三种方式，在编写程序代码时，逻辑比前面两种都复杂。大多数网络服务器采用了第三种方式，逐渐形成了所谓的`事件驱动处理库`。

`事件驱动处理库`又被称为`多路IO复用方法`，最常见的包括以下三种：select模型，poll模型和epoll模型。

其中Nginx就默认使用的是`epoll`模型，同时也支持其他事件模型。

`epoll`的帮助就在于其提供了一种机制，可以让进程同时处理多个并发请求，不用关心IO调用的具体状态。IO调用完全由事件驱动模型来管理，这样一来，当某个工作进程接收到客户端的请求以后，调用IO进行处理，如果不能立即得到结果，就去处理其他的请求；而工作进程在此期间也无需等待响应，可以去处理其他事情；当IO返回时，`epoll`就会通知此工作进程；该进程得到通知后，会来继续处理未完的请求

## **9.Nginx配置的最佳实践**

在生产环境或者开发环境中Nginx一般会代理多个虚拟主机，如果把所有的配置文件写在默认的`nginx.conf`中，看起来会非常臃肿，因此建议将每一个虚拟文件单独放置一个文件夹，Nginx支持这样的配置，如下：

```
http {
	# 省略中间配置

	# 引用该目录下以 .conf 文件结尾的配置
    include /etc/nginx/conf.d/*.conf;
}
```

具体文件配置如：

```
# Demo
upstream web_pro_testin {
	server 10.42.46.70:6003 max_fails=3 fail_timeout=20s;
	ip_hash;
}
 
server {
	listen 80;
	server_name web.pro.testin.cn;
	location / {
		proxy_pass http://web_pro_testin;
		proxy_redirect off;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
    }
 
    location ~ ^/(WEB-INF)/ {
		deny all;
    }
}
```

## 10.**Nginx全量配置参数说明**

```
# 运行用户
user www-data;    

# 启动进程,通常设置成和cpu的数量相等
worker_processes  6;

# 全局错误日志定义类型，[debug | info | notice | warn | error | crit]
error_log  logs/error.log;
error_log  logs/error.log  notice;
error_log  logs/error.log  info;

# 进程pid文件
pid        /var/run/nginx.pid;

# 工作模式及连接数上限
events {
    # 仅用于linux2.6以上内核,可以大大提高nginx的性能
    use   epoll; 
    
    # 单个后台worker process进程的最大并发链接数
    worker_connections  1024;     
    
    # 客户端请求头部的缓冲区大小
    client_header_buffer_size 4k;
    
    # keepalive 超时时间
    keepalive_timeout 60;      
    
    # 告诉nginx收到一个新连接通知后接受尽可能多的连接
    # multi_accept on;            
}

#设定http服务器，利用它的反向代理功能提供负载均衡支持
http {
    # 文件扩展名与文件类型映射表义
    include       /etc/nginx/mime.types;
    
    # 默认文件类型
    default_type  application/octet-stream;
    
    # 默认编码
    charset utf-8;
    
    # 服务器名字的hash表大小
    server_names_hash_bucket_size 128;
    
    # 客户端请求头部的缓冲区大小
    client_header_buffer_size 32k;
    
    # 客户请求头缓冲大小
	large_client_header_buffers 4 64k;
	
	# 设定通过nginx上传文件的大小
    client_max_body_size 8m;
    
    # 开启目录列表访问，合适下载服务器，默认关闭。
    autoindex on;

    # sendfile 指令指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，对于普通应用，
    # 必须设为 on,如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度
    sendfile        on;
    
    # 此选项允许或禁止使用socke的TCP_CORK的选项，此选项仅在使用sendfile的时候使用
    #tcp_nopush     on;

    # 连接超时时间（单秒为秒）
    keepalive_timeout  65;
    
    
    # gzip模块设置
    gzip on;               #开启gzip压缩输出
    gzip_min_length 1k;    #最小压缩文件大小
    gzip_buffers 4 16k;    #压缩缓冲区
    gzip_http_version 1.0; #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
    gzip_comp_level 2;     #压缩等级
    gzip_types text/plain application/x-javascript text/css application/xml;
    gzip_vary on;

    # 开启限制IP连接数的时候需要使用
    #limit_zone crawler $binary_remote_addr 10m;
   
	# 指定虚拟主机的配置文件，方便管理
    include /etc/nginx/conf.d/*.conf;


    # 负载均衡配置
    upstream mysvr {
        # 请见上文中的五种配置
    }


   # 虚拟主机的配置
    server {
        
        # 监听端口
        listen 80;

        # 域名可以有多个，用空格隔开
        server_name www.jd.com jd.com;
        
        # 默认入口文件名称
        index index.html index.htm index.php;
        root /data/www/jd;

        # 图片缓存时间设置
        location ~ .*.(gif|jpg|jpeg|png|bmp|swf)${
            expires 10d;
        }
         
        #JS和CSS缓存时间设置
        location ~ .*.(js|css)?${
            expires 1h;
        }
         
        # 日志格式设定
        #$remote_addr与$http_x_forwarded_for用以记录客户端的ip地址；
        #$remote_user：用来记录客户端用户名称；
        #$time_local： 用来记录访问时间与时区；
        #$request： 用来记录请求的url与http协议；
        #$status： 用来记录请求状态；成功是200，
        #$body_bytes_sent ：记录发送给客户端文件主体内容大小；
        #$http_referer：用来记录从那个页面链接访问过来的；
        log_format access '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" $http_x_forwarded_for';
         
        # 定义本虚拟主机的访问日志
        access_log  /usr/local/nginx/logs/host.access.log  main;
        access_log  /usr/local/nginx/logs/host.access.404.log  log404;
         
        # 对具体路由进行反向代理
        location /connect-controller {
 
            proxy_pass http://127.0.0.1:88;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
             
            # 后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;

            # 允许客户端请求的最大单文件字节数
            client_max_body_size 10m;

            # 缓冲区代理缓冲用户端请求的最大字节数，
            client_body_buffer_size 128k;

            # 表示使nginx阻止HTTP应答代码为400或者更高的应答。
            proxy_intercept_errors on;

            # nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_connect_timeout 90;

            # 后端服务器数据回传时间_就是在规定时间之内后端服务器必须传完所有的数据
            proxy_send_timeout 90;

            # 连接成功后，后端服务器响应的超时时间
            proxy_read_timeout 90;

            # 设置代理服务器（nginx）保存用户头信息的缓冲区大小
            proxy_buffer_size 4k;

            # 设置用于读取应答的缓冲区数目和大小，默认情况也为分页大小，根据操作系统的不同可能是4k或者8k
            proxy_buffers 4 32k;

            # 高负荷下缓冲大小（proxy_buffers*2）
            proxy_busy_buffers_size 64k;

            # 设置在写入proxy_temp_path时数据的大小，预防一个工作进程在传递文件时阻塞太长
            # 设定缓存文件夹大小，大于这个值，将从upstream服务器传
            proxy_temp_file_write_size 64k;
        }
        
        # 动静分离反向代理配置（多路由指向不同的服务端或界面）
        location ~ .(jsp|jspx|do)?$ {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8080;
        }
    }
}
```

## 11.**Nginx还能做什么**

### 11.1**解决CORS跨域问题**

思路有两个：

- 基于多路由，把跨域的两个请求发到各自的服务器，然后统一访问入口即可避免该问题
- 利用Nginx配置Headerd的功能，为其附上相应的请求头

### 11.2**适配 PC 或移动设备**

根据用户设备不同返回不同样式的站点，以前经常使用的是纯前端的自适应布局，但无论是复杂性和易用性上面还是不如分开编写的好，比如我们常见的淘宝、京东......这些大型网站就都没有采用自适应，而是用分开制作的方式，根据用户请求的 `user-agent` 来判断是返回 PC 还是 H5 站点

### 11.3**请求限流**

Nginx按请求速率限速模块使用的是漏桶算法，即能够强行保证请求的实时处理速度不会超过设置的阈值，如：

```
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
    server {
        location /search/ {
            limit_req zone=one burst=5 nodelay;
        }
    }
}
```

### 11.4**其他技巧**

如：图片防盗链，请求过滤，泛域名转发，配置HTTPS等等

## **12.常见问题**

### **Q1：Nginx一般用作什么？**

> 见上文中Nginx在架构体系中的作用，配合Nginx还能做什么作答即可

### **Q2：为什么要用Nginx？**

> 理解网关的必要性，以及Nginx保证高可用，负载均衡的能力

### **Q3：为什么Nginx这么快？**

如果一个server采用一个进程负责一个request的方式，那么进程数就是并发数。那么显而易见的，就是会有很多进程在等待中。等什么？最多的应该是等待网络传输。

而nginx 的异步非阻塞工作方式正是利用了这点等待的时间。在需要等待的时候，这些进程就空闲出来待命了。因此表现为少数几个进程就解决了大量的并发问题。

nginx是如何利用的呢，简单来说：同样的4个进程，如果采用一个进程负责一个request的方式，那么，同时进来4个request之后，每个进程就负责其中一个，直至会话关闭。期间，如果有第5个request进来了。就无法及时反应了，因为4个进程都没干完活呢，因此，一般有个调度进程，每当新进来了一个request，就新开个进程来处理。

nginx不这样，每进来一个request，会有一个worker进程去处理。但不是全程的处理，处理到什么程度呢？处理到可能发生阻塞的地方，比如向上游（后端）服务器转发request，并等待请求返回。那么，这个处理的worker不会这么傻等着，他会在发送完请求后，注册一个事件：“如果upstream返回了，告诉我一声，我再接着干”。于是他就休息去了。此时，如果再有request 进来，他就可以很快再按这种方式处理。而一旦上游服务器返回了，就会触发这个事件，worker才会来接手，这个request才会接着往下走。

由于web server的工作性质决定了每个request的大部份生命都是在网络传输中，实际上花费在server机器上的时间片不多。这是几个进程就解决高并发的秘密所在。

> 总结：事件模型，异步非阻塞，多进程模型加上细节优化的共同作用

### **Q4：什么是正向代理和反向代理**

> 见上文

### **Q5：Nginx负载均衡的算法有哪些？**

> 见上文

### **Q6：Nginx如何解决的惊群现象？**

> 见上文

### **Q7：Nginx为什么不用多线程模型？**

> 深入理解多进程模型加上异步非阻塞IO的好处以及多线程模型中上下文切换的劣势

### **Q8：Nginx压缩功能有什么坏处吗？**

> 非常耗费服务器的CPU

### **Q9：Nginx有几种进程模型?**

> 实际上有两种，多进程和单进程，但是实际工作中都是多进程的

原文链接：https://zhuanlan.zhihu.com/p/582188212