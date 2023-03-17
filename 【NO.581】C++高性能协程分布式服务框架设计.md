# 【NO.581】C++高性能协程分布式服务框架设计

本项目将从零开始搭建出一个基于[协程]的异步RPC框架。

> 学习本项目需要有一定的C++，网络，RPC知识

## 1.RPC框架设计

本项目将从零开始搭建出一个基于协程的异步RPC框架。

通过本项目你将学习到：

- 协程同步原语
- 序列化协议
- 通信协议
- 连接复用
- 服务注册
- 服务发现
- 负载均衡
- 健康检查
- 三种异步调用方式

相信大家对RPC的相关概念都已经很熟悉了，这里不做过多介绍，直接进入重点。

本文将简单介绍框架的设计，在最后的 examples 会给出完整的使用范例。

更多的细节需要仔细阅读源码。

本RPC框架主要有网络模块， 序列化模块，通信协议模块，客户端模块，服务端模块，服务注册中心模块，负载均衡模块

主要有以下三个角色：

![img](https://pic2.zhimg.com/80/v2-cfc863dfb3e441791735fa2b2906abbd_720w.webp)

**注册中心 Registry**

主要是用来完成服务注册和服务发现的工作。同时需要维护服务下线机制，管理了服务器的存活状态。

**服务提供方 Service Provider**

其需要对外提供服务接口，它需要在应用启动时连接注册中心，将服务名发往注册中心。服务端还需要启动Socket服务监听客户端请求。

**服务消费方 Service Consumer**

客户端需要有从注册中心获取服务的基本能力，它需要在发起调用时，

从注册中心拉取开放服务的服务器地址列表存入本地缓存，

然后选择一种负载均衡策略从本地缓存中筛选出一个目标地址发起调用，

并将这个连接存入连接池等待下一次调用。

### 1.1 **协程同步原语**

我们都知道，一旦协程阻塞后整个协程所在的线程都将阻塞，这也就失去了协程的优势。编写协程程序时难免会对一些数据进行同步，而Linux下常见的同步原语互斥量、条件变量、信号量等基本都会堵塞整个线程，使用原生同步原语协程性能将大幅下降，甚至发生死锁的概率大大增加。

框架实现了一套协程同步原语来解决原生同步原语带来的阻塞问题，在协程同步原语之上实现更高层次同步的抽象——Channel用于协程之间的便捷通信。

### 1.2 序列化协议

本模块支持了基本类型以及标准库容器的序列化，包括：

- 顺序容器：string, list, vector
- 关联容器：set, multiset, map, multimap
- 无序容器：unordered_set, unordered_multiset, unordered_map, unordered_multimap
- 异构容器：tuple

以及通过以上任意组合嵌套类型的序列化

序列化有以下规则：

1. 默认情况下序列化，8，16位类型以及浮点数不压缩，32，64位有符号/无符号数采用 zigzag 和 varints 编码压缩
2. 针对 std::string 会将长度信息压缩序列化作为元数据，然后将原数据直接写入。char数组会先转换成 std::string 后按此规则序列化
3. 调用 writeFint 将不会压缩数字，调用 writeRowData 不会加入长度信息

对于任意用户自定义类型，只要实现了以下的重载，即可参与传输时的序列化。

```text
template<typename T>
Serializer &operator >> (Serializer& in, T& i){
   return *this;
}
template<typename T>
Serializer &operator << (Serializer& in, T i){
    return *this;
}
```

rpc调用过程：

- 调用方发起过程调用时，自动将参数打包成tuple，然后序列化传输。
- 被调用方收到调用请求时，先将参数包反序列回tuple，再解包转发给函数。

### 1.3 通信协议

```text
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|  BYTE  |        |        |        |        |        |        |        |        |        |        |             ........                                                           |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|  magic | version|  type  |          sequence id              |          content length           |             content byte[]                                                     |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
```

封装通信协议，使RPC Server和RPC Client 可以基于同一协议通信。

采用私有通信协议，协议如下：

第一个字节是魔法数。

第二个字节代表协议版本号，以便对协议进行扩展，使用不同的协议解析器。

第四个字节开始是一个32位序列号，用来识别请求顺序。

第七个字节开始的四字节表示消息长度，即后面要接收的内容长度。

除了协议头固定7字节，消息不定长。

目前提供了以下几种请求

```text
enum class MsgType : uint8_t {
    HEARTBEAT_PACKET,       // 心跳包
    RPC_PROVIDER,           // 向服务中心声明为provider
    RPC_CONSUMER,           // 向服务中心声明为consumer
    
    RPC_REQUEST,            // 通用请求
    RPC_RESPONSE,           // 通用响应
    
    RPC_METHOD_REQUEST ,    // 请求方法调用
    RPC_METHOD_RESPONSE,    // 响应方法调用
    
    RPC_SERVICE_REGISTER,   // 向中心注册服务
    RPC_SERVICE_REGISTER_RESPONSE,
    
    RPC_SERVICE_DISCOVER,   // 向中心请求服务发现
    RPC_SERVICE_DISCOVER_RESPONSE
};
```

### 1.4 连接复用

对于短连接来说，每次发起rpc调用就创建一条连接，由于没有竞争实现起来比较容易，但开销太大。所以本框架实现了rpc连接复用来支持更高的并发。

连接复用的问题在于，在一条连接上可以有多个并发的调用请求，由于服务器也是并发处理这些请求的，所以导致了服务器返回的响应顺序与请求顺序不一致。

### 1.5 服务注册

每一个服务提供者对应的机器或者实例在启动运行的时候，
都去向注册中心注册自己提供的服务以及开放的端口。
注册中心维护一个服务名到服务地址的多重映射，一个服务下有多个服务地址，
同时需要维护连接状态，断开连接后移除服务。

```text
/**
 * 维护服务名和服务地址列表的多重映射
 * serviceName -> serviceAddress1
 *             -> serviceAddress2
 *             ...
 */
std::multimap<std::string, std::string> m_services;
```

### 1.6 服务发现

虽然服务调用是服务消费方直接发向服务提供方的，但是分布式的服务，都是集群部署的，

服务的提供者数量也是动态变化的，

所以服务的地址也就无法预先确定。

因此如何发现这些服务就需要一个统一注册中心来承载。



客户端从注册中心获取服务，它需要在发起调用时，

从注册中心拉取开放服务的服务器地址列表存入本地缓存，

### 1.7 负载均衡

实现通用类型负载均衡路由引擎（工厂）。
通过路由引擎获取指定枚举类型的负载均衡器，
降低了代码耦合，规范了各个负载均衡器的使用，减少出错的可能。

提供了三种路由策略（随机、轮询、一致性哈希）,
由客户端使用，在客户端实现负载均衡

```text
/**
 * @brief: 路由均衡引擎
 */
template<class T>
class RouteEngine {
public:
    static typename RouteStrategy<T>::ptr queryStrategy(typename RouteStrategy<T>::Strategy routeStrategyEnum) {
        switch (routeStrategyEnum){
            case RouteStrategy<T>::Random:
                return s_randomRouteStrategy;
            case RouteStrategy<T>::Polling:
                return std::make_shared<impl::PollingRouteStrategyImpl<T>>();
            case RouteStrategy<T>::HashIP:
                return s_hashIPRouteStrategy ;
            default:
                return s_randomRouteStrategy ;
        }
    }
private:
    static typename RouteStrategy<T>::ptr s_randomRouteStrategy;
    static typename RouteStrategy<T>::ptr s_hashIPRouteStrategy;
};
```

选择客户端负载均衡策略，根据路由策略选择服务地址。默认随机策略。

```text
RouteStrategy<int>::ptr strategy =
            RouteEngine<int>::queryStrategy(Strategy::Random);
```

客户端同时会维护RPC连接池，以及服务发现结果缓存，减少频繁建立连接。

通过上述策略尽量消除或减少系统压力及系统中各节点负载不均衡的现象。

### 1.8 健康检查

服务中心必须管理服务器的存活状态，也就是健康检查。

注册服务的这一组机器，当这个服务组的某台机器如果出现宕机或者服务死掉的时候，

就会剔除掉这台机器。这样就实现了自动监控和管理。



项目采用了心跳发送的方式来检查健康状态。



服务器端：

开启一个定时器，定期给注册中心发心跳包，注册中心会回一个心跳包。



注册中心：

开启一个定时器倒计时，每次收到一个消息就更新一次定时器。如果倒计时结束还没有收到任何消息，则判断服务掉线。

### 1.9 三种异步调用方式

整个框架最终都要落实在服务消费者。为了方便用户，满足用户的不同需求，项目设计了三种异步调用方式。
三种调用方式的模板参数都是返回值类型，对void类型会默认转换uint8_t 。

1、以同步的方式异步调用

整个框架本身基于协程池，所以在遇到阻塞时会自动调度实现以同步的方式异步调用

```text
Result<int> res = con->call<int>("add", 123, 321);
ACID_LOG_INFO(g_logger) << res.getVal();
```

2、future 形式的异步调用

调用时会立即返回一个future

```text
future<Result<int>> res = con->async_call<int>("add", 123, 321);
ACID_LOG_INFO(g_logger) << res.get().getVal();
```

3、异步回调

async_call的第一个参数为函数时，启用回调模式，回调参数必须是返回类型的包装。收到消息时执行回调。

```text
con->async_call<int>([](Result<int> res){
    ACID_LOG_INFO(g_logger) << res.getVal();
}, "add", 123, 321); 
```

对调用结果及状态的封装如下

```text
/**
 * @brief RPC调用状态
 */
enum RpcState{
    RPC_SUCCESS = 0,    // 成功
    RPC_FAIL,           // 失败
    RPC_NO_METHOD,      // 没有找到调用函数
    RPC_CLOSED,         // RPC 连接被关闭
    RPC_TIMEOUT         // RPC 调用超时
};

/**
 * @brief 包装 RPC调用结果
 */
template<typename T = void>
class Result {
    ...
private:
    /// 调用状态
    code_type m_code = 0;
    /// 调用消息
    msg_type m_msg;
    /// 调用结果
    type m_val;
}
```

## 2.最后

通过以上介绍，我们粗略地了解了分布式服务的大概流程。但篇幅有限，无法面面俱到，
更多细节就需要去阅读代码来理解了。

这并不是终点，项目只是实现了简单的服务注册、发现。后续将考虑加入注册中心集群，限流，熔断，监控节点等。

## 3.示例

rpc服务注册中心

```text
#include "acid/rpc/rpc_service_registry.h"

// 服务注册中心
void Main() {
    acid::Address::ptr address = acid::Address::LookupAny("127.0.0.1:8080");
    acid::rpc::RpcServiceRegistry::ptr server = std::make_shared<acid::rpc::RpcServiceRegistry>();
    // 服务注册中心绑定在8080端口
    while (!server->bind(address)){
        sleep(1);
    }
    server->start();
}

int main() {
    acid::IOManager loop;
    loop.submit(Main);
}
```

rpc 服务提供者

```text
#include "acid/rpc/rpc_server.h"

int add(int a,int b){
    return a + b;
}

// 向服务中心注册服务，并处理客户端请求
void Main() {
    int port = 9000;
    acid::Address::ptr local = acid::IPv4Address::Create("127.0.0.1",port);
    acid::Address::ptr registry = acid::Address::LookupAny("127.0.0.1:8080");

    acid::rpc::RpcServer::ptr server = std::make_shared<acid::rpc::RpcServer>();;

    // 注册服务，支持函数指针
    server->registerMethod("add",add);
    // 支持函数对象
    server->registerMethod("echo", [](std::string str){
        return str;
    });
    // 支持标准库容器
    server->registerMethod("revers", [](std::vector<std::string> vec) -> std::vector<std::string>{
        std::reverse(vec.begin(), vec.end());
        return vec;
    });

    // 先绑定本地地址
    while (!server->bind(local)){
        sleep(1);
    }
    // 绑定服务注册中心
    server->bindRegistry(registry);
    // 开始监听并处理服务请求
    server->start();
}

int main() {
    acid::IOManager loop;
    loop.submit(Main);
}
```

rpc 服务消费者，并不直接用RpcClient，而是采用更高级的封装，RpcConnectionPool。

提供了连接池和服务地址缓存。

```text
#include "acid/log.h"
#include "acid/rpc/rpc_connection_pool.h"

static acid::Logger::ptr g_logger = ACID_LOG_ROOT();

// 连接服务中心，自动服务发现，执行负载均衡决策，同时会缓存发现的结果
void Main() {
    acid::Address::ptr registry = acid::Address::LookupAny("127.0.0.1:8080");

    // 设置连接池的数量
    acid::rpc::RpcConnectionPool::ptr con = std::make_shared<acid::rpc::RpcConnectionPool>();

    // 连接服务中心
    con->connect(registry);

    // 第一种调用接口，以同步的方式异步调用，原理是阻塞读时会在协程池里调度
    acid::rpc::Result<int> sync_call = con->call<int>("add", 123, 321);
    ACID_LOG_INFO(g_logger) << sync_call.getVal();

    // 第二种调用接口，future 形式的异步调用，调用时会立即返回一个future
    std::future<acid::rpc::Result<int>> async_call_future = con->async_call<int>("add", 123, 321);
    ACID_LOG_INFO(g_logger) << async_call_future.get().getVal();

    // 第三种调用接口，异步回调
    con->async_call<int>([](acid::rpc::Result<int> res){
        ACID_LOG_INFO(g_logger) << res.getVal();
    }, "add", 123, 321);
    
    // 测试并发
    int n=0;
    while(n != 1000) {
        n++;
        con->async_call<int>([](acid::rpc::Result<int> res){
            ACID_LOG_INFO(g_logger) << res.getVal();
        }, "add", 0, n);
    }
}

int main() {
    acid::IOManager loop;
    loop.submit(Main);
}
```

原文地址：https://zhuanlan.zhihu.com/p/607919437

作者：linux