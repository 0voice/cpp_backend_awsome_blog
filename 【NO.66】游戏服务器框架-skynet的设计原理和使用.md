# 【NO.66】游戏服务器框架-skynet的设计原理和使用

## 1.多核并发编程方式

（1）多线程。

在一个进程中开启多线程，为了充分利用多核，一般设置工作线程的个数为 cpu 的核心数。memcached 就是采用这种方式。

多线程在一个进程当中，所以数据共享来自进程当中的虚拟内存；这里会涉及到很多临界资源的访问，所以需要考虑加锁。

（2）多进程。

在一台机器当中，开启多个进程充分利用多核，一般设置工作进程的个数为 cpu 的核心数。nginx 就是采用这种方式，nginx 当中的 worker 进程，通过共享内存来进行共享数据；也需要考虑使用锁。

（3）CSP。

以 go 语言为代表，并发实体是协程（用户态线程、轻量级线程）；内部也是采用多少个核心开启多少个线程来充分利用多核。

（4）Actor。

erlang 从语言层面支持 actor 并发模型，并发实体是 actor（在skynet 中称之为服务）；skynet 采用 c + lua来实现 actor 并发模型；底层也是通过采用多少个核心开启多少个内核线程来充分利用多核。

## 2.skynet

### 2.1 skynet简介

它是一个轻量级游戏服务器框架，但也不仅仅用于游戏。

轻量级体现在：

1. 实现了 actor 模型，以及相关的脚手架（工具集）：actor 间数据共享机制以及c 服务扩展机制。
2. 实现了服务器框架的基础组件。实现了 reactor 并发网络库；并提供了大量连接的接入方案；基于自身网络库，实现了常用的数据库驱动（异步连接方案），并融合了 lua 数据结构；实现了网关服务；时间轮用于处理定时消息。

skynet抽象了actor并发模型，用户层抽象进程；sknet通过消息的方式共享内存；通过消息驱动actor运行。

skynet的actor模型使用lua虚拟，lua虚拟机非常小（只有几十kb），代价不高；每个actor对应一个lua虚拟机；系统中不能启动过多的进程就是因为资源受限，lua虚拟机占用的资源很少，可以开启很多个，这就能抽象出很多个用户层的进程。lua虚拟机可以共享一些数据，比如常量，达到资源复用。

抽象进程而不抽象线程的原因在于进程有独立的工作空间，隔离的运行环境。

sknet的所有actor都是对等的，通过公平调度实现。

### 2.2 环境准备

ubuntu：

```text
sudo apt-get install git build-essential readline-dev autoconf
# 或者
sudo apt-get install git build-essential libreadline-dev autoconf
```

centos：

```text
yum install -y git gcc readline-devel autoconf
```

mac：

```text
yum install -y git gcc readline-devel autoconf
```

### 2.3 编译安装

```text
git clone https://github.com/cloudwu/skynet.git
cd skynet
# centos or ubuntu
make linux
# mac
make macosx
```

### 2.4 Actor 模型

有消息的 actor 为活跃的 actor，没有消息为非活跃的 actor。

**定义：**

1. 用于并行计算；
2. Actor 是最基本的计算单元；
3. 基于消息计算；
4. Actor 通过消息进行沟通。

**组成：**

1. 隔离的环境。主要通过 lua 虚拟机来实现。
2. 消息队列。用来存放有序（先后到达）的消息。
3. 回调函数。用来运行 Actor；从 Actor 的消息队列中取出消息，并作为该回调函数的参数来运行 Actor。

### 2.5 消息队列

Actor 模型基于消息计算，在 skynet 框架中，消息包含 Actor（之间）消息、网络消息以及定时消息。

生产者生产消息，消息放入消息队列中，由线程池去消费消息。



### 2.6 actor公平调度

所有的actor都是对等的，每个actor都有自己的消息队列；skynet的并发实体是actor，因此需要公平的调度actor。

skynet会有非常的actor，公平调度就非常重要；采用两级队列。

首先，查找活跃的消息队列（有消息的消息队列）；

然后，通过队列组织所有活跃的消息队列；

最后，调度队列。

![img](https://pic1.zhimg.com/80/v2-a62db5cbff2f41b94a0e207e66c73f4c_720w.webp)

线程池从一级队列取出一个消息，消费消息，消费完消息后如果还有消息，则添加到一级队列的末尾。

因为用户问题造成消息不均匀，在应用上不一定公平；skynet在工作线程赋予权重来解决这个问题。

```text
// 工作线程权重图   32个核心
static int weight[] = {
    -1, -1, -1, -1, 0, 0, 0, 0,
    1, 1, 1, 1, 1, 1, 1, 1,  // 1/2
    2, 2, 2, 2, 2, 2, 2, 2,  // 1/4
    3, 3, 3, 3, 3, 3, 3, 3, }; // 1/8
```

当工作线程的权重为 -1 时，该工作线程每次只 pop 一条消息；当工作线程的权重为 0 时，该工作线程每次消费完所有的消息；当工作线程的权重为 1 时，每次消费消息队列中二分之一的消息；当工作线程的权重为 2 时，每次消费消息队列中四分之一 的消息；以此类推；通过这种方式，完成消息队列梯度消费，从而不至于让某些队列过长。

## 3.skynet的使用

github 上打开 skynet 点击 [wiki](https://link.zhihu.com/?target=https%3A//github.com/cloudwu/skynet/wiki) 里面有所有函数的详细说明。

### 3.1 第一个skynet程序

（1）在sknet源码的父目录下创建一个config文件，用于定制skynet的行为。
内容可参考如下：

```text
thread=4
logger=nil
harbor=0
start="main"
lua_path="./skynet/lualib/?.lua;./skynet/lualib/?/init.lua;"
luaservice="./skynet/service/?.lua;./app/?.lua"
lualoader="./skynet/lualib/loader.lua"
cpath="./skynet/cservice/?.so"
lua_cpath="./skynet/luaclib/?.so"
```

参数说明：

![img](https://pic4.zhimg.com/80/v2-d2f5d3953971737865686f33a04ba51f_720w.webp)

（2）在app文件下面创建main.lua。每次启动时都是从skynet.start(…)中运行，相当于入口函数。

```text
local skynet = require("skynet")

skynet.start(function()
    -- body
    print("hello skynet")
end)
```

（3）编写一个自己项目的makefile。首先编译skynet的源码，如果有自己的c代码直接加入makefile文件。

```text
SKYNET_PATH?=./skynet

all:
	cd $(SKYNET_PATH) && $(MAKE) PLAT='linux'

clean:
	cd $(SKYNET_PATH) && $(MAKE) clean
```

（4）编译，执行自己的makefile。

```text
make
```

（5）执行skynet，指定config。

```text
./skynet/skynet config
```



```text
[:00000002] LAUNCH snlua bootstrap
[:00000003] LAUNCH snlua launcher
[:00000004] LAUNCH snlua cdummy
[:00000005] LAUNCH harbor 0 4
[:00000006] LAUNCH snlua datacenterd
[:00000007] LAUNCH snlua service_mgr
[:00000008] LAUNCH snlua main
hello skynet
[:00000002] KILL self
```

可以看到打印了hello skynet。

### 3.2 skynet网络消息

skynet 当中采用一个 socket 线程来处理网络信息；skynet 基于 reactor 网络模型。

网络消息驱动actor运行。需要skynet.socket模块。

首先绑定一个监听 listenfd，然后调用socket.start(listenfd, function)绑定fd到进程函数function中处理。

示例：

main.lua

```text
local skynet = require "skynet"
local socket=require "skynet.socket"


skynet.start(function()
    -- body
    print("hello skynet")

    local listenfd=socket.listen("0.0.0.0",8888);
    socket.start(listenfd,function(clientfd,addr)
        -- body
        print("receive a client: ",clientfd,addr);
    end)

end)
```



```text
[:00000002] LAUNCH snlua bootstrap
[:00000003] LAUNCH snlua launcher
[:00000004] LAUNCH snlua cdummy
[:00000005] LAUNCH harbor 0 4
[:00000006] LAUNCH snlua datacenterd
[:00000007] LAUNCH snlua service_mgr
[:00000008] LAUNCH snlua main
hello skynet
[:00000002] KILL self
receive a client: 	2	192.168.0.105:50024
```

### 3.3 skynet定时消息

定时任务推送给定时线程，定时线程检测完时间后往消息队列推送一个消息，然后actor开始执行callback函数。

使用示例：
main.lua

```text
local skynet = require "skynet"


skynet.start(function()
    -- body
    print("hello skynet")
    skynet.timeout(100,function()
        -- body
        print("timer 1s");
    end)

end)
```

### 3.4 skynet actor间消息

所有的服务都是通过消息来交换数据。actor间消息通信通过将消息放到对方的消息队列实现。

示例，创建两个lua文件。

main发送“ping”给slave，协程挂起；slave回复“pong”给main，协程唤醒。整个过程使用协程实现异步操作，消除异步中的回调。

main.lua

```text
local skynet = require("skynet")

skynet.start(function()
    -- body
    print("hello skynet")

    local slave=skynet.newservice("slave")
    local response=skynet.call(slave,"lua","ping")
    print("main ",response)
end)
```

slave.lua

```text
local skynet = require "skynet"

local CMD={}

function CMD.ping()
    -- body
    skynet.retpack("pong")
end

skynet.start(function()
    skynet.dispatch("lua",function(session,source,cmd,...)
        local func=assert(CMD[cmd])
        func(...)
    end)
end)
```

执行效果：

```text
[:00000002] LAUNCH snlua bootstrap
[:00000003] LAUNCH snlua launcher
[:00000004] LAUNCH snlua cdummy
[:00000005] LAUNCH harbor 0 4
[:00000006] LAUNCH snlua datacenterd
[:00000007] LAUNCH snlua service_mgr
[:00000008] LAUNCH snlua main
hello skynet
[:00000009] LAUNCH snlua slave
main    pong
[:00000002] KILL self
```

## 4.vscode调试skynet

首先要知道整个系统/框架 是怎么应用的。skynet会自己产生一个skynet执行程序，需要指定配置文件来启动服务。

项目下的.vscode文件夹需要如下三个文件。

launch.json

```text
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "启动 app",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/skynet/skynet",
            "args": ["config"],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "build-skynet",
            "miDebuggerPath": "/usr/bin/gdb"
        }
    ]
}
```

settings.json

```text
{
    "C_Cpp.default.configurationProvider": "ms-vscode.makefile-tools",
    "files.associations": {
        "string.h": "c",
        "stdlib.h": "c",
        "pthread.h": "c"
    },
}
```

tasks.json

```text
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "build-skynet",
            "command": "/usr/bin/make",
            "args": [],
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "..."
        }
    ],
    "version": "2.0.0"
}
```

实现这三个文件就可以开始调试了。

skynet是一个网络服务，需要处理网络连接，需要知道数据是怎么流向，理解框架的运行机制（多线程并发）；所有的actor的运行都是由线程池来驱动，actor的运行又是由消息队列的消息驱动（网络消息、定时消息、actor间消息）。

线程池从消息队列中取出消息，找到回调函数，驱动actor运行。

## 5.总结

不要通过共享内存来通信，而应该通过通信来共享内存。CSP 和 Actor 都符合这一哲学；通过通信来共享数据，其实是一种解耦合的过程。并lua发实体之间可以分别开发并单独优化，而它们唯一的耦合在于消息；这能让我们快速地进行开发；同时也符合我们开发的思路，将一个大的问题拆分成若干个小问题。

actor模型是在用户层抽象了进程。

actor调度：将活跃的 actor 通过全局队列组织起来；actor 当中的消息队列有消息就是活跃的 actor； 线程池去全局队列中取出 actor 的消息队列，接着运行actor。

原文地址：https://zhuanlan.zhihu.com/p/591754195