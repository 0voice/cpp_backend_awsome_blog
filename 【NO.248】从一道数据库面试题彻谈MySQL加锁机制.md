# 【NO.248】从一道数据库面试题彻谈MySQL加锁机制

以下内容来自于腾讯工程师joeycheng。

| 导语有一道关于数据库锁的面试题，发现其实很多DBA包括工作好几年的DBA都答的不太好，说明MySQL锁的机制其实还是比较复杂，值得深入研究。本文对3条简单的查询语句加锁情况进行分析，彻底搞清楚加锁细节。

首先来看这个面试题：
已知表t是innodb引擘，有主键：id（int类型) ，下面3条语句是否加锁？加锁的话，是什么锁？
\1. select * from t where id=X;
\2. begin;select * from t where id=X;
\3. begin;select * from t where id=X for update;

这里其实有很多依赖条件，并不能一开始就给出一个很确定的答复。我们一层层展开来分析。

## 1.MySQL有哪些锁？

![img](https://pic4.zhimg.com/80/v2-1c319cbb084c605470802c180b3dfdaf_720w.webp)

首先要知道MySQL有哪些锁，如上图所示，至少有12类锁（其中**自增锁**是事务向包含了AUTO_INCREMENT列的表中新增数据时会持有，**predicate locks for spatial index** 为空间索引专用，本文不讨论这2类锁）。

锁按**粒度**可分为分为**全局，表级，行级**3类。

### **1.1 全局锁**

对整个数据库实例加锁。
加锁表现：数据库处于只读状态，阻塞对数据的所有DML/DDL
加锁方式：**Flush tables with read lock** 释放锁：**unlock tables**(发生异常时会自动释放)
作用场景：全局锁主要用于做数据库实例的逻辑备份，与设置数据库只读命令**set global readonly=true**相比，全局锁在发生异常时会自动释放

### 1.2 表锁

对操作的整张表加锁， 锁定颗粒度大，资源消耗少，不会出现死锁，但会导致写入并发度低。具体又分为3类：
**1）显式表锁：**分为共享锁（S）和排他锁（X）
显示加锁方式：**lock tables ... read/write**
释放锁：**unlock tables**(连接中断也会自动释放)
**2）Metadata-Lock（元数据锁）**：MySQL5.5版本开始引入，主要功能是并发条件下，防止session1的查询事务未结束的情况下，session2对表结构进行修改，保护元数据的一致性。在session1持有 metadata-lock的情况下，session2处于等待状态：show processlist可见**Waiting for table metadata lock**。其具体加锁机制如下：

- DML->先加MDL 读锁（SHARED_READ，SHARED_WRITE）
- DDL->先加MDL 写锁（EXCLUSIVE）
- 读锁之间兼容
- 读写锁之间、写锁之间互斥

**3）Intention Locks（意向锁）**：**意向锁**为表锁（表示为IS或者IX），由存储引擎自己维护，用户无法干预。下面举一个例子说明其功能：
假设有2个事务：T1和T2
T1: 锁住表中的一行，只能读不能写（行级读锁）。
T2: 申请整个表的写锁（表级写锁）。
如T2申请成功，则能任意修改表中的一行，但这与T1持有的行锁是冲突的。故数据库应识别这种冲突，让T2的锁申请被阻塞，直到T1释放行锁。
有2种方法可以实现冲突检测：
\1. 判断表是否已被其它事务用表锁锁住。
\2. 判断表中的每一行是否已被行锁锁住。
其中2需要遍历整个表，**效率太低**。因此innodb使用意向锁来解决这个问题：
T1需要先申请表的**意向共享锁（IS）**，成功后再申请某一行的**记录锁S**。
在意向锁存在的情况下，上面的判断可以改为：
T2发现表上有意向共享锁IS，因此申请表的写锁被阻塞。


**1.3 行锁**

InnoDB引擘支持行级别锁，行锁粒度小，并发度高，但加锁开销大，也可能会出现死锁。
加锁机制：innodb行锁锁住的是索引页，回表时，主键的聚簇索引也会加上锁。

![img](https://pic2.zhimg.com/80/v2-d736695de3e65e17a52737669895e56d_720w.webp)

行锁具体类别如上图所示，包括：**Record lock/Gap Locks/Next-Key Locks**，每类又可分为**共享锁（S）**或者**排它锁（X）**，一共2*3=6类，最后还有1类插入意向锁：
**Record lock（记录锁）：**最简单的行锁，仅仅锁住一行。记录锁永远都是加在索引上的，即使一个表没有索引，InnoDB也会隐式的创建一个索引，并使用这个索引实施记录锁。
**Gap Locks（间隙锁）：**加在两个索引值之间的锁，或者加在第一个索引值之前，或最后一个索引值之后的间隙。使用间隙锁锁住的是一个区间，而不仅仅是这个区间中的每一条数据。间隙锁只阻止其他事务插入到间隙中，不阻止其它事务在同一个间隙上获得间隙锁，所以 gap x lock 和 gap s lock 有相同的作用。它是一个**左开右开**区间：如（1，3）
**Next-Key Locks：记录锁和间隙锁的组合，它指的是加在某条记录以及这条记录前面间隙上的锁。它是一个左开右闭**区间：如（1，3】
**Insert Intention（插入意向锁）**：该锁只会出现在insert操作执行前（并不是所有insert操作都会出现），目的是为了提高并发插入能力。它在插入一行记录操作之前设置一种特殊的间隙锁，多个事务在相同的索引间隙插入时，如果不是插入间隙中相同的位置就不需要互相等待。

**TIPS:**

1.不存在unlock tables … read/write，只有unlock tables
2.If a session begins a transaction, an implicit UNLOCK TABLES is performed

## 2.锁的兼容情况

引入意向锁后，表锁之间的兼容性情况如下表：

![img](https://pic2.zhimg.com/80/v2-eb3fda6550b257f3f9f101ed349de9c9_720w.webp)

总结：

1. 意向锁之间都兼容
2. X,IX和其它都不兼容（除了1）
3. S,IS和其它都兼容（除了1,2）

## 3.锁信息查看方式

- MySQL 5.6.16版本之前，需要建立一张特殊表innodb_lock_monitor，然后使用**show engine innodb status**查看

CREATE TABLE innodb_lock_monitor (a INT) ENGINE=INNODB;

DROP TABLE innodb_lock_monitor;

- MySQL 5.6.16版本之后，修改系统参数innodb_status_output后，使用**show engine innodb status**查看

set GLOBAL innodb_status_output=ON;

set GLOBAL innodb_status_output_locks=ON;

每15秒输出一次INNODB运行状态信息到错误日志

![img](https://pic4.zhimg.com/80/v2-64281e84309ecc4678faf76a9b6acc0b_720w.webp)

- MySQL5.7版本之后

可以通过information_schema.innodb_locks查看事务的锁情况，但只能看到阻塞事务的锁；如果事务并未被阻塞，则在该表中看不到该事务的锁情况

- MySQL8.0

删除information_schema.innodb_locks，添加performance_schema.data_locks，即使事务并未被阻塞，依然可以看到事务所持有的锁，同时通过performance_schema.table_handles、performance_schema.metadata_locks可以非常方便的看到元数据锁等表锁。

## 4.测试环境搭建

### 4.1 建立测试表

该表包含一个主键，一个唯一键和一个非唯一键：

CREATE TABLE `t` (

`id` int(11) NOT NULL,

`a` int(11) DEFAULT NULL,

`b` int(11) DEFAULT NULL,

`c` varchar(10),

PRIMARY KEY (`id`),

unique KEY `a` (`a`),

key b(b))

ENGINE=InnoDB;

### 4.2 写入测试数据

insert into t values(1,10,100,'a'),(3,30,300,'c'),(5,50,500,'e');

## 5.记录存在时的加锁

对于innodb引擘来说，加锁的2个决定因素：

1）当前的**事务隔离级别**
2）当前**记录是否存在**

假设id为3的记录存在，则在不同的4个隔离级别下3个语句的加锁情况汇总如下表(select 3表示 select * from t where id=3)：

| 隔离级别 | select 3 | begin;select 3                | begin;select 3 for update      |
| -------- | -------- | ----------------------------- | ------------------------------ |
| RU       | 无       | SHARED_READ                   | SHARED_WRITEIXX,REC_NOT_GAP：3 |
| RC       | 无       | SHARED_READ                   | SHARED_WRITEIXX,REC_NOT_GAP：3 |
| RR       | 无       | SHARED_READ                   | SHARED_WRITEIXX,REC_NOT_GAP：3 |
| Serial   | 无       | SHARED_READISS,REC_NOT_GAP：1 | SHARED_WRITEIXX,REC_NOT_GAP：3 |

分析：

1. 使用以下语句在4种隔离级别之间切换：
   set global transaction_isolation='READ-UNCOMMITTED';
   set global transaction_isolation='READ-COMMITTED';
   set global transaction_isolation='REPEATABLE-READ';
   set global transaction_isolation='Serializable';
2. 对于auto commit=true，select 没有显式开启事务（begin）的语句，元数据锁和行锁都不加，是真的“**读不加锁**”
3. 对于begin; select ... where id=3这种只读事务，会加**元数据锁SHARED_READ**，防止事务执行期间表结构变化，查询**performance_schema.metadata_locks**表可见此锁：

![img](https://pic4.zhimg.com/80/v2-09617f06b0066268a365d3d2b528e6b3_720w.webp)

4.对于begin; select ... where id=3这种只读事务，MySQL在RC和RR隔离级别下，使用MVCC快照读，不加行锁，但在Serial隔离级别下，读写互斥，会加**意向共享锁（表锁）**和**共享记录锁（行锁）**

5.对于begin; select ... where id=3 for update，会加**元数据锁SHARED_WRITE**

6.对于begin; select ... where id=3 or update，4种隔离级别都会加**意向排它锁（表锁）**和**排它记录锁（行锁）,**查询**performance_schema.data_locks**可见此2类锁

![img](https://pic4.zhimg.com/80/v2-f05caee858dd450aed01c78de8385f43_720w.webp)

6 记录不存在时的加锁

| 隔离级别 | select 2 | begin;select 2        | begin;select 2 for update |
| -------- | -------- | --------------------- | ------------------------- |
| RU       | 无       | SHARED_READ           | SHARED_WRITEIX            |
| RC       | 无       | SHARED_READ           | SHARED_WRITEIX            |
| RR       | 无       | SHARED_READ           | SHARED_WRITEIXX,GAP：3    |
| Serial   | 无       | SHARED_READISS,GAP：3 | SHARED_WRITEIXX,GAP：3    |

分析：

1. 当记录不存在的时候，RU和RC隔离级别只有意向锁，没有行锁了
2. RR，Serial隔离级别下，记录锁变成了**Gap Locks（间隙锁），**可以防止**幻读，**lock_data为3的GAP lock锁住区间（1，3），此时ID=2的记录插入会被阻塞。

![img](https://pic2.zhimg.com/80/v2-6bb1c557a252a61d90f2cb52722d8259_720w.webp)

![img](https://pic2.zhimg.com/80/v2-b35275724285a207814701d26d886f3d_720w.webp)

那么对于主键范围查询，唯一键查询，非唯一键查询，在不同隔离级别下又是如何加锁的呢？
《从一道数据库面试题彻谈MySQL加锁机制（下）》再做分享。

原文作者：鹅厂架构师

原文链接：https://zhuanlan.zhihu.com/p/566194828