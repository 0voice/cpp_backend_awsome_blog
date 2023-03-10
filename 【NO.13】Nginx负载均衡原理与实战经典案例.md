# 【NO.13】Nginx负载均衡原理与实战经典案例

## **1.负载均衡的概念**

nginx应用场景之一就是负载均衡。在访问量较多的时候，可以通过负载均衡，将多个请求分摊到多台服务器上，相当于把一台服务器需要承担的负载量交给多台服务器处理，进而提高系统的吞吐率；另外如果其中某一台服务器挂掉，其他服务器还可以正常提供服务，以此来提高系统的可伸缩性与可靠性。

![img](https://pic1.zhimg.com/80/v2-0f0b3dc13daf7b304ba00eabea95d260_720w.webp)

上图为负载均衡示例图，当用户请求发送后，首先发送到负载均衡服务器，而后由负载均衡服务器根据配置规则将请求转发到不同的web服务器上。

## **2.nginx负载均衡策略**

nginx内置负载均衡策略主要分为三大类，分别是轮询、最少连接和ip hash

- 最少连接 请求分配给活动连接数最少的服务器，哪台服务器连接数最少，则把请求交给哪台服务器，由nginx统计服务器连接数
- ip hash 基于客户端ip的分配方式
- 轮询 以循环方式分发对应用服务器的请求，将请求平均分发到每台服务器上。

### **2-1轮询详解**

#### **2-1-1 普通轮询方式**

该方式是默认方式，轮询适合服务器配置相当，无状态且短平快的服务使用。另外在轮询中，如果服务器挂掉，会自动剔除该服务器。

```
http {
    # 定义转发分配规则
    upstream myapp1 {
        server srv1.com; # 要转发到的服务器，如ip、ip:端口号、域名、域名:端口号
        server srv2.com:8088;
        server 192.168.0.100:8088;
    }

    server {
        listen 80; # nginx监听的端口

        location / {
          # 使用myapp1分配规则，即刚自定义添加的upstream节点
          # 将所有请求转发到myapp1服务器组中配置的某一台服务器上
            proxy_pass http://myapp1; 
        }
    }
}
```

#### **2-1-2 权重轮询方式**

如果在 upstream 中配置的server参数后追加 weight 配置，则会根据配置的权重进行请求分发。此策略可以与least_conn和ip_hash结合使用，适合服务器的硬件配置差别比较大的情况。

```
# 定义转发分配规则
upstream myapp1 {
  server srv1.com weight=1; # 该台服务器接受1/6的请求量
  server srv2.com:8088 weight=2; # 该台服务器接受2/6的请求量
  server 192.168.0.100:8088 weight=3; # 该台服务器接受3/6的请求量;
}
```

### **2-2最少连接**

轮询算法是把请求平均的转发给各个后端，使它们的负载大致相同；但是，有些请求占用的时间很长，会导致其所在的后端负载较高。这种情况下，least_conn这种方式就可以达到更好的负载均衡效果，适合请求处理时间长短不一造成服务器过载的情况。

```
# 定义转发分配规则
upstream myapp1 {
  least_conn; # 把请求分派给连接数最少的服务器
  server srv1.com;
  server srv2.com:8088;
  server 192.168.0.100:8088;
}
```

### **2-3IP HASH**

这个方法确保了相同的客户端的请求一直发送到相同的服务器，这样每个访客都固定访问一个后端服务器。如用户需要分片上传文件到服务器下，然后再由服务器将分片合并，这时如果用户的请求到达了不同的服务器，那么分片将存储于不同的服务器目录中，导致无法将分片合并，该场景则需要使用ip hash策略。

需要注意的是，ip_hash不能与backup同时使用，另外当有服务器需要剔除，必须手动down掉，此模式适合有状态服务，比如session。

```
# 定义转发分配规则
upstream myapp1 {
  ip_hash; # #保证每个请求固定访问一个后端服务器
  server srv1.com;
  server srv2.com:8088;
  server 192.168.0.100:8088;
}
```



## **3.Nginx的负载均衡案例**

### **1.环境介绍**

该示例使用一台nginx作为负载均衡服务器，两台tomcat作为web服务器；可以把三个服务均在一台机器进行搭建，也可以使用虚拟机虚拟三台机器，然后进行测试。教程这里就只在一台机器进行搭建，采用默认的权重方式进行配置。

![img](https://pic2.zhimg.com/80/v2-16f961a829e95659ee6c5eb5d3493ec5_720w.webp)

如拓扑所示：**客户端要访问资源链接**如下：

```
http://www.ywflinux.com:8090/linux/linux.html
```

通过配置Nginx负载均衡服务器，它会将客户端请求平均转发到真实服务器站点1和真实服务器站点2上。

### **2.环境准备**

（1）客户端主机配置好域名解析，使得在客户端主机浏览器上通过访问[http://www.ywflinux.com](https://link.zhihu.com/?target=http%3A//www.ywflinux.com)，能够直接解析到192.168.13.199，我客户端在windows环境，现在通过修改相关配置文件，配置IP与域名的映射关系，配置如下：

```
192.168.13.199  www.ywflinux.com
```

![img](https://pic1.zhimg.com/80/v2-f479b2df0d5bbf7bab562fa085a72750_720w.webp)

（2）**真实服务器站点1和真实服务器站点2的tomcat环境准备好**，这里准备两个tomcat服务器，对应端口分别为8888和9999，主要是安装配置tomcat相关操作，如我配置好相关操作，能够访问到的真实服务器站点如下：

![img](https://pic3.zhimg.com/80/v2-ba44c92b5a228cfee09a1791b10bbe1a_720w.webp)

```
[root@www local]# cd tomcat2
[root@www tomcat2]# 
[root@www tomcat2]# ls
bin   lib      logs    RELEASE-NOTES  temp     work
conf  LICENSE  NOTICE  RUNNING.txt    webapps
[root@www tomcat2]# cd webapps/
[root@www webapps]# ls
docs  examples  host-manager  manager  ROOT  vod
[root@www webapps]# mkdir linux
[root@www webapps]# ls
docs  examples  host-manager  linux  manager  ROOT  vod
[root@www webapps]# cd vod/
[root@www vod]# ls
a.html
[root@www vod]# cp a.html /usr/local/tomcat2/webapps/linux/
[root@www vod]# cd /usr/local/tomcat2/webapps/linux/
[root@www linux]# ls
a.html
[root@www linux]# mv a.html linux.html
[root@www linux]# vi linux.html 
[root@www linux]# cat linux.html 
<h1>This is ywf 9999 Port</h1>
[root@www vod]#
```

![img](https://pic3.zhimg.com/80/v2-26ad476480c1282c448f14f695f0b75e_720w.webp)

![img](https://pic4.zhimg.com/80/v2-68ec5d498cfc92c03377581668bd5d8b_720w.webp)

![img](https://pic3.zhimg.com/80/v2-4290ef78641d9f2eb76a19919a338c6a_720w.webp)

（3）**负载均衡服务器环境准备**，这里主要是准备并且配置好负载均衡服务器，使得负载均衡服务器能够将客户端请求平均转发给真实服务器站点1和真实服务器站点2。具体配置如下所示：

修改nginx配置文件nginx.conf，在nginx.conf文件中的http块中添加、修改server{}标签相关配置。如下所示：

```
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    upstream ywfserver {
        server 192.168.13.199:8888;
        server 192.168.13.199:9999;
    }
    server {
        listen      8090;
        server_name  192.168.13.199;

        charset utf-8;

        #access_log  logs/host.access.log  main;

        location  / {
            proxy_pass http://ywfserver;
            root html;
            index index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
```

![img](https://pic2.zhimg.com/80/v2-a3d99c666553056b012491fb42affc79_720w.webp)

### **3.检查nginx配置文件语法，确认语法无误，命令如下：**

```
[root@localhost conf]# nginx -t
```

![img](https://pic1.zhimg.com/80/v2-c8735b1943e7b5d403a3406b31638dc4_720w.webp)

![img](https://pic1.zhimg.com/80/v2-2d2c4e33544900244eea44a093ef5b50_720w.webp)

### **4.重启nginx服务：**

```
[root@localhost conf]# nginx -s stop[root@localhost conf]# nginx
```

### **5.验证实验配置结果**

客户端通过浏览器访问站点资源[http://www.ywflinux.com:8090/linux/linux.html](https://link.zhihu.com/?target=http%3A//www.ywflinux.com%3A8090/linux/linux.html)，返回结果如下图所示，说明负载均衡配置成功。

![img](https://pic2.zhimg.com/80/v2-91d7060fcd5e83e545f68d0133c1c1fd_720w.webp)

此时第一次访问成功的时候，只需要重新载入页面就可以发现自动切换到我们的真实站点目录2.

![img](https://pic3.zhimg.com/80/v2-6acfa5eed034603d648b46f021b1d6e6_720w.webp)

**由此可见，负载均衡服务器将客户端请求平均分摊转发给了两个不同端口号的tomcat服务器进行处理。**

## 4.Nginx的动静分离介绍

### **1. 动静分离的概念**

#### 1.1 动态页面与静态页面区别

- 静态资源：当用户多次访问这个资源，资源的源代码永远不会改变的资源。
- 动态资源：当用户多次访问这个资源，资源的源代码可能会发送改变。

#### 1.2 什么是动静分离

- 动静分离是让动态网站里的动态网页根据一定规则把不变的资源和经常变的资源区分开来，动静资源做好了拆分以后，我们就可以根据静态资源的特点将其做缓存操作，这就是网站静态化处理的核心思路
- 动静分离简单的概括是：动态文件与静态文件的分离。
- 伪静态：网站如果想被搜索引擎搜素到，动态页面静态技术freemarker等模版引擎技术
- 1.3 为什么要用动静分离
- 在我们的软件开发中，有些请求是需要后台处理的（如：.jsp,.do等等），有些请求是不需要经过后台处理的（如：css、html、jpg、js等等文件），这些不需要经过后台处理的文件称为静态文件，否则动态文件。因此我们后台处理忽略静态文件。这会有人又说那我后台忽略静态文件不就完了吗。当然这是可以的，但是这样后台的请求次数就明显增多了。在我们对资源的响应速度有要求的时候，我们应该使用这种动静分离的策略去解决。
- 动静分离将网站静态资源（HTML，JavaScript，CSS，img等文件）与后台应用分开部署，提高用户访问静态代码的速度，降低对后台应用访问。这里我们将静态资源放到nginx中，动态资源转发到tomcat服务器中。
- 因此，动态资源转发到tomcat服务器我们就使用到了前面讲到的反向代理了。

### **2 .nginx实现动静分离**

**架构分析**

![img](https://pic3.zhimg.com/80/v2-5224f7575f16a287c60a976f7006b35e_720w.webp)

原文链接：https://zhuanlan.zhihu.com/p/584070725