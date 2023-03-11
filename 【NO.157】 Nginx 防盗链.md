# 【NO.157】 Nginx 防盗链

> 本篇主要介绍一下 nginx 中 防盗链的具体配置 , 以及http 的 referer 头

![image-20221205135337893](https://img2023.cnblogs.com/other/1898722/202212/1898722-20221213091423499-169252935.png)

## 1.概述

```
防盗链其实就是 防止别的站点来引用你的 资源, 占用你的流量
```

在了解nginx 防盗链之前 我们先了解一下 什么是 HTTP 的头信息 Referer,当浏览器访问网站的时候,一般会带上Referer,`告诉后端该是从哪个页面过来的`

nginx的 防盗链'功能基于 HTTP协议的Referer机制,通过判断Referer对来源进行 识别和判断 做出一定的处理

nginx会通就过`查看referer自动和valid_referers后面的内容进行匹配`，如果匹配到了就将`invalid_referer 变量置位1` ， 如果没有匹配到 ， 则 将 invalid_referer变量置0 , 匹配的过程中不区分大小写.

| 语法   | valid_referers none blocked server_names string… |
| ------ | ------------------------------------------------ |
| 默认值 | —                                                |
| 位置   | server、location                                 |

## 2.Nginx 防盗链演示

这里我拿图片等资源 作为案例演示

### 2.1 未配置 防盗链的情况

> 如果你的图片没有做防盗链的控制 , 像如下配置一样, 那么其他人就可以直接使用你的文件图片等等

```properties
http {
    include       mime.types;

    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;


       server {
        listen       80;
        server_name  www.testfront.com;
				
        location ~* .(gif|jpg|png) {
           root /www/static;
        }

        location / {
            proxy_pass http://www.testbackend.com;
        }
    }

}
```

可以看到在其他机器上如 www.testbackend.com 直接引入 www.testfront.com的资源文件 , 也是可以展示的

![image-20221205132512265](https://img2023.cnblogs.com/other/1898722/202212/1898722-20221213091423917-1213037371.png)

### 2.2 配置nginx防盗链

如果配置了valid_referers

nginx会通就过`查看referer自动和valid_referers后面的内容进行匹配`，如果匹配到了就将`invalid_referer 变量置位1` ， 如果没有匹配到 ， 则 将 invalid_referer变量置0 , 匹配的过程中不区分大小写.

```properties
    location ~* .(gif|jpg|png) {
       # 配置校验 referer , 意思就是如果referer 是172.16.225.111 或者 www.testfront.com 都通过
       valid_referers 172.16.225.111 www.testfront.com;
       if ($invalid_referer) {
          return 403;
       }
       root /www/static;
    }
```

此时再访问 www.testbackend.com 去引用 www.testfront.com 的资源 就不能访问了

![image-20221205133441066](https://img2023.cnblogs.com/other/1898722/202212/1898722-20221213091424461-1384825207.png)

## 3. 防盗链的 具体配置

从上面可以看出, 通过配置 valid_referers 后面添加校验的域名和ip , nginx 会自动进行 http的 referer 的匹配

```
防盗链除了可以配置 ip 域名外, 还能配置 如 none 和 blocked
```

- **none**: 如果Header中的Referer为空，允许访问

- blocked:在Header中的Referer不为空，但是该值被防火墙或代理进行伪装过，如不带"http://" 、"https://"等协议头的资源允许访问。

- server_names:指定具体的域名或者IP

  可以支持正则表达式和*的字符串。如果是正则表达式，需要以~开头表示

## 4.扩展Curl 访问

可以通过curl 直接进行访问 并且指定 referer 来快速测试

```
curl -i 只看返回头信息
```

![image-20221205134817848](https://img2023.cnblogs.com/other/1898722/202212/1898722-20221213091424753-2031673148.png)

```
curl -e "" 指定 referer
```

![image-20221205134908232](https://img2023.cnblogs.com/other/1898722/202212/1898722-20221213091425637-297318403.png)

## 5.总结

本篇主要介绍了 nginx中如何配置 防盗链, 来限制别人随意的引用你的资源 造成占用你的网络资源情况, 配置还是毕竟简单的.

原文作者：[Johnny小屋](https://www.askajohnny.com/)

原文链接：https://www.cnblogs.com/askajohnny/p/16977685.html