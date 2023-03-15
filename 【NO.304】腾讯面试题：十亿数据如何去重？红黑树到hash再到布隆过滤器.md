# 【NO.304】腾讯面试题：十亿数据如何去重？红黑树到hash再到布隆过滤器

在开始之前我们先思考以下几个问题

a1.在使⽤word⽂档时，word如何判断某个单词是否拼写正确？

a2.网络爬虫程序，怎么让它不去爬相同的url⻚⾯？

a3.垃圾邮件（短信）过滤算法如何设计？

a4.公安办案时，如何判断某嫌疑⼈是否在⽹逃名单中？

a5.缓存穿透问题如何解决？

对于前四个问题，这些大量的数据我们如何进行存储判断

- 应该将需要对比的数据以何种方式存储才能提高对比速率？
- 如何才能确保进行数据对比时候的高效性？

如果数据量比较少，我们可以直接挨个与数据库存储的信息进行对比得到答案，但如果数据量十分庞大，甚至达到上亿的级别，我们还能直接与数据库的信息进行对比吗？

想要明白上一点，我们先来聊聊关于缓存穿透的问题，它是如何发生的，以及我们该如何解决它。

![img](https://pic1.zhimg.com/80/v2-97ea446c2cf0fa5b4b2a8ac68ce90f90_720w.webp)

当server端向数据库请求数据时，中间缓存组件(redis)和数据库(此处是MySQL）都不存在相应数据，此时如果请求量较大，由于中间缓冲组件(redis)没有对应的缓存数据，于是压力全部给到数据库上，导致压力过大出现系统瘫痪的情况被称之为缓存穿透。

需求：

从海量数据中查询某字符串是否存在。

学过C++的朋友都知道set和map，但你清除它们的内部是如何实现的吗？

set和map

c++标准库（STL）中的set和map结构都是采⽤红⿊树实现的，它增删改查的时间复杂度是O(log2 N）；

图结构示例：

![img](https://pic1.zhimg.com/80/v2-42d699e7090c4232db531c455df97b60_720w.webp)

对于严格平衡二叉搜索树(AVL)，100w条数据组成的红黑树，只需要比较20次就能找到该值；对于10亿条数据只需要比较30次就能找到该数据；也就是查找次数跟树的维度是一致的；

对于红黑树来说平衡的是⿊节点⾼度，所以研究⽐较次数需要考虑树的⾼度差，最好情况某条树链路全是⿊节点，假设此时⾼度为h1，最差情况某条树链路全是⿊红节点间隔，那么此时树⾼度为2*h1;

在红⿊树中每⼀个节点都存储key和val字段，key是⽤来做⽐较的字段；红⿊树并没有要求key字段唯⼀，在set和map实现过程中限制了key字段唯⼀。我们来看nginx的红⿊树实现：

```text
// 这个是截取 nginx 的红⿊树的实现，这段代码是 insert 操作中的⼀部分，执⾏完这个函数还需要检测插⼊节点后是否平衡（主要是看他的⽗节点是否也是红⾊节点）
// 调⽤ ngx_rbtree_insert_value 时，temp传的参数为 红⿊树的根节点，node传的参数为待插⼊的节点
void ngx_rbtree_insert_value(ngx_rbtree_node_t *temp, ngx_rbtree_node_t
*node,
 	ngx_rbtree_node_t *sentinel) 
 {
 ngx_rbtree_node_t **p;
  for ( ;; ) {
	 p = (node->key < temp->key) ? &temp->left : &temp->right;// 这⾏很重要
	 if (*p == sentinel) {
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
// 不插⼊相同节点 如果插⼊相同 让它变成修改操作 此时 红⿊树当中就不会有相同的key了
定时器 key 时间戳
// 如果我们插⼊key = 12，如上图红⿊树，12号节点应该在哪个位置？ 如果我们要实现插⼊存在的节点变成修改操作，该怎么改上⾯的函数
void ngx_rbtree_insert_value_ex(ngx_rbtree_node_t *temp, ngx_rbtree_node_t
*node,  ngx_rbtree_node_t *sentinel) {
  ngx_rbtree_node_t **p;
   for ( ;; ) {
// {-------------add-------------
  if (node->key == temp->key) {
  temp->value = node->value;
 return;
 }
// }-------------add-------------
 p = (node->key < temp->key) ? &temp->left : &temp->right;// 这⾏很重要
  if (*p == sentinel) {
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
```

另外set和map的关键区别是set不存储val字段；

优点：存储效率⾼，访问速度⾼效；

缺点：对于数据量⼤且查询字符串⽐较⻓且查询字符串相似时将会是噩梦；

unordered_map

**过渡点:**

通过上边的内容我们已经知道了map内部是由红黑树实现的，时间复杂度相对较低了，但如果是一个字符单词，甚至是一个url(很长的字符串),如果数据量较大时，比如10亿条数据需要⽐较30次能找到该数据，但对于字符串的比较相对也是比较耗时的，有没有一种办法能避免或者减少比较呢，于是出现了hash结构(通过数组和hash函数来实现的一种数据结构)。

map内部是红黑树实现的，unordered_map则采用了hashtable



c++标准库（STL）中的unordered_map<string, bool>是采⽤hashtable实现的；

构成：数组+hash函数；

它是将字符串通过hash函数⽣成⼀个整数再映射到数组当中；它增删改查的时间复杂度是o(1);

图结构示例：

![img](https://pic4.zhimg.com/80/v2-02591ffa6eea47a41dea979d26213cbf_720w.webp)

hash函数的作⽤：避免插⼊的时候字符串的⽐较；hash函数计算出来的值通过对数组⻓度的取模

能随机分布在数组当中；

hash函数⼀般返回的是64位整数，将多个⼤数映射到⼀个⼩数组中，必然会产⽣冲突；

**如何选取hash函数？**

1. 选取计算速度快；
2. 哈希相似字符串能保持强随机分布性（防碰撞）；

murmurhash1，murmurhash2，murmurhash3，siphash（redis6.0当中使⽤，rust等⼤多数语⾔选⽤的hash算法来实现hashmap），cityhash都具备强随机分布性；测试地址如下：

[GitHub - aappleby/smhasher: Automatically exported from code.google.com/p/smhasher](https://link.zhihu.com/?target=https%3A//github.com/aappleby/smhasher)

负载因⼦：数组存储元素的个数/数组⻓度；负载因⼦越⼩，冲突越⼩；负载因⼦越⼤，冲突越⼤；

hash冲突解决⽅案：

链表法

引⼊链表来处理哈希冲突；也就是将冲突元素⽤链表链接起来；这也是常⽤的处理冲突的⽅式；但是可能出现⼀种极端情况，冲突元素⽐较多，该冲突链表过⻓，这个时候可以将这个链表转换为红⿊树；由原来链表时间复杂度 o(n) 转换为红⿊树时间复杂度 ；那么判

断该链表过⻓的依据是多少？可以采⽤超过256（经验值）个节点的时候将链表结构转换为红⿊树结构；

开放寻址法

将所有的元素都存放在哈希表的数组中，不使⽤额外的数据结构；⼀般使⽤线性探查的思路解决；

1 当插⼊新元素的时，使⽤哈希函数在哈希表中定位元素位置；

\2. 检查数组中该槽位索引是否存在元素。如果该槽位为空，则插⼊，否则3；

3 在 2 检测的槽位索引上加⼀定步⻓接着检查2；

加⼀定步⻓分为以下两种：

![img](https://pic4.zhimg.com/80/v2-5964c0a456f7ff55ae4c3340a4fe9a33_720w.webp)

这两种都会导致同类hash聚集；也就是近似值它的hash值也近似，那么它的数组槽位也靠近，形成hash聚集；第⼀种同类聚集冲突在前，**第⼆种只是将聚集冲突延后**；

另外还可以使⽤**双重哈希**来解决上⾯出现hash聚集现象：

![img](https://pic4.zhimg.com/80/v2-ab96b799847a9787c724166b1fa78763_720w.webp)

重点：何为同类hash聚集；也就是近似值它的hash值也近似，比如（nihao,nihaoa）这两个字符串由于很接近，所有通过一个hash函数得到值可能比较解决，就会出现hash聚集的情况，所以我们采用了双重哈希来解决这个问题。 双重哈希其实就是（双重散列分布算法）来保证相近的字符串通过hash函数得到的结果具有强随机分布性（防碰撞）从而避免hash聚集。

具体原理请点击：[双重哈希原理](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/organic/p/6283476.html)

同样的hashtable中节点存储了key和val，hashtable并没有要求key的⼤⼩顺序，我们同样可以修改代码让插⼊存在的数据变成修改操作；

优点：访问速度更快；不需要进⾏字符串⽐较；

缺点：需要引⼊策略避免冲突，存储效率不⾼；空间换时间；

**细节总结**

到目前为止，我们已经明白了红黑树以及hash对于数据查找带来的各自的优势，但它们都有一个最大的共同缺点，就是无法解决十分庞大的数据问题。因为不管是红黑树还是hashtable，它们都需要存储具体字符串，如果是十亿的数据，显然提供不了几百G的内存给它存放；所以在不断的发掘一种不用存放key值，同时具有hash结构的优点的数据结构。于是出现了我们今天的主角

**布隆过滤器**

定义：布隆过滤器是⼀种概率型数据结构，它的特点是⾼效的插⼊和查询，能明确告知某个字符串⼀定不存在或者可能存在

布隆过滤器相⽐传统的查询结构（例如：hash，set，map等数据结构）更加⾼效，占⽤空间更⼩；但是其缺点是它返回的结果是概率性的，也就是说结果存在误差的，虽然这个误差是可控的；

同时它不⽀持删除操作(至于为何不支持，继续往后看)

组成：位图（bit数组）+ n个hash函数

![img](https://pic3.zhimg.com/80/v2-f96586a424dadb2ba6765aab2f50117a_720w.webp)

原理：当⼀个元素加⼊位图时，通过k个hash函数将这个元素映射到位图的k个点，并把它们置为1；当检索时，再通过k个hash函数运算检测位图的k个点是否都为1；如果有不为1的点，那么认为不存在；如果全部为1，则可能存在（存在误差）；

![img](https://pic1.zhimg.com/80/v2-df8e2903a871f414a9b34fa0ef976bd8_720w.webp)

在位图中每个槽位只有两种状态（0或者1），⼀个槽位被设置为1状态，但不明确它被设置了多少次；也就是不知道被多少个str1哈希映射以及是被哪个hash函数映射过来的；所以不⽀持删除操作；

在实际应⽤过程中，布隆过滤器该如何使⽤？要选择多少个hash函数，要分配多少空间的位图，存储多少元素？另外如何控制假阳率（布隆过滤器能明确⼀定不存在，不能明确⼀定存在，那么存在的判断是有误差的，假阳率就是错误判断存在的概率）？

![img](https://pic1.zhimg.com/80/v2-0abf3e8791f0959ffc2caf0180d35e60_720w.webp)

假定我们选取这四个值为：
n = 4000
p = 0.000000001
m = 172532
k = 30

- 四个值的关系：

![img](https://pic2.zhimg.com/80/v2-8bca720422fa42f27ac8f2f9505f8c91_720w.webp)

![img](https://pic4.zhimg.com/80/v2-e5ca5e8976286a362569d793248e61d7_720w.webp)

![img](https://pic1.zhimg.com/80/v2-1f0dfcafcfe1bd15d1b1c2fa1600f8a0_720w.webp)

在实际应⽤中，我们确定n和p，通过上⾯的计算算出m和k；也可以在⽹站上选取合适的值：[计算m和k的网站(点我https://hur.st/bloomfilter）](https://link.zhihu.com/?target=https%3A//hur.st/bloomfilter)

已知k，如何选择k个hash函数？

**布隆过滤器的细节点：**

1:这里的k个hash函数是什么意思，真的是k个不同的哈希函数吗？其实通过下边的代码我们就可以明白，实际上只有一个哈希函数，但上边讲hashtable的时候我们说过可以采用双重hash的结构来避免同类哈希聚集，所以下边的代码我们可以看出同样采用了双重hash的结构，同时使用一个循环来实现这些命中的点的不同位置(以防命中同一个点)

2:还有一个细节点要明白：当⼀个元素加⼊位图时，通过k个hash函数将这个元素映射到位图的k个点，并把它们全置为1；当检索时，再通过k个hash函数运算检测位图的k个点是否都为1；如果有不为1的点，那么认为不存在；如果全部为1，则可能存在（存在误差）； 这里一定要明白，加入过程和检索过程是不一样的，加入是全设为1，检索是判断是否全为1，如果不是全为1，说明从来没有加入过，所以一定不存在，而就算全是1，也不一定存在，因为确实有可能同一个数据通过多次哈希都映射到了相同的几个位图点上。

// 采⽤⼀个hash函数，给hash传不同的种⼦偏移值

```text
// #define MIX_UINT64(v) ((uint32_t)((v>>32)^(v)))
uint64_t hash1 = MurmurHash2_x64(key, len, Seed);
uint64_t hash2 = MurmurHash2_x64(key, len, MIX_UINT64(hash1));
for (i = 0; i < k; i++) // k 是hash函数的个数
{
 Pos[i] = (hash1 + i*hash2) % m; // m 是位图的⼤⼩
}
```

// 通过这种⽅式来模拟 k 个hash函数 跟我们前⾯开放寻址法 双重hash是⼀样的思路

[应⽤源码http://gitlab.0voice.com/0voice/bloomfilter/tree/master：](https://link.zhihu.com/?target=http%3A//gitlab.0voice.com/0voice/bloomfilter/tree/master)

**大总结：**

本文章学习了从红黑树到hashtable再到布隆过滤器的演变，阐述了它们各自的优缺点，那么我们也应该深入了解它们各自都使用于哪些场景下：

对于普通类型的数据，可以采用红黑树来快速进行查找，而对于字符串类型的数据，如果采用红黑树进行查找，由于字符串的对比会花费较多的比较时间，所以我们采用了更好的方法hashtable(数组和hash函数配合)，从而避免或者减少了字符串的比较，提高查找效率，再后来引申到对于海量数据的存储管理，这两种结构都不适合，因为要进行存储的数据量实在太大，所以出现了布隆过滤器这种不需要存储key值就能进行判断的结构。

原文地址：https://zhuanlan.zhihu.com/p/536749623

作者：Linux