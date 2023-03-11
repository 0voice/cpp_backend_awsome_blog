# 【NO.172】MySQL 深入学习总结

## **1.数据库基础**

### 1.1 MySQL 架构

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNj8xvPB51Q6fuLFiatUpBPyMMAccahJwLgia74sxRXvCtQ8icm3pXyPOS2w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

和其它数据库相比，MySQL 有点与众不同，它的架构可以在多种不同场景中应用并发挥良好作用。主要体现在存储引擎的架构上，插件式的存储引擎架构将查询处理和其它的系统任务以及数据的存储提取相分离。这种架构可以根据业务的需求和实际需要选择合适的存储引擎，各层介绍：

#### **1.1.1 连接层**

最上层是一些客户端和连接服务，包含本地 sock 通信和大多数基于客户端/服务端工具实现的类似于 tcp/ip 的通信。主要完成一些类似于连接处理、授权认证、及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于 SSL 的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

#### **1.1.2 服务层**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjRnaus2X6PTY5qjzfoSnbOuiadJGpzbp8VBf1ic8xasqy0FgAx6NibDtRg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.1.3 引擎层**

存储引擎层，存储引擎真正的负责了 MySQL 中数据的存储和提取，服务器通过 API 与存储引擎进行通信。不同的存储引擎具有的功能不同，这样我们可以根据自己的实际需要进行选取。

#### **1.1.4 存储层**

数据存储层，主要是将数据存储在运行于裸设备的文件系统之上，并完成与存储引擎的交互。

### 1.2 数据引擎

不同的存储引擎都有各自的特点，以适应不同的需求，如表所示。为了做出选择，首先要考虑每一个存储引擎提供了哪些不同的功能。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjqZpsuouicEx4PVBcwWvWUicNDbFjncRMryh9S4IUJ3zn4klqoiaoc56sQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.2.1 MyISAM**

使用这个存储引擎，每个 MyISAM 在磁盘上存储成三个文件。

1. frm 文件：存储表的定义数据
2. MYD 文件：存放表具体记录的数据
3. MYI 文件：存储索引

#### **1.2.2 InnoDB**

InnoDB 是默认的数据库存储引擎，他的主要特点有：

1. 可以通过自动增长列，方法是 auto_increment；
2. 支持事务。默认的事务隔离级别为可重复度，通过 MVCC（并发版本控制）来实现的；
3. 使用的锁粒度为行级锁，可以支持更高的并发；
4. 支持外键约束；外键约束其实降低了表的查询速度，但是增加了表之间的耦合度；
5. 配合一些热备工具可以支持在线热备份；
6. 在 InnoDB 中存在着缓冲管理，通过缓冲池，将索引和数据全部缓存起来，加快查询的速度；
7. 对于 InnoDB 类型的表，其数据的物理组织形式是聚簇表。所有的数据按照主键来组织。数据和索引放在一块，都位于 B+数的叶子节点上。

#### **1.2.3 Memory**

将数据存在内存，为了提高数据的访问速度，每一个表实际上和一个磁盘文件关联。文件是 frm。

1. 支持的数据类型有限制，比如：不支持 TEXT 和 BLOB 类型，对于字符串类型的数据，只支持固定长度的行；VARCHAR 会被自动存储为 CHAR 类型；
2. 支持的锁粒度为表级锁。所以，在访问量比较大时，表级锁会成为 MEMORY 存储引擎的瓶颈；
3. 由于数据是存放在内存中，一旦服务器出现故障，数据都会丢失；
4. 查询的时候，如果有用到临时表，而且临时表中有 BLOB，TEXT 类型的字段，那么这个临时表就会转化为 MyISAM 类型的表，性能会急剧降低；
5. 默认使用 hash 索引；
6. 如果一个内部表很大，会转化为磁盘表。

### 1.3 表与字段设计

#### **1.3.1 数据库基本设计规范**

1. 尽量控制单表数据量的大小，建议控制在 500 万以。500 万并不是 MySQL 数据库的限制，过大会造成修改表结构、备份、恢复都会有很大的问题，可以用历史数据归档（应用于日志数据），分库分表（应用于业务数据）等手段来控制数据量大小；
2. 谨慎使用 MySQL 分区表。分区表在物理上表现为多个文件，在逻辑上表现为一个表 谨慎选择分区键，跨分区查询效率可能更低 建议采用物理分表的方式管理大数据；
3. 禁止在数据库中存储图片，文件等大的二进制数据。通常文件很大，会短时间内造成数据量快速增长，数据库进行数据库读取时，通常会进行大量的随机 IO 操作，文件很大时，IO 操作很耗时 通常存储于文件服务器，数据库只存储文件地址信息；
4. 禁止在线上做数据库压力测试。

#### **1.3.2 数据库字段设计规范**

1. 优先选择符合存储需要的最小的数据类型。列的字段越大，建立索引时所需要的空间也就越大，这样一页中所能存储的索引节点的数量也就越少也越少，在遍历时所需要的 IO 次数也就越多， 索引的性能也就越差；
2. 避免使用 TEXT、BLOB 数据类型，最常见的 TEXT 类型可以存储 64k 的数据；
3. 尽可能把所有列定义为 NOT NULL。

#### **1.3.3 索引设计规范**

1. 限制每张表上的索引数量，建议单张表索引不超过 5 个；
2. 禁止给表中的每一列都建立单独的索引；
3. 每个 InnoDB 表必须有个主键；
4. 建立索引的目的是：希望通过索引进行数据查找，减少随机 IO，增加查询性能 ，索引能过滤出越少的数据，则从磁盘中读入的数据也就越少。区分度最高的放在联合索引的最左侧（区分度 = 列中不同值的数量 / 列的总行数）。尽量把字段长度小的列放在联合索引的最左侧（因为字段长度越小，一页能存储的数据量越大，IO 性能也就越好）。使用最频繁的列放到联合索引的左侧（这样可以比较少的建立一些索引）。

#### **1.3.4 数据库 SQL 开发规范**

1. 充分利用表上已经存在的索引,避免使用双 % 号的查询条件。如 a like '%123%'，（如果无前置 %，只有后置 %，是可以用到列上的索引的）一个 SQL 只能利用到复合索引中的一列进行范围查询，如：有 a,b,c 列的联合索引，在查询条件中有 a 列的范围查询，则在 b,c 列上的索引将不会被用到，在定义联合索引时，如果 a 列要用到范围查找的话，就要把 a 列放到联合索引的右侧；使用 left join 或 not exists 来优化 not in 操作， 因为 not in 也通常会使用索引失效；
2. 禁止使用 SELECT * 必须使用 SELECT <字段列表> 查询；
3. 避免使用子查询，可以把子查询优化为 JOIN 操作；
4. 避免使用 JOIN 关联太多的表。

#### 

### 1.4 范式与反范式

#### **1.4.1 第一范式**

该范式是为了排除 重复组 的出现，因此要求数据库的每个列的值域都由原子值组成；每个字段的值都只能是单一值。1971 年埃德加·科德提出了第一范式。即表中所有字段都是不可再分的。解决方案：想要消除重复组的话，只要把每笔记录都转化为单一记录即可。

#### **1.4.2 第二范式**

表中必须存在业务主键，并且非主键依赖于全部业务主键。解决方案：拆分将依赖的字段单独成表。

#### **1.4.3 第三范式**

表中的非主键列之间不能相互依赖，将不与 PK 形成依赖关系的字段直接提出单独成表即可。

### 1.5 sql 索引

1. B 树只适合随机检索，适合文件操作，B+树同时支持随机检索和顺序检索；
2. B+树的磁盘读写代价更低， B+树的内部结点并没有指向关键字具体信息的指针；
3. B+树的查询效率更加稳定。B 树搜索有可能会在非叶子结点结束；
4. 只要遍历叶子节点就可以实现整棵树的遍历，数据库中基于范围的查询是非常频繁，B 树这样的操作效率非常低。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjfne8jicVmEJiaMMuS1j9FNQ1eib60O9UUubcwupIibRIs5EQwic3ykerNkg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 1.6 join 连表

#### **1.6.1 JOIN 按照功能大致分为如下三类：**

1. INNER JOIN（内连接,或等值连接）：获取两个表中字段匹配关系的记录。![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvaufKISYMVA6xk1p9XpneMNjHzm5WKGPnqvnyBuUKqh4HyabWMSHHn9DouFK6ia84jr7mpyT2WdpU3g/640?wx_fmt=gif&wxfrom=5&wx_lazy=1&wx_co=1)
2. LEFT JOIN（左连接）：获取左表所有记录，即使右表没有对应匹配的记录。![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvaufKISYMVA6xk1p9XpneMNjFA9iaDKpvdyuOS4W1NNNL0Z4PticHdB4bJiaFicH8RAAPibXmvVVk35E0icQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1&wx_co=1)
3. RIGHT JOIN（右连接）：与 LEFT JOIN 相反，用于获取右表所有记录，即使左表没有对应匹配的记录。![图片](https://mmbiz.qpic.cn/mmbiz_png/j3gficicyOvaufKISYMVA6xk1p9XpneMNjLG9BszRWO07ZaCysDfUsosia2MPTYYAGp8Kiah4GplRerPMIYzeicRfWw/640?wx_fmt=gif&wxfrom=5&wx_lazy=1&wx_co=1)

#### **1.6.2 join 的原理**

MySQL 使用了嵌套循环（Nested-Loop Join）的实现方式。Nested-Loop Join 需要区分驱动表和被驱动表，先访问驱动表，筛选出结果集，然后将这个结果集作为循环的基础，访问被驱动表过滤出需要的数据。Nested-Loop Join 分下面几种类型：

1. SNLJ，简单嵌套循环。这是最简单的方案，性能也一般。实际上就是通过驱动表的结果集作为循环基础数据，然后一条一条的通过该结果集中的数据作为过滤条件到下一个表中查询数据，然后合并结果。如果还有第三个参与 Join，则再通过前两个表的 Join 结果集作为循环基础数据，再一次通过循环查询条件到第三个表中查询数据，如此往复。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjQfBpDaMUiadkYynibYCats5l27FC4nn8PQBGM52td3MCqb1xnokvJsoQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



相关图片来源于网络

这个算法相对来说就是很简单了，从驱动表中取出 R1 匹配 S 表所有列，然后 R2，R3,直到将 R 表中的所有数据匹配完，然后合并数据，可以看到这种算法要对 S 表进行 RN 次访问，虽然简单，但是相对来说开销还是太大了

1. INLJ，索引嵌套循环。索引嵌套联系由于非驱动表上有索引，所以比较的时候不再需要一条条记录进行比较，而可以通过索引来减少比较，从而加速查询。这也就是平时我们在做关联查询的时候必须要求关联字段有索引的一个主要原因。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjIoXSJee6o9WBibHQrkvsyHtOkSM9VXf9xZmjxiawJgEnPEr3aicFRHWTQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



相关图片来源于网络

1. BNLJ，块嵌套循环。如果 join 字段没索引，被驱动表需要进行扫描。这里 MySQL 并不会简单粗暴的应用前面算法，而是加入了 buffer 缓冲区，降低了内循环的个数，也就是被驱动表的扫描次数。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjutXPdYZukU6K4R14OHCLU5cTPp15DcaxJ86DINdddhvIBceAzgn6Vw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

这个 buffer 被称为 join buffer，顾名思义，就是用来缓存 join 需要的字段。MySQL 默认 buffer 大小 256K，如果有 n 个 join 操作，会生成 n-1 个 join buffer。

#### **1.6.3 join 的优化**

1. 小结果集驱动大结果集。用数据量小的表去驱动数据量大的表，这样可以减少内循环个数，也就是被驱动表的扫描次数。
2. 用来进行 join 的字段要加索引，会触发 INLJ 算法，如果是主键的聚簇索引，性能最优。例子：第一个子查询是 72075 条数据，join 的第二条子查询是 50w 数据，主要的优化还是驱动表是小表，后面的是大表，on 的条件加上了唯一索引。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjGzCfc2mhkhPwCxY5kDOZbHbCFpDM8ETTCicIRjlwpWVNIEsZSUn5ia6A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

1. 如果无法使用索引，那么注意调整 join buffer 大小，适当调大些
2. 减少不必要的字段查询（字段越少，join buffer 所缓存的数据就越多）

## **2.数据进阶**

### 2.1 sql 执行过程

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjZrBQdiadbZNDWdc0mibfGJbK2Gard1VtHmEJ65U4hPCcdugeL3FCleZw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，当向 MySQL 发送一个请求的时候，MySQL 到底做了什么：

1. 客户端发送一条查询给服务器。服务器先检查查询缓存，如果命中了缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段；
2. 在解析一个查询语句之前，如果查询缓存是打开的，那么 MYSQL 会优先检查这个查询是否命中查询缓存中的数据；
3. 这个检查是通过一个对大小写敏感的哈希查找的。查询和缓存中的查询即使只有一个不同，也不会匹配缓存结果；
4. 如果命中缓存，那么在但会结果前 MySQL 会检查一次用户权限，有权限则跳过其他步骤直接返回数据；
5. 服务器端进行 SQL 解析、预处理，再由优化器生成对应的执行计划。MySQL 解析器将使用 MySQL 语法规则验证和解析查询。例如验证是否使用错误的关键字、关键字顺序、引号前后是否匹配等；预处理器则根据一些 MySQL 规则进一步解析树是否合法，例如检查数据表和数据列是否存在，解析名字和别名是否有歧义等；
6. MySQL 根据优化器生成的执行计划，再调用存储引擎的 API 来执行查询。

MySQL 的查询优化器使用很多策略来生成一个最优的执行计划。优化策略可以简单的分为两种：

1. 静态优化：静态优化可以直接对解析树进行分析，并完成优化。例如优化器可以通过简单的代数变化将 WHERE 条件转换成另外一种等价形式，静态优化在第一次完成后就一直有效，即使使用不同的参数重复执行查询也不会变化。可以认为是一种”编译时优化“。

1. 动态优化：和查询的上下文有关，也可能和其他因素有关，例如 WHERE 中取值、索引中条目对应的数据行数等。这需要在每次查询的时候重新评估，可以让那位 u 是"运行时优化"。

使用 show status like ‘Last_query_cost’ 可以查询上次执行的语句的成本，单位为数据页。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjeNuApvD7wlLiafM66u5kibxvnwZhW4H9GjkGBhcX95CbajvdClY7mVJw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjRYrfOr34WQuVQEVufdusrffU2YPrqXXTm9l1oCibJpMH85R5sEXkP0A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.2 sql 查询计划

使用 explain 进行执行计划分析：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjGMsDq0rNFqdJLW8icBLhUsBib4IbibDTfVkOmh5DRL0Guo87SB15PCvOA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjzahuIvAwpafVlhwr2ytAdR7bm1ibRw5VuOcwZEwdUAxeic2uKLbSElvQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNj5S7iayr7UySToEibxPwDticX2y1OGmY9j3fiajkd2VibicwPTKJaahBNX1Ww/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.3 sql 索引优化

遵循索引原则适合大部分的常规数据库查询场景，但不是所有的索引都能符合预期，从索引原理本身来分析对索引的创建会更有帮助。

1. 小表的全表扫描往往会比索引更快 ；
2. 中大型表使用索引会有很大的查询效率提升；超大型表，索引也无法解决慢查询，过多和过大的索引会带来更多的磁盘占用和降低 INSERT 效率。

#### **2.3.1 前缀索引**

当要索引的列字符很多时 索引则会很大且变慢( 可以只索引列开始的部分字符串 节约索引空间 从而提高索引效率 )

例如：一个数据表的 x_name 值都是类似 23213223.434323.4543.4543.34324 这种值，如果以整个字段值做索引，会使索引文件过大，但是如果设置前 7 位来做索引则不会出现重复索引值的情况了。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjdlz0kUvNGK6d1VhWE6d2FqweqGibH0aGFYb6HXX52vdqicHDosEcQF5g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

查询效率会大大提升：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjlsEOxicwvJJic5swOk1MlE0g3BY6eYKlurw50ibiay1VLyaUMOAT2icITVg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **2.3.2 联合索引顺序**

```
alter table table1 add key （distribute_type，plat_id）
```

使用选择基数更高（不重复的数据）的字段作为最左索引：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjDbXCtiamka1bwSgzKuzpiaiaODh5Bd6tKcbfoV8LrGoGGMVqqLGPe5ozQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **2.3.3 联合索引左前缀匹配**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjPkAlD5sg1tqibj4jSmzTO6D62sAj0o7MjVT7gVAnFXRicKdzAuGqHW2g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

1. a=? and b=? and c=?；查询效率最高，索引全覆盖
2. a=? and b=?；索引覆盖 a 和 b
3. a=? or b=?；索引覆盖 a 和 b
4. b=? or a=?；无法覆盖索引(>、<、between、like)
5. b=? and a=?；经过 mysql 的查询分析器的优化，索引覆盖 a 和 b
6. a=?；索引覆盖 a
7. b=? and c=?；没有 a 列，不走索引，索引失效
8. c=?；没有 a 列，不走索引，索引失效

### 2.4 慢查询分析

#### **2.4.1 先对 sql 语句进行 explain，查看语句存在的问题**

#### **2.4.2 使用 show profile 查看执行耗时，分析具体耗时原因**

show profile 的使用指引：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjusUmiblTqicyibxHPiaia7XHosUoyV3QwYgficeWbibVhjLfGGISuDErmVialg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.5 改表与 sql 日志

#### **2.5.1 改表**

改表会直接触发表锁，改表过程非常耗时，对于大表修改，无论是字段类型调整还是字段增删，都需要谨慎操作，防止业务表操作被阻塞，大表修改往往有以下几种方式。

1. 主备改表切换，先改冷库表，再执行冷热切换；
2. 直接操作表数据文件，拷贝文件替换；
3. 使用类似 percona-toolkit 工具操作表。

常用方法：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjDGeFQibKHXlOo3vJz5O2gSMLG1BzLp1KxRiad0IB3LU82fVicFXsu6icGA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

#### **2.5.2 sql 日志**

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjbrD7UomRQd6b4ZsNjtgR4RLsfcGH7Uc4pLGQV1Jt0QKTwOCkWsbmuw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjgKS26768iaRZ86tyZdI5dypric0vITkj1bLJAqiaX1Okt68tPFMQSvxnA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.6 分库与分表

#### **2.6.1 数据库瓶颈**

不管是 IO 瓶颈，还是 CPU 瓶颈，最终都会导致数据库的活跃连接数增加，进而逼近甚至达到数据库可承载活跃连接数的阈值。在业务 Service 来看就是，可用数据库连接少甚至无连接可用。接下来就可以想象了吧（并发量、吞吐量、崩溃）。

1. IO 瓶颈 第一种：磁盘读 IO 瓶颈，热点数据太多，数据库缓存放不下，每次查询时会产生大量的 IO，降低查询速度 -> 分库和垂直分表。

第二种：网络 IO 瓶颈，请求的数据太多，网络带宽不够 -> 分库。

1. CPU 瓶颈 第一种：SQL 问题，如 SQL 中包含 join，group by，order by，非索引字段条件查询等，增加 CPU 运算的操作 -> SQL 优化，建立合适的索引，在业务 Service 层进行业务计算。

第二种：单表数据量太大，查询时扫描的行太多，SQL 效率低，CPU 率先出现瓶颈 -> 水平分表。

#### **2.6.2 分库分表**

1. 水平分库![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNj8Ia5K9ibTicpLibzzT6Nyco07Tgyr5V4O1qc875OXbnbS0vFjgcurMj3Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

相关图片来源于网络

概念：以字段为依据，按照一定策略（hash、range 等），将一个库中的数据拆分到多个库中。结果：每个库的结构都一样；每个库的数据都不一样，没有交集；所有库的并集是全量数据；场景：系统绝对并发量上来了，分表难以根本上解决问题，并且还没有明显的业务归属来垂直分库。分析：库多了，io 和 cpu 的压力自然可以成倍缓解。

1. 水平分表![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjTNs8uqR60d9OGjYB4WKlIiaIQ0diaskXNoRx56sRR8fhIicqLUkRyTYiaw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)相关图片来源于网络

概念：以字段为依据，按照一定策略（hash、range 等），将一个表中的数据拆分到多个表中。结果：每个表的结构都一样；每个表的数据都不一样，没有交集；所有表的并集是全量数据。

场景：系统绝对并发量并没有上来，只是单表的数据量太多，影响了 SQL 效率，加重了 CPU 负担，以至于成为瓶颈。

分析：表的数据量少了，单次 SQL 执行效率高，自然减轻了 CPU 的负担。

1. 垂直分库![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjicT4ZWCEXMnv6yNA1NTqibAHVsG10k1DiaquQrO5KDVYJ21ibtfjfibSSCA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

   相关图片来源于网络

概念：以表为依据，按照业务归属不同，将不同的表拆分到不同的库中。

结果：每个库的结构都不一样；每个库的数据也不一样，没有交集；所有库的并集是全量数据。

场景：系统绝对并发量上来了，并且可以抽象出单独的业务模块。

分析：到这一步，基本上就可以服务化了。例如，随着业务的发展一些公用的配置表、字典表等越来越多，这时可以将这些表拆到单独的库中，甚至可以服务化。再有，随着业务的发展孵化出了一套业务模式，这时可以将相关的表拆到单独的库中，甚至可以服务化。

1. 垂直分表![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjV1lXZJWVANqECRdbPFtV6HqVYx4lJDgY9dtWhiaLBpFEs1OhEQ0Z7RQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

相关图片来源于网络

概念：以字段为依据，按照字段的活跃性，将表中字段拆到不同的表（主表和扩展表）中。

结果：每个表的结构都不一样；每个表的数据也不一样，一般来说，每个表的字段至少有一列交集，一般是主键，用于关联数据；所有表的并集是全量数据。

#### **2.6.3 分库分表工具**

目前市面上的分库分表中间件相对较多，其中基于代理方式的有 MySQL Proxy 和 Amoeba， 基于 Hibernate 框架的是 Hibernate Shards，基于 jdbc 的有当当 sharding-jdbc， 基于 mybatis 的类似 maven 插件式的有蘑菇街的蘑菇街 TSharding， 通过重写 spring 的 ibatis template 类的 Cobar Client。

还有一些大公司的开源产品：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjOdRiaA53bchSbSLvEraC0M1Bu3uoOdc0BAiaBTuXibSsgvrj4IKQsv2IA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## **3.分布式数据库**

### 3.1 什么是分布式数据库

分布式系统数据库系统原理(第三版)中的描述：“我们把分布式数据库定义为一群分布在计算机网络上、逻辑上相互关联的数据库。分布式数据库管理系统(分布式 DBMS)则是支持管理分布式数据库的软件系统，它使得分布对于用户变得透明。有时，分布式数据库系统(Distributed Database System,DDBS)用于表示分布式数据库和分布式 DBMS 这两者。

在以上表述中，“一群分布在网络上、逻辑上相互关联”是其要义。在物理上一群逻辑上相互关联的数据库可以分布式在一个或多个物理节点上。当然，主要还是应用在多个物理节点。这一方面是 X86 服务器性价比的提升有关，另一方面是因为互联网的发展带来了高并发和海量数据处理的需求，原来的单物理服务器节点不足以满足这个需求。

### 3.2 分布式数据库的理论基础

**1. CAP 理论**首先，分布式数据库的技术理论是基于单节点关系数据库的基本特性的继承，主要涉及事务的 ACID 特性、事务日志的容灾恢复性、数据冗余的高可用性几个要点。

其次，分布式数据的设计要遵循 CAP 定理，即：一个分布式系统不可能同时满足 一致性( Consistency ) 、可用性 ( Availability ) 、分区容 忍 性 ( Partition tolerance ) 这三个基本需求，最 多只能同时满足其中的两项， 分区容错性 是不能放弃的，因此架构师通常是在可用性和一致性之间权衡。这里的权衡不是简单的完全抛弃，而是考虑业务情况作出的牺牲，或者用互联网的一个术语“降级”来描述。

CAP 三个特性描述如下 ：一致性：确保分布式群集中的每个节点都返回相同的 、 最近 更新的数据 。一致性是指每个客户端具有相同的数据视图。有多种类型的一致性模型 ， CAP 中的一致性是指线性化或顺序一致性，是强一致性。

可用性：每个非失败节点在合理的时间内返回所有读取和写入请求的响应。为了可用，网络分区两侧的每个节点必须能够在合理的时间内做出响应。

分区容忍性：尽管存在网络分区，系统仍可继续运行并 保证 一致性。网络分区已成事实。保证分区容忍度的分布式系统可以在分区修复后从分区进行适当的恢复。

**2. BASE 理论**

基于 CAP 定理的权衡，演进出了 BASE 理论 ，BASE 是 Basically Available(基本可用)、Soft state(软状态)和 Eventually consistent(最终一致性)三个短语的缩写。BASE 理论的核心思想是：即使无法做到强一致性，但每个应用都可以根据自身业务特点，采用适当的方式来使系统达到最终一致性。

BA：Basically Available 基本可用，分布式系统在出现故障的时候，允许损失部分可用性，即保证核心可用；S：Soft state 软状态，允许系统存在中间状态，而该中间状态不会影响系统整体可用性；E：Consistency 最终一致性，系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。

BASE 理论本质上是对 CAP 理论的延伸，是对 CAP 中 AP 方案的一个补充。

### 3.3 分布式数据库的架构演变

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/j3gficicyOvaufKISYMVA6xk1p9XpneMNjCHJ9wcmptqYxsYnaMXaXcoevzUOWCzZ8koib3sEuL20ToDH0BFE3xJw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)三类数据库架构特点：

1. Shard-everting：共享数据库引擎和数据库存储，无数据存储问题。一般是针对单个主机，完全透明共享 CPU/MEMORY/IO，并行处理能力是最差的，典型的代表 SQLServer；
2. Shared-storage：引擎集群部署，分摊接入压力，无数据存储问题；
3. Shard-noting：引擎集群部署，分摊接入压力，存储分布式部署，存在数据存储问题。各个处理单元都有自己私有的 CPU/内存/硬盘等，不存在共享资源，类似于 MPP（大规模并行处理）模式，各处理单元之间通过协议通信，并行处理和扩展能力更好。典型代表 DB2 DPF 和 hadoop ，各节点相互独立，各自处理自己的数据，处理后的结果可能向上层汇总或在节点间流转。

原文作者：yandeng，腾讯 PCG 应用开发工程师

原文链接：https://mp.weixin.qq.com/s/sRFmW57KUY3yyyRkyw0L4A