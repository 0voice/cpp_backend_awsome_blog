# 【NO.579】Nginx反向代理与系统参数配置conf原理

## 1.nignx安装

### 1.1 安装步骤

安装pcre（用来正则表达式的）
下载地址 http://www.pcre.org/
安装zlib（用来压缩）
下载地址 http://www.zlib.net/
安装nginx（反向代理）
下载地址 http://www.nginx.net/

分别进入目录，执行以下命令就行

```
./configure
make 
sudo make install
```



### 1.2 查看

在/usr/local/nginx文件夹中，有4个文件

![在这里插入图片描述](https://img-blog.csdnimg.cn/0be8d03e57a04a1d8d05b2d2ca1912b5.png)

conf 存放配置文件
html 存放网页
logs 存放日志
sbin 运行时可执行程序

### 1.3 测试

```
./sbin/nginx -c conf/nginx.conf
```


在浏览器中，输入当前主机的ip地址即可
比如我的是192.168.192.128，输入到浏览器即可
在页面中可以看到下图内容（nginx中默认的）

![在这里插入图片描述](https://img-blog.csdnimg.cn/7f2ff520a7fe4c5b91d0e68ed66a6bef.png)

如果出现错误，查看端口是否被占用，如果占用就kill -9 [pid]

```
netstat -ntlp|grep 80
```


补充:
查看运行的nginx进程

```
ps aux|grep nginx
```


停止nginx

```
./sbin/nginx -s stop
```



## 2.配置conf文件

conf文件中有两类数据，一种是单行的，另外一种是block块

![在这里插入图片描述](https://img-blog.csdnimg.cn/09ddf854681d405899c23e79dc2bae61.png)

配置

### 2.1 worker_processes和worker_connections

worker_processes 4 表示会在主进程中启动4个子进程，也就是说总共会有5个进程，1个master进程，4个worker进程
worker_connections 1024;表示连接池的最大数量为1024，每一个work进程都有一个连接池，每个连接池最大数量是1024，如果有4个work进程，那么总共最大连接数是4096

```
worker_processes 4;
events {
	worker_connections 1024;
}
```


测试

启动该配置

在浏览器访问服务器ip，会看到connect refuse

然后查看进程可以看到，1个主进程和4个子进程

![在这里插入图片描述](https://img-blog.csdnimg.cn/0e2e19cbb5cd480e87750031989474e9.png)

查看网络状态，现在没有启动server，所以也就看不到东西

![在这里插入图片描述](https://img-blog.csdnimg.cn/38aaa689509346bc9c7438daf112dfc3.png)

### 2.2 http server

在原来基础上，再增加http server相关配置，
表示意思：启动一个http server，监听在端口8888上，

```
worker_processes 4;

events {
	worker_connections 1024;
}

http {
	server {
		listen 8888;
	}
}
```

测试

这时候再启动，进入浏览器，输入 192.168.192.128:8888是可以进入相关界面的，是nginx默认的页面

查看网络状态

第一行中代表 主进程，36384就是主进程的pid
但是第二行、第三行分别对应2个子进程，其中一个子进程是进行http请求，第二个子进程是(keepalive
)发送心跳包的

![在这里插入图片描述](https://img-blog.csdnimg.cn/1aa8cc13977846d3a138bf558951b5fa.png)

### 2.3 http中多个server

有4个server都去处理http协议

```
worker_processes 4;

events {
	worker_connections 1024;
}

http {
	server {
		listen 8888;
	}
	server {
		listen 8889;
	}
	server {
		listen 8890;
	}
	server {
		listen 8891;
	}
}
```

测试

可以发现4个端口都在一个进程（主进程）上面。listen在master进程上面，后面的处理在work进程上面

这时候访问192.169.192.128:8888，还是192.169.192.128:8889，都是nginx默认的界面，因为缺省了html

![在这里插入图片描述](https://img-blog.csdnimg.cn/4403b7d7fbe146d8bb8a090769dfbb71.png)

### 2.4 proxy_pass（反向代理）

根据http请求头，找到对应的location

![在这里插入图片描述](https://img-blog.csdnimg.cn/cc4fb8ffeda44b75ab320bf736979714.png)

也是11个状态里面的一个 ,NGX_HTTP_POST_REWRITE_PHASE

![在这里插入图片描述](https://img-blog.csdnimg.cn/53203820ad1b40a2879b95d4bc2ace67.png)

/前面是ip、端口或者 域名，/后面对应的是资源

![在这里插入图片描述](https://img-blog.csdnimg.cn/f353e349a67d4faba01581bda1349026.png)

下图中，server 8891访问的 html文件是/home/king/share/html
在server 8888中proxy_pass http://192.168.232.137:8891;重定向到8891。因此访问8888和8891是一样的内容

```
worker_processes 4;

events {
	worker_connections 1024;
}

http {
	server {
		listen 8888;
		location / {
			proxy_pass http://192.168.232.137:8891;
		}
	}
	server {
		listen 8889;
	}
	server {
		listen 8890;
	}

	server {
		listen 8891;
	
		location / {
			root /home/king/share/html;
			proxy_pass http://backend;
		}
	}

}
```



### 2.5 加入负载均衡

假设已经有两个已启动的nginx
通过设置upstream（把这个流转发出去的意思），，通过当前的主机去访问这两个服务器
proxy_pass http://backend;
去访问当前服务器的时候，就重定向到了指定的网页（upstream中的设置的两个）
weigth为权重，通过这种方式来实现负载均衡，weight可以自己调节

```
upstream backend {
		server 192.168.232.128 weight=2;
		server 192.168.232.132:8888 weight=1;
	}
	
server {
		listen 8891;

		location / {
			root /home/king/share/html;
			proxy_pass http://backend;
		}
	}


```

完整配置

```
#进程数量
worker_processes 4;

events {
	worker_connections 1024;
}

http {
	server {
		listen 8888;
		location / {
			proxy_pass http://192.168.232.137:8891;
		}
	}
	server {
		listen 8889;
	}
	server {
		listen 8890;
	}

	upstream backend {
		server 192.168.232.128 weight=2;
		server 192.168.232.132:8888 weight=1;
	}
	server {
		listen 8891;
	
		location / {
			root /home/king/share/html;
			proxy_pass http://backend;
		}
	}

}
```


测试

访问192.168.192.128:8891，可以发现，每3次就有2次进入192.168.232.128的页面，每3次就有1次进入192.168.232.132:8888

## 3.源码阅读

### 3.1 worker_processes设置

在nginx.conf中有

![在这里插入图片描述](https://img-blog.csdnimg.cn/f85ca96e474b4db692cf64a14da6fa9f.png)

ngx_core_commands中的命令如果找到nignx.conf配置中有话，那么就会执行相应的部分，比如下面"worker_processes"如果在conf文件中找到，那么就会执行ngx_set_worker_processes

```
static ngx_command_t  ngx_core_commands[] = {

	...
	
	{ ngx_string("worker_processes"),
	  NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE1,
	  ngx_set_worker_processes,
	  0,
	  0,
	  NULL },
	
	...

};
```


ngx_set_worker_processes
将解析的参数通过变量ccf->worker_processes赋值就行了

ngx_set_worker_processes
将解析的参数通过变量ccf->worker_processes赋值就行了

通过后续设置值，内容就存储到核心配置文件的变量里ngx_core_conf_t *ccf

```
static char *
ngx_set_worker_processes(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_str_t        *value;
    ngx_core_conf_t  *ccf;

    ccf = (ngx_core_conf_t *) conf;
    
    if (ccf->worker_processes != NGX_CONF_UNSET) {
        return "is duplicate";
    }
    
    value = cf->args->elts;
    
    if (ngx_strcmp(value[1].data, "auto") == 0) {
        ccf->worker_processes = ngx_ncpu;
        return NGX_CONF_OK;
    }
    //value[0].data就是 命令本身，value[1].data是命令参数  value[1]这里指的是线程数，是字符串形式，要转成int
    ccf->worker_processes = ngx_atoi(value[1].data, value[1].len);
    
    if (ccf->worker_processes == NGX_ERROR) {
        return "invalid value";
    }
    
    return NGX_CONF_OK;

}
```

### 3.2 http

![在这里插入图片描述](https://img-blog.csdnimg.cn/0e2eba2b9d3e4df8adafbc09211f3072.png)


同样也有http的命令，这是一个块NGX_CONF_BLOCK，也就是上图用花括号括起来，称为BLOCK。
通过读取conf中的http配置来启动http server，执行ngx_http_block

```
static ngx_command_t  ngx_http_commands[] = {

    { ngx_string("http"),
      NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
      ngx_http_block,
      0,
      0,
      NULL },
    
      ngx_null_command

};
```

ngx_http_block
ngx_http_block这个启动过程中，包含着tcp server和http协议的处理

在ngx_http_init_phase_handlers是http状态机启动，phase就是状态的意思，里面有很多枚举状态，如NGX_HTTP_SERVER_REWRITE_PHASE等

在ngx_http_block中，有ngx_http_optimize_servers就是启动tcp server，其中cmcf->ports是包含端口号的数组。

```
static char *
ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
	...
	//http状态机启动
	if (ngx_http_init_phase_handlers(cf, cmcf) != NGX_OK) {
	        return NGX_CONF_ERROR;
	    }
	//启动tcp server
	if (ngx_http_optimize_servers(cf, cmcf, cmcf->ports) != NGX_OK) {
	        return NGX_CONF_ERROR;
	    }
	...
}
```

如何添加黑白名单？
方法1：tcp连接后：可以在建立连接（三次握手）之后，看它的ip地址，根据黑白名单，来看要不要执行后面的步骤
方法2：处理http头阶段：不对ip作限制，比如对pos或get请求作处理，在处理http头的阶段
方法3：对具体body 表单部分，进行处理，确定该部分是否要发送
————————————————
版权声明：本文为CSDN博主「菊头蝙蝠」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_21539375/article/details/124736034