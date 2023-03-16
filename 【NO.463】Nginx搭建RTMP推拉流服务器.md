# 【NO.463】Nginx搭建RTMP推拉流服务器

## 1.安装Nginx

在Mac上有一个很好用的包管理插件，名为homebrew。 具体的安装可以自行去搜索下。下面就借助Homebrew来安装Nginx。

### 1.1 首先是拉取Nginx

```text
$ brew tap home/nginx
```

### **1.2 执行安装**

```text
$ brew install nginx-full --with-rtmp-module
```

这里需要注意的就是后面的–with-rtmp-module参数，其意思就是带上rtmp的模块，这样我们才能借助Nginx实现一个rtmp的推拉流服务器。

安装过程中，homebrew或帮我们自动的安装如pcre,openssl等模块。因此相对于其他平台的安装方式或者源码安装方式，homebrew贼省心。

经过稍长的等待时间，带有rtmp模块的Nginx就安装好了。查看安装详情的命令为：

```text
brew info nginx-full
```

就可以看到具体的安装信息了，比如配置文件在哪里，可执行文件又在哪里。

我这里有如下路径：

\- 配置文件路径 /usr/local/etc/nginx/

\- web容器路径 /usr/local/var/www

\- 可执行文件路径/usr/local/Ceallar/nginx/

配置rtmp

在nginx.conf的HTTP节点后面添加一个同级别的rtmp接单。具体内容如下：

```text
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

error_log   /usr/local/var/logs/nginx/error.log debug;
pid         /usr/local/var/run/nginx.pid;

#pid        logs/nginx.pid;


events {
    worker_connections  256;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log   /usr/local/var/logs/access.log  main;
    #access_log  logs/access.log  main;

    sendfile        on;
    port_in_redirect off;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       8080;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            #root   html;
            root    /usr/local/var/www;
            index  index.html index.htm index.php;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/local/var/www;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        location ~ \.php$ {
            proxy_pass   http://127.0.0.1;
        }

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        location ~ \.php$ {
            fastcgi_intercept_errors on;
            #root           html;
            root            /usr/local/var/www;
            fastcgi_pass   127.0.0.1:9000;
            #fastcgi_pass unix:/run/php/php7.0-fpm.sock;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
    include servers/*;
}

rtmp {
server {
  listen 1935;
  chunk_size 4000;
  application rtmplive {
    live on;
    max_connections 1024;
  }

  application hls {
    live on;
    hls on;
    hls_path /usr/local/var/www/hls;
    hls_fragment 1s;
  }
}
}
```

最后面hls_path就是待会要用到的推流文件目录了。一般来说不必创建，如果出现文件夹权限问题的话，手动添加下可读可写权限就可以了。

## 2.安装ffmpeg

安装它在其他的平台上可能会超级费劲，但是在Mac上，有了homebrew，那就真的不是事了。

```text
➜  $/opt nginx brew install ffmpeg
Updating Homebrew...
==> Auto-updated Homebrew!
Updated 1 tap (caskroom/cask).

==> Installing dependencies for ffmpeg: lame, x264, xvid
==> Installing ffmpeg dependency: lame
==> Downloading https://homebrew.bintray.com/bottles/lame-3.99.5.high_sierra.bottle.1.tar.gz
######################################################################## 100.0%
==> Pouring lame-3.99.5.high_sierra.bottle.1.tar.gz
��  /usr/local/Cellar/lame/3.99.5: 27 files, 2MB
==> Installing ffmpeg dependency: x264
==> Downloading https://homebrew.bintray.com/bottles/x264-r2795.high_sierra.bottle.tar.gz
######################################################################## 100.0%
==> Pouring x264-r2795.high_sierra.bottle.tar.gz
��  /usr/local/Cellar/x264/r2795: 11 files, 3.2MB
==> Installing ffmpeg dependency: xvid
==> Downloading https://homebrew.bintray.com/bottles/xvid-1.3.4.high_sierra.bottle.tar.gz
######################################################################## 100.0%
==> Pouring xvid-1.3.4.high_sierra.bottle.tar.gz
��  /usr/local/Cellar/xvid/1.3.4: 10 files, 1.2MB
==> Installing ffmpeg
==> Downloading https://homebrew.bintray.com/bottles/ffmpeg-3.4.high_sierra.bottle.tar.gz
######################################################################## 100.0%
==> Pouring ffmpeg-3.4.high_sierra.bottle.tar.gz
��  /usr/local/Cellar/ffmpeg/3.4: 248 files, 50.9MB
```

## 3.安装VLC客户端

VLC客户端是一个很好用的可以拉流并进行读取的软件，Mac上挺好用的。

## 4.开始推流，拉流

### 4.1 推流

推流之前，先准备一个视频软件。我就直接用QQ的录屏来录制了一个视频，放在桌面上，名为demo.mp4

然后在命令行里面输入：

```text
ffmpeg -re -i 本地视频路径如(如/Users/changba/Desktop/Player/demo.mp4)  -vcodec copy -f flv rtmp://localhost:1935/rtmplive/home
```

> 这里rtmplive是上面的配置文件中,配置的应用的路径名称;后面的room可以随便写，待会使用拉流软件的时候把地址对应上就可以了。

### 4.2 rtmp的配置

输入完之后，就可以打开VLC客户端了。

### 4.3 推流效果

### 4.4 拉流

具体操作为：file–>>Open Network
然后在弹出的URL框中输入如下链接。

```text
rtmp://localhost:1935/rtmplive/home
```

记得对应上名字就可以了,大致的拉流效果大家可以自己试下

## 5.总结

至此，基于rtmp的推拉流的Nginx服务器就算是完成了。如果感兴趣不妨来尝试一下，其实还是挺有意思的。

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/451814135