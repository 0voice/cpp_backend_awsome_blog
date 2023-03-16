# 【NO.476】nginx自定义实现一个计量模块

## 1.意义

HTTP 过滤模块与普通模块的功能是i完全不同的，下面先回顾一下HTTP模块有哪些功能。
首先HTTP框架为HTTP请求的处理过程定义了11个阶段，相关代码如下：

```
typedef enum {
    // server 读完之后怎么处理
    NGX_HTTP_POST_READ_PHASE = 0,   // ngx_http_init_phases

    // 读完之后对中间数据修改
    NGX_HTTP_SERVER_REWRITE_PHASE,  // ngx_http_init_phases
    
    // load 配置文件
    // 重写
    // 重写之后的操作
    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,         // ngx_http_init_phases
    NGX_HTTP_POST_REWRITE_PHASE,
    
    // 读取文件之前、之中、之后怎么操作
    NGX_HTTP_PREACCESS_PHASE,       // ngx_http_init_phases
    NGX_HTTP_ACCESS_PHASE,          // ngx_http_init_phases
    NGX_HTTP_POST_ACCESS_PHASE,
    
    // 回复内容之前、之后的操作
    NGX_HTTP_PRECONTENT_PHASE,      // ngx_http_init_phases
    NGX_HTTP_CONTENT_PHASE,         // ngx_http_init_phases
    
    // log信息 
    NGX_HTTP_LOG_PHASE              // ngx_http_init_phases

} ngx_http_phases;
```


HTTP 框架允许普通的HTTP处理模块介入其中的7个阶段处理请求，但是通常大部分HTTP模块（官方模块或者第三方模块）都只在NGX_HTTP_CONTENT_PHASE 阶段处理请求。在这个阶段处理请求有一个特点，即HTTP模块有两种介入方法，第一种方法是，任一个HTTP模块会对所有的用户请求产生作用，第二种方法是，只对请求的URI 匹配了nginx.conf 中的某些location 表达式下的HTTP模块起作用。也就是说如果多个HTTP模块共同处理一个请求，则多半是由subrequest功能完成，即将原始请求分为多个子请求，每个子请求在由一个HTTP模块在NGX_HTTP_CONTENT_PHASE 阶段处理。

HTTP 框架允许普通的HTTP处理模块介入其中的7个阶段处理请求，但是通常大部分HTTP模块（官方模块或者第三方模块）都只在NGX_HTTP_CONTENT_PHASE 阶段处理请求。在这个阶段处理请求有一个特点，即HTTP模块有两种介入方法，第一种方法是，任一个HTTP模块会对所有的用户请求产生作用，第二种方法是，只对请求的URI 匹配了nginx.conf 中的某些location 表达式下的HTTP模块起作用。也就是说如果多个HTTP模块共同处理一个请求，则多半是由subrequest功能完成，即将原始请求分为多个子请求，每个子请求在由一个HTTP模块在NGX_HTTP_CONTENT_PHASE 阶段处理。

然而,HTTP过滤模块则不同于此，一个请求可以被任意多个HTTP 过滤模块处理。

## 2.实现一个自己的模块能用在哪些方面呢

黑白名单ip, 固定的ip 能访问
特定的请求不响应
流量控制
转发的功能
源ip地址修改
对具体的请求加一些信息
…
其实我们会发现上面实现模块对应的功能是对应http 对应的11个阶段， 对其中的某一个阶段或者多个阶段进行修改就是需要实现自己模块的功能了。

## 3.自定义模块

下面是自定义实现的一个ip 计量的HTTP 模块， 在N个客户端进行访问时候统计访问次数

### 3.1 首先需要明确下面几个问题

如何快速查找对应的客户端
工作在http 的哪一个阶段
客户端在多次请求时不同进程上，如何进行统计（多进程间如何共享）

### 3.2 用到的技术

数据结构 （nginx内部的红黑树 rbtree）
slab (共享内存)

### 3.3 nginx 中配置文件如何用代码结合

```
ngx_command_t ngx_http_location_count_cmd[] = {
    {
        ngx_string("count"),
        NGX_HTTP_LOC_CONF | NGX_CONF_NOARGS,   // 写在location里面， 不带参数
        ngx_http_location_count_create_cmd_set,
        NGX_HTTP_LOC_CONF_OFFSET,
        0,
        NULL
    }, 
    ngx_null_command                        // 结尾标识符, 类似字符串中的 \0
};
```

### 3.4 nginx 解析http 的顺序和对应的函数

```
static ngx_http_module_t ngx_http_location_count_ctx = {
    // 提供8个方法，解析配置文件的
    // 解析 http {}
    // 1 8 2 7 3 6 4 5 这种顺序解析的
    NULL,   // preconfiguration
    NULL,   // postconfiguration

    NULL,   // create_main_conf
    NULL,   //init_main_conf
    
    NULL,   // create_srv_conf
    NULL,   // merge_srv_conf
    
    ngx_http_location_count_create_loc_conf,   // create_loc_conf
    NULL,   // merge_loc_conf

};
```



### 3.5节点（客户端）的统计

节点（客户端）的统计

```
// 这里添加红黑树的初始化
ngx_rbtree_init(&conf->lcshm->rbtree, &conf->lcshm->sentinel,  \
gx_rbtree_insert_value);

// 节点的插入
node->key = key;
node->data = 1;
ngx_rbtree_insert(&conf->lcshm->rbtree, node);
    
// 遍历节点
ngx_rbtree_node_t *node = ngx_rbtree_min(conf->lcshm->rbtree.root, \
conf->lcshm->rbtree.sentinel);

do {
	// 提取node 信息
	// ...


	node = ngx_rbtree_next(&conf->lcshm->rbtree, node);

} while (node);
```

### 3.6 全部代码如下

```
#include <ngx_http.h>
#include <ngx_config.h>
#include <ngx_core.h>

typedef struct {
    ngx_rbtree_t rbtree;
    ngx_rbtree_node_t sentinel;
}ngx_http_location_count_shm_t;   // 共享内存

typedef struct  {

	ssize_t shmsize;
	ngx_slab_pool_t *pool;
	
	ngx_http_location_count_shm_t* lcshm;
	
	//ngx_uint_t interval;    // 后面参数的属性
	//ngx_uint_t client_count;

} ngx_http_location_conf_t;


static void* ngx_http_location_count_create_loc_conf(ngx_conf_t *cf);
static char *ngx_http_location_count_create_cmd_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
static ngx_int_t ngx_http_location_count_shm_zone_init(ngx_shm_zone_t *zone, void *data);
static ngx_int_t ngx_http_location_count_handler(ngx_http_request_t *r);

// static void ngx_http_location_count_rbtree_insert (ngx_rbtree_node_t *root,
//     ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel);

// 红黑树的遍历   方便html的显示各个客户端访问次数
static int ngx_encode_http_page_rb(ngx_http_location_conf_t *conf, char *html);

// 查找函数，在共享内存中找到相应的信息
// 查找    插入
static ngx_int_t ngx_http_pagecount_lookup(ngx_http_request_t *r, ngx_http_location_conf_t *conf, ngx_uint_t key);


ngx_command_t ngx_http_location_count_cmd[] = {
    {
        ngx_string("count"),
        NGX_HTTP_LOC_CONF | NGX_CONF_NOARGS,   // 写在location里面， 不带参数
        ngx_http_location_count_create_cmd_set,
        NGX_HTTP_LOC_CONF_OFFSET,
        0,
        NULL
    }, 
    ngx_null_command                        // 结尾标识符
};


static ngx_http_module_t ngx_http_location_count_ctx = {
    // 提供8个方法，解析配置文件的
    // 解析 http {}
    // 1 8 2 7 3 6 4 5 这种顺序解析的
    NULL,   // preconfiguration
    NULL,   // postconfiguration

    NULL,   // create_main_conf
    NULL,   //init_main_conf
    
    NULL,   // create_srv_conf
    NULL,   // merge_srv_conf
    
    ngx_http_location_count_create_loc_conf,   // create_loc_conf
    NULL,   // merge_loc_conf

};

ngx_module_t ngx_http_location_count_module = {
    NGX_MODULE_V1,                  // 前面7个参数
    &ngx_http_location_count_ctx,
    ngx_http_location_count_cmd,
    NGX_HTTP_MODULE,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NGX_MODULE_V1_PADDING  // 全部是填充 
};

static void* ngx_http_location_count_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_location_conf_t* conf = ngx_palloc(cf->pool, sizeof(ngx_http_location_conf_t));
    if (conf == NULL)
        return NULL;

    // ngx_log_error(NGX_LOG_EMERG, cf->log, ngx_errno, "ngx_http_location_count_create_loc_conf");
    return conf;

}

// nginx.conf --> count
// curl -I http://localhost/test
static char *ngx_http_location_count_create_cmd_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    // count 配置文件中的
    // 如果设置的多个参数是放在这里的  cf->args->elts

    // 由上面的带入进来
    ngx_http_location_conf_t* lconf = (ngx_http_location_conf_t*)conf;
    ngx_str_t name = ngx_string("location_count_slab");
    lconf->shmsize = 128 * 1024;
    
    // 向系统提交一个我们分配这么大的内存给我们自定义模块使用
    // 这里是提交申请
    ngx_shm_zone_t* zone = ngx_shared_memory_add(cf, &name, lconf->shmsize, &ngx_http_location_count_module);
    
    if(zone == NULL) // 分配失败
    {
        return NGX_CONF_ERROR;
    }
    
    // 真正申请
    zone->init =  ngx_http_location_count_shm_zone_init;
    zone->data = lconf;
    
    // handler 
    ngx_http_core_loc_conf_t *corecf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    corecf->handler = ngx_http_location_count_handler;


    // ngx_log_error(NGX_LOG_EMERG, cf->log, ngx_errno, "ngx_http_location_count_create_cmd_set");
    
    return NULL;

}


static ngx_int_t ngx_http_location_count_shm_zone_init(ngx_shm_zone_t *zone, void *data)
{

    ngx_http_location_conf_t* conf;
    
    //这里需要添加东西
    ngx_http_location_conf_t* oconf = data;
    conf = (ngx_http_location_conf_t*)zone->data;
    if(oconf) 
    {
        conf->lcshm = oconf->lcshm;
        conf->pool = oconf->pool;
        return NGX_OK;
    }
    // printf("ngx_http_location_count_shm_zone_init 00000\n");
    conf->pool = (ngx_slab_pool_t*)zone->shm.addr;
    conf->lcshm = ngx_slab_alloc(conf->pool, sizeof(ngx_http_location_count_shm_t));
    if(conf->lcshm == NULL)
    {
        return NGX_ERROR;
    }
    
    conf->pool->data = conf->lcshm;
    // printf("ngx_http_location_count_shm_zone_init 11111\n");


    // 这里添加红黑树的初始化
    ngx_rbtree_init(&conf->lcshm->rbtree, &conf->lcshm->sentinel, ngx_rbtree_insert_value);
    // ngx_rbtree_init(&conf->lcshm->rbtree, &conf->lcshm->sentinel, ngx_http_location_count_rbtree_insert);


    return NGX_OK;

}

ngx_int_t ngx_http_location_count_handler(ngx_http_request_t *r)
{
    u_char html[1024] = {0};
	int len = sizeof(html);
    ngx_uint_t key = 0;


    struct sockaddr_in* client_addr = (struct sockaddr_in*)r->connection->sockaddr;
    
    ngx_http_location_conf_t* conf = ngx_http_get_module_loc_conf(r, ngx_http_location_count_module);
    key = client_addr->sin_addr.s_addr;
    
    // key , value          // ip地址和访问次数
    ngx_shmtx_lock(&conf->pool->mutex);
    ngx_http_pagecount_lookup(r, conf, key);
    ngx_shmtx_unlock(&conf->pool->mutex);
    
    ngx_encode_http_page_rb(conf, (char*)html);
    
    // ngx_log_error(NGX_LOG_EMERG, r->connection->log, ngx_errno, "ngx_http_location_count_handler");
    
    // 返回显示
    
    // 发送http头
    r->headers_out.status = 200;
    ngx_str_set(&r->headers_out.content_type, "text/html");
    ngx_http_send_header(r);  
    
    // body  body中可能返回多个客户端访问次数的列表，是一个前端显示的过程
    ngx_buf_t* b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
    
    ngx_chain_t out;
    out.buf = b;
    out.next = NULL;
    
    b->pos = html;
    b->last = html + len;
    b->memory = 1;
    b->last_buf = 1;
    
    return ngx_http_output_filter(r, &out);

}

// 红黑树插入方法  ngx_rbtree_insert_value (直接用nginx中的函数)
#if 0
void ngx_http_location_count_rbtree_insert (ngx_rbtree_node_t *temp,
    ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel)
{
    ngx_rbtree_node_t **p;
   //ngx_http_testslab_node_t *lrn, *lrnt;

    for (;;)
    {
        if (node->key < temp->key)
        {
            p = &temp->left;
        }
        else if (node->key > temp->key) {
           	p = &temp->right;
        }
        else
        {
          	return ;
        }
     
        if (*p == sentinel)
        {
            break;
        }
     
        temp = *p;
    }
     
    *p = node;
     
    node->parent = temp;
    node->left = sentinel;
    node->right = sentinel;
    ngx_rbt_red(node);

}
#endif

// 红黑树的查找，插入
// 参数r 方便日志的打印
ngx_int_t ngx_http_pagecount_lookup(ngx_http_request_t *r, ngx_http_location_conf_t *conf, ngx_uint_t key)
{
    ngx_rbtree_node_t *node, *sentinel;

	node = conf->lcshm->rbtree.root;
	sentinel = conf->lcshm->rbtree.sentinel;
	
	while(node != sentinel)   //  node == sentinel 需要进行下面操作，在slab中分配节点
	{
	    if(key < node->key)
	    {
	        node = node->left;
	        continue;
	    } 
	    else if (key > node->key)
	    {
	        node = node->right;
	        continue;
	    }
	    else
	    {
	        node->data ++;   // 找到了,  需要做一个原子操作
	        return NGX_OK;
	    }
	}
	
	// 添加之前 需要在slab中分配一个节点
	node = ngx_slab_alloc_locked(conf->pool, sizeof(ngx_rbtree_node_t));
	if (NULL == node) {
		return NGX_ERROR;
	}
	
	node->key = key;
	node->data = 1;
	ngx_rbtree_insert(&conf->lcshm->rbtree, node);
	
	return NGX_OK;

}

static int ngx_encode_http_page_rb(ngx_http_location_conf_t *conf, char *html)
{

    sprintf(html, "<h1>Ip Access Count</h1>");
    strcat(html, "<h2>");
    
    ngx_rbtree_node_t *node = ngx_rbtree_min(conf->lcshm->rbtree.root, conf->lcshm->rbtree.sentinel);
    
    do {
    
    	char str[INET_ADDRSTRLEN] = {0};
    	char buffer[128] = {0};
    
    	sprintf(buffer, "req from : %s, count: %d <br/>",
    		inet_ntop(AF_INET, &node->key, str, sizeof(str)), node->data);
    
    	strcat(html, buffer);
    
    	node = ngx_rbtree_next(&conf->lcshm->rbtree, node);
    
    } while (node);
    
    strcat(html, "</h2>");
    
    return NGX_OK;

}
```


上述的编译命令，环境我全部放在自定义中的镜像中了， 现在可以动动小手就可以拉取学习的环境了~

上述的编译命令，环境我全部放在自定义中的镜像中了， 现在可以动动小手就可以拉取学习的环境了~

docker pull registry.cn-hangzhou.aliyuncs.com/aclj/nginx:1.0.1

如果需要项目环境源代码，可以参考此github， 后续可根据需求增加东西，欢迎star~

参考：
后台服务器：https://course.0voice.com/v1/course/intro?courseId=5&agentId=0
————————————————
版权声明：本文为CSDN博主「_刘小雨」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_39486027/article/details/124733639