# 【NO.216】Nginx-接入层Nginx架构及模块介绍

文章简介：

**1）帮助大家对Nginx有一定的认识**
**2）熟悉Nginx有哪些应用场景**
**3）熟悉Nginx特点和架构模型以及相关流程**
**4）熟悉Nginx定制化开发的几种模块分类**

## 1. Nginx简介以及特点

**Nginx简介:**

Nginx (engine x) 是一个高性能的web服务器和反向代理服务器，也是一个IMAP/POP3/SMTP服务器

- 俄罗斯程序员Igor Sysoev于2002年开始
- Nginx是增长最快的Web服务器，市场份额已达33.3％
- 全球使用量排名第二2011年成立商业公司

**Nginx社区分支：**

- Openresty作者 [@agentzh](https://github.com/agentzh)(章宜春)开发的，最大特点是引入了ngx_lua模块，支持使用lua开发插件，并且集合了很多丰富的模块，以及lua库。
- Tengine主要是淘宝团队开发。特点是融入了因淘宝自身的一些业务带来的新功能。
- Nginx官方版本，更新迭代比较快，并且提供免费版本和商业版本。

**Nginx源码结构：**

- 代码量大约11万行C代码
- 源代码目录结构
- - core （主干和基础设置）
  - event （事件驱动模型和不同的IO复用模块）
  - http （HTTP服务器和模块）
  - mail （邮件代理服务器和模块）
  - os （操作系统相关的实现）
  - misc （杂项）

**Nginx特点：**

- 反向代理，负载均衡器
- 高可靠性、单master多worker模式
- 高可扩展性、高度模块化
- 非阻塞
- 事件驱动
- 低内存消耗
- 热部署

## 2. Nginx应用场景

**场景如下：**

- 静态文件服务器
- 反向代理，负载均衡
- 安全防御
- 智能路由(企业级灰度测试、地图POI一键切流)
- 灰度发布
- 静态化
- 消息推送
- 图片实时压缩
- 防盗链

## 3. Nginx框架模型介绍

**进程组件角色：**

- master进程
- - 监视工作进程的状态
  - 当工作进程死掉后重启一个新的
  - 处理信号和通知工作进程
- worker进程
- - 处理客户端请求
  - 从主进程处获得信号做相应的事情
- cache loader进程
- - 加载缓存索引文件信息，然后退出
- cache manager进程
- - 管理磁盘的缓存大小，超过预定值大小后最少使用数据将被删除

**框架模型：**

![img](https://pic1.zhimg.com/80/v2-6cc21dc22cb69f589b2331c5a708a338_720w.webp)

**框架模型流程：**

![img](https://pic1.zhimg.com/80/v2-f635e3ab2d216e1d52fcad678aeb1634_720w.webp)

## 4. Nginx内部流程介绍

### 4.1. 框架模型流程

![img](https://pic4.zhimg.com/80/v2-cfb33c24986d001c289d94232c76bcef_720w.webp)

![img](https://pic2.zhimg.com/80/v2-378f87b6eb04da3cbe84cf417e6980e1_720w.webp)

### 4.2. master初始化流程

![img](https://pic2.zhimg.com/80/v2-f9ca34cf514787041ab71c592b79f2c5_720w.webp)

### 4.3. worker初始化

![img](https://pic3.zhimg.com/80/v2-ae7b6abff4be9b31f39e5699d510f8c2_720w.webp)

### 4.4. worker初始化流程

![img](https://pic1.zhimg.com/80/v2-1a1be68c874107712f98573318ed69fc_720w.webp)

### 4.5. 静态文件请求IO流程

![img](https://pic2.zhimg.com/80/v2-5d685acdb3bec4588db1c3efcefe4601_720w.webp)

### 4.6. http请求流程

![img](https://pic1.zhimg.com/80/v2-2f20e35650f447c7b68bbf461e1225d4_720w.webp)

### 4.7. http请求11个阶段

![img](https://pic4.zhimg.com/80/v2-2a1db7e6ed3b2f1fb48a7382a6966ac7_720w.webp)

### 4.8. upstream模块

- 访问第三方Server服务器
- 底层HTTP通信非常完善
- 异步非阻塞
- 上下游内存零拷贝，节省内存
- 支持自定义模块开发

#### 4.8.1 upstream框架流程

![img](https://pic1.zhimg.com/80/v2-16827e0fbdc6f621925c08f7a0f6e210_720w.webp)

#### 4.8.2 upstream内部流程

![img](https://pic2.zhimg.com/80/v2-735b470e9498c78f2b893f057c2127dd_720w.webp)

### 4.9 反向代理流程

![img](https://pic2.zhimg.com/80/v2-4512a75c9866c2fca26c2ca215a9d039_720w.webp)

## 5. Nginx定制化模块开发

### 5.1. Nginx的模块化设计特点

- 高度抽象的模块接口
- 模块接口非常简单，具有很高的灵活性
- 配置模块的设计
- 核心模块接口的简单化
- 多层次、多类别的模块设计

### 5.2. 内部核心模块

![img](https://pic4.zhimg.com/80/v2-0a2f149846bb6d463e70fb0c74ea3e97_720w.webp)

![img](https://pic1.zhimg.com/80/v2-9fe9c202c6c489c6a742176fa666d144_720w.webp)

### 5.3. handler模块

- 接受来自客户端的请求并构建响应头和响应体。

![img](https://pic4.zhimg.com/80/v2-d9e02f5fde12fedd98dde9d8a4d6d17b_720w.webp)

### 5.4. filter模块

- 过滤（filter）模块是过滤响应头和内容的模块，可以对回复的头和内容进行处理。它的处理时间在获取回复内容之后，向用户发送响应之前。

![img](https://pic3.zhimg.com/80/v2-4973e0693ac310c6feac07640d6fc746_720w.webp)

### 5.5. upstream模块

- 使nginx跨越单机的限制，完成网络数据的接收、处理和转发，纯异步的访问后端服务。

![img](https://pic4.zhimg.com/80/v2-72fbc635533e1a898b568cf95687e923_720w.webp)

**load_balance：**

- 负载均衡模块，实现特定的算法，在众多的后端服务器中，选择一个服务器出来作为某个请求的转发服务器。

![img](https://pic4.zhimg.com/80/v2-36296ebc2f69ba725d531ecca66776c7_720w.webp)

### 5.6. ngx_lua模块

- 脚本语言
- 内存开销小
- 运行速度快
- 强大的 Lua 协程
- 非阻塞
- 业务逻辑以自然逻辑书写

![img](https://pic2.zhimg.com/80/v2-7e33a6c32db06083ed320598c4b2887d_720w.webp)

### 5.7. 定制化开发Demo

**Handler模块：**

- 编写config文件
- 编写模块产生内容响应信息

1. \#配置文件：
2. server {
3. …
4. location test {
5. test_counter on;
6. }
7. }
8. \#config
9. ngx_addon_name=ngx_http_test_module
10. HTTP_MODULES=”$HTTP_MODULES ngx_http_test_module”
11. NGX_ADDON_SRCS=”$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_test_module.c”
12. \#ngx_http_test_module.c
13. staticngx_int_t
14. ngx_http_test_handler(ngx_http_request_t*r)
15. {
16. ngx_int_t rc;
17. ngx_buf_t*b;
18. ngx_chain_tout;
19. ngx_http_test_conf_t*lrcf;
20. ngx_str_t ngx_test_string = ngx_string(“hello test”);
21. lrcf = ngx_http_get_module_loc_conf(r, ngx_http_test_module);
22. if( lrcf->test_counter ==0){
23. return NGX_DECLINED;
24. }
25. /* we response to ‘GET’ and ‘HEAD’ requests only */
26. if(!(r->method &(NGX_HTTP_GET|NGX_HTTP_HEAD))){
27. return NGX_HTTP_NOT_ALLOWED;
28. }
29. /* discard request body, since we don’t need it here */
30. rc = ngx_http_discard_request_body(r);
31. if( rc != NGX_OK ){
32. return rc;
33. }
34. /* set the ‘Content-type’ header */
35. /*
36. *r->headers_out.content_type.len = sizeof(“text/html”) - 1;
37. *r->headers_out.content_type.data = (u_char *)”text/html”;
38. */
39. ngx_str_set(&r->headers_out.content_type,”text/html”);
40. /* send the header only, if the request type is http ‘HEAD’ */
41. if( r->method == NGX_HTTP_HEAD ){
42. r->headers_out.status = NGX_HTTP_OK;
43. r->headers_out.content_length_n = ngx_test_string.len;
44. return ngx_http_send_header(r);
45. }
46. /* set the status line */
47. r->headers_out.status = NGX_HTTP_OK;
48. r->headers_out.content_length_n = ngx_test_string.len;
49. /* send the headers of your response */
50. rc = ngx_http_send_header(r);
51. if( rc == NGX_ERROR || rc > NGX_OK || r->header_only ){
52. return rc;
53. }
54. /* allocate a buffer for your response body */
55. b = ngx_pcalloc(r->pool,sizeof(ngx_buf_t));
56. if( b == NULL ){
57. return NGX_HTTP_INTERNAL_SERVER_ERROR;
58. }
59. /* attach this buffer to the buffer chain */
60. out.buf = b;
61. out.next= NULL;
62. /* adjust the pointers of the buffer */
63. b->pos = ngx_test_string.data;
64. b->last= ngx_test_string.data + ngx_test_string.len;
65. b->memory =1;/* this buffer is in memory */
66. b->last_buf =1;/* this is the last buffer in the buffer chain */
67. /* send the buffer chain of your response */
68. return ngx_http_output_filter(r,&out);
69. }

## 6. Nginx核心时间点模块介绍

解决接入层故障定位慢的问题，帮助OP快速判定问题根因，优先自证清白，提高接入层高效的生产力。

![img](https://pic2.zhimg.com/80/v2-891074a145f92b5d0c367823ca619681_720w.webp)

## 7. Nginx分流模块介绍

**特点：**
实现非常灵活的动态的修改策略从而进行切流量。
实现平滑无损的方式进行流量的切换。
通过秒级切换流量可以缩小影响范围，从而减少损失。
按照某一城市或者某个特征，秒级进行切换流量或者禁用流量。
容忍单机房级别容量故障，缩短了单机房故障的止损时间。
快速的将流量隔离或者流量抽样。
高效的灰度测试，提高生产力。

![img](https://pic2.zhimg.com/80/v2-1049b286fa13f53b4bbf814dfe406419_720w.webp)

## 8. Nginx动态upstream模块介绍

让接入层可以适配动态调度的云环境，实现服务的平滑上下线、弹性扩/缩容。
从而提高接入层高效的生产力以及稳定性，保证业务流量的平滑无损。

![img](https://pic2.zhimg.com/80/v2-dddc164ce367d6cf0733e98db42c5249_720w.webp)

## 9. Nginx query_upstream模块介绍

链路追踪，梳理接口到后端链路的情况。查询location接口对应upstream server信息。

![img](https://pic2.zhimg.com/80/v2-121ef7f1c9c7a8fc8d87d4c5a43547d1_720w.webp)

## 10. Nginx query_conf模块介绍

获取nginx配置文件格式化为json格式信息。

![img](https://pic2.zhimg.com/80/v2-87dbc0cf71ae43ec8656726cb0e715a1_720w.webp)

## 11.Nginx 共享内存支持redis协议模块介绍

根据配置文件来动态的添加共享内存。
[http://github.com/lidaohang/n](https://link.zhihu.com/?target=http%3A//github.com/lidaohang/n)…

- ngx_shm_dict
  共享内存核心模块(红黑树，队列)
- ngx_shm_dict_manager
  添加定时器事件，定时的清除共享内存中过期的key
  添加读事件，支持redis协议，通过redis-cli get,set,del,ttl
- ngx_shm_dict_view
  共享内存查看

![img](https://pic1.zhimg.com/80/v2-95e9acb1c6d279e3ad2c97a354558d4c_720w.webp)

## 12. Nginx日志回放压测工具

- 解析日志进行回放压测，模拟后端服务器慢等各种异常情况
  [http://github.com/lidaohang/p](https://link.zhihu.com/?target=http%3A//github.com/lidaohang/p)…

## 13.方案说明

- 客户端解析access.log构建请求的host,port,url,body
- 把后端响应时间，后端响应状态码，后端响应大小放入header头中
- 后端服务器获取相应的header，进行模拟响应body大小，响应状态码，响应时间

## 14.使用方式

- 拷贝需要测试的access.log的日志到logs文件夹里面
- 搭建需要测试的nginx服务器，并且配置upstream 指向后端服务器断端口
- 启动后端服务器实例 server/backserver/main.go
- 进行压测 bin/wrk -c30 -t1 -s conf/nginx_log.lua [http://localhost:8095](http://localhost:8095/)

原文链接：https://zhuanlan.zhihu.com/p/372403172

作者：Hu先生的Linux