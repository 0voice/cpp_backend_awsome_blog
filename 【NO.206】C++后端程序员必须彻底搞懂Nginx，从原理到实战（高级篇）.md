# 【NO.206】C++后端程序员必须彻底搞懂Nginx，从原理到实战（高级篇）

本文为 Nginx 实操高级篇。通过配置 Nginx 配置文件，实现正向代理、反向代理、负载均衡、Nginx 缓存、动静分离和高可用 Nginx 6种功能，并对 Nginx 的原理作进一步的解析。当需要使用 Nginx 配置文件时，参考本文实例即可，建议收藏。

## 1. 正向代理

正向代理的代理对象是客户端。正向代理就是代理服务器替客户端去访问目标服务器。

### 1.1. 实战一

**实现效果：**
在浏览器输入 *[http://www.google.com](https://link.zhihu.com/?target=http%3A//www.google.com)* , 浏览器跳转到*[www.google.com](http://www.google.com/)* 。
**具体配置：**

```
server{
    resolver 8.8.8.8;
    listen 80;
    location / {
        proxy_pass http://$http_host$request_uri;
    }
}
```

在需要访问外网的客户端上执行以下一种操作即可：

```
1. 方法1（推荐）
export http_proxy=http://你的正向代理服务器地址：代理端口   
2. 方法2
vim ~/.bashrc
export http_proxy=http://你的正向代理服务器地址：代理端口
```

## 2. 反向代理

反向代理指代理后端服务器响应客户端请求的一个中介服务器，代理的对象是服务端。

### 2.1 .实战一

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

### 2.2. 实战二

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

## 3 负载均衡

### 3.1. 实战一

**实现效果：**
在浏览器地址栏输入 *[http://192.168.4.32/example/a.html](https://link.zhihu.com/?target=http%3A//192.168.4.32/example/a.html)* ，平均到 5000 和 8080 端口中，实现负载均衡效果。
**具体配置：**

```
upstream myserver {   
      server 192.167.4.32:5000;
      server 192.168.4.32:8080;
    }
    server {
        listen       80;   #监听端口
        server_name  192.168.4.32;   #监听地址
        location  / {       
           root html;  #html目录
           index index.html index.htm;  #设置默认页
           proxy_pass  http://myserver;  #请求转向 myserver 定义的服务器列表      
        } 
    }
```

**nginx 分配服务器策略**

- **轮询**（默认）
  按请求的时间顺序依次逐一分配，如果服务器down掉，能自动剔除。
- **权重**
  weight 越高，被分配的客户端越多，默认为 1。比如： upstream myserver { server 192.167.4.32:5000 weight=10; server 192.168.4.32:8080 weight=5; } 复制代码
- **ip**
  按请求 ip 的 hash 值分配，每个访客固定访问一个后端服务器。比如： upstream myserver { ip_hash; server 192.167.4.32:5000; server 192.168.4.32:8080; } 复制代码
- **fair**
  按后端服务器的响应时间来分配，响应时间短的优先分配到请求。比如： upstream myserver { fair; server 192.168.4.32:5000; server 192.168.4.32:8080; } 复制代码

## 4. Nginx 缓存

### 4.1. 实战一

**实现效果：**
在3天内，通过浏览器地址栏访问 *[http://192.168.4.32/a.jpg](https://link.zhihu.com/?target=http%3A//192.168.4.32/a.jpg)* ，不会从服务器抓取资源，3天后（过期）则从服务器重新下载。
**具体配置：**

```
# http 区域下添加缓存区配置
proxy_cache_path /tmp/nginx_proxy_cache levels=1 keys_zone=cache_one:512m inactive=60s max_size=1000m;
# server 区域下添加缓存配置
location ~ \.(gif|jpg|png|htm|html|css|js)(.*) {
     proxy_pass http://192.168.4.32:5000；#如果没有缓存则转向请求
     proxy_redirect off;
     proxy_cache cache_one;
     proxy_cache_valid 200 1h;            #对不同的 HTTP 状态码设置不同的缓存时间
     proxy_cache_valid 500 1d;
     proxy_cache_valid any 1m;
     expires 3d;
}
```

**expires** 是给一个资源设定一个过期时间，通过 expires 参数设置，可以使浏览器缓存过期时间之前的内容，减少与服务器之间的请求和流量。也就是说无需去服务端验证，直接通过浏览器自身确认是否过期即可，所以不会产生额外的流量。此种方法非常适合不经常变动的资源。

## 5. 动静分离

### 5.1. 实战一

**实现效果：**
通过浏览器地址栏访问 *[http://www.abc.com/a.html](https://link.zhihu.com/?target=http%3A//www.abc.com/a.html)* ，访问静态资源服务器的静态资源内容。通过浏览器地址栏访问 *[http://www.abc.com/a.jsp](https://link.zhihu.com/?target=http%3A//www.abc.com/a.jsp)* ，访问动态资源服务器的动态资源内容。
**具体配置：**

```
upstream static {   
    server 192.167.4.31:80;
}
upstream dynamic {   
    server 192.167.4.32:8080;
}
server {
    listen       80;   #监听端口
    server_name  www.abc.com; 监听地址
    # 拦截动态资源
    location ~ .*\.(php|jsp)$ {
       proxy_pass http://dynamic;
    }
    # 拦截静态资源
    location ~ .*\.(jpg|png|htm|html|css|js)$ {       
       root /data/;  #html目录
       proxy_pass http://static;
       autoindex on;;  #自动打开文件列表
    }  
}
```

## 6. 高可用

一般情况下，通过 nginx 主服务器访问后台目标服务集群，当主服务器挂掉后，自动切换至备份服务器，此时由备份服务器充当主服务器的角色，访问后端目标服务器。

### 6.1 .实战一

**实现效果：**
准备两台 nginx 服务器，通过浏览器地址栏访问虚拟 ip 地址，把主服务器的 nginx 停止，再次访问虚拟 ip 地址仍旧有效。
**具体配置：**
（1）在两台 nginx 服务器上安 keepalived。
keepalived 相当于一个路由，它通过一个脚本来检测当前服务器是否还活着，如果还活着则继续访问，否则就切换到另一台备份服务器。

```
# 安装 keepalived
yum install keepalived -y
# 检查版本
rpm -q -a keepalived
keepalived-1.3.5-16.el7.x86_64
```

（2）修改主备服务器 /etc/keepalived/keepalivec.conf 配置文件（可直接替换），完成高可用主从配置。
keepalived 将 nginx 服务器绑定到一个虚拟 ip ， nginx 高可用集群对外统一暴露这个虚拟 ip，客户端都是通过访问这个虚拟 ip 来访问 nginx 服务器 。

```
global_defs {
    notification_email {
        acassen@firewall.loc
        failover@firewall.loc
        sysadmin@firewall.loc
    }
    notification_email_from_Alexandre.Cassen@firewall.loc
    smtp_server 192.168.4.32
    smtp_connect_timeout 30
    router_id LVS_DEVEL  # 在 /etc/hosts 文件中配置，通过它能访问到我们的主机
}
vrrp_script_chk_http_port {   
    script "/usr/local/src/nginx_check.sh"
    interval 2      # 检测脚本执行的时间间隔
    weight 2        # 权重每次加2
}
vrrp_instance VI_1 {
    interface ens7f0 # 网卡，需根据情况修改
    state MASTER    # 备份服务器上将 MASTER 改为 BACKUP
    virtual_router_id 51 # 主备机的 virtual_router_id 必须相同
    priority 100   # 主备机取不同的优先级，主机值较大，备份机值较小
    advert_int 1  # 每隔多长时间（默认1s）发送一次心跳，检测服务器是否还活着
    authentication {
      auth_type PASS
      auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100 # VRRP H 虚拟地址，可以绑定多个
    }
}
```

**字段说明**

- router_id： 在 /etc/hosts 文件中配置，通过它能访问到我们的主机。 127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4 ::1 localhost localhost.localdomain localhost6 localhost6.localdomain6 127.0.0.1 LVS_DEVEL 复制代码
- interval： 设置脚本执行的间隔时间
- weight： 当脚本执行失败即 keepalived 或 nginx 挂掉时，权重增加的值（可为负数）。
- interface： 输入 ifconfig 命令查看当前的网卡名是什么。 ens7f0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500 inet 192.168.4.32 netmask 255.255.252.0 broadcast 192.168.7.255 inet6 fe80::e273:9c3c:e675:7c60 prefixlen 64 scopeid 0x20 … … 复制代码

（3）在 /usr/local/src 目录下添加检测脚本 nginx_check.sh。

```
#!/bin/bash
A=`ps -C nginx -no-header |wc -l`
if [ $A -eq 0 ];then
    /usr/local/nginx/sbin/nginx
    sleep 2
    if [ ps -C nginx -no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi
```

（4）启动两台服务器的 nginx 和 keepalived。

```
# 启动 nginx
./nginx
# 启动 keepalived
systemctl start keepalived.service
```

（5）查看虚拟 ip 地址 ip a 。把主服务器 192.168.4.32 nginx 和 keepalived停止，再访问虚拟 ip 查看高可用效果。

## 7. 原理解析

![img](https://pic2.zhimg.com/80/v2-3856323981ade758ea0aba92d26c0c01_720w.webp)

Nginx 启动之后，在 Linux 系统中有两个进程，一个为 master，一个为 worker。master 作为管理员不参与任何工作，只负责给多个 worker 分配不同的任务（worker 一般有多个）。

```
ps -ef |grep nginx
root     20473     1  0  2019 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx     4628 20473  0 Jan06 ?        00:00:00 nginx: worker process
nginx     4629 20473  0 Jan06 ?        00:00:00 nginx: worker process
```

**worker 是如何工作的？**
客户端发送一个请求首先要经过 master，管理员收到请求后会将请求通知给 worker，多个 worker 以**争抢**的机制来抢夺任务，得到任务的 worker 会将请求经由 tomcat 等做请求转发、反向代理、访问数据库等（nginx 本身是不直接支持 java 的）。

![img](https://pic2.zhimg.com/80/v2-7bcd042eb50b14229f52738c472a7685_720w.webp)

**一个 master 和多个 worker 的好处？**

- 可以使用 nginx -s reload 进行热部署。
- 每个 worker 是独立的进程，如果其中一个 worker 出现问题，其它 worker 是独立运行的，会继续争抢任务，实现客户端的请求过程，而不会造成服务中断。

**设置多少个 worker 合适？**
Nginx 和 redis 类似，都采用了 io 多路复用机制，每个 worker 都是一个独立的进程，每个进程里只有一个主线程，通过异步非阻塞的方式来处理请求，每个 worker 的线程可以把一个 cpu 的性能发挥到极致，因此，**worker 数和服务器的 cpu 数相等是最为适宜的**。

**思考：**
（1）发送一个请求，会占用 worker 几个连接数？
（2）有一个 master 和 4个 worker，每个 worker 支持的最大连接数为 1024，该系统支持的最大并发数是多少？

恭喜！目前为止你已经掌握了 Nginx 6种功能的配置方式，并和我一起进一步探讨了 Nginx 的原理。最后两个面试中可能会问到的思考题，欢迎大家评论区积极讨论。如果本文对你有所帮助，点赞互相鼓励一下吧~

原文链接：https://zhuanlan.zhihu.com/p/356100901

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)