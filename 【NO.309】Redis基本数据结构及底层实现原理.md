# 【NO.309】Redis基本数据结构及底层实现原理

**前言**

面试必问之Redis，大部分人都知道Redis的几种数据类型，也知道怎么用。但具体底层是怎么实现的呢，面试过程中面试官问：Redis底层是怎么实现的，你能答上来吗？

## 1.Redis支持的数据类型

一、Redis支持的数据类型

Redis 主要有以下几种数据类型：

- String 字符串对象
- Hash 哈希Map对象
- List 列表对象
- Set 集合对象
- ZSet 有序集合

还有三种特殊数据类型:

- geospatial: Redis 在 3.2 推出 Geo 类型，该功能可以推算出地理位置信息，两地之间的距离。
- hyperloglog:基数：数学上集合的元素个数，是不能重复的。这个数据结构常用于统计网站的 UV。
- bitmap: bitmap 就是通过最小的单位 bit 来进行0或者1的设置，表示某个元素对应的值或者状态。一个 bit 的值，或者是0，或者是1；也就是说一个 bit 能存储的最多信息是2。bitmap 常用于统计用户信息比如活跃粉丝和不活跃粉丝、登录和未登录、是否打卡等。

补充说明：

基数是一种算法。举个例子 一本英文著作由数百万个单词组成，你的内存却不足以存储它们，那么我们先分析下业务。英文单词本身是有限的，在这本书的几百万个单词中有许许多多重复单词，扣去重复的单词，这本书中也就 千到 万多个单词而己，那么内存就足够存储它 了。比如数字集合｛ l,2,5 1,5,9 ｝的基数集合为｛ 1,2,5 ｝那么 基数（不重复元素）就是基数的作用是评估大约需要准备多少个存储单元去存储数据，基数并不是存储元素，存储元素消耗内存空间比较大，而是给某个有重复元素的数据集合（ 般是很大的数据集合〉评估需要的空间单元数。

![img](https://pic2.zhimg.com/80/v2-a29461535103d79589cb4dbeafebd435_720w.webp)

几种特殊类型的使用场景会在文末详细地补充介绍，请耐心看完。

## 2.redisObject对象

Redis存储的所有值对象在内部都定义为redisObject结构体，内部结构如下图所示：

![img](https://pic1.zhimg.com/80/v2-f6189e01322c8ead0f5d07676c5ca330_720w.webp)

Redis存储的包括string,hash,list,set,zset在内的所有数据类型，都使用redisObject来封装的。

下面针对每个字段做详细说明：

1.type字段:表示当前对象使用的数据类型，Redis主要支持5种数据类型:string,hash,list,set,zset。可以使用type {key}命令查看对象所属类型，type命令返回的是值对象类型，键都是string类型。

2.encoding字段:表示Redis内部编码类型，encoding在Redis内部使用，代表当前对象内部采用哪种数据结构实现。理解Redis内部编码方式对于优化内存非常重要，同一个对象采用不同的编码实现内存占用存在明显差异，具体细节见之后编码优化部分。

3.lru字段:记录对象最后一次被访问的时间，当配置了 maxmemory和maxmemory-policy=volatile-lru | allkeys-lru 时，用于辅助LRU算法删除键数据。可以使用object idletime {key}命令在不更新lru字段情况下查看当前键的空闲时间。开发提示：可以使用scan + object idletime 命令批量查询哪些键长时间未被访问，找出长时间不访问的键进行清理降低内存占用。

4.refcount字段:记录当前对象被引用的次数，用于通过引用次数回收内存，当refcount=0时，可以安全回收当前对象空间。使用object refcount {key}获取当前对象引用。当对象为整数且范围在[0-9999]时，Redis可以使用共享对象的方式来节省内存。具体细节见之后共享对象池部分。

5.ptr字段:与对象的数据内容相关，如果是整数直接存储数据，否则表示指向数据的指针。Redis在3.0之后对值对象是字符串且长度<=39字节的数据，内部编码为embstr类型，字符串sds和redisObject一起分配，从而只要一次内存操作。开发提示：高并发写入场景中，在条件允许的情况下建议字符串长度控制在39字节以内，减少创建redisObject内存分配次数从而提高性能。

可以简单的理解成下图：

![img](https://pic2.zhimg.com/80/v2-e0aeccf7ce2e6e23caddedd7fae6429d_720w.webp)

每一种类型都有自己特有的数据结构，下面我们要探讨的就是每种数据类型的具体的底层结构。

## 3.String

- string 是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。
- value其实不仅是String，也可以是数字
- string 类型是二进制安全的，可以包含任何数据，比如jpg图片或者序列化的对象
- string 类型的值最大能存储 512MB
- 常用命令：get、set、incr、decr、mget等

**应用场景**

常规key-value缓存应用。常规计数: 微博数, 粉丝数。

**String 底层实现**

字符串是我们日常工作中用得最多的对象类型，它对应的编码可以是int、raw和embstr。通过 object encoding key 命令来查看具体的编码格式：

![img](https://pic1.zhimg.com/80/v2-bc4129437b63c1751c3b8578776c6f38_720w.webp)

如果一个字符串对象保存的是不超过long类型的整数值，此时编码类型即为int，其底层数据结构直接就是long类型。例如执行set number 10086，就会创建int编码的字符串对象作为number键的值。

![img](https://pic2.zhimg.com/80/v2-8d23810026e6ae1ac89b8b23b329c9c9_720w.webp)

如果字符串对象保存的是一个长度大于39字节的字符串，此时编码类型即为raw，其底层数据结构是简单动态字符串(SDS)。

如果长度小于等于39个字节，编码类型则为embstr，底层数据结构就是embstr编码SDS。下面，我们详细理解下什么是简单动态字符串。

**SDS**

SDS是"simple dynamic string"的缩写。redis中所有场景中出现的字符串，由SDS来实现。

![img](https://pic4.zhimg.com/80/v2-42e8434873abb9efeeb2031bbfbfb0eb_720w.webp)

free:还剩多少空间 len:字符串长度 buf:存放的字符数组。

在源码的 src目录下，找到了 sds.h 这样一个文件，规定了 SDS 的结构：

```text
struct sdsshr<T>{
    T len;//数组长度
    T alloc;//数组容量
    unsigned  flags;//sdshdr类型
    char buf[];//数组内容
}
```

可以看出，SDS 的结构有点类似于 Java 中的 ArrayList。buf[]表示真正存储的字符串内容，alloc 表示所分配的数组的长度，len 表示字符串的实际长度，并且由于 len 这个属性的存在，Redis 可以在 O(1)的时间复杂度内获取数组长度。

**空间预分配**

为减少修改字符串带来的内存重分配次数，sds采用了“一次管够”的策略：

- 若修改之后sds长度小于1MB,则多分配现有len长度的空间
- 若修改之后sds长度大于等于1MB，则扩充除了满足修改之后的长度外，额外多1MB空间

由于Redis的字符串是动态字符串，可以修改，内部结构类似于Java的ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配。如上图所示，内部为当前字符串实际分配的空间capacity，一般高于实际字符串长度len。

![img](https://pic3.zhimg.com/80/v2-49ad3d15d55b9d89627f852d723d0412_720w.webp)

假设我们要存储的结构是：

```text
{
    "name": "xiaowang",
    "age": "35"
}
```

如果此时将此用户信息的name改为“xiaoli”，再存到redis中，redis是不需要重新分配空间的，使用已分配空间即可。

**惰性空间释放**

为避免缩短字符串时候的内存重分配操作，sds在数据减少时，并不立刻释放空间。

**SDS与C字符串的区别**

C语言使用长度为N+1的字符数组来表示长度为N的字符串，并且字符串的最后一个元素是空字符\0。Redis采用SDS相对于C字符串有如下几个优势：

- 常数复杂度获取字符串长度
- 杜绝缓冲区溢出
- 减少修改字符串时带来的内存重分配次数
- 二进制安全

**raw和embstr编码的SDS区别**

长度大于39字节的字符串，编码类型为raw，底层数据结构是简单动态字符串(SDS)。

比如当我们执行set story "Long, long, long ago there lived a king ..."(长度大于39)之后，Redis就会创建一个raw编码的String对象。数据结构如下：

![img](https://pic1.zhimg.com/80/v2-34a8f7eb0e27c97fa5dc43bead1bfd54_720w.webp)

长度小于等于39个字节的字符串，编码类型为embstr，底层数据结构则是embstr编码SDS。

embstr编码是专门用来保存短字符串的，它和raw编码最大的不同在于：raw编码会调用两次内存分配分别创建redisObject结构和sdshdr结构，而embstr编码则是只调用一次内存分配，在一块连续的空间上同时包含redisObject结构和sdshdr`结构。

![img](https://pic2.zhimg.com/80/v2-124275da03659bd4c16ba0074e0564d9_720w.webp)

**编码转换**

int编码和embstr编码的字符串对象在条件满足的情况下会自动转换为raw编码的字符串对象。

- 对于int编码来说，当我们修改这个字符串为不再是整数值的时候，此时字符串对象的编码就会从int变为raw；
- 对于embstr编码来说，只要我们修改了字符串的值，此时字符串对象的编码就会从embstr变为raw。embstr编码的字符串对象可以认为是只读的，因为Redis为其编写任何修改程序。当我们要修改embstr编码字符串时，都是先将转换为raw编码，然后再进行修改。

**Redis字符串结构特点**

- O(1) 时间复杂度获取：字符串长度，已用长度，未用长度。
- 可用于保存字节数组，支持安全的二进制数据存储。
- 内部实现空间预分配机制，降低内存再分配次数。
- 惰性删除机制，字符串缩减后的空间不释放，作为预分配空间保留。

## 4.List

列表对象的编码可以是linkedlist或者ziplist，对应的底层数据结构是链表和压缩列表。

默认情况下，当列表对象保存的所有字符串元素的长度都小于64字节，且元素个数小于512个时，列表对象采用的是ziplist编码，否则使用linkedlist编码。可以通过配置文件修改该上限值。

**链表**

提供了高效的节点重排能力以及顺序性的节点访问方式。在Redis中，每个链表节点使用listNode结构表示：

```text
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode
```

多个listNode通过prev和next指针组成双端链表，如下图所示：



![img](https://pic2.zhimg.com/80/v2-a6bb9af8ff3343c28842ecee72868f2d_720w.webp)

为了操作起来比较方便，Redis使用了list结构持有链表。list结构为链表提供了表头指针head、表尾指针tail，以及链表长度计数器len，而dup、free和match成员则是实现多态链表所需类型的特定函数。

```text
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表包含的节点数量
    unsigned long len;
    // 节点复制函数
    void *(*dup)(void *ptr);
    // 节点释放函数
    void (*free)(void *ptr);
    // 节点对比函数
    int (*match)(void *ptr, void *key);
} list;
```

![img](https://pic3.zhimg.com/80/v2-74d57bfb24b6c9418a3a9fe484f784ea_720w.webp)

Redis链表实现的特征总结如下：

- 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(n)。
- 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点。
- 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)。
- 带链表长度计数器：程序使用list结构的len属性来对list持有的节点进行计数，程序获取链表中节点数量的复杂度为O(1)。
- 多态：链表节点使用void*指针来保存节点值，可以保存各种不同类型的值。

**压缩列表**

压缩列表。redis的列表键和哈希键的底层实现之一。此数据结构是为了节约内存而开发的。和各种语言的数组类似，它是由连续的内存块组成的，这样一来，由于内存是连续的，就减少了很多内存碎片和指针的内存占用，进而节约了内存。

![img](https://pic1.zhimg.com/80/v2-3bbf2bc934d9351e75afcd29dedf1680_720w.webp)

entry的结构是这样的：

![img](https://pic2.zhimg.com/80/v2-c4c20af18eaed68f84bd38813ed64f89_720w.webp)

压缩列表记录了各组成部分的类型、长度以及用途:

![img](https://pic2.zhimg.com/80/v2-3ae8233fbdad51600e76e49208b264c1_720w.webp)

## 5.Hash

哈希对象的编码可以是ziplist或者hashtable。

哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节并且保存的键值对数量小于 512 个，使用ziplist 编码；否则使用hashtable；

**hash-ziplist**

ziplist底层使用的是压缩列表实现，前面已经详细介绍了压缩列表的实现原理。每当有新的键值对要加入哈希对象时，先把保存了键的节点推入压缩列表表尾，然后再将保存了值的节点推入压缩列表表尾。比如，我们执行如下三条HSET命令：

```text
HSET profile name "tom"
HSET profile age 25
HSET profile career "Programmer"
```

如果此时使用ziplist编码，那么该Hash对象在内存中的结构如下：

![img](https://pic3.zhimg.com/80/v2-393b86ab3257d0078043a79d91e5e8e6_720w.webp)

**hash-hashtable**

hashtable 编码的哈希对象使用字典dictht作为底层实现。字典是一种保存键值对的数据结构。

```text
typedef struct dictht{
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size-1
    unsigned long sizemask;
    // 该哈希表已有节点数量
    unsigned long used;
} dictht
```

table属性是一个数组，数组中的每个元素都是一个指向dictEntry结构的指针，每个dictEntry结构保存着一个键值对。size属性记录了哈希表的大小，即table数组的大小。used属性记录了哈希表目前已有节点数量。sizemask总是等于size-1，这个值主要用于数组索引。比如下图展示了一个大小为4的空哈希表。

![img](https://pic3.zhimg.com/80/v2-b2fc8a2c2e1eac193e79c41e6cb378b2_720w.webp)

**哈希表节点**

哈希表节点使用dictEntry结构表示，每个dictEntry结构都保存着一个键值对：

```text
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        unit64_t u64;
        nit64_t s64;
    } v;
    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

key属性保存着键值对中的键，而v属性则保存了键值对中的值。值可以是一个指针，一个uint64_t整数或者是int64_t整数。next属性指向了另一个dictEntry节点，在数组桶位相同的情况下，将多个dictEntry节点串联成一个链表，以此来解决键冲突问题。(链地址法)

**字典**

Redis字典由dict结构表示：

```text
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    //rehash索引
    // 当rehash不在进行时，值为-1
    int rehashidx;
}
```

ht是大小为2，且每个元素都指向dictht哈希表。一般情况下，字典只会使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用。rehashidx记录了rehash的进度，如果目前没有进行rehash，值为-1。

![img](https://pic2.zhimg.com/80/v2-c64e440195ff4a807fa701348cac3c09_720w.webp)

**rehash**

为了使hash表的负载因子(ht[0]).used/ht[0]).size)维持在一个合理范围，当哈希表保存的元素过多或者过少时，程序需要对hash表进行相应的扩展和收缩。rehash（重新散列）操作就是用来完成hash表的扩展和收缩的。rehash的步骤如下：

1. 为ht[1]哈希表分配空间，如果是扩展操作，那么ht[1]的大小为第一个大于ht[0].used*2的2^n。比如ht[0].used=5，那么此时ht[1]的大小就为16。(大于10的第一个2^n的值是16)如果是收缩操作，那么ht[1]的大小为第一个大于ht[0].used的2^n。比如ht[0].used=5，那么此时ht[1]的大小就为8。(大于5的第一个2^n的值是8)
2. 将保存在ht[0]中的所有键值对rehash到ht[1]中。
3. 迁移完成之后，释放掉ht[0]，并将现在的ht[1]设置为ht[0]，在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

**哈希表的扩展和收缩时机：**

当服务器没有执行BGSAVE或者BGREWRITEAOF命令时，负载因子大于等于1触发哈希表的扩展操作。

当服务器在执行BGSAVE或者BGREWRITEAOF命令，负载因子大于等于5触发哈希表的扩展操作。

当哈希表负载因子小于0.1，触发哈希表的收缩操作。

**渐进式rehash**

前面讲过，扩展或者收缩需要将ht[0]里面的元素全部rehash到ht[1]中，如果ht[0]元素很多，显然一次性rehash成本会很大，从影响到Redis性能。为了解决上述问题，Redis使用了渐进式rehash技术，具体来说就是分多次，渐进式地将ht[0]里面的元素慢慢地rehash到ht[1]中。下面是渐进式rehash的详细步骤：

1. 为ht[1]分配空间。
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash正式开始。
3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新时，除了会执行相应的操作之外，还会顺带将ht[0]在rehashidx索引位上的所有键值对rehash到ht[1]中，rehash完成之后，rehashidx值加1。
4. 随着字典操作的不断进行，最终会在某个时刻迁移完成，此时将rehashidx值置为-1，表示rehash结束。

渐进式rehash一次迁移一个桶上所有的数据，设计上采用分而治之的思想，将原本集中式的操作分散到每个添加、删除、查找和更新操作上，从而避免集中式rehash带来的庞大计算。

因为在渐进式rehash时，字典会同时使用ht[0]和ht[1]两张表，所以此时对字典的删除、查找和更新操作都可能会在两个哈希表进行。比如，如果要查找某个键时，先在ht[0]中查找，如果没找到，则继续到ht[1]中查找。

**hash对象中的hashtable**

```text
HSET profile name "tom"
HSET profile age 25
HSET profile career "Programmer"
```

还是上述三条命令，保存数据到Redis的哈希对象中，如果采用hashtable编码保存的话，那么该Hash对象在内存中的结构如下：

![img](https://pic2.zhimg.com/80/v2-af0ae1983921d4676344cec4f2a960ad_720w.webp)

## 6.Set

集合对象的编码可以是intset或者hashtable。

当集合对象保存的元素都是整数，并且个数不超过512个时，使用intset编码，否则使用hashtable编码。

**set-intset**

整数集合(intset)是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且保证集合中的数据不会重复。Redis使用intset结构表示一个整数集合。

```text
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

contents数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数组项，各个项在数组中按值大小从小到大有序排列，并且数组中不包含重复项。虽然contents属性声明为int8_t类型的数组，但实际上，contents数组不保存任何int8_t类型的值，数组中真正保存的值类型取决于encoding。如果encoding属性值为INTSET_ENC_INT16，那么contents数组就是int16_t类型的数组，以此类推。

当新插入元素的类型比整数集合现有类型元素的类型大时，整数集合必须先升级，然后才能将新元素添加进来。这个过程分以下三步进行。

1. 根据新元素类型，扩展整数集合底层数组空间大小。
2. 将底层数组现有所有元素都转换为与新元素相同的类型，并且维持底层数组的有序性。
3. 将新元素添加到底层数组里面。

还有一点需要注意的是，整数集合不支持降级，一旦对数组进行了升级，编码就会一直保持升级后的状态。

举个栗子，当我们执行SADD numbers 1 3 5向集合对象插入数据时，该集合对象在内存的结构如下：

![img](https://pic1.zhimg.com/80/v2-a20ed4b2ace422201981cf8312768c64_720w.webp)

**set-hashtable**

hashtable编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象对应一个集合元素，字典的值都是NULL。当我们执行SADD fruits "apple" "banana" "cherry"向集合对象插入数据时，该集合对象在内存的结构如下：

![img](https://pic4.zhimg.com/80/v2-93a5b34d2985c38f9bc327a3f804c88f_720w.webp)

## 7.Zset

有序集合的编码可以是ziplist或者skiplist。

当有序集合保存的元素个数小于128个，且所有元素成员长度都小于64字节时，使用ziplist编码，否则，使用skiplist编码。

**zset-ziplist**

ziplist编码的有序集合使用压缩列表作为底层实现，每个集合元素使用两个紧挨着一起的两个压缩列表节点表示，第一个节点保存元素的成员(member)，第二个节点保存元素的分值(score)。

压缩列表内的集合元素按照分值从小到大排列。如果我们执行ZADD price 8.5 apple 5.0 banana 6.0 cherry命令，向有序集合插入元素，该有序集合在内存中的结构如下：

![img](https://pic2.zhimg.com/80/v2-58799bceef062c08b18d6c092525c805_720w.webp)

**zset-skiplist**

skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表。

```text
typedef struct zset {

    zskiplist *zs1;
    dict *dict;
}
```

继续介绍之前，我们先了解一下什么是跳跃表。

**跳跃表**

跳跃表(skiplist)是一种有序的数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

Redis的跳跃表由zskiplistNode和zskiplist两个结构定义，zskiplistNode结构表示跳跃表节点，zskiplist保存跳跃表节点相关信息，比如节点的数量，以及指向表头和表尾节点的指针等。

**跳跃表节点 zskiplistNode**

跳跃表节点zskiplistNode结构定义如下：

```text
typedef struct zskiplistNode {
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

下图是一个层高为5，包含4个跳跃表节点(1个表头节点和3个数据节点)组成的跳跃表:

![img](https://pic4.zhimg.com/80/v2-7e1efd9467a252e3fba13bca85f1ddcb_720w.webp)

1. 每次创建一个新的跳跃表节点的时候，会根据幂次定律(越大的数出现的概率越低)随机生成一个1-32之间的值作为当前节点的"层高"。每层元素都包含2个数据，前进指针和跨度。
   前进指针:每层都有一个指向表尾方向的前进指针，用于从表头向表尾方向访问节点。
   跨度:层的跨度用于记录两个节点之间的距离。
2. 后退指针(BW)
   节点的后退指针用于从表尾向表头方向访问节点，每个节点只有一个后退指针，所以每次只能后退一个节点。
3. 分值和成员
   节点的分值(score)是一个double类型的浮点数，跳跃表中所有节点都按分值从小到大排列。节点的成员(obj)是一个指针，指向一个字符串对象。在跳跃表中，各个节点保存的成员对象必须是唯一的，但是多个节点的分值确实可以相同。

需要注意的是，表头节点不存储真实数据，并且层高固定为32，从表头节点第一个不为NULL最高层开始，就能实现快速查找。

**跳跃表 zskiplist**

实际上，仅靠多个跳跃表节点就可以组成一个跳跃表，但是Redis使用了zskiplist结构来持有这些节点，这样就能够更方便地对整个跳跃表进行操作。比如快速访问表头和表尾节点，获得跳跃表节点数量等等。zskiplist结构定义如下：

```text
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct skiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 最大层数
    int level;
} zskiplist;
```

下图是一个完整的跳跃表结构示例：

![img](https://pic4.zhimg.com/80/v2-89b03e4542c1b22156ee9936ed27b1d7_720w.webp)

**有序集合对象的skiplist实现**

前面讲过，skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表。

```text
typedef struct zset {
    zskiplist *zs1;
    dict *dict;
}
```

zset结构中的zs1跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素。通过跳跃表，可以对有序集合进行基于score的快速范围查找。zset结构中的dict字典为有序集合创建了从成员到分值的映射，字典的键保存了成员，字典的值保存了分值。通过字典，可以用O(1)复杂度查找给定成员的分值。

假如还是执行ZADD price 8.5 apple 5.0 banana 6.0 cherry命令向zset保存数据，如果采用skiplist编码方式的话，该有序集合在内存中的结构如下：

![img](https://pic1.zhimg.com/80/v2-cf193c3801410ca94c3fe61c1edb9158_720w.webp)

## 8.总结

Redis底层数据结构主要包括简单动态字符串(SDS)、链表、字典、跳跃表、整数集合和压缩列表六种类型，并且基于这些基础数据结构实现了字符串对象、列表对象、哈希对象、集合对象以及有序集合对象五种常见的对象类型。每一种对象类型都至少采用了2种数据编码，不同的编码使用的底层数据结构也不同。

原文地址：https://zhuanlan.zhihu.com/p/531323771

作者：linux