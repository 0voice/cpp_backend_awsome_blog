# 【NO.16】Nginx源码分析与实践---编写一个简单的Http模块

本节通过亲自实践，写一个经典的Hello World模块来了解相应的流程是如何进行的。我们采用自上向下的方法，先让模块跑起来，再去向下分析模块的具体细节。

## 1.模块源码

准备工作：新建的模块名为ngx_http_mytest_module，于是在/home/zxin/nginx下新建个目录ngx_http_mytest_module。该目录下新建两个文件config，ngx_http_mytest_module.c。要将编写的模块加进nginx里，需要和nginx源码一起编译，因此，config就是告诉nginx如何将我们的模块编译进nginx。

ngx_http_mytest_module.c：

```
#include <ngx_core.h>
#include <ngx_http.h>
#include <ngx_config.h>
 
static char *ngx_http_mytest(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
static ngx_int_t ngx_http_mytest_handler(ngx_http_request_t *r);
 
/* 配置指令 */
static ngx_command_t ngx_http_mytest_commands[] = {
        {
             ngx_string("mytest"),
             NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LMT_CONF|NGX_CONF_NOARGS,
             ngx_http_mytest,
             NGX_HTTP_LOC_CONF_OFFSET,
             0,
             NULL },
 
        ngx_null_command
};
 
/* 模块上下文 */
static ngx_http_module_t ngx_http_mytest_module_ctx = {
        NULL,
        NULL,
 
        NULL,
        NULL,
 
        NULL,
        NULL,
 
        NULL,
        NULL
};
 
/* 模块定义 */
ngx_module_t ngx_http_mytest_module = {
        NGX_MODULE_V1,
        &ngx_http_mytest_module_ctx,
        ngx_http_mytest_commands,
        NGX_HTTP_MODULE,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NGX_MODULE_V1_PADDING
};
 
/* 挂载handler函数 */
static char *ngx_http_mytest(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
        ngx_http_core_loc_conf_t *clcf;
        clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
        clcf->handler = ngx_http_mytest_handler;
 
        return NGX_CONF_OK;
}
 
/* handler函数 */
static ngx_int_t ngx_http_mytest_handler(ngx_http_request_t *r)
{
        //必须是GET或HEAD方法
        if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD)))
            return NGX_HTTP_NOT_ALLOWED;
 
        //丢弃请求包体
        ngx_int_t rc = ngx_http_discard_request_body(r);
        if (rc != NGX_OK) {
            return rc;
        }
 
        ngx_str_t type = ngx_string("text/plain");//响应的Content-Type
        ngx_str_t response = ngx_string("Hello World !");//响应的包体
 
        r->headers_out.status = NGX_HTTP_OK;//响应的状态码
        r->headers_out.content_length_n = response.len;//包体的长度
        r->headers_out.content_type = type;//响应的Content-Type
 
        rc = ngx_http_send_header(r);//发送响应头部
        if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
            return rc;
        }
 
        ngx_buf_t *b;//构造ngx_buf_t结构，准备发送包体
        b = ngx_create_temp_buf(r->pool, response.len);
        if (b == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
 
        ngx_memcpy(b->pos, response.data, response.len);//将"Hello World !"拷贝到ngx_buf_t中
        b->last = b->pos + response.len;//设置last指针
        b->last_buf = 1;//声明是最后一块缓冲区
 
        ngx_chain_t out;//构造发送的ngx_chain_t结构
        out.buf = b;
        out.next = NULL;
 
        return ngx_http_output_filter(r, &out);//发送包体
}
```

config:

```
ngx_addon_name=ngx_http_mytest_module
HTTP_MODULES="$HTTP_MODULES ngx_http_mytest_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_mytest_module.c"
```

将模块编译进nginx内：

1.先进入目录：/usr/local/src/nginx-1.9.15

2.添加模块：./configure --add-module=/home/zxin/nginx/ngx_http_mytest_module

3.make

4.make install

然后修改我们的nginx.conf文件，在server{...}配置块内加入

```
location /test {
                mytest;
        }
```

重启nginx：

/usr/local/nginx/sbin/nginx -s reload

接着便可访问：[http://localhost/test](https://link.zhihu.com/?target=http%3A//localhost/test)

**相关视频推荐**

[手把手带你实现一个nginx模块，更加深入了解nginx（搭建好环境）](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1uU4y1j7i2/)

[16万行Nginx源码，就该这么读](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1hz411h7Go/)

[c/c++后台开发岗位，如何精进技术，8个维度来说清楚](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1gP411n7rJ/)

学习地址：**[c/c++ linux服务器开发/后台架构师](https://link.zhihu.com/?target=https%3A//ke.qq.com/course/417774%3FflowToken%3D1013300)**

需要C/C++ Linux服务器架构师学习资料加qun**[812855908](https://link.zhihu.com/?target=https%3A//jq.qq.com/%3F_wv%3D1027%26k%3DnVddKID0)**获取（资料包括**C/C++，Linux，golang技术，Nginx，ZeroMQ，MySQL，Redis，fastdfs，MongoDB，ZK，流媒体，CDN，P2P，K8S，Docker，TCP/IP，协程，DPDK，ffmpeg**等），免费分享

![img](https://pic3.zhimg.com/80/v2-0fb9145c6e395f48de7c1d3a1d500dc2_720w.webp)

## 2.实验结果

浏览器访问：

![img](https://pic4.zhimg.com/80/v2-62bedd02881737df53d74005b9f646ef_720w.webp)

linux curl 访问：

![img](https://pic1.zhimg.com/80/v2-5524fc1e9bcd9be8ec7992bc2fd40234_720w.webp)

## 3.模块分析

下面开始分析具体细节：

首先要知道nginx的配置是和模块相对应的，配置文件里有一个配置，则存在一个模块去处理这个配置。先看下面这幅流程图：

![img](https://pic3.zhimg.com/80/v2-77656ad2976f3bfc3a41bf8695f42d32_720w.webp)

1.在我们输入URL：http://localhost/test后，由HTTP框架去接收请求的HTTP包头，框架分析出包头的uri为/test，然后就去配置文件nginx.conf里去找相应的配置项/test。找到了我们的location块，和其/test相匹配。找到后则进入location，发现mytest配置项，则调用ngx_http_mytest_module模块去处理后面的请求。ngx_http_mytest_module模块处理完后，返回到HTTP框架，然后将响应返回给浏览器。下面看下模块内部是如何操作的。

2.模块和配置信息是紧密相关的。先看下配置指令相关的信息吧：

```
/* 配置指令 */
static ngx_command_t ngx_http_mytest_commands[] = {
        {
             ngx_string("mytest"),//配置项名称
             NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LMT_CONF|NGX_CONF_NOARGS,//位置及参数
             ngx_http_mytest,//挂载handler函数
             NGX_HTTP_LOC_CONF_OFFSET,//配置项所在内存池
             0,//配置所在精确位置
             NULL },//一般为NULL
 
        ngx_null_command
};
```

我们用一个结构数组来存储配置信息，数组元素为ngx_command_t。

第一项：用函数ngx_string()来设置配置名称，这里的名称应和nginx.conf配置文件里自定义的配置项名称相同。模块就是通过这个名称检测到配置的。

第二项：说明了配置项可以出现在哪些配置块以及配置项后可以跟的参数的个数。

第三项：ngx_http_mytest是用来挂载handler函数的函数，框架在解析配置的时候，会将解析到的参数都传给ngx_http_mytest这个函数，在其内部调用真正的处理请求的函数。即在此函数内调用handler函数。

第四项：该字段指定当前配置项存储的内存位置。实际上是使用哪个内存池的问题。因为http模块对所有http模块所要保存的配置信息，划分了main, server和location三个地方进行存储，每个地方都有一个内存池用来分配存储这些信息的内存。

第五项：如果location配置块内有多个配置项，就可以利用此项来精确定位每一项在location内的具体位置。因为我们只有一个配置项，所以可设为0。

第六项：一般为NULL

3.下面看下在挂载函数内部是如何挂载handler函数的：

```
/* 挂载handler函数 */
static char *ngx_http_mytest(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
        ngx_http_core_loc_conf_t *clcf;
        clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
        clcf->handler = ngx_http_mytest_handler;
 
        return NGX_CONF_OK;
}
```

首先调用ngx_http_conf_get_module_loc_conf，找到mytest配置项所在的配置块。接下来去挂载我们的handler函数ngx_http_mytest_handler。HTTP框架在解析到请求与我们的配置块相匹配时，就会调用handler函数来处理请求。关于挂载函数的部分见： Nginx源码分析与实践---挂载函数

挂载函数的具体实现放最后再说，配置相关的内容说完了。接下来是模块相关的内容：

4.首先是模块上下文：

```
/* 模块上下文 */
static ngx_http_module_t ngx_http_mytest_module_ctx = {
        NULL,
        NULL,
 
        NULL,
        NULL,
 
        NULL,
        NULL,
 
        NULL,
        NULL
};
```

这是一个ngx_http_module_t类型的静态变量。这个变量实际上是提供一组回调函数指针，这些函数有在创建存储配置信息的对象的函数，也有在创建前和创建后会调用的函数。这些函数都将被nginx在合适的时间进行调用。

然后是最重要的模块本身：

```
/* 模块定义 */
ngx_module_t ngx_http_mytest_module = {
        NGX_MODULE_V1,
        &ngx_http_mytest_module_ctx,
        ngx_http_mytest_commands,
        NGX_HTTP_MODULE,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NULL,
        NGX_MODULE_V1_PADDING
};
```

通过这个模块的定义，我们将模块本身、模块上下文以及配置指令就结合起来了。关于模块上下文和模块的详细介绍见：Nginx源码分析与实践---ngx_module_t/ngx_http_module_t

5.下面看下handler函数内部是如何处理请求的：

```
/* handler函数 */
static ngx_int_t ngx_http_mytest_handler(ngx_http_request_t *r)
{
        //必须是GET或HEAD方法
        if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD)))
            return NGX_HTTP_NOT_ALLOWED;
 
        //丢弃请求包体
        ngx_int_t rc = ngx_http_discard_request_body(r);
        if (rc != NGX_OK) {
            return rc;
        }
 
        ngx_str_t type = ngx_string("text/plain");//响应的Content-Type
        ngx_str_t response = ngx_string("Hello World !");//响应的包体
 
        r->headers_out.status = NGX_HTTP_OK;//响应的状态码
        r->headers_out.content_length_n = response.len;//包体的长度
        r->headers_out.content_type = type;//响应的Content-Type
 
        rc = ngx_http_send_header(r);//发送响应头部
        if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
            return rc;
        }
 
        ngx_buf_t *b;//构造ngx_buf_t结构，准备发送包体
        b = ngx_create_temp_buf(r->pool, response.len);
        if (b == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
 
        ngx_memcpy(b->pos, response.data, response.len);//将"Hello World !"拷贝到ngx_buf_t中
        b->last = b->pos + response.len;//设置last指针
        b->last_buf = 1;//声明是最后一块缓冲区
 
        ngx_chain_t out;//构造发送的ngx_chain_t结构
        out.buf = b;
        out.next = NULL;
 
        return ngx_http_output_filter(r, &out);//发送包体
}
```

请求的所有信息(如：方法、URI、协议版本号和头部等)都传给了结构体ngx_http_request_t，关于此结构体的详细信息见：Nginx源码分析与实践---ngx_http_request_t。该函数只处理GET和HEAD两种方法，其他方法不处理。请求由请求头和请求体组成，由于我们只需要请求头的信息，所以调用 ngx_http_discard_request_body(r)丢弃请求体。请求收到了，下面我们就可以构造响应的包头和包体了。响应的包头和包体要分别发送，r->headers_out成员是专门用于构造响应的包头的。该成员又是一个结构体，通过该结构体的成员，可设置包头的各项内容。包头构造结束后就调用ngx_http_send_header(r)来发回包头。对于包体的发送，我们是通过数据类型ngx_chain_t来完成的，而out是利用结构体ngx_buf_t来完成包体的存储的，我们将包体内容"Hello World !"存储到结构体b中，因此可以利用out来发送出去包体，构造好out后，便可调用函数ngx_http_output_filter(r, &out)来将包体发给浏览器。

## 4.总结

由上述讲解可知：模块开发最重要的就是配置信息和模块两部分的内容，它们是相对应的。正是因为Nginx将整个开发分为配置和模块两部分，使得我们在使用Nginx的时候，只用去简单修改下配置文件就可以得到我们想要的功能了。我们只需在配置文件中设置想要的配置项，剩下的Nginx全部会给你解决的很好的。当然前提是要有很好的模块支撑。看看源码src目录下的那些模块的实现，定义ngx_command_t、ngx_http_module_t、ngx_module_t这几个部分的内容肯定少不了的，这就是模块开发中的主要内容！

原文链接：https://zhuanlan.zhihu.com/p/583410850