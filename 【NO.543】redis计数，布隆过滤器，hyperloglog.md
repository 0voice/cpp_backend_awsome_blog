# 【NO.543】redis计数，布隆过滤器，hyperloglog

# 1.redis计数



![img](https://img-community.csdnimg.cn/images/92ed170fa1b045df9424d0f3b041be73.png)



# 2.布隆过滤器

## 2.1 redis扩展

redis通过对外提供一套API和一些数据结构，可以供开发者开发自己的模块并加载到redis中。

### 2.1.1 本质

在不侵入redis源码的基础上，提供一种高效的扩展数据结构的方式。

### 2.1.2 API及数据结构

参考redismodule.h

## 2.2 RedisBloom

RedisBloom是redis的一个扩展，我们主要使用了它的布隆过滤器。

关于布隆过滤器的原理，参考[《hash，bloomfilter，分布式一致性hash》](https://blog.csdn.net/congchp/article/details/122882760)

### 2.2.1 加载到redis中的方法

```shell
git clone https://github.com/RedisBloom/RedisBloom.git



cd RedisBloom



git submodule update --init --recursive



make



cp redisbloom.so /path/to



 



# 命令方式加载



redis-server --loadmodule /path/to/redisbloom.so



 



#配置文件加载



vi redis.conf



# loadmodules /path/to/redisbloom.so
```

### 2.2.2 命令

redis计数中的使用方法如下：

```shell
# 为 bfkey 所对应的布隆过滤器分配内存，参数是误差率以及预期元素数量，根据运算得出需要多少hash函数



以及所需位图大小



bf.reserve bfkey 0.0000001 10000



# 检测 bfkey 所对应的布隆过滤器是否包含 userid



bf.exists bfkey userid



# 往 key 所对应的布隆过滤器中添加 userid 字段



bf.add bfkey userid
```

# 3.hyperloglog

布隆过滤器提供了一种节省内存的概率型数据结构，它不保存具体数据，存在误差。

hyperloglog也是一种概率型数据结构，相对于布隆过滤器，使用的内存更少。

为什么叫hyperloglog？

是因为空间复杂度非常低，是$O(log_2log_2n)$。

redis提供了hyperloglog。在redis实现中，每个hyperloglog只是用**固定的12kb**进行计数，使用16384($2^{14}$)个桶子，标准误差为`0.8125%`，可以统计$2^{64}$个元素。



![img](https://img-community.csdnimg.cn/images/013f90d6c04f46058abd4b1e29cce768.png)



## 3.1 本质

使用少量内存统计集合的基数的近似值。存在一定的误差。

HLL的误差率：$\frac{1.04}{\sqrt{m}}$, m是桶子的个数；对于redis，误差率就是`0.8125%`。

## 3.2 原理

HyperLogLog 原理是通过给定 n 个的元素集合，记录集合中数字的比特串第一个1出现位置的最大值k，也可以理解为统计二进制低位连续为零（前导零）的最大个数。通过k值可以估算集合中不重复元素的数量m，m近似等于$2^k$。

但是这种预估方法存在较大误差，为了改善误差情况，HyperLogLog中引入分桶平均的概念，计算 m 个桶的调和平均值。下面公式中的const是一个修正常量。

$DV_HLL = const*m*\frac{m}{\sum_{j=1}^m\frac{1}{2^{R_j}}}$

$\frac{1}{2^{R_j}}$: 每个桶的估算值

$\frac{m}{\sum_{j=1}^m\frac{1}{2^{R_j}}}$：所有桶估值计值的调和平均值。

redis的hyperloglog中有16384($2^{14}$)个桶，每一个桶占6bit；

对于一个要放入集合中的字符串，hash生成64位整数，其中后14位用来索引桶子；前面50位用来统计低位连续为0的最大个数, 最大个数49。6bit对应的是$2^6$对应的整数值64可以存储49。在设置前，要设置进桶的值是否大于桶中的旧值，如果大于才进行设置，否则不进行设置。



![img](https://img-community.csdnimg.cn/images/a3ffcbeccef64e329af797c9b68bb51b.png)



## 3.3 去重

相同元素通过hash函数会生成相同的 64 位整数，它会索引到相同的桶子中，累计0的个数也会相同，按照上面计算最长累计0的规则，它不会改变该桶子的最长累计0；

## 3.4 存储

redis 中的 hyperloglog 存储分为稀疏存储和紧凑存储；

当元素很少的时候，redis采用节约内存的策略，hyperloglog采用稀疏存储方案；当存储大小超过3000 的时候，hyperloglog 会从稀疏存储方案转换为紧凑存储方案；紧凑存储不会转换为稀疏存储，因为hyperloglog数据只会增加不会减少（不存储具体元素，所以没法删除）；

## 3.5 命令

```shell
# 往key对应的hyperloglog对象添加元素



pfadd key userid_1 userid_2 userid_3



# 获取key对应hyperloglog中集合的基数



pfcount key



# 往key对应过的hyperloglog中添加重复元素。已存在，并不会添加



pfadd key userid_1



pfadd key1 userid_2 userid_3 userid_4



# 合并key，key1到key2，会去重



pfmerge key2 key key1
```

原文作者：[congchp](https://blog.csdn.net/congchp)

原文链接：https://bbs.csdn.net/topics/604985776