# 【NO.203】C++后端程序员必须彻底搞懂Nginx，从原理到实战详解

本文首先介绍 Nginx 的反向代理、负载均衡、动静分离和高可用的原理，随后详解 Nginx 的配置文件，最后通过实际案例实现 Nginx 反向代理和负载均衡的具体配置。学会 Nginx ，一篇足够了。

## 1. 简介

**Nginx** 是开源的轻量级 Web 服务器、反向代理服务器，以及负载均衡器和 HTTP 缓存器。其特点是高并发，高性能和低内存。
**Nginx** 专为性能优化而开发，性能是其最重要的考量，实现上非常注重效率，能经受高负载的考验，最大能支持 50000 个并发连接数。 Nginx 还支持热部署，它的使用特别容易，几乎可以做到 7x24 小时不间断运行。 Nginx 的网站用户有：百度、淘宝、京东、腾讯、新浪、网易等。

## 2. 反向代理

### 2.1 正向代理

**Nginx** 不仅可以做反向代理，实现负载均衡，还能用做正向代理来进行上网等功能。

![img](https://pic4.zhimg.com/80/v2-587440070f083e4c6f73c3491ce9b54f_720w.webp)

### 2.2 反向代理

客户端对代理服务器是无感知的，客户端不需要做任何配置，用户只请求反向代理服务器，反向代理服务器选择目标服务器，获取数据后再返回给客户端。反向代理服务器和目标服务器对外而言就是一个服务器，只是暴露的是代理服务器地址，而隐藏了真实服务器的IP地址。

![img](https://pic2.zhimg.com/80/v2-e97ffd9d3bc482f891e87266a5bf5ab1_720w.webp)

## 3. 负载均衡

将原先请求集中到单个服务器上的情况改为增加服务器的数量，然后将请求分发到各个服务器上，将负载分发到不同的服务器，即负载均衡。

![img](https://pic3.zhimg.com/80/v2-0a2403d192f45dfc60cc1e36a6cb03fe_720w.webp)

## 4. 动静分离

为了加快网站的解析速度，可以把静态页面和动态页面由不同的服务器来解析，加快解析速度，降低原来单个服务器的压力。

![img](https://pic1.zhimg.com/80/v2-863bdaf857975baee5c42fb31adc2fc0_720w.webp)

## 5. 高可用

为了提高系统的可用性和容错能力，可以增加nginx服务器的数量，当主服务器发生故障或宕机，备份服务器可以立即充当主服务器进行不间断工作。

![img](https://pic2.zhimg.com/80/v2-6ee22d09fe07287addc4611e96c8f659_720w.webp)

## 6. Nginx配置文件

### 6.1 文件结构

Nginx 配置文件由三部分组成。

```
...              #全局块
events {         #events块
   ...
}
http      #http块
{
    ...   #http全局块
    server        #server块
    { 
        ...       #server全局块
        location [PATTERN]   #location块
        {
            ...
        }
        location [PATTERN] 
        {
            ...
        }
    }
    server
    {
      ...
    }
    ...     #http全局块
}
```

- **第一部分 全局块**
  主要设置一些影响 nginx 服务器整体运行的配置指令。
  比如： worker_processes 1; ， worker_processes 值越大，可以支持的并发处理量就越多。
- **第二部分 events块**
  events 块涉及的指令主要影响Nginx服务器与用户的网络连接。
  比如： worker_connections 1024; ，支持的最大连接数。
- **第三部分 http块**
  http 块又包括 http 全局块和 server 块，是服务器配置中最频繁的部分，包括配置代理、缓存、日志定义等绝大多数功能。 **server块**：配置虚拟主机的相关参数。 **location块**：配置请求路由，以及各种页面的处理情况。

### 6.2 配置文件

```
########### 每个指令必须有分号结束。#################
#user administrator administrators;  #配置用户或者组，默认为nobody nobody。
#worker_processes 2;  #允许生成的进程数，默认为1
#pid /nginx/pid/nginx.pid;   #指定nginx进程运行文件存放地址
error_log log/error.log debug;  #制定日志路径，级别。这个设置可以放入全局块，http块，server块，级别以此为：debug|info|notice|warn|error|crit|alert|emerg
events {
    accept_mutex on;   #设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;  #设置一个进程是否同时接受多个网络连接，默认为off
    #use epoll;      #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
    worker_connections  1024;    #最大连接数，默认为512
}
http {
    include       mime.types;   #文件扩展名与文件类型映射表
    default_type  application/octet-stream; #默认文件类型，默认为text/plain
    #access_log off; #取消服务日志    
    log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
    access_log log/access.log myFormat;  #combined为日志格式的默认值
    sendfile on;   #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。
    sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    keepalive_timeout 65;  #连接超时时间，默认为75s，可以在http，server，location块。
    upstream mysvr {   
      server 127.0.0.1:7878;
      server 192.168.10.121:3333 backup;  #热备
    }
    error_page 404 https://www.baidu.com; #错误页
    server {
        keepalive_requests 120; #单连接请求上限次数。
        listen       4545;   #监听端口
        server_name  127.0.0.1;   #监听地址       
        location  ~*^.+$ {       #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
           #root path;  #根目录
           #index vv.txt;  #设置默认页
           proxy_pass  http://mysvr;  #请求转向mysvr 定义的服务器列表
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip           
        } 
    }
}   
```

## 7.配置实例

### 7.1 反向代理

#### 7.1.1 实战一

**实现效果：**
在浏览器输入 *[http://www.abc.com](https://link.zhihu.com/?target=http%3A//www.abc.com)* , 从 nginx 服务器跳转到 linux 系统 tomcat 主页面。
**具体配置：**

```
server {
        listen       80;   
        server_name  192.168.4.32;   #监听地址
        location  / {       
           root html;  #/html目录
           proxy_pass http://127.0.0.1:8080;  #请求转向
           index  index.html index.htm;      #设置默认页       
        } 
    }
```

#### 7.1.2 实战二

**实现效果：**
根据在浏览器输入的路径不同，跳转到不同端口的服务中。
**具体配置：**

```
server {
        listen       9000;   
        server_name  192.168.4.32;   #监听地址       
        location  ~ /example1/ {  
           proxy_pass http://127.0.0.1:5000;         
        } 
        location  ~ /example2/ {  
           proxy_pass http://127.0.0.1:8080;         
        } 
    }
```

**location** 指令说明：

- **~ :** 表示uri包含正则表达式，且区分大小写。
- **~\* :** 表示uri包含正则表达式，且不区分大小写。
- **= :** 表示uri不含正则表达式，要求严格匹配。

### 7.2 负载均衡

#### 7.2.1 实战一

**实现效果：**
在浏览器地址栏输入 *[http://192.168.4.32/example/a.html](https://link.zhihu.com/?target=http%3A//192.168.4.32/example/a.html)* ，平均到 5000 和 8080 端口中，实现负载均衡效果。
**具体配置：**

```
upstream myserver {         server 192.167.4.32:5000;      server 192.168.4.32:8080;    }    server {        listen       80;   #监听端口        server_name  192.168.4.32;   #监听地址        location  / {                  root html;  #html目录           index index.html index.htm;  #设置默认页           proxy_pass  http://myserver;  #请求转向 myserver 定义的服务器列表              }     }复制代码
```

**nginx 分配服务器策略**

- **轮询**（默认）
  按请求的时间顺序依次逐一分配，如果服务器down掉，能自动剔除。
- **权重**
  weight 越高，被分配的客户端越多，默认为 1。比如： upstream myserver { server 192.167.4.32:5000 weight=10; server 192.168.4.32:8080 weight=5; } 复制代码
- **ip**
  按请求 ip 的 hash 值分配，每个访客固定访问一个后端服务器。比如： upstream myserver { ip_hash; server 192.167.4.32:5000; server 192.168.4.32:8080; } 复制代码
- **fair**
  按后端服务器的响应时间来分配，响应时间短的优先分配到请求。比如： upstream myserver { fair; server 192.167.4.32:5000; server 192.168.4.32:8080; } 复制代码

恭喜！目前为止你已经掌握了 Nginx 的基本原理，并且能够配置反向代理和负载均衡。

原文链接：https://zhuanlan.zhihu.com/p/354598764

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)