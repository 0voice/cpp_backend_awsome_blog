# 【NO.46】一文读懂网关中间件-Nginx

## **1.Nginx介绍**

1.nginx是一个高性能HTTP服务器，反向代理服务器，邮件代理服务器，TCP/UDP反向代理服务器.

2.nginx处理请求是异步非阻塞的，在高并发下nginx 能保持低资源低消耗高性能,主要用在集群系统中用于支持负载均衡.

3.nginx对静态文件的处理速度也相当快，也可以用于前端站点的服务器.

## **2.为什么要使用Nginx?**

单个系统主要用于处理客户端请求，一个系统处理客户端的请求量是有限的，当客户端的并发量超过了系统的处理能力的时候，就会导致服务器性能降低，速度变慢，直接影响用户体验，所以为了提升性能，我们会创建多个服务实例，形成`集群系统`用于保证`高可用`

那么什么样的系统业务适合使用集群系统呢？我觉得主要从2个方面来看，第一请求人数多,导致次数多，第二请求量密集，例如我们近两年常用的`防疫健康码`查询，我们排除与其他的业务系统接入的因素，可以说他的99%针对用户的业务其实就是查询，而且并发量和请求数也是非常庞大的，所以就很适合使用集群系统。

## **3.查询分流Nginx原理**

### **3.1模块化设计**

高度模块化的设计是 Nginx的架构基础。在Nginx中，除了少量的核心代码，其他一切皆为模块，所有模块间是分层次、分类别的，Nginx 官方共有五大类型的模块：`核心模块`、`配置模块`、`事件模块`、`HTTP模块`、`mail模块`，5种模块中，`配置模块`和`核心模块`是与 Nginx 框架密切相关的。而事件模块则是 HTTP 模块和 mail 模块的基础。`HTTP 模块`和 `mail 模块`的“地位”类似，它们都是更关注于`应用层面`并且引用基础核心模块。

### **3.2多进程模型**

与Memcached的经典多线程模型相比，Nginx是经典的多进程模型，Nginx启动后在后台运行，后台进程包含一个master进程和多个worker进程，可以在配置中设置工作进程数，一般根据服务器的Cpu核心数，来决定工作进程数是多少，例如我的电脑核心数是12，那可以在配置文件中设置worker_processes为12，那么在进程中可以看到 一个13个nginx运行实例。

![img](https://pic3.zhimg.com/80/v2-485acc009963ea368519f27f78069d26_720w.webp)

### **3.3事件驱动架构**

![img](https://pic2.zhimg.com/80/v2-8190d1063be66d6acf0e5d7e489215f5_720w.webp)

处理请求事件时，Nginx 的事件消费者只是被事件分发者进程短期调用而已，这种设计使得网络性能、用户感知的请求时延都得到了提升，每个用户的请求所产生的事件会及时响应，整个服务器的网络吞吐量都会由于事件的及时响应而增大。当然，这也带来一定的要求，即每个事件消费者都不能有阻塞行为，否则将会由于长时间占用事件分发者进程而导致其他事件得不到及时响应，Nginx 的非阻塞特性就是由于它的模块都是满足这个要求，其实Nginx最佳的部署应该在linux ,linux的io及事件驱动优于windows，我们可以通过配置文件中设置events的数量表示当前的nginx能处理多少个请求，这个没有一个绝对的标准，可以基于服务的性能和本身业务需求而定。

### **3.4虚拟主机、反向代理、负载均衡**

1.虚拟主机就是为了对所有应用系统进行反向代理。
2.反向代理是指代理后端服务器，正向代理代表代理客户端。
3.负载均衡将流量均分到指定后端实例。

## **4.落地Nginx**

我们首先结合实际业务场景分析，然后对不同的业务用例进行落地实践的方案选择。

### **4.1负载均衡业务实践**

1.首先我们应该准备一个业务系统，在这就用上面说的“健康码查询”业务，模拟一个查询的服务，**`注意在这仅仅只是引用场景示例，不代表健康码真实场景如此， 因为我没有参与真正的防疫健康码的开发和设计,也不了解它业务和技术架构上真正的复杂度，单纯只是由此引入业务场景而已，如果您在阅读时觉得这样不合适，您可以把他当做你想当做的任何系统，或者忘记这件事，都是可以的`**
接下来我们应该下载Nginx作为我们的服务器，在这里我使用的是在Windows环境下的演示，其实不管在Linux或者Docker中部署都可以，但是开发在windows，所以基于Windows比较方便。

- 1.创建健康码服务

![img](https://pic3.zhimg.com/80/v2-f27abe32b4666e744f3c6025b4be520a_720w.webp)

2.我们应该创建服务集群，在这为了演示创建2个服务实例，然后使用nginx来进行负载均衡查询分流

- 1.命令行启动2个服务实例模拟集群，分别绑定端口8081和8082，其实真实环境不会只有2个实例或者在同一台服务器上部署。
- 2.配置nginx的反向代理和负载均衡

```text
worker_processes  1;
   error_log  logs/error.log  info;
   events {
       worker_connections  1024;
   }
   http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    access_log  logs/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    
    #虚拟主机
    server{
      listen       8080;
      server_name  localhost;
      #配置反向代理
      location / {
               proxy_pass  http://HealthCode;
           }
      error_page   500 502 503 504  /50x.html;
     }
   }
   #负载均衡配置
   upstream HealthCode{
           server localhost:8081;
           server localhost:8082;
   }
```

- 3.启动nginx,并访问虚拟服务器

![img](https://pic1.zhimg.com/80/v2-2d60590547845212ea5a3fba6c9db920_720w.webp)

### **4.2负载均衡业务场景分析**

针对上面的业务，我们使用到nginx实现负载均衡，那对于我们来说，应该知道Nginx虚拟主机，是如何将我们的请求转发到各个不同的服务实例的，下面结合业务用例来介绍Nginx中的各种`负载均衡算法`，并设置对应的配置文件节点。

**1.轮训算法**

根据字面理解就是轮训处理 1 2 3 4 周而复始,它是upstream模块默认的负载均衡默认策略，适合服务器处理能力相同的实例，在轮询中，如果某个服务实例宕机了，会自动剔除该服务器。，也可以加入权重，指定轮询几率，weight和访问比率成正比，用于服务器性能不均的情况下，权重越高，在被访问的概率越大，如下面分别是20%，80%。

```text
#负载均衡配置
upstream HealthCode{
    server localhost:8081;
    server localhost:8082;

    #server localhost:8081 weight=2;
    #server localhost:8082 weight=8;
}
```

**2.最小连接数算法 least_conn**

当客户端给Nginx发送查询健康码的请求时，Nginx把请求转发给8081和8082 ,如果8082 处理请求比较慢，会导致请求堆积在8082,那我们就需要解决请求堆积的问题，在这种场景下，我们可以把请求转发给连接数较少的服务器处理，能够达到更好的负载均衡效果，使用least_conn算法,在nginx配置文件中，负载均衡节点加入配置least_conn

```text
#负载均衡配置
upstream HealthCode{
    #配置最小连接数算法
    least_conn;
    server localhost:8081;
    server localhost:8082;
}
```

**3. hash一致性算法 ip_hash**

由于查询压力过大，为了提升可用性，我们在服务端加入缓存，例如3分钟之内请求，就直接将缓存的信息丢出去，客户端给Nginx发送查询健康码的请求时，Nginx把请求转发给8081和8082，甚至更多实例，使用轮训或者最小连接数时，会导致在缓存的情况下命中率下降，基于这种缓存`状态丢失`的情况，请求依然会给到没有缓存的服务实例，并去数据库中去查询数据，导致性能下降。

**（当第一次请求发送到8081去查询了数据库，但是在8082 或者其他的节点没有缓存，如果使用轮训算法及其他算法，会导致下次请求时，并不会访问缓存，所以叫`缓存命中率下降`）**
在这种场景下我们应该使用`Hash一致性算法`,将某一个请求客户端的ip地址与nginx的负载均衡中的某一个实例绑定。

```text
#负载均衡配置
upstream HealthCode{
  #配置iphash算法
    ip_hash;
    server localhost:8081;
    server localhost:8082;
}
```

**4.容灾策略**

```
重试机制
```

1.当客户端给Nginx发送查询请求时，Nginx把请求转发给8081和8082 ,如果转发到8081的时候，8081服务器被人拉闸，临时宕机了，会导致请求失败。如何保证请求成功？
在这种场景下我们应该使用nginx的`失败重试`机制,将某一个请求客户端的ip地址与nginx的负载均衡中的某一个实例绑定。

```text
#动态负载均衡配置
upstream HealthCode{
     ip_hash;
     #设置最大失败次数2次，超时时间10s钟
     server localhost:8081 max_fails=2 fail_timeout=10s;
     server localhost:8082 max_fails=2 fail_timeout=10s;
}
```

`主机备份 backup`
1.查询时请求转发给8081和8082 ，假设此时两个实例同时宕机了，会导致系统不可用，在这种异常业务情况下，我们可以使用`主机备份`来解决，注意在正常节点在运行时 ，备份节点是不工作的，如果使用`ip_hash`将不会生效，因为ip和主机已经绑定。

```text
#动态负载均衡配置
upstream HealthCode{
    ip_hash;
    server localhost:8081 max_fails=2 fail_timeout=10s;
    server localhost:8082 max_fails=2 fail_timeout=10s;
    #主机备份
    server localhost:8083 backup;
}
```

## **5.配置HTTPS**

我们要保证我们的请求的安全，所以需要使用Https通信,同样需要对我们的虚拟主机设置https，设置https的前提需要证书，一个是秘钥（server-key.pem），一个是证书（server-cert.pem）。

### **5.1基本设置**

1.下载openSSL,然后使用openSSL工具生成，教程连接
2.找到证书生成路径
3.然后在nginx配置文件添加对应的虚拟主机节点，然后配置Https

```text
# https 虚拟主机
    server {
        listen       4435 ssl;
        server_name  localhost;

        ssl_certificate      D:/cert/server-cert.pem;
        ssl_certificate_key  D:/cert/server-key.pem;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

        location / {
            proxy_pass  http://HealthCode;
        }
    }
```

### **5.2http转https**

系统当中总是有很多默认的Http请求，我们需要使用nginx的ngx_http_rewrite_module模块来将http请求转换成https

```text
worker_processes  1;
    error_log  logs/error.log  info;
    events {
        worker_connections  1024;
    }
    http {
     include       mime.types;
     default_type  application/octet-stream;
     log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
     access_log  logs/access.log  main;
     sendfile        on;
     keepalive_timeout  65;
     
     #虚拟主机
     server{
       listen       8080;
       server_name  localhost;

        #默认重定向到https
        if($scheme = http)
        {
            return 301 https://$host:4435$request_url
        }

       #配置反向代理
       location / {
                proxy_pass  http://HealthCode;
            }
       error_page   500 502 503 504  /50x.html;
      }
    }
#负载均衡配置
upstream HealthCode{
            server localhost:8081;
            server localhost:8082;
    }
```



原文链接：https://zhuanlan.zhihu.com/p/483693498

原文作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)