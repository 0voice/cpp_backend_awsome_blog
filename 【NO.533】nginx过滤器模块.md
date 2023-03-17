# 【NO.533】nginx过滤器模块

**什么是过滤器呢？**

你在申请网页的时候，打开浏览器
可以看到这里有两个字叫做广告，

![在这里插入图片描述](https://img-blog.csdnimg.cn/edd89fff2f81433bb34af658486439f3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



在我们申请网页的过程，这个广告是怎么做的？
我们去申请这个网页的时候，请注意广告的显示，它不是在我们请求的时候，在我们请求的过程中间，在http十一个阶段的时候，它是没有插入广告地方，在response里面这个过程中间它在哪个地方去插入广告？
就是在过滤器的时候

![在这里插入图片描述](https://img-blog.csdnimg.cn/00399bf7557947479d95a1e6e9acb4c8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_15,color_FFFFFF,t_70,g_se,x_16)



在这里讲另外一个新的方法，就是我们在过滤器中间同样也可以做，

![在这里插入图片描述](https://img-blog.csdnimg.cn/c598d979bfb74d3bab511e08d24536db.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



Handler模块：接收到请求并返回结果，

![在这里插入图片描述](https://img-blog.csdnimg.cn/f0ac3500615141dab20a40933242787f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_13,color_FFFFFF,t_70,g_se,x_16)


Upstream:

![在这里插入图片描述](https://img-blog.csdnimg.cn/653cd9e13c5b4ceebaa93bb827a77ef1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


Filter:

![在这里插入图片描述](https://img-blog.csdnimg.cn/512c329d147a40afb62f4f49b63215b0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



**过滤器模块它主要是用来做什么？**
过滤器模块主要起到的作用就是在response的时候返回结果的时候，我们加上一些想要的,加上一些特别的信息往里面

也就是我们从后端返回结果，然后再返回前端的时候，这时候我们可以加东西，比如说哪些功能，我们对一个结果，我们请求结果返回参数，我们对它作MD5的校验，验证对还是不对？

就是俄罗斯人的这个英文，我们不能用正常人的思维来理解，它过滤器其实是在这个过程中要去添加一些东西，把这个 filter也就在这个过程数据就像漏斗一样，然后往外流的时候我们可以这个进行删减添加
比如说我们想屏蔽一些一些参数，我们也可以通过过滤器去实现。
过滤器主要是作用在response返回的这个流程中间，

```cpp
void  *ngx_http_ads_filter_create_loc_conf(ngx_conf_t *cf);



char  *ngx_http_ads_filter_merge_loc_conf(ngx_conf_t *cf, void *prev, void *conf);



ngx_int_t ngx_http_ads_filter_init(ngx_conf_t *cf);



 



ngx_int_t ngx_http_ads_filter_header_filter(ngx_http_request_t *r);



ngx_int_t ngx_http_ads_filter_body_filter(ngx_http_request_t *r, ngx_chain_t *chain);



 



ngx_str_t ads_content = ngx_string("<h2>Author : King</h2><p><a href=\"http://www.0voice.com\">0voice</a></p>");



 



 



typedef struct {



    ngx_flag_t enable;



} ngx_http_ads_filter_conf_t;



 



 



// nginx.conf --> add_ads on/off



static ngx_command_t ngx_http_ads_filter_cmd[] = {



    // 从两个维度来描述command,



    //第一个就是形容command在conf文件的那个位置？ 那个位置怎么理解，



    //你是放location里面还是server里面



    //第二个维度就是他带的参数，可以带一个参数两个参数三个参数或者on,off参数



    {



        ngx_string("add_ads"), // on / off



        NGX_HTTP_LOC_CONF | NGX_CONF_FLAG,



        ngx_conf_set_flag_slot,



        NGX_HTTP_LOC_CONF_OFFSET,



        offsetof(ngx_http_ads_filter_conf_t, enable),



        NULL



    },



    ngx_null_command



}



 



// nginx -c nginx.conf



static ngx_http_module_t ngx_http_ads_filter_ctx = {



 



    NULL, //pre



    ngx_http_ads_filter_init, //post



 



    NULL, 



    NULL,



 



    NULL,



    NULL,



 



    ngx_http_ads_filter_create_loc_conf, //nginx.conf 



    ngx_http_ads_filter_merge_loc_conf,  // 



    



 



}



 



// config --> 



// ./configure --> objs/ngx_modules.c 



ngx_module_t ngx_http_ads_filter_module = {



    NGX_MODULE_V1,



    &ngx_http_ads_filter_ctx,



    ngx_http_ads_filter_cmd,



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



 



static ngx_http_output_header_filter_pt ngx_http_next_header_filter;



static ngx_http_output_body_filter_pt ngx_http_next_body_filter;



 



// nginx -c nginx.conf



ngx_int_t ngx_http_ads_filter_init(ngx_conf_t *cf) {



 



    // head



    ngx_http_next_header_filter = ngx_http_top_header_filter;



    ngx_http_top_header_filter = ngx_http_ads_filter_header_filter;



 



    // body



    ngx_http_next_body_filter = ngx_http_top_body_filter;



    ngx_http_top_body_filter = ngx_http_ads_filter_body_filter;



}



 



// content_length + strlen(add_ads)



ngx_int_t ngx_http_ads_filter_header_filter(ngx_http_request_t *r) {



 



    //ngx_http_ads_filter_conf_t *conf = ngx_http_get_module_loc_conf(r, ngx_http_ads_filter_module)



    



    if (r->headers_out.content_length_n > 0) {



        r->headers_out.content_length_n += ads_content.len;



    }



    return ngx_http_next_header_filter(r);



}



 



// add content 



// content.str



 



// html = h1 + h2 + h3 + h4



 



// send(fd, html, length, 0);



 



 



// send(fd, h1);



// send(h2);



// send(h3);



// send(h4);



 



 



 



ngx_int_t ngx_http_ads_filter_body_filter(ngx_http_request_t *r, ngx_chain_t *chain) {



 



    // ads_content --> body



    //if () {



    //    return ngx_http_next_body_filter(r, chain);



    //}



 



    ngx_buf_t *b = ngx_create_temp_buf(r->pool, ads_content->len);



    b->start = b->pos = ads_content.data;



    b->last = b->pos + ads_content.len;



    



    // html = h1 + h2 + h3 + h4类似



 



    ngx_chain_t *cl = ngx_alloc_chain_link(r->pool);



    cl->buf = b;



    cl->next = chain;



    



    return ngx_http_next_body_filter(r, cl);



    //return ngx_http_ads_filter_body_filter(r, chain);



 



}



 



 
```

从两个维度来描述command,第一个就是形容command在conf文件的那个位置？ 那个位置怎么理解，你是放location里面还是server里面
第二个维度就是他带的参数，可以带一个参数两个参数三个参数或者on,off参数

container_of这个宏就是通过成员找结构体，通过这个成员变量的地址去找到结构体的指针



![在这里插入图片描述](https://img-blog.csdnimg.cn/b72ba9e51c49437ba53e0274f2781dda.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



现在只要清楚这个执行完了之后，我们这个enable里面就有值就可以了。



![在这里插入图片描述](https://img-blog.csdnimg.cn/a4b4eeddb35a4438bbea84dd30afec3d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



这个过滤器它的入口函数在哪里？这个请求我们怎么就使得他能够请求到我们对应的这个模块里面，我们在整个配置文件
这个结束完了之后，我们还要实现一个东西，我们去初始化这个过滤器模块的入口，也就是说这个入口是在这个配置环节解析完以后，nginx -c nginx.conf执行完后，再去执行ngx_http_ads_filter_init

接下来给大家写的就是函数的编写。



![在这里插入图片描述](https://img-blog.csdnimg.cn/95e56ec5904f4094ae0b3259ee3a55db.png)



三个过滤性模板怎么组织起来的，我在加一个模块怎么加入呢？
首先这些模块已经是一个链了，过滤器的模块是通过头插法这个方式来做的
nginx提供了两个过滤器模块

![在这里插入图片描述](https://img-blog.csdnimg.cn/11dea5da5d3646c4bdce115bfcdc5554.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/54660a941b7f4218b36422cf389fe55e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_19,color_FFFFFF,t_70,g_se,x_16)





![在这里插入图片描述](https://img-blog.csdnimg.cn/0b2409bc17344f1489516d40d1bed2a0.png)



过滤器模块请注意是一个结构体里面，它指向一个next，那这个next里面有一个指针，有一个函数指针，它是一个函数调用，每一个结构体里面都有一个函数指针，我们通过for循环去遍历这里头每一个节点，遍历到节点一的时候调用它的回调函数，遍历节点二的时候调用它的回调函数，这个过程中间就形成了一种责任链的方式来理解

调用完ngx_http_ads_filter_init以后会去分配一个节点，在这个节点里面它有个指指向next，然后再把当前的Top的这个指针赋值给这个结构体里面这个函数指针，从而在我们循环的时候，它就能够走到这个函数

我们任何一个模块在跑起来的时候，他有两个地方会跑到模块里面跑到，第一个就是在nginx初始化的时候，就在nginx启动的时候，这时候主要是去解析这个配置文件，那配置文件的信息拿到对应的模块里面，
这是第一个地方

这个模块在这个配置文件解析完了以后，会去设置对应的每一次请求的入口函数，也就是说这两个地方，
一个就是解析conf的数据放到模块内，并且设置每一次请求的入口函数。
第二个就是在我们每一次请求的时候，它能够走到对应这个函数。
每一个模块都在执行的时候，都是这两个地方

如果去捋这个流程的时候，就看哪一些是这个解析配置文件的时候走的，哪一些它是在我们每一次请求的时候走的



![在这里插入图片描述](https://img-blog.csdnimg.cn/3bbf931949f5445cad353555013edc37.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



Handler他是拿到了请求的信息去修改，
后端返回过来的信息，我们对后端的信息修改用过滤器

抓住两个点，一个初始化的地方，另外一个是每次请求以及它的具体在运行时走的地方，

http入口在哪个地方
找到这个关键字，

第一个哪些是在初始化的时候，
第二个就是我们现在的入口函数在哪个地方。
第三个就是每次请求



![在这里插入图片描述](https://img-blog.csdnimg.cn/3e348229da82478287e1977b1d6b632b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



这个ngx_http_block这个函数它是在什么时候执行的？
是在配置文件的时候执行还是在我们每一次请求是执行？

原文作者：[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605225430