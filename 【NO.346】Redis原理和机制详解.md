# 【NO.346】Redis原理和机制详解

## 1.什么是Redis?

Redis 是开源免费的，遵守BSD协议，是一个高性能的key-value非关系型数据库。

## 2.Redis特点：

Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。

Redis不仅仅支持简单的key-value类型的数据，同时还提供String，list，set，zset，hash等数据结构的存储。

Redis支持数据的备份，即master-slave模式的数据备份。

性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 。

原子 – Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。

丰富的特性 – Redis还支持 publish/subscribe, 通知, 设置key有效期等等特性。

## 3.redis作用:

可以减轻数据库压力，查询内存比查询数据库效率高。

## 4.Redis应用：

token生成、session共享、分布式锁、自增id、验证码等。

## 5.比较重要的3个可执行文件：

redis-server：Redis服务器程序

redis-cli：Redis客户端程序，它是一个命令行操作工具。也可以使用telnet根据其纯文本协议操作。

redis-benchmark：Redis性能测试工具，测试Redis在你的系统及配置下的读写性能。

## 6.I/O复用模型和Reactor 设计模式

Redis内部实现采用epoll+自己实现的简单的事件框架。 epoll中的读、写、关闭、连接都转化成了事件，然后利用epoll的多路复用特性， 绝不在io上浪费一点时间：

**1、I/O 多路复用的封装**

I/O 多路复用其实是在单个线程中通过记录跟踪每一个sock（I/O流） 的状态来管理多个I/O流。



![img](https://pic3.zhimg.com/80/v2-bf25427a55b4fa1d9856edbcdfa29f0a_720w.webp)



因为 Redis 需要在多个平台上运行，同时为了最大化执行的效率与性能，所以会根据编译平台的不同选择不同的 I/O 多路复用函数作为子模块，提供给上层统一的接口。

redis的多路复用， 提供了select, epoll, evport, kqueue几种选择，在编译的时候来选择一种。

select是POSIX提供的， 一般的操作系统都有支撑；
epoll 是LINUX系统内核提供支持的；
evport是Solaris系统内核提供支持的；
kqueue是Mac 系统提供支持的；

```text
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

为了将所有 IO 复用统一，Redis 为所有 IO 复用统一了类型名 aeApiState，对于 epoll 而言，类型成员就是调用 epoll_wait所需要的参数
接下来就是一些对epoll接口的封装了：包括创建 epoll(epoll_create)，注册事件(epoll_ctl)，删除事件(epoll_ctl)，阻塞监听(epoll_wait)等
创建 epoll 就是简单的为 aeApiState 申请内存空间，然后将返回的指针保存在事件驱动循环中
注册事件和删除事件就是对 epoll_ctl 的封装，根据操作不同选择不同的参数
阻塞监听是对 epoll_wait 的封装，在返回后将激活的事件保存在事件驱动中

**2、Reactor 设计模式：事件驱动循环流程**

**Redis 服务采用 Reactor 的方式来实现文件事件处理器（每一个网络连接其实都对应一个文件描述符）**

当 main 函数初始化工作完成后，就需要进行事件驱动循环，而在循环中，会调用 IO 复用函数进行监听
在初始化完成后，main 函数调用了 aeMain 函数，传入的参数就是服务器的事件驱动

Redis 对于时间事件是采用链表的形式记录的，这导致每次寻找最早超时的那个事件都需要遍历整个链表，容易造成性能瓶颈。而 libevent 是采用最小堆记录时间事件，寻找最早超时事件只需要 O(1) 的复杂度



![img](https://pic2.zhimg.com/80/v2-1928c86722f8d82fbf9c4dc8aa90e7e9_720w.webp)



如上图，IO多路复用模型使用了Reactor设计模式实现了这一机制。

通过Reactor的方式，可以将用户线程轮询IO操作状态的工作统一交给handle_events事件循环进行处理。

用户线程注册事件处理器之后可以继续执行做其他的工作（异步），而Reactor线程负责调用内核的select/epoll函数检查socket状态。当有socket被激活时，则通知相应的用户线程（或执行用户线程的回调函数），执行handle_event进行数据读取、处理的工作。由于select/epoll函数是阻塞的，因此多路IO复用模型也被称为异步阻塞IO模型。注意，这里的所说的阻塞是指select函数执行时线程被阻塞，而不是指socket。一般在使用IO多路复用模型时，socket都是设置为NONBLOCK的，不过这并不会产生影响，因为用户发起IO请求时，数据已经到达了，用户线程一定不会被阻塞。

**3、redis线程模型：**

**如图所示：**



![img](https://pic4.zhimg.com/80/v2-cdbd833e4e8ac5405f36f8f4ebd7500f_720w.webp)



简单来说，就是。我们的redis-client在操作的时候，会产生具有不同事件类型的socket。在服务端，有一段I/0多路复用程序，将其置入队列之中。然后，IO事件分派器，依次去队列中取，转发到不同的事件处理器中。

## 7.redis数据结构

> **存储字符串**
> 1.set key value：设定key持有指定的字符串value，如果该key存在则进行覆盖操作,总是返回OK
> 2.get key: 获取key的value。如果与该key关联的value不是String类型，redis将返回错误信息，因为get命令只能用于获取String value；如果该key不存在，返回null。
> 3.getset key value：先获取该key的值，然后在设置该key的值。
> 4.incr key：将指定的key的value原子性的递增1. 如果该key不存在，其初始值为0，在incr之后其值为1。如果value的值不能转成整型，如hello，该操作将执行失败并返回相应的错误信息
> 5.decr key：将指定的key的value原子性的递减1.如果该key不存在，其初始值为0，在incr之后其值为-1。如果value的值不能转成整型，如hello，该操作将执 行失败并返回相应的错误信息。
> 6.incrby key increment：将指定的key的value原子性增加increment，如果该key不存在，器初始值为0，在incrby之后，该值为increment。如果该值不能转成 整型，如hello则失败并返回错误信息
> 7.decrby key decrement：将指定的key的value原子性减少decrement，如果该key不存在，器初始值为0，在decrby之后，该值为decrement。如果该值不能 转成整型，如hello则失败并返回错误信息
> 8.append key value：如果该key存在，则在原有的value后追加该值；如果该key 不存在，则重新创建一个key/value



> **存储list类型**
> 1.lpush key value1 value2...：在指定的key所关联的list的头部插入所有的values，如果该key不存在，该命令在插入的之前创建一个与该key关联的空链表，之后再向该链表的头部插入数据。插入成功，返回元素的个数。
> 2.rpush key value1、value2…：在该list的尾部添加元素
> 3.lrange key start end：获取链表中从start到end的元素的值，start、end可为负数，若为-1则表示链表尾部的元素，-2则表示倒数第二个，依次类推…
> 4.lpushx key value：仅当参数中指定的key存在时（如果与key管理的list中没有值时，则该key是不存在的）在指定的key所关联的list的头部插入value。
> 5.rpushx key value：在该list的尾部添加元素
> 6.lpop key：返回并弹出指定的key关联的链表中的第一个元素，即头部元素
> 7.rpop key：从尾部弹出元素
> 8.rpoplpush resource destination：将链表中的尾部元素弹出并添加到头部
> 9.llen key：返回指定的key关联的链表中的元素的数量。
> 10.lset key index value：设置链表中的index的脚标的元素值，0代表链表的头元素，-1代表链表的尾元素。



> **存储Set**
> 添加或删除元素
> 1.sadd key values[value1、value2……]:向set中添加数据，如果该key的值有则不会重复添加
> 例如:sadd myset a b c
> 2.srem key members[member1、menber2…]:删除set中的指定成员
> 例如:srem myset 1 2 3
> 获得集合中的元素
> 1.smembers key :获取set中所有的成员
> smembers myset
> 2.sismember key menber :判断参数中指定的成员是否在该set中，1表示存在，0表示不存在或者该key本身就不存在(无论集合中有多少元素都可以极速的返回结果)
> 集合的差集运算 A-B
> sdiff key1 key2 … : 返回key1与key2中相差的成员，而且与key的顺序有关。即返回差集。
> 集合的交集运算 
> sinter key1 key2 key3… :返回交集
> 集合的并集运算 
> sunion key1 key2 key3… : 返回并集
> 扩展命令(了解)
> scard key : 获取set中的成员数量
> 例子:scard myset
> srandmember key : 随机返回set中的一个成员
> sdiffstore destination key1 key2 …: 将key1 key2 相差的成员存储到destination中
> sinterstore destination key[key…] : 将返回的交集存储在destination上
> suninonstore destination key[key…] : 将返回的并集存储在destination上



> **存储hash**
> 1.赋值
> hset key field value : 为指定的key设定field/value对
> hmset key field1 value1 field2 value2 field3 value3 为指定的key设定多个field/value对
> 2.取值
> hget key field : 返回指定的key中的field的值
> hmget key field1 field2 field3 : 获取key中的多个field值
> hkeys key : 获取所有的key
> hvals key :获取所有的value
> hgetall key : 获取key中的所有field 中的所有field-value
> 3.删除
> hdel key field[field…] : 可以删除一个或多个字段，返回是被删除的字段个数
> del key : 删除整个list
> 4.增加数字
> hincrby key field increment ：设置key中field的值增加increment，如: age增加20
> hincrby myhash age 5
> 自学命令:
> hexists key field : 判断指定的key中的field是否存在
> hlen key : 获取key所包含的field的数量
> hkeys key ：获得所有的key 
> hkeys myhash
> hvals key ：获得所有的value
> hvals myhash



> **存储sortedset**
> 1.添加元素
> zadd key score member score2 member2…:将所有成员以及该成员的分数存放到sorted-set中。如果该元素已经存在则会用新的分数替换原有的分数。返回值是新加入到集合中的元素个数。(根据分数升序排列)
> 2.获得元素
> zscore key member ：返回指定成员的分数
> zcard key ：获得集合中的成员数量
> 3.删除元素
> zrem key member[member…] ：移除集合中指定的成员，可以指定多个成员
> 4.范围查询
> zrange key strat end [withscores]：获取集合中角标为start-end的成员，[withscore]参数表明返回的成员包含其分数。
> zremrangebyrank key start stop ：按照排名范围删除元素
> zremrangescore key min max ：按照分数范围删除元素
> 扩展命令(了解)
> zrangebyscore key min max [withscore] [limit offset count] ：返回分数在[min,max]的成员并按照分数从低到高排序。[withscore]：显示分数；[limit offset count]；offset，表明从脚标为offset的元素开始并返回count个成员
> zincrby key increment member ：设置指定成员的增加分数。返回值是修改后的分数
> zcount key min max：获取分数在[min，max]之间的成员个数
> zrank key member：返回成员在集合中的排名(从小到大)
> zrevrank key member ：返回成员在集合中的排名(从大到小)



> key的通用操作 
> keys pattern : 获取所有与pattern匹配的key ，返回所有与该key匹配的keys。 *表示任意一个或者多个字符， ?表示任意一个字符
> del key1 key2… ：删除指定的key 
> del my1 my2 my3
> exists key ：判断该key是否存在，1代表存在，0代表不存在
> rename key newkey ：为key重命名
> expire key second：设置过期时间，单位秒
> ttl key：获取该key所剩的超时时间，如果没有设置超时，返回-1，如果返回-2表示超时不存在。
> persist key:持久化key
> 192.168.25.153:6379> expire Hello 100
> (integer) 1
> 192.168.25.153:6379> ttl Hello
> (integer) 77
> type key：获取指定key的类型。该命令将以字符串的格式返回。返回的字符串为string 、list 、set 、hash 和 zset，如果key不存在返回none。
> 例如: type newcompany
> none

## 8.redis的数据类型，以及每种数据类型的使用场景

(一)String
这个其实没啥好说的，最常规的set/get操作，value可以是String也可以是数字。一般做**一些复杂的计数功能的缓存。**

(二)hash
这里value存放的是结构化的对象，比较方便的就是操作其中的某个字段。博主在做**单点登录**的时候，就是用这种数据结构存储用户信息，以cookieId作为key，设置30分钟为缓存过期时间，能很好的模拟出类似session的效果。

(三)list
使用List的数据结构，可以**做简单的消息队列的功能**。另外还有一个就是，可以利用lrange命令，**做基于redis的分页功能**，性能极佳，用户体验好。

(四)set
因为set堆放的是一堆不重复值的集合。所以可以做**全局去重的功能**。为什么不用JVM自带的Set进行去重？因为我们的系统一般都是集群部署，使用JVM自带的Set，比较麻烦，难道为了一个做一个全局去重，再起一个公共服务，太麻烦了。
另外，就是利用交集、并集、差集等操作，可以**计算共同喜好，全部的喜好，自己独有的喜好等功能**。

(五)sorted set

sorted set多了一个权重参数score,集合中的元素能够按score进行排列。可以做**排行榜应用，取TOP N操作**。另外，参照另一篇《分布式之延时任务方案解析》，该文指出了sorted set可以用来做**延时任务**。最后一个应用就是可以做**范围查找**。

## 9.redis的过期策略以及内存淘汰机制

**分析**:这个问题其实相当重要，到底redis有没用到家，这个问题就可以看出来。比如你redis只能存5G数据，可是你写了10G，那会删5G的数据。怎么删的，这个问题思考过么？还有，你的数据已经设置了过期时间，但是时间到了，内存占用率还是比较高，有思考过原因么?
**回答**:
redis采用的是**定期删除+惰性删除策略**。
**为什么不用定时删除策略?**
定时删除,用一个定时器来负责监视key,过期则自动删除。虽然内存及时释放，但是十分消耗CPU资源。在大并发请求下，CPU要将时间应用在处理请求，而不是删除key,因此没有采用这一策略.
**定期删除+惰性删除是如何工作的呢?**
定期删除，redis默认每个100ms检查，是否有过期的key,有过期key则删除。需要说明的是，redis不是每个100ms将所有的key检查一次，而是随机抽取进行检查(如果每隔100ms,全部key进行检查，redis岂不是卡死)。因此，如果只采用定期删除策略，会导致很多key到时间没有删除。
于是，惰性删除派上用场。也就是说在你获取某个key的时候，redis会检查一下，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除。
**采用定期删除+惰性删除就没其他问题了么?**
不是的，如果定期删除没删除key。然后你也没及时去请求key，也就是说惰性删除也没生效。这样，redis的内存会越来越高。那么就应该采用**内存淘汰机制**。
在redis.conf中有一行配置

```text
# maxmemory-policy allkeys-lru
```

该配置就是配内存淘汰策略的(什么，你没配过？好好反省一下自己)
1）noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。**应该没人用吧。**
2）allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。**推荐使用。**
3）allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。**应该也没人用吧，你不删最少使用Key,去随机删。**
4）volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。**这种情况一般是把redis既当缓存，又做持久化存储的时候才用。不推荐**
5）volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。**依然不推荐**
6）volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。**不推荐**
ps：如果没有设置 expire 的key, 不满足先决条件(prerequisites); 那么 volatile-lru, volatile-random 和 volatile-ttl 策略的行为, 和 noeviction(不删除) 基本上一致。

## 10.redis和数据库双写一致性问题

**分析**:一致性问题是分布式常见问题，还可以再分为最终一致性和强一致性。数据库和缓存双写，就必然会存在不一致的问题。答这个问题，先明白一个前提。就是**如果对数据有强一致性要求，不能放缓存。**我们所做的一切，只能保证最终一致性。另外，我们所做的方案其实从根本上来说，只能说**降低不一致发生的概率**，无法完全避免。因此，有强一致性要求的数据，不能放缓存。
**回答**:首先，采取正确更新策略，先更新数据库，再删缓存。其次，因为可能存在删除缓存失败的问题，提供一个补偿措施即可，例如利用消息队列。

## 11.如何应对缓存穿透和缓存雪崩问题

**分析**:这两个问题，说句实在话，一般中小型传统软件企业，很难碰到这个问题。如果有大并发的项目，流量有几百万左右。这两个问题一定要深刻考虑。
**回答**:如下所示

**缓存穿透：**即黑客故意去请求缓存中不存在的数据，导致所有的请求都怼到数据库上，从而数据库连接异常。

**解决方案**:
(一)利用互斥锁，缓存失效的时候，先去获得锁，得到锁了，再去请求数据库。没得到锁，则休眠一段时间重试
(二)采用异步更新策略，无论key是否取到值，都直接返回。value值中维护一个缓存失效时间，缓存如果过期，异步起一个线程去读数据库，更新缓存。需要做**缓存预热**(项目启动前，先加载缓存)操作。
(三)提供一个能迅速判断请求是否有效的拦截机制，比如，利用布隆过滤器，内部维护一系列合法有效的key。迅速判断出，请求所携带的Key是否合法有效。如果不合法，则直接返回。



**缓存雪崩**，即缓存同一时间大面积的失效，这个时候又来了一波请求，结果请求都怼到数据库上，从而导致数据库连接异常。

**解决方案**:
(一)给缓存的失效时间，加上一个随机值，避免集体失效。
(二)使用互斥锁，但是该方案吞吐量明显下降了。
(三)双缓存。我们有两个缓存，缓存A和缓存B。缓存A的失效时间为20分钟，缓存B不设失效时间。自己做缓存预热操作。然后细分以下几个小点

- I 从缓存A读数据库，有则直接返回
- II A没有数据，直接从B读数据，直接返回，并且异步启动一个更新线程。
- III 更新线程同时更新缓存A和缓存B。

## 12.如何解决redis的并发竞争key问题

**分析**:这个问题大致就是，同时有多个子系统去set一个key。这个时候要注意什么呢？大家思考过么。需要说明一下，博主提前百度了一下，发现答案基本都是推荐用redis事务机制。博主**不推荐使用redis的事务机制。**因为我们的生产环境，基本都是redis集群环境，做了数据分片操作。你一个事务中有涉及到多个key操作的时候，这多个key不一定都存储在同一个redis-server上。因此，**redis的事务机制，十分鸡肋。**

**回答:**如下所示
(1)如果对这个key操作，**不要求顺序**
这种情况下，准备一个分布式锁，大家去抢锁，抢到锁就做set操作即可，比较简单。
(2)如果对这个key操作，**要求顺序**
假设有一个key1,系统A需要将key1设置为valueA,系统B需要将key1设置为valueB,系统C需要将key1设置为valueC.
期望按照key1的value值按照 valueA-->valueB-->valueC的顺序变化。这种时候我们在数据写入数据库的时候，需要保存一个时间戳。假设时间戳如下

```text
系统A key 1 {valueA  3:00}
系统B key 1 {valueB  3:05}
系统C key 1 {valueC  3:10}
```

那么，假设这会系统B先抢到锁，将key1设置为{valueB 3:05}。接下来系统A抢到锁，发现自己的valueA的时间戳早于缓存中的时间戳，那就不做set操作了。以此类推。

其他方法，比如利用队列，将set方法变成串行访问也可以。总之，灵活变通。

原文地址：https://zhuanlan.zhihu.com/p/222697530

作者：linux