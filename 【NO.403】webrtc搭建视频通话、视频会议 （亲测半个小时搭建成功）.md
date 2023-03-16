# 【NO.403】webrtc搭建视频通话、视频会议 （亲测半个小时搭建成功）

## 1. 搭建前置条件

- 首先你需要有一台linux服务器，windows的也可以，请自行搞定
- 一些 简单工具应该先装好
  如：git、make、gcc之类的

## 2. 安装node和npm

下载官网最新nodejs：[https://nodejs.org/en/download/](https://link.zhihu.com/?target=https%3A//nodejs.org/en/download/)

```text
wget https://nodejs.org/dist/v10.16.0/node-v10.16.0-linux-x64.tar.xz
```

安装

```text
# 解压
tar -xvf node-v10.16.0-linux-x64.tar.xz
# 改名
mv node-v10.16.0-linux-x64 nodejs
# 进入目录
cd nodejs/

# 确认一下nodejs下bin目录是否有node和npm文件，如果有就可以执行软连接
sudo ln -s /home/dds/webrtc/nodejs/bin/npm /usr/local/bin/
sudo ln -s /home/dds/webrtc/nodejs/bin/node /usr/local/bin/

**看清楚，上面软链接的路径是你自己创建的路径，我的路径是/home/dds/webrtc/nodejs**

#查看是否安装
node -v 
npm -v 

# 注意，ubuntu 有的是需要sudo,如果不想sudo,可以软链接到当前用户目录
sudo ln -s /home/dds/webrtc/nodejs/bin/node /usr/bin/
```

## 3. coturn穿透和转发服务器

### 3.1 ubuntu安装

ubuntu的话直接用apt安装就行了

```text
sudo apt install coturn 
```

### 3.2 centos安装

centos或者其他的系统根据下面的方式进行安装

安装依赖

```text
Ubuntu, Debian, Mint:		
		$ sudo apt-get install libssl-dev（必须）
		$ sudo apt-get install libsqlite3 (or sqlite3)
		$ sudo apt-get install libsqlite3-dev (or sqlite3-dev)
		$ sudo apt-get install libevent-dev（必须）
		$ sudo apt-get install libpq-dev 
		$ sudo apt-get install mysql-client
		$ sudo apt-get install libmysqlclient-dev
		$ sudo apt-get install libhiredis-dev

Fedora:		
    	$ sudo yum install openssl-devel
		$ sudo yum install sqlite
		$ sudo yum install sqlite-devel
		$ sudo yum install libevent
		$ sudo yum install libevent-devel
		$ sudo yum install postgresql-devel
		$ sudo yum install postgresql-server
		$ sudo yum install mysql-devel
		$ sudo yum install mysql-server
		$ sudo yum install hiredis
		$ sudo yum install hiredis-devel		
```

编译安装coturn

```text
git clone https://github.com/coturn/coturn 
cd coturn 
./configure 
make 
sudo make install
```

### 3.3 配置相关

查看是否安装成功

```text
## 如果能够找到就说明已经安装成功
which turnserver
```

根据自己的安装目录，配置文件`/usr/local/etc/turnserver.conf` 或者`/etc/turnserver.conf`

我的目录是 `/usr/local/etc/turnserver.conf`
配置 如下

```text
verbose
fingerprint
lt-cred-mech
realm=test 
user=ddssingsong:123456
stale-nonce
no-loopback-peers
no-multicast-peers
mobility
no-cli
```

或者下面这个配置，只配置stun（stun-only）

```text
listening-ip=本地ip
listening-port=3478
#relay-ip=0.0.0.0
external-ip=外网ip
min-port=59000
max-port=65000
Verbose
fingerprint
no-stdout-log
syslog
user=ddssingsong:123456
no-tcp
no-tls
no-tcp-relay
stun-only
# 下面是配置证书，不懂就问后端人员怎么用openssl生成这个
cert=pem/turn_server_cert.pem 
pkey=pem/turn_server_pkey.pem 
#secure-stun
```

### 3.4 启动相关

```text
# 如果按照上面的配置直接运行
turnserver

# 如果没有配置上述配置文件，可采用其他运行方法
/usr/local/bin/turnserver --syslog -a -f --min-port=32355 --max-port=65535 --user=dds:123456 -r dds --cert=turn_server_cert.pem --pkey=turn_server_pkey.pem --log-file=stdout -v

--syslog 使用系统日志
-a 长期验证机制
-f 使用指纹
--min-port   起始用的最小端口
--max-port   最大端口号
--user=dds:123456  turn用户名和密码
-r realm组别
--cert PEM格式的证书
--pkey PEM格式的私钥文件
-l, --log-file,<filename> 指定日志文件
-v verbose


#请根据需要选择
```

测试地址，请分别测试stun和turn，relay代表turn

https://[webrtc](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3Dwebrtc).github.io/samples/src/content/peerconnection/trickle-ice/

![img](https://pic4.zhimg.com/80/v2-abfd35c48a156569f2be20d3d8e3832f_720w.webp)

## 4. 安装webrtc服务器和浏览器端

1. 下载 代码

```text
# 代码检出来
git clone https://github.com/ddssingsong/webrtc_server_node.git  
cd webrtc_server
```

2.修改`/public/dist/js/SkyRTC-client.js`，设置穿透服务器

```text
   var iceServer = {
        "iceServers": [
          {
            "url": "stun:stun.l.google.com:19302"
          },
          {
            "url": "stun:118.25.25.147:3478"
          },
          {
             "url": "turn:118.25.25.147:3478",
             "username":"ddssingsong",
             "credential":"123456"
          }
        ]
    };
```

3.修改`/public/dist/js/conn.js`

```text
## 最后一行

##  如果没有配wss代理

rtc.connect("ws:" + window.location.href.substring(window.location.protocol.length).split('#')[0], window.location.hash.slice(1));

如果配了nginx wss代理
rtc.connect("wss:" + window.location.href.substring(window.location.protocol.length).split('#')[0]+"/wss", window.location.hash.slice(1));

# 后面的那个“/wss”是根据自己配的代理路径
```

4.运行

```text
# cd到项目路径

# 安装依赖
npm install

# 运行
node server.js
```

**其实到了这一步就可以测试客户端了，往下看获取线上部署详情**

客户端测试可以不使用nginx配置代理，只需要使用ws即可

## 5. 安装nginx

如果是ubuntu的话还是可以使用apt安装

```text
sudo apt-get install nginx
```

centos按照下面的方式进行

1. 安装依赖

```text
yum install -y gcc gcc-c++ autoconf automake make zlib zlib-devel openssl openssl-devel pcre pcre-devel
```

2.编译安装nginx

```text
wget -C http://nginx.org/download/nginx-1.12.0.tar.gz
tar xvf nginx-1.12.0.tar.gz
cd nginx-1.12.0

./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module

make 

sudo make install 
```

3.生成证书，这个只是简单的生成，请慎重对待

```text
# 移动到目录，下面会用到
cd /
sudo mkdir cert
ce cert

# 生成服务器证书key
sudo openssl genrsa -out cert.pem 1024

# 生成证书请求,需要你输入信息，一路回车就行，不要输入内容
sudo openssl req -new -key cert.pem -out cert.csr

# 生成crt证书
sudo openssl x509 -req -days 3650 -in cert.csr -signkey cert.pem -out cert.crt
```

4.修改 配置文件`/usr/local/nginx/conf/nginx.conf`或者`/etc/nginx/nginx.conf`,没有的话自己找一下

将下面的内容帖进去就行了

```text
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
	# multi_accept on;
}

http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;


	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	gzip on;

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
	
	 #代理https
	upstream web {
			server 0.0.0.0:3000;      
        }
	#代理websocket
	upstream websocket {
			server 0.0.0.0:3000;   
        }
        
	server { 
		listen       443; 
		server_name  localhost;
		ssl          on;

		ssl_certificate     /cert/cert.crt;#配置证书
		ssl_certificate_key  /cert/cert.key;#配置密钥

		ssl_session_cache    shared:SSL:1m;
		ssl_session_timeout  50m;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2 SSLv2 SSLv3;
		ssl_ciphers  HIGH:!aNULL:!MD5;
		ssl_prefer_server_ciphers  on;

    
	#wss 反向代理  
	location /wss {
		proxy_pass http://websocket/; # 代理到上面的地址去
		proxy_read_timeout 300s;
		proxy_set_header Host $host;
		proxy_set_header X-Real_IP $remote_addr;
		proxy_set_header X-Forwarded-for $remote_addr;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'Upgrade';	
  }
	#https 反向代理
	location / {
		proxy_pass         http://web/;
		proxy_set_header   Host             $host;
		proxy_set_header   X-Real-IP        $remote_addr;
		proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
  }
 }
}
```

5.开启nginx

```text
#查看是否开启
ps -ef|grep nginx

#改变配置文件重启nginx
sudo nginx -s reload
```

## 6. 测试浏览器

```text
#访问

https://serverIp#roomName

如：
外网：https://192.168.1.123/#123
内网：http:192.168.1.123:3000#123

# 查看效果，其中roomName为进入的房间名，不同房间的用户无法互相通话
```

## 7. 测试客户端

将这个项目下下来使用 android studio 编译并安装

```text
## 看清楚分支，项目一直在开发中，所以请使用固定分支测试，一般使用branch_nodejs分支测试，master和dev是最新代码
https://github.com/ddssingsong/webrtc_android
```

修改`WebrtcUtil.java`，要去掉界面上的地址哦

```text
    // turn and stun
    // 外网测试才需要
    private static MyIceServer[] iceServers = {
            new MyIceServer("stun:stun.l.google.com:19302"),
            new MyIceServer("118.25.25.147:3478?transport=udp"),
            new MyIceServer("118.25.25.147:3478?transport=udp",
                    "ddssingsong",
                    "123456"),
            new MyIceServer("118.25.25.147:3478?transport=tcp",
                    "ddssingsong",
                    "123456"),

    };

    // 外网测试
    private static String WSS = "wss://47.254.34.146/wss";

    //本地内网信令地址
    private static String WSS = "ws://192.168.1.122:3000";
```

## 8. 到这里，基本完成

原文作者：零声音视频开发

原文链接：https://zhuanlan.zhihu.com/p/455942818