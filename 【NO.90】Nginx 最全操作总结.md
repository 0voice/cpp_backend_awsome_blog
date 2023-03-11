# 【NO.90】Nginx 最全操作总结

> 本文将会从：安装 -> 全局配置 -> 常用的各种配置 来书写，其中常用配置写的炒鸡详细，需要的童鞋可以直接滑倒相应的位置查看。

## 1.**安装 nginx**

**下载 nginx 的压缩包文件到根目录，官网下载地址：nginx.org/download/nginx-x.xx.xx.tar.gz**

```
yum update #更新系统软件cd /wget nginx.org/download/nginx-1.17.2.tar.gz
```

**解压 tar.gz 压缩包文件，进去 nginx-1.17.2**

```
tar -xzvf nginx-1.17.2.tar.gzcd nginx-1.17.2
```

**进入文件夹后进行配置检查**

```
./configure
```

**通过安装前的配置检查，发现有报错。检查中发现一些依赖库没有找到，这时候需要先安装 nginx 的一些依赖库**

```
yum -y install pcre* #安装使nginx支持rewriteyum -y install gcc-c++yum -y install zlib*yum -y install openssl openssl-devel
```

**再次进行检查操作 ./configure 没发现报错显示，接下来进行编译并安装的操作**

```
 // 检查模块支持  ./configure  --prefix=/usr/local/nginx  --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_auth_request_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-mail --with-mail_ssl_module --with-stream --with-stream_ssl_module --with-stream_realip_module --with-stream_ssl_preread_module --with-threads --user=www --group=www
```

这里得特别注意下，你以后需要用到的功能模块是否存在，不然以后添加新的包会比较麻烦。

**查看默认安装的模块支持**

命令 `ls nginx-1.17.2` 查看 nginx 的文件列表，可以发现里面有一个 auto 的目录。

在这个 auto 目录中有一个 options 文件，这个文件里面保存的就是 nginx 编译过程中的所有选项配置。

通过命令：`cat nginx-1.17.2/auto/options | grep YES`就可以查看

[nginx 编译安装时，怎么查看安装模块](https://jingyan.baidu.com/article/454316ab354edcf7a7c03a81.html)

**编译并安装**

```
make && make install
```

这里需要注意，模块的支持跟后续的 nginx 配置有关，比如 SSL，gzip 压缩等等，编译安装前最好检查需要配置的模块存不存在。

**查看 nginx 安装后在的目录，可以看到已经安装到 /usr/local/nginx 目录了**

```
whereis nginx$nginx: /usr/local/nginx
```

**启动 nginx 服务**

```
cd /usr/local/nginx/sbin/./nginx
```

服务启动的时候报错了：`nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)` ，通过命令查看本机网络地址和端口等一些信息，找到被占用的 80 端口 `netstat -ntpl` 的 tcp 连接，并杀死进程(kill 进程 pid)

```
netstat -ntplkill 进程PID
```

继续启动 nginx 服务，启动成功

```
./nginx
```

在浏览器直接访问 ip 地址，页面出现 Welcome to Nginx! 则安装成功。

## **2.nginx 配置**

### 2.1 基本结构

```
main        # 全局配置，对全局生效├── events  # 配置影响 nginx 服务器或与用户的网络连接├── http    # 配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置│   ├── upstream # 配置后端服务器具体地址，负载均衡配置不可或缺的部分│   ├── server   # 配置虚拟主机的相关参数，一个 http 块中可以有多个 server 块│   ├── server│   │   ├── location  # server 块可以包含多个 location 块，location 指令用于匹配 uri│   │   ├── location│   │   └── ...│   └── ...└── ...
```

### 2.2 主要配置含义

- main:nginx 的全局配置，对全局生效。
- events:配置影响 nginx 服务器或与用户的网络连接。
- http：可以嵌套多个 server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。
- server：配置虚拟主机的相关参数，一个 http 中可以有多个 server。
- location：配置请求的路由，以及各种页面的处理情况。
- upstream：配置后端服务器具体地址，负载均衡配置不可或缺的部分。

### 2.3 nginx.conf 配置文件的语法规则

1. 配置文件由指令与指令块构成
2. 每条指令以 “;” 分号结尾，指令与参数间以空格符号分隔
3. 指令块以 {} 大括号将多条指令组织在一起
4. include 语句允许组合多个配置文件以提升可维护性
5. 通过 # 符号添加注释，提高可读性
6. 通过 $ 符号使用变量
7. 部分指令的参数支持正则表达式，例如常用的 location 指令

### 2.4 内置变量

nginx 常用的内置全局变量，你可以在配置中随意使用：
![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151537491057483.png)

### 2.5 常用命令

这里列举几个常用的命令：

```
nginx -s reload  # 向主进程发送信号，重新加载配置文件，热重启nginx -s reopen  # 重启 Nginxnginx -s stop    # 快速关闭nginx -s quit    # 等待工作进程处理完成后关闭nginx -T         # 查看当前 Nginx 最终的配置nginx -t -c <配置路径>  # 检查配置是否有问题，如果已经在配置目录，则不需要 -c
```

以上命令通过 `nginx -h` 就可以查看到，还有其它不常用这里未列出。

Linux 系统应用管理工具 systemd 关于 nginx 的常用命令：

```
systemctl start nginx    # 启动 Nginxsystemctl stop nginx     # 停止 Nginxsystemctl restart nginx  # 重启 Nginxsystemctl reload nginx   # 重新加载 Nginx，用于修改配置后systemctl enable nginx   # 设置开机启动 Nginxsystemctl disable nginx  # 关闭开机启动 Nginxsystemctl status nginx   # 查看 Nginx 运行状态
```

### 2.6 配置 nginx 开机自启

**利用 systemctl 命令**：

如果用 yum install 命令安装的 nginx，yum 命令会自动创建 nginx.service 文件，直接用命令:

```
systemctl enable nginx   # 设置开机启动 Nginxsystemctl disable nginx  # 关闭开机启动 Nginx
```

就可以设置开机自启，否则需要在系统服务目录里创建 nginx.service 文件。

创建并打开 nginx.service 文件：

```
vi /lib/systemd/system/nginx.service
```

内容如下：

```
[Unit]Description=nginxAfter=network.target[Service]Type=forkingExecStart=/usr/local/nginx/sbin/nginxExecReload=/usr/local/nginx/sbin/nginx -s reloadExecStop=/usr/local/nginx/sbin/nginx -s quitPrivateTmp=true[Install]WantedBy=multi-user.target
```

`:wq` 保存退出，运行 `systemctl daemon-reload` 使文件生效。

这样便可以通过以下命令操作 nginx 了：

```
systemctl start nginx.service # 启动nginx服务systemctl enable nginx.service # 设置开机启动systemctl disable nginx.service # 停止开机自启动systemctl status nginx.service # 查看服务当前状态systemctl restart nginx.service # 重新启动服务systemctl is-enabled nginx.service #查询服务是否开机启动
```

**通过开机启动命令脚本实现开机自启**

创建开机启动命令脚本文件：

```
vi /etc/init.d/nginx
```

在这个 nginx 文件中插入一下启动脚本代码，启动脚本代码来源网络复制，实测有效：

```
#! /bin/bash# chkconfig: - 85 15PATH=/usr/local/nginxDESC="nginx daemon"NAME=nginxDAEMON=$PATH/sbin/$NAMECONFIGFILE=$PATH/conf/$NAME.confPIDFILE=$PATH/logs/$NAME.pidscriptNAME=/etc/init.d/$NAMEset -e[ -x "$DAEMON" ] || exit 0do_start() {$DAEMON -c $CONFIGFILE || echo -n "nginx already running"}do_stop() {$DAEMON -s stop || echo -n "nginx not running"}do_reload() {$DAEMON -s reload || echo -n "nginx can't reload"}case "$1" instart)echo -n "Starting $DESC: $NAME"do_startecho ".";;stop)echo -n "Stopping $DESC: $NAME"do_stopecho ".";;reload|graceful)echo -n "Reloading $DESC configuration..."do_reloadecho ".";;restart)echo -n "Restarting $DESC: $NAME"do_stopdo_startecho ".";;*)echo "Usage: $scriptNAME {start|stop|reload|restart}" >&2exit 3;;esacexit 0
```

设置所有人都有对这个启动脚本 nginx 文件的执行权限：

```
chmod a+x /etc/init.d/nginx
```

把 nginx 加入系统服务中：

```
chkconfig --add nginx
```

把服务设置为开机启动：

```
chkconfig nginx on
```

reboot 重启系统生效，可以使用上面 systemctl 方法相同的命令：

```
systemctl start nginx.service # 启动nginx服务systemctl enable nginx.service # 设置开机启动systemctl disable nginx.service # 停止开机自启动systemctl status nginx.service # 查看服务当前状态systemctl restart nginx.service # 重新启动服务systemctl is-enabled nginx.service #查询服务是否开机启动
```

如果服务启动的时候出现 `Restarting nginx daemon: nginxnginx: [error] open() "/usr/local/nginx/logs/nginx.pid" failed (2: No such file or directory) nginx not running` 的错误，通过 nginx -c 参数指定配置文件即可解决

```
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

如果服务启动中出现 `nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)` 的错误，可以先通过 `service nginx stop` 停止服务，再启动就好。

### 2.7 配置 nginx 全局可用

当你每次改了 `nginx.conf` 配置文件的内容都需要重新到 nginx 启动目录去执行命令，或者通过 -p 参数指向特定目录，会不会感觉很麻烦？

例如：直接执行 `nginx -s reload` 会报错 `-bash: nginx: command not found`，需要到 `/usr/local/nginx/sbin` 目录下面去执行，并且是执行 `./nginx -s reload`。

这里有两种方式可以解决，一种是通过脚本对 nginx 命令包装，这里介绍另外一种比较简单：通过把 nginx 配置到环境变量里，用 nginx 执行指令即可。步骤如下：

1、编辑 /etc/profile

```
vi /etc/profile
```

2、在最后一行添加配置，:wq 保存

```
export PATH=$PATH:/usr/local/nginx/sbin
```

3、使配置立即生效

```
source /etc/profile
```

这样就可以愉快的直接在全局使用 nginx 命令了。

## 3.**nginx 常用功能**

### 3.1 反向代理

我们最常说的反向代理的是通过反向代理解决跨域问题。

其实反向代理还可以用来控制缓存（代理缓存 proxy cache），进行访问控制等等，以及后面说的负载均衡其实都是通过反向代理来实现的。

```
server {    listen    8080;        # 用户访问 ip:8080/test 下的所有路径代理到 github        location /test {         proxy_pass   https://github.com;        }        # 所有 /api 下的接口访问都代理到本地的 8888 端口        # 例如你本地运行的 java 服务的端口是 8888，接口都是以 /api 开头        location /api {            proxy_pass   http://127.0.0.1:8888;        }}
```

### 3.2 访问控制

```
server {   location ~ ^/index.html {       # 匹配 index.html 页面 除了 127.0.0.1 以外都可以访问       deny 192.168.1.1;       deny 192.168.1.2;       allow all; }}
```

上面的命令表示禁止 192.168.1.1 和 192.168.1.2 两个 ip 访问，其它全部允许。从上到下的顺序，匹配到了便跳出，可以按你的需求设置。

### 3.3 负载均衡

通过负载均衡充利用服务器资源，nginx 目前支持自带 4 种负载均衡策略，还有 2 种常用的第三方策略。

**轮询策略（默认）**

每个请求按时间顺序逐一分配到不同的后端服务器，如果有后端服务器挂掉，能自动剔除。但是如果其中某一台服务器压力太大，出现延迟，会影响所有分配在这台服务器下的用户。

```
http {    upstream test.com {        server 192.168.1.12:8887;        server 192.168.1.13:8888;    }    server {        location /api {            proxy_pass  http://test.com;        }    }}
```

**根据服务器权重**

例如要配置：10 次请求中大概 1 次访问到 8888 端口，9 次访问到 8887 端口：

```
http {    upstream test.com {        server 192.168.1.12:8887 weight=9;        server 192.168.1.13:8888 weight=1;    }    server {        location /api {            proxy_pass  http://test.com;        }    }}
```

**客户端 ip 绑定（ip_hash）**

来自同一个 ip 的请求永远只分配一台服务器，有效解决了动态网页存在的 session 共享问题。例如：比如把登录信息保存到了 session 中，那么跳转到另外一台服务器的时候就需要重新登录了。

所以很多时候我们需要一个客户只访问一个服务器，那么就需要用 ip_hash 了。

```
http {    upstream test.com {     ip_hash;        server 192.168.1.12:8887;        server 192.168.1.13:8888;    }    server {        location /api {            proxy_pass  http://test.com;        }    }}
```

**最小连接数策略**

将请求优先分配给压力较小的服务器，它可以平衡每个队列的长度，并避免向压力大的服务器添加更多的请求。

```
http {    upstream test.com {     least_conn;        server 192.168.1.12:8887;        server 192.168.1.13:8888;    }    server {        location /api {            proxy_pass  http://test.com;        }    }}
```

**最快响应时间策略（依赖于第三方 NGINX Plus）**

依赖于 NGINX Plus，优先分配给响应时间最短的服务器。

```
http {    upstream test.com {     fair;        server 192.168.1.12:8887;        server 192.168.1.13:8888;    }    server {        location /api {            proxy_pass  http://test.com;        }    }}
```

**按访问 url 的 hash 结果（第三方）**

按访问 url 的 hash 结果来分配请求，使每个 url 定向到同一个后端服务器，后端服务器为缓存时比较有效。在 upstream 中加入 hash 语句，server 语句中不能写入 weight 等其他的参数，hash_method 是使用的 hash 算法

```
http {    upstream test.com {     hash $request_uri;     hash_method crc32;     server 192.168.1.12:8887;     server 192.168.1.13:8888;    }    server {        location /api {            proxy_pass  http://test.com;        }    }}
```

采用 HAproxy 的 loadbalance uri 或者 nginx 的 upstream_hash 模块，都可以做到针对 url 进行哈希算法式的负载均衡转发。

### 3.4 gzip 压缩

开启 gzip 压缩可以大幅减少 http 传输过程中文件的大小，可以极大的提高网站的访问速度，基本是必不可少的优化操作：

```
gzip  on; # 开启gzip 压缩# gzip_types# gzip_static on;# gzip_proxied expired no-cache no-store private auth;# gzip_buffers 16 8k;gzip_min_length 1k;gzip_comp_level 4;gzip_http_version 1.0;gzip_vary off;gzip_disable "MSIE [1-6]\.";
```

解释一下：

1. gzip_types：要采用 gzip 压缩的 MIME 文件类型，其中 text/html 被系统强制启用；
2. gzip_static：默认 off，该模块启用后，Nginx 首先检查是否存在请求静态文件的 gz 结尾的文件，如果有则直接返回该 .gz 文件内容；
3. gzip_proxied：默认 off，nginx 做为反向代理时启用，用于设置启用或禁用从代理服务器上收到相应内容 gzip 压缩；
4. gzip_buffers：获取多少内存用于缓存压缩结果，16 8k 表示以 8k*16 为单位获得；
5. gzip_min_length：允许压缩的页面最小字节数，页面字节数从 header 头中的 Content-Length 中进行获取。默认值是 0，不管页面多大都压缩。建议设置成大于 1k 的字节数，小于 1k 可能会越压越大；
6. gzip_comp_level：gzip 压缩比，压缩级别是 1-9，1 压缩级别最低，9 最高，级别越高压缩率越大，压缩时间越长，建议 4-6；
7. gzip_http_version：默认 1.1，启用 gzip 所需的 HTTP 最低版本；
8. gzip_vary：用于在响应消息头中添加 Vary：Accept-Encoding，使代理服务器根据请求头中的 Accept-Encoding 识别是否启用 gzip 压缩；
9. gzip_disable 指定哪些不需要 gzip 压缩的浏览器

其中第 2 点，普遍是结合前端打包的时候打包成 gzip 文件后部署到服务器上，这样服务器就可以直接使用 gzip 的文件了，并且可以把压缩比例提高，这样 nginx 就不用压缩，也就不会影响速度。一般不追求极致的情况下，前端不用做任何配置就可以使用啦~

附前端 webpack 开启 gzip 压缩配置，在 vue-cli3 的 vue.config.js 配置文件中：

```
const CompressionWebpackPlugin = require('compression-webpack-plugin')module.exports = {  // gzip 配置  configureWebpack: config => {    if (process.env.NODE_ENV === 'production') {      // 生产环境      return {        plugins: [new CompressionWebpackPlugin({          test: /\.js$|\.html$|\.css/,    // 匹配文件名          threshold: 1024,               // 文件压缩阈值，对超过 1k 的进行压缩          deleteOriginalAssets: false     // 是否删除源文件        })]      }    }  },  ...}
```

### 3.5 HTTP 服务器

nginx 本身也是一个静态资源的服务器，当只有静态资源的时候，就可以使用 nginx 来做服务器：

```
server {  listen       80;  server_name  localhost;  location / {      root   /usr/local/app;      index  index.html;  }}
```

这样如果访问 [http://ip](http://ip/) 就会默认访问到 /usr/local/app 目录下面的 index.html，如果一个网站只是静态页面的话，那么就可以通过这种方式来实现部署，比如一个静态官网。

### 3.6 动静分离

就是把动态和静态的请求分开。方式主要有两种：

- 一种是纯粹把静态文件独立成单独的域名，放在独立的服务器上，也是目前主流推崇的方案
- 一种方法就是动态跟静态文件混合在一起发布， 通过 nginx 配置来分开

```
# 所有静态请求都由nginx处理，存放目录为 htmllocation ~ \.(gif|jpg|jpeg|png|bmp|swf|css|js)$ {    root    /usr/local/resource;    expires     10h; # 设置过期时间为10小时}# 所有动态请求都转发给 tomcat 处理location ~ \.(jsp|do)$ {    proxy_pass  127.0.0.1:8888;}
```

注意上面设置了 expires，当 nginx 设置了 expires 后，例如设置为：expires 10d; 那么，所在的 location 或 if 的内容，用户在 10 天内请求的时候，都只会访问浏览器中的缓存，而不会去请求 nginx 。

### 3.7 请求限制

对于大流量恶意的访问，会造成带宽的浪费，给服务器增加压力。可以通过 nginx 对于同一 IP 的连接数以及并发数进行限制。合理的控制还可以用来防止 DDos 和 CC 攻击。

关于请求限制主要使用 nginx 默认集成的 2 个模块：

- limit_conn_module 连接频率限制模块
- limit_req_module 请求频率限制模块

涉及到的配置主要是：

- limit_req_zone 限制请求数
- limit_conn_zone 限制并发连接数

**通过 limit_req_zone 限制请求数**

```
http{    limit_conn_zone $binary_remote_addrzone=limit:10m; // 设置共享内存空间大    server{     location /{            limit_conn addr 5; # 同一用户地址同一时间只允许有5个连接。        }    }}
```

如果共享内存空间被耗尽，服务器将会对后续所有的请求返回 503 (Service Temporarily Unavailable) 错误。

当多个 limit_conn_zone 指令被配置时，所有的连接数限制都会生效。比如，下面配置不仅会限制单一 IP 来源的连接数，同时也会限制单一虚拟服务器的总连接数：

```
limit_conn_zone $binary_remote_addr zone=perip:10m;limit_conn_zone $server_name zone=perserver:10m;server {    limit_conn perip 10; # 限制每个 ip 连接到服务器的数量    limit_conn perserver 2000; # 限制连接到服务器的总数}
```

**通过 limit_conn_zone 限制并发连接数**

```
limit_req_zone $binary_remote_addr zone=creq:10 mrate=10r/s;server{    location /{        limit_req zone=creq burst=5;    }}
```

限制平均每秒不超过一个请求，同时允许超过频率限制的请求数不多于 5 个。如果不希望超过的请求被延迟，可以用 nodelay 参数,如：

```
limit_req zone=creq burst=5 nodelay;
```

这里只是简单讲讲，让大家有这个概念，配置的时候可以深入去找找资料。

### 3.8 正向代理

正向代理，意思是一个位于客户端和原始服务器(origin server)之间的服务器，为了从原始服务器取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端。客户端才能使用正向代理，比如我们使用的 VPN 服务就是正向代理，直观区别：

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/202212151538333925963.png)

配置正向代理：

```
resolver 8.8.8.8 # 谷歌的域名解析地址server {    resolver_timeout 5s; // 设超时时间    location / {        # 当客户端请求我的时候，我会把请求转发给它        # $host 要访问的主机名 $request_uri 请求路径        proxy_pass http://$host$request_uri;    }}
```

正向代理的对象是客户端，服务器端看不到真正的客户端。

### 3.9 图片防盗链

```
server {    listen       80;    server_name  *.test;    # 图片防盗链    location ~* \.(gif|jpg|jpeg|png|bmp|swf)$ {        valid_referers none blocked server_names ~\.google\. ~\.baidu\. *.qq.com;  # 只允许本机 IP 外链引用，将百度和谷歌也加入白名单有利于 SEO        if ($invalid_referer){            return 403;        }    }}
```

以上设置就能防止其它网站利用外链访问我们的图片，有利于节省流量

### 3.10 适配 PC 或移动设备

根据用户设备不同返回不同样式的站点，以前经常使用的是纯前端的自适应布局，但是复杂的网站并不适合响应式，无论是复杂性和易用性上面还是不如分开编写的好，比如我们常见的淘宝、京东。

根据用户请求的 user-agent 来判断是返回 PC 还是 H5 站点：

```
server {    listen 80;    server_name test.com;    location / {     root  /usr/local/app/pc; # pc 的 html 路径        if ($http_user_agent ~* '(Android|webOS|iPhone|iPod|BlackBerry)') {            root /usr/local/app/mobile; # mobile 的 html 路径        }        index index.html;    }}
```

### 3.11 设置二级域名

新建一个 server 即可：

```
server {    listen 80;    server_name admin.test.com; // 二级域名    location / {        root  /usr/local/app/admin; # 二级域名的 html 路径        index index.html;    }}
```

### 3.12 配置 HTTPS

这里我使用的是 certbot 免费证书，但申请一次有效期只有 3 个月（好像可以用 crontab 尝试配置自动续期，我暂时没试过）：

先安装 certbot

```
wget https://dl.eff.org/certbot-autochmod a+x certbot-auto
```

申请证书（注意：需要把要申请证书的域名先解析到这台服务器上，才能申请）:

```
sudo ./certbot-auto certonly --standalone --email admin@abc.com -d test.com -d www.test.com
```

执行上面指令，按提示操作。

Certbot 会启动一个临时服务器来完成验证（会占用 80 端口或 443 端口，因此需要暂时关闭 Web 服务器），然后 Certbot 会把证书以文件的形式保存，包括完整的证书链文件和私钥文件。

文件保存在 /etc/letsencrypt/live/ 下面的域名目录下。

修改 nginx 配置：

```
server{    listen 443 ssl http2; // 这里还启用了 http/2.0    ssl_certificate /etc/letsencrypt/live/test.com/fullchain.pem; # 证书文件地址    ssl_certificate_key /etc/letsencrypt/live/test.com/privkey.pem; # 私钥文件地址    server_name test.com www.test.com; // 证书绑定的域名}
```

### 3.13 配置 HTTP 转 HTTPS

```
server {    listen      80;    server_name test.com www.test.com;    # 单域名重定向    if ($host = 'www.sherlocked93.club'){        return 301 https://www.sherlocked93.club$request_uri;    }    # 全局非 https 协议时重定向    if ($scheme != 'https') {        return 301 https://$server_name$request_uri;    }    # 或者全部重定向    return 301 https://$server_name$request_uri;}
```

以上配置选择自己需要的一条即可，不用全部加。

### 3.14 单页面项目 history 路由配置

```
server {    listen       80;    server_name  fe.sherlocked93.club;    location / {        root       /usr/local/app/dist;  # vue 打包后的文件夹        index      index.html index.htm;        try_files  $uri $uri/ /index.html @rewrites; # 默认目录下的 index.html，如果都不存在则重定向        expires -1;                          # 首页一般没有强制缓存        add_header Cache-Control no-cache;    }    location @rewrites { // 重定向设置        rewrite ^(.+)$ /index.html break;    }}
```

[vue-router](https://router.vuejs.org/zh/guide/essentials/history-mode.html#后端配置例子) 官网只有一句话 `try_files $uri $uri/ /index.html;`，而上面做了一些重定向处理。

### 3.15 配置高可用集群（双机热备）

当主 nginx 服务器宕机之后，切换到备份的 nginx 服务器

首先安装 keepalived:

```
yum install keepalived -y
```

然后编辑 `/etc/keepalived/keepalived.conf` 配置文件，并在配置文件中增加 `vrrp_script` 定义一个外围检测机制，并在 `vrrp_instance` 中通过定义 `track_script` 来追踪脚本执行过程，实现节点转移：

```
global_defs{   notification_email {        cchroot@gmail.com   }   notification_email_from test@firewall.loc   smtp_server 127.0.0.1   smtp_connect_timeout 30 // 上面都是邮件配置   router_id LVS_DEVEL     // 当前服务器名字，用 hostname 命令来查看}vrrp_script chk_maintainace { // 检测机制的脚本名称为chk_maintainace    script "[[ -e/etc/keepalived/down ]] && exit 1 || exit 0" // 可以是脚本路径或脚本命令    // script "/etc/keepalived/nginx_check.sh"    // 比如这样的脚本路径    interval 2  // 每隔2秒检测一次    weight -20  // 当脚本执行成立，那么把当前服务器优先级改为-20}vrrp_instanceVI_1 {   // 每一个vrrp_instance就是定义一个虚拟路由器    state MASTER      // 主机为MASTER，备用机为BACKUP    interface eth0    // 网卡名字，可以从ifconfig中查找    virtual_router_id 51 // 虚拟路由的id号，一般小于255，主备机id需要一样    priority 100      // 优先级，master的优先级比backup的大    advert_int 1      // 默认心跳间隔    authentication {  // 认证机制        auth_type PASS        auth_pass 1111   // 密码    }    virtual_ipaddress {  // 虚拟地址vip       172.16.2.8    }}
```

其中检测脚本 `nginx_check.sh`，这里提供一个：

```
#!/bin/bashA=`ps -C nginx --no-header | wc -l`if [ $A -eq 0 ];then    /usr/sbin/nginx # 尝试重新启动nginx    sleep 2         # 睡眠2秒    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then        killall keepalived # 启动失败，将keepalived服务杀死。将vip漂移到其它备份节点    fifi
```

复制一份到备份服务器，备份 nginx 的配置要将 `state` 后改为 `BACKUP`，`priority` 改为比主机小。设置完毕后各自 `service keepalived start` 启动，经过访问成功之后，可以把 Master 机的 keepalived 停掉，此时 Master 机就不再是主机了 `service keepalived stop`，看访问虚拟 IP 时是否能够自动切换到备机 ip addr。

再次启动 Master 的 keepalived，此时 vip 又变到了主机上。

配置高可用集群的内容来源于：[Nginx 从入门到实践，万字详解！](https://juejin.im/post/6844904144235413512#heading-11)

## 4. **其它功能和技巧**

### 4.1 代理缓存

nginx 的 http_proxy 模块，提供类似于 Squid 的缓存功能，使用 proxy_cache_path 来配置。

nginx 可以对访问过的内容在 nginx 服务器本地建立副本，这样在一段时间内再次访问该数据，就不需要通过 nginx 服务器再次向后端服务器发出请求，减小数据传输延迟，提高访问速度：

```
proxy_cache_path usr/local/cache levels=1:2 keys_zone=my_cache:10m;server {  listen       80;  server_name  test.com;  location / {      proxy_cache my_cache;      proxy_pass http://127.0.0.1:8888;      proxy_set_header Host $host;  }}
```

上面的配置表示：nginx 提供一块 10 M 的内存用于缓存，名字为 my_cache, levels 等级为 1:2，缓存存放的路径为 `usr/local/cache`。

### 4.2 访问日志

访问日志默认是注释的状态，需要可以打开和进行更详细的配置，一下是 nginx 的默认配置：

```
http {    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '                      '$status $body_bytes_sent "$http_referer" '                      '"$http_user_agent" "$http_x_forwarded_for"';    access_log  logs/access.log  main;}
```

### 4.3 错误日志

错误日志放在 main 全局区块中，童鞋们打开 nginx.conf 就可以看见在配置文件中和下面一样的代码了：

```
#error_log  logs/error.log;#error_log  logs/error.log  notice;#error_log  logs/error.log  info;
```

nginx 错误日志默认配置为：

```
error_log logs/error.log error;
```

### 4.4 静态资源服务器

```
server {    listen       80;    server_name  static.bin;    charset utf-8;    # 防止中文文件名乱码    location /download {        alias           /usr/share/nginx/static;  # 静态资源目录        autoindex               on;    # 开启静态资源列目录，浏览目录权限        autoindex_exact_size    off;   # on(默认)显示文件的确切大小，单位是byte；off显示文件大概大小，单位KB、MB、GB        autoindex_localtime     off;   # off(默认)时显示的文件时间为GMT时间；on显示的文件时间为服务器时间    }}
```

### 4.5 禁止指定 user_agent

nginx 可以禁止指定的浏览器和爬虫框架访问：

```
# http_user_agent 为浏览器标识# 禁止 user_agent 为baidu、360和sohu，~*表示不区分大小写匹配if ($http_user_agent ~* 'baidu|360|sohu') {    return 404;}# 禁止 Scrapy 等工具的抓取if ($http_user_agent ~* (Scrapy|Curl|HttpClient)) {    return 403;
```

### 4.6 请求过滤

**根据请求类型过滤**

```
# 非指定请求全返回 403if ( $request_method !~ ^(GET|POST|HEAD)$ ) {    return 403;}
```

**根据状态码过滤**

```
error_page 502 503 /50x.html;location = /50x.html {    root /usr/share/nginx/html;}
```

这样实际上是一个内部跳转，当访问出现 502、503 的时候就能返回 50x.html 中的内容，这里需要注意是否可以找到 50x.html 页面，所以加了个 location 保证找到你自定义的 50x 页面。

**根据 URL 名称过滤**

```
if ($host = zy.com' ) {     #其中 $1是取自regex部分()里的内容,匹配成功后跳转到的URL。     rewrite ^/(.*)$  http://www.zy.com/$1  permanent；}location /test {    // /test 全部重定向到首页    rewrite  ^(.*)$ /index.html  redirect;}
```

### 4.7 ab 命令

ab 命令全称为：Apache bench，是 Apache 自带的压力测试工具，也可以测试 Nginx、IIS 等其他 Web 服务器:

- -n 总共的请求数
- -c 并发的请求数
- -t 测试所进行的最大秒数，默认值 为 50000
- -p 包含了需要的 POST 的数据文件
- -T POST 数据所使用的 Content-type 头信息

```
ab -n 1000 -c 5000 http://127.0.0.1/ # 每次发送1000并发的请求数，请求数总数为5000。
```

测试前需要安装 httpd-tools：`yum install httpd-tools`

### 4.8 泛域名路径分离

这是一个非常实用的技能，经常有时候我们可能需要配置一些二级或者三级域名，希望通过 nginx 自动指向对应目录，比如：

1. test1.doc.test.club 自动指向 /usr/local/html/doc/test1 服务器地址；
2. test2.doc.test.club 自动指向 /usr/local/html/doc/test2 服务器地址；

```
server {    listen       80;    server_name  ~^([\w-]+)\.doc\.test\.club$;    root /usr/local/html/doc/$1;}
```

### 4.9 泛域名转发

和之前的功能类似，有时候我们希望把二级或者三级域名链接重写到我们希望的路径，让后端就可以根据路由解析不同的规则：

1. test1.serv.test.club/api?name=a 自动转发到 127.0.0.1:8080/test1/api?name=a
2. test2.serv.test.club/api?name=a 自动转发到 127.0.0.1:8080/test2/api?name=a

```
server {    listen       80;    server_name ~^([\w-]+)\.serv\.test\.club$;    location / {        proxy_set_header        X-Real-IP $remote_addr;        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;        proxy_set_header        Host $http_host;        proxy_set_header        X-NginX-Proxy true;        proxy_pass              http://127.0.0.1:8080/$1$request_uri;    }}
```

## 5.**常见问题**

### 5.1 nginx 中怎么设置变量

或许你不知道，nginx 的配置文件使用的是一门微型的编程语言。既然是编程语言，一般也就少不了“变量”这种东西，但是在 nginx 配置中，变量只能存放一种类型的值，因为也只存在一种类型的值，那就是字符串。

例如我们在 nginx.conf 中有这样一行配置：

```
set $name "chroot";
```

上面使用了 set 配置指令对变量 `$name`进行了赋值操作，把 “chroot” 赋值给了 `$name`。nginx 变量名前面有一个 `$` 符号，这是记法上的要求。所有的 Nginx 变量在 Nginx 配置文件中引用时都须带上 `$` 前缀。这种表示方法和 Perl、PHP 这些语言是相似的。

这种表示方法的用处在哪里呢，那就是可以直接把变量嵌入到字符串常量中以构造出新的字符串，例如你需要进行一个字符串拼接：

```
server {  listen       80;  server_name  test.com;  location / {     set $temp hello;     return "$temp world";  }}
```

以上当匹配成功的时候就会返回字符串 “hello world” 了。需要注意的是，当引用的变量名之后紧跟着变量名的构成字符时（比如后跟字母、数字以及下划线），我们就需要使用特别的记法来消除歧义，例如：

```
server {  listen       80;  server_name  test.com;  location / {     set $temp "hello ";     return "${temp}world";  }}
```

这里，我们在配置指令的参数值中引用变量 `$temp` 的时候，后面紧跟着 `world` 这个单词，所以如果直接写作 `"$tempworld"` 则 nginx 的计算引擎会将之识别为引用了变量 `$tempworld`. 为了解决这个问题，nginx 的字符串支持使用花括号在 `$` 之后把变量名围起来，比如这里的 `${temp}`，所以 上面这个例子返回的还是 “hello world”：

```
$ curl 'http://test.com/'    hello world
```

还需要注意的是，若是想输出 `$` 符号本身，可以这样做：

```
geo $dollar {    default "$";}server {    listen       80;    server_name  test.com;    location / {        set $temp "hello ";        return "${temp}world: $dollar";    }}
```

上面用到了标准模块 ngx_geo 提供的配置指令 geo 来为变量 `$dollar` 赋予字符串 `"$"` ，这样，这里的返回值就是 “hello world: $” 了。

## 6.**附 nginx 内置预定义变量**

按字母顺序，变量名与对应定义：

- `$arg_PARAMETER` #GET 请求中变量名 PARAMETER 参数的值
- `$args` #这个变量等于 GET 请求中的参数，例如，foo=123&bar=blahblah;这个变量可以被修改
- `$binary_remote_addr` #二进制码形式的客户端地址
- `$body_bytes_sent` #传送页面的字节数
- `$content_length` #请求头中的 Content-length 字段
- `$content_type` #请求头中的 Content-Type 字段
- `$cookie_COOKIE` #cookie COOKIE 的值
- `$document_root` #当前请求在 root 指令中指定的值
- `$document_uri` #与 $uri 相同
- `$host` #请求中的主机头(Host)字段，如果请求中的主机头不可用或者空，则为处理请求的 server 名称(处理请求的 server 的 server_name 指令的值)。值为小写，不包含端口
- `$hostname` #机器名使用 gethostname 系统调用的值
- `$http_HEADER` #HTTP 请求头中的内容，HEADER 为 HTTP 请求中的内容转为小写，-变为*(破折号变为下划线)，例如：`$http*user_agent`(Uaer-Agent 的值)
- `$sent_http_HEADER` #HTTP 响应头中的内容，HEADER 为 HTTP 响应中的内容转为小写，-变为*(破折号变为下划线)，例如：`$sent*http_cache_control`、`$sent_http_content_type`…
- `$is_args` #如果 $args 设置，值为”?”，否则为””
- `$limit_rate` #这个变量可以限制连接速率
- `$nginx_version` #当前运行的 nginx 版本号
- `$query_string` #与 $args 相同
- `$remote_addr` #客户端的 IP 地址
- `$remote_port` #客户端的端口
- `$remote_port` #已经经过 Auth Basic Module 验证的用户名
- `$request_filename` #当前连接请求的文件路径，由 root 或 alias 指令与 URI 请求生成
- `$request_body` #这个变量（0.7.58+）包含请求的主要信息。在使用 proxy_pass 或 fastcgi_pass 指令的 location 中比较有意义
- `$request_body_file` #客户端请求主体信息的临时文件名
- `$request_completion` #如果请求成功，设为”OK”；如果请求未完成或者不是一系列请求中最后一部分则设为空
- `$request_method` #这个变量是客户端请求的动作，通常为 GET 或 POST。包括 0.8.20 及之前的版本中，这个变量总为 main request 中的动作，如果当前请求是一个子请求，并不使用这个当前请求的动作
- `$request_uri` #这个变量等于包含一些客户端请求参数的原始 URI，它无法修改，请查看 $uri 更改或重写 URI
- `$scheme` #所用的协议，例如 http 或者是 https，例如 `rewrite ^(.+)$$scheme://example.com$1 redirect`
- `$server_addr` #服务器地址，在完成一次系统调用后可以确定这个值，如果要绕开系统调用，则必须在 listen 中指定地址并且使用 bind 参数
- `$server_name` #服务器名称
- `$server_port` #请求到达服务器的端口号
- `$server_protocol` #请求使用的协议，通常是 HTTP/1.0、HTTP/1.1 或 HTTP/2
- `$uri` #请求中的当前 URI(不带请求参数，参数位于 args ) ， 不 同 于 浏 览 器 传 递 的 args)，不同于浏览器传递的 args)，不同于浏览器传递的 request_uri 的值，它可以通过内部重定向，或者使用 index 指令进行修改。不包括协议和主机名，例如 /foo/bar.html

## 7.**附 nginx 模块**

### 7.1 nginx 模块分类

- 核心模块：nginx 最基本最核心的服务，如进程管理、权限控制、日志记录；
- 标准 HTTP 模块：nginx 服务器的标准 HTTP 功能；
- 可选 HTTP 模块：处理特殊的 HTTP 请求
- 邮件服务模块：邮件服务
- 第三方模块：作为扩展，完成特殊功能

### 7.2 模块清单

**核心模块**：

- ngx_core
- ngx_errlog
- ngx_conf
- ngx_events
- ngx_event_core
- ngx_epll
- ngx_regex

**标准 HTTP 模块**：

- ngx_http
- ngx_http_core #配置端口，URI 分析，服务器相应错误处理，别名控制 (alias) 等
- ngx_http_log #自定义 access 日志
- ngx_http_upstream #定义一组服务器，可以接受来自 proxy, Fastcgi,Memcache 的重定向；主要用作负载均衡
- ngx_http_static
- ngx_http_autoindex #自动生成目录列表
- ngx_http_index #处理以/结尾的请求，如果没有找到 index 页，则看是否开启了 random_index；如开启，则用之，否则用 autoindex
- ngx_http_auth_basic #基于 http 的身份认证 (auth_basic)
- ngx_http_access #基于 IP 地址的访问控制 (deny,allow)
- ngx_http_limit_conn #限制来自客户端的连接的响应和处理速率
- ngx_http_limit_req #限制来自客户端的请求的响应和处理速率
- ngx_http_geo
- ngx_http_map #创建任意的键值对变量
- ngx_http_split_clients
- ngx_http_referer #过滤 HTTP 头中 Referer 为空的对象
- ngx_http_rewrite #通过正则表达式重定向请求
- ngx_http_proxy
- ngx_http_fastcgi #支持 fastcgi
- ngx_http_uwsgi
- ngx_http_scgi
- ngx_http_memcached
- ngx_http_empty_gif #从内存创建一个 1×1 的透明 gif 图片，可以快速调用
- ngx_http_browser #解析 http 请求头部的 User-Agent 值
- ngx_http_charset #指定网页编码
- ngx_http_upstream_ip_hash
- ngx_http_upstream_least_conn
- ngx_http_upstream_keepalive
- ngx_http_write_filter
- ngx_http_header_filter
- ngx_http_chunked_filter
- ngx_http_range_header
- ngx_http_gzip_filter
- ngx_http_postpone_filter
- ngx_http_ssi_filter
- ngx_http_charset_filter
- ngx_http_userid_filter
- ngx_http_headers_filter #设置 http 响应头
- ngx_http_copy_filter
- ngx_http_range_body_filter
- ngx_http_not_modified_filter

**可选 HTTP 模块**:

- ngx_http_addition #在响应请求的页面开始或者结尾添加文本信息
- ngx_http_degradation #在低内存的情况下允许服务器返回 444 或者 204 错误
- ngx_http_perl
- ngx_http_flv #支持将 Flash 多媒体信息按照流文件传输，可以根据客户端指定的开始位置返回 Flash
- ngx_http_geoip #支持解析基于 GeoIP 数据库的客户端请求
- ngx_google_perftools
- ngx_http_gzip #gzip 压缩请求的响应
- ngx_http_gzip_static #搜索并使用预压缩的以.gz 为后缀的文件代替一般文件响应客户端请求
- ngx_http_image_filter #支持改变 png，jpeg，gif 图片的尺寸和旋转方向
- ngx_http_mp4 #支持.mp4,.m4v,.m4a 等多媒体信息按照流文件传输，常与 ngx_http_flv 一起使用
- ngx_http_random_index #当收到 / 结尾的请求时，在指定目录下随机选择一个文件作为 index
- ngx_http_secure_link #支持对请求链接的有效性检查
- ngx_http_ssl #支持 https
- ngx_http_stub_status
- ngx_http_sub_module #使用指定的字符串替换响应中的信息
- ngx_http_dav #支持 HTTP 和 WebDAV 协议中的 PUT/DELETE/MKCOL/COPY/MOVE 方法
- ngx_http_xslt #将 XML 响应信息使用 XSLT 进行转换

**邮件服务模块**:

- ngx_mail_core
- ngx_mail_pop3
- ngx_mail_imap
- ngx_mail_smtp
- ngx_mail_auth_http
- ngx_mail_proxy
- ngx_mail_ssl

**第三方模块**：

- echo-nginx-module #支持在 nginx 配置文件中使用 echo/sleep/time/exec 等类 Shell 命令
- memc-nginx-module
- rds-json-nginx-module #使 nginx 支持 json 数据的处理
- lua-nginx-module

原文作者：chrootliu，腾讯 QQ 音乐前端开发工程师

原文链接：https://mp.weixin.qq.com/s/LmtHTOVOvdcnMBuxv7a9_A