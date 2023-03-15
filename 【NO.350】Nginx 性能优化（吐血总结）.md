# 【NO.350】Nginx 性能优化（吐血总结）

## 1.性能优化考虑点

当我需要进行性能优化时，说明我们服务器无法满足日益增长的业务。性能优化是一个比较大的课题，需要从以下几个方面进行探讨

- 当前系统结构瓶颈
- 了解业务模式
- 性能与安全

### 1.1 当前系统结构瓶颈

首先需要了解的是当前系统瓶颈，用的是什么，跑的是什么业务。里面的服务是什么样子，每个服务最大支持多少并发。比如针对Nginx而言，我们处理静态资源效率最高的瓶颈是多大？

可以通过查看当前cpu负荷，内存使用率，进程使用率来做简单判断。还可以通过操作系统的一些工具来判断当前系统性能瓶颈，如分析对应的日志，查看请求数量。

也可以通过nginx http_stub_status_module模块来查看对应的连接数，总握手次数，总请求数。也可以对线上进行压力测试，来了解当前的系统的性能，并发数，做好性能评估。

### 1.2 了解业务模式

虽然我们是在做性能优化，但还是要熟悉业务，最终目的都是为业务服务的。我们要了解每一个接口业务类型是什么样的业务，比如电子商务抢购模式，这种情况平时流量会很小，但是到了抢购时间，流量一下子就会猛涨。也要了解系统层级结构，每一层在中间层做的是代理还是动静分离，还是后台进行直接服务。需要我们对业务接入层和系统层次要有一个梳理

### 1.3 性能与安全

性能与安全也是一个需要考虑的因素，往往大家注重性能忽略安全或注重安全又忽略性能。比如说我们在设计防火墙时，如果规则过于全面肯定会对性能方面有影响。如果对性能过于注重在安全方面肯定会留下很大隐患。所以大家要评估好两者的关系，把握好两者的孰重孰轻，以及整体的相关性。权衡好对应的点。

## 2.系统与Nginx性能优化

大家对相关的系统瓶颈及现状有了一定的了解之后，就可以根据影响性能方面做一个全体的评估和优化。

- 网络（网络流量、是否有丢包，网络的稳定性都会影响用户请求）
- 系统（系统负载、饱和、内存使用率、系统的稳定性、硬件磁盘是否有损坏）
- 服务（连接优化、内核性能优化、http服务请求优化都可以在nginx中根据业务来进行设置）
- 程序（接口性能、处理请求速度、每个程序的执行效率）
- 数据库、底层服务

上面列举出来每一级都会有关联，也会影响整体性能，这里主要关注的是Nginx服务这一层。

### 2.1 文件句柄

linux/Unix上，一切皆文件，每一次用户发起请求就会生成一个文件句柄，文件句柄可以理解为就是一个索引，所以文件句柄就会随着请求量的增多，而进程调用的频率增加，文件句柄的产生就越多，系统对文件句柄默认的限制是1024个，对Nginx来说非常小了，需要改大一点

**（1）设置方式**

- 系统全局性修改
- 用户局部性修改
- 进程局部性修改

（2）系统全局性修改和用户局部性修改

```text
vim /etc/security/limits.conf
```

在End of file前面添加4个参数

![img](https://pic2.zhimg.com/80/v2-a40ab09229527a47632b52f105b6c8c9_720w.webp)

- soft：软控制，到达设定值后，操作系统不会采取措施，只是发提醒
- hard：硬控制，到达设定值后，操作系统会采取机制对当前进程进行限制，这个时候请求就会受到影响
- root：这里代表root用户（系统全局性修改）
- *：代表全局，即所有用户都受此限制（用户局部性修改）
- nofile：指限制的是文件数的配置项。后面的数字即设定的值，一般设置10000左右

尤其在企业新装的系统，这个地方应该根据实际情况进行设置，可以设置全局的，也可以设置用户级别的

**（3）进程局部性修改**

```text
vim /etc/nginx/nginx.conf
```

每个进程的最大文件打开数，所以最好与ulimit -n的值保持一致。

```text
worker_rlimit_nofile 35535; #进程限制
```

![img](https://pic3.zhimg.com/80/v2-797db8e4186ca36d83d0ccb0cb96229e_720w.webp)

### 2.2 cpu的亲和配置

cpu的亲和能够使nginx对于不同的work工作进程绑定到不同的cpu上面去。就能够减少在work间不断切换cpu，把进程通常不会在处理器之间频繁迁移，进程迁移的频率小，来减少性能损耗。

![img](https://pic3.zhimg.com/80/v2-d46fcec7e3c0eb31d8ca80ab0b732e7e_720w.webp)

**（1）具体设置**

Nginx运行工作进程个数一般设置CPU的核心或者核心数x2。如果不了解cpu的核数，可以top命令之后按1看出来，也可以查看/proc/cpuinfo文件 grep ^processor /proc/cpuinfo | wc -l。

```text
[root@lx~]# vi/usr/local/nginx1.10/conf/nginx.conf
worker_processes 4;
[root@lx~]# /usr/local/nginx1.10/sbin/nginx-s reload
[root@lx~]# ps -aux | grep nginx |grep -v grep
root 9834 0.0 0.0 47556 1948 ?     Ss 22:36 0:00 nginx: master processnginx
www 10135 0.0 0.0 50088 2004 ?       S   22:58 0:00 nginx: worker process
www 10136 0.0 0.0 50088 2004 ?       S   22:58 0:00 nginx: worker process
www 10137 0.0 0.0 50088 2004 ?       S   22:58 0:00 nginx: worker process
www 10138 0.0 0.0 50088 2004 ?       S   22:58 0:00 nginx: worker process
```

比如4核配置：

```text
worker_processes 4;
worker_cpu_affinity 0001 0010 0100 1000
```

比如8核配置：

```text
worker_processes 8;
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
```

worker_processes最多开启8个，8个以上性能提升不会再提升了，而且稳定性变得更低，所以8个进程够用了。

为每个进程分配cpu，上例中将8个进程分配到8个cpu，当然可以写多个，或者将一个进程分配到多个cpu。

在nginx 1.9版本之后，就帮我们自动绑定了cpu;

```text
worker_cpu_affinity auto;
```

**（2）相关命令**

查看cpu核心数

```text
cat /proc/cpuinfo|grep "cpu cores"|uniq
```

显示物理cpu数量：

```text
cat /proc/cpuinfo | grep "physical id"|sort|uniq|wc -l
```

查看cpu使用率

```text
top  回车后按 1
```

查看nginx使用cpu核心和对应的nginx进程号

```text
ps -eo pid,args,psr | grep [n]ginx
```

### 2.3 事件处理模型优化

nginx的连接处理机制在于不同的操作系统会采用不同的I/O模型，Linux下，nginx使用epoll的I/O多路复用模型，在freebsd使用kqueue的IO多路复用模型，在solaris使用/dev/pool方式的IO多路复用模型，在windows使用的icop等等。要根据系统类型不同选择不同的事务处理模型，我们使用的是Centos，因此将nginx的事件处理模型调整为epoll模型。

```text
events {
    worker_connections  10240;    //
    use epoll;
}
```

说明：在不指定事件处理模型时，nginx默认会自动的选择最佳的事件处理模型服务。

### 2.4 设置work_connections 连接数

```text
 worker_connections  10240;
```

### 2.5 keepalive timeout会话保持时间

```text
keepalive_timeout  60;
```

### 2.6 GZIP压缩性能优化

```text
gzip on;       #表示开启压缩功能
gzip_min_length  1k; #表示允许压缩的页面最小字节数，页面字节数从header头的Content-Length中获取。默认值是0，表示不管页面多大都进行压缩，建议设置成大于1K。如果小于1K可能会越压越大
gzip_buffers     4 32k; #压缩缓存区大小
gzip_http_version 1.1; #压缩版本
gzip_comp_level 6; #压缩比率， 一般选择4-6，为了性能gzip_types text/css text/xml application/javascript;　　#指定压缩的类型 gzip_vary on;　#vary header支持
```

### 2.7 proxy超时设置

```text
proxy_connect_timeout 90;
proxy_send_timeout  90;
proxy_read_timeout  4k;
proxy_buffers 4 32k;
proxy_busy_buffers_size 64k
```

### 2.8 高效传输模式

```text
sendfile on; # 开启高效文件传输模式。
tcp_nopush on; #需要在sendfile开启模式才有效，防止网路阻塞，积极的减少网络报文段的数量。将响应头和正文的开始部分一起发送，而不一个接一个的发送。
```

### 2.9 Linux系统内核层面

Nginx要达到最好的性能，出了要优化Nginx服务本身之外，还需要在nginx的服务器上的内核参数。

这些参数追加到/etc/sysctl.conf,然后执行sysctl -p 生效。

1）调节系统同时发起的tcp连接数

```text
net.core.somaxconn = 262144
```

2）允许等待中的监听

```text
net.core.somaxconn = 4096 
```

3） tcp连接重用

```text
net.ipv4.tcp_tw_recycle = 1 
net.ipv4.tcp_tw_reuse = 1   
```

4）不抵御洪水攻击

```text
net.ipv4.tcp_syncookies = 0  
net.ipv4.tcp_max_orphans = 262144  #该参数用于设定系统中最多允许存在多少TCP套接字不被关联到任何一个用户文件句柄上，主要目的为防止Ddos攻击
```

5）最大文件打开数

在命令行中输入如下命令，即可设置Linux最大文件打开数。

```text
ulimit -n 30000
```

以上，就把Nginx服务器高性能优化的配置介绍完了，大家可以根据我提供的方法，每个参数挨个设置一遍，看看相关的效果。这些都是一点点试出来的，这样才能更好的理解各个参数的意义。

## 3.nginx通用配置优化

```text
#将nginx进程设置为普通用户，为了安全考虑
user nginx; 

#当前启动的worker进程，官方建议是与系统核心数一致
worker_processes 2;
#方式一，就是自动分配绑定
worker_cpu_affinity auto;

#日志配置成warn
error_log /var/log/nginx/error.log warn; 
pid /var/run/nginx.pid;

#针对 nginx 句柄的文件限制
worker_rlimit_nofile 35535;
#事件模型
events {
    #使用epoll内核模型
    use epoll;
    #每一个进程可以处理多少个连接，如果是多核可以将连接数调高 worker_processes * 1024
    worker_connections 10240;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    charset utf-8;  #设置字符集

    #设置日志输出格式，根据自己的情况设置
    log_format  main  '$http_user_agent' '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      '"$args" "$request_uri"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;   #对静态资源的处理比较有效
    #tcp_nopush     on;   #如果做静态资源服务器可以打开

    keepalive_timeout  65; 

    ########
    #Gzip module
    gzip  on;    #文件压缩默认可以打开

    include /etc/nginx/conf.d/*.conf;
}
```

## 4.实战配置

1、整体配置

```text
worker_processes  1;
pid  /var/run/nginx.pid;

events {
    worker_connections  2048;
	multi_accept on;
	use epoll;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
					  
	log_format main '{"@timestamp":"$time_iso8601",'   
	'"host":"$server_addr",'
	'"clientip":"$remote_addr",'
	'"size":$body_bytes_sent,'
	'"responsetime":$request_time,'
	'"upstreamtime":"$upstream_response_time",'
	'"upstreamhost":"$upstream_addr",'
	'"http_host":"$host",'
	'"url":"$uri",'
	'"xff":"$http_x_forwarded_for",'
	'"referer":"$http_referer",'
	'"agent":"$http_user_agent",'
	'"status":"$status"}';
	
	sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
	
    server_names_hash_bucket_size 128;
    server_names_hash_max_size 512;
    keepalive_timeout  65;
    client_header_timeout 15s;
    client_body_timeout 15s;
    send_timeout 60s;
	
	limit_conn_zone $binary_remote_addr zone=perip:10m;
	limit_conn_zone $server_name zone=perserver:10m;
	limit_conn perip 2;
	limit_conn perserver 20;
	limit_rate 300k; 

    proxy_cache_path /data/nginx-cache levels=1:2 keys_zone=nginx-cache:20m max_size=50g inactive=168h;
	
	client_body_buffer_size 512k;
	client_header_buffer_size 4k;
	client_max_body_size 512k;
	large_client_header_buffers 2 8k;
	proxy_connect_timeout 5s;
	proxy_send_timeout 120s;
	proxy_read_timeout 120s;
	proxy_buffer_size 16k;
	proxy_buffers 4 64k;
	proxy_busy_buffers_size 128k;
	proxy_temp_file_write_size 128k;
	proxy_next_upstream http_502 http_504 http_404 error timeout invalid_header;
	
	gzip on;
	gzip_min_length 1k;
	gzip_buffers 4 16k;
	gzip_http_version 1.1;
	gzip_comp_level 4;
	gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
	gzip_vary on;
	gzip_disable "MSIE [1-6].";

    include /etc/nginx/conf.d/*.conf;
}
```

2、负载均衡

```text
upstream ygoapi{ 
  server 0.0.0.0:8082 fail_timeout=5 max_fails=3;
  server 0.0.0.0:8083 fail_timeout=5 max_fails=3;
  ip_hash;
}
```

3、HTTP 配置

```text
#隐藏版本信息
server_tokens off;
server {
    listen       80;
    server_name  素材管理平台;
    charset utf-8;
	
	#重定向HTTP请求到HTTPS
	return 301 https://$server_name$request_uri;
}
```

4、HTTPS 配置

```text
准备条件：需要先去下载 HTTPS 证书
server {
    listen 443;
    server_name 素材管理平台;
    ssl on;
    ssl_certificate cert/证书名称.pem;
    ssl_certificate_key cert/证书名称.key;
    ssl_session_timeout 5m;
    # SSL协议配置
    ssl_protocols SSLv2 SSLv3 TLSv1.2;
    ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
    ssl_prefer_server_ciphers on;
	
	valid_referers none blocked server_names
               *.ygoclub.com;
			   
	#日志配置
    access_log  /Users/jackson/Desktop/www.ygoclub.com-access.log  main gzip=4 flush=5m;
    error_log  /Users/jackson/Desktop/www.ygoclub.com-error.log  error;
	
	location ~ .*\.(eot|svg|ttf|woff|jpg|jpeg|gif|png|ico|cur|gz|svgz|mp4|ogg|ogv|webm) {
		proxy_cache nginx-cache;
		proxy_cache_valid 200 304 302 5d;
		proxy_cache_key '$host:$server_port$request_uri';
		add_header X-Cache '$upstream_cache_status from $host';
		#所有静态文件直接读取硬盘
		root /usr/share/nginx/html;
		expires 30d; #缓存30天
	}

	location ~ .*\.(js|css)?$
	{
		proxy_cache nginx-cache;
		proxy_cache_valid 200 304 302 5d;
		proxy_cache_key '$host:$server_port$request_uri';
		add_header X-Cache '$upstream_cache_status from $host';
		#所有静态文件直接读取硬盘
		root /usr/share/nginx/html;
		expires      12h;
	}

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    
	location /druid {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_cache nginx-cache;
        proxy_cache_valid 200 10m;
        proxy_pass http://ygoapi/druid;
	}
	
	location /api {
       proxy_set_header X-Real-IP $remote_addr;
       proxy_pass http://ygoapi/api;
	}
	
    # redirect server error pages to the static page /50x.html
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

## 5.ab接口压力测试工具

ab是Apache超文本传输协议(HTTP)的性能测试工具。其设计意图是描绘当前所安装的Apache的执行性能，主要是显示你安装的Apache每秒可以处理多少个请求。

```text
yum install httpd-tools -y
ab -n 2000 -c 2 http://127.0.0.1/
```

- -n ：总的请求数
- -c ：并发数
- -k 是否开启长连接

![img](https://pic4.zhimg.com/80/v2-5ac2b2c212ce5885270e8f3b35650a23_720w.webp)

1、参数选项

（1）完整测试报告

这段展示的是web服务器的信息，可以看到服务器采用的是nginx，[域名是wan.bigertech.com](https://link.zhihu.com/?target=http%3A//%E5%9F%9F%E5%90%8D%E6%98%AFwan.bigertech.com)，端口是80

![img](https://pic4.zhimg.com/80/v2-d42dd87fc0fcd4207628f4e0fabe5fdb_720w.webp)

（2）服务器信息

这段是关于请求的文档的相关信息，所在位置“/”，文档的大小为338436 bytes（此为http响应的正文长度）

![img](https://pic4.zhimg.com/80/v2-35e864681af15b78a40f0085d03fede3_720w.webp)

（3）文档信息

这段展示了压力测试的几个重要指标

![img](https://pic4.zhimg.com/80/v2-bffc316bf636dfd25604dad9a1cbf203_720w.webp)

![img](https://pic1.zhimg.com/80/v2-a1b516e0cb4306f476536352593a0fd8_720w.webp)

（4）这段表示网络上消耗的时间的分解

![img](https://pic4.zhimg.com/80/v2-bc53c081e005c0bdb3f00918cbe05d0b_720w.webp)

（5）网络消耗时间

这段是每个请求处理时间的分布情况，50%的处理时间在4930ms内，66%的处理时间在5008ms内…，重要的是看90%的处理时间。

![img](https://pic2.zhimg.com/80/v2-1746332fd0e44f689b361e50681c74b1_720w.webp)

响应情况

原文地址：https://zhuanlan.zhihu.com/p/456376971

作者：linux 