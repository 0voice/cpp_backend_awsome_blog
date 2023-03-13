# 【NO.209】mysql锁机制详解

## 1.前言

　　之前项目中用到事务，需要保证数据的强一致性，期间也用到了mysql的锁，但当时对mysql的锁机制只是管中窥豹，所以本文打算总结一下mysql的锁机制。

　　本文主要论述关于mysql锁机制，mysql版本为5.7，引擎为innodb，由于实际中关于innodb锁相关的知识及加锁方式很多，所以没有那么多精力罗列所有场景下的加锁过程并加以分析，仅根据现在了解的知识，结合官方文档，说说自己的理解，如果发现有不对的地方，欢迎指正。

## 2.概述

　　总的来说，InnoDB共有七种类型的锁：

- 共享/排它锁(Shared and Exclusive Locks)
- 意向锁(Intention Locks)
- 记录锁(Record Locks)
- 间隙锁(Gap Locks)
- 临键锁(Next-key Locks)
- 插入意向锁(Insert Intention Locks)
- 自增锁(Auto-inc Locks)

## 3.mysql锁详解

### 3.1. 共享/排它锁(Shared and Exclusive Locks)

- 共享锁（Share Locks，记为S锁），读取数据时加S锁
- 排他锁（eXclusive Locks，记为X锁），修改数据时加X锁

　　使用的语义为：

- 共享锁之间不互斥，简记为：读读可以并行
- 排他锁与任何锁互斥，简记为：写读，写写不可以并行

　　可以看到，一旦写数据的任务没有完成，数据是不能被其他任务读取的，这对并发度有较大的影响。对应到数据库，可以理解为，写事务没有提交，读相关数据的select也会被阻塞，这里的select是指加了锁的，普通的select仍然可以读到数据(快照读)。

### 3.2. 意向锁(Intention Locks)

　　InnoDB为了支持多粒度锁机制(multiple granularity locking)，即允许行级锁与表级锁共存，而引入了意向锁(intention locks)。意向锁是指，未来的某个时刻，事务可能要加共享/排它锁了，先提前声明一个意向。

1. 意向锁是一个表级别的锁(table-level locking)；
2. 意向锁又分为：
3. 意向共享锁(intention shared lock, IS)，它预示着，事务有意向对表中的某些行加共享S锁；
4. 意向排它锁(intention exclusive lock, IX)，它预示着，事务有意向对表中的某些行加排它X锁；

　　加锁的语法为：

```
select ... lock in share mode;　　要设置IS锁；select ... for update;　　　　　　 要设置IX锁；
```

　　事务要获得某些行的S/X锁，必须先获得表对应的IS/IX锁，意向锁仅仅表明意向，意向锁之间相互兼容，兼容互斥表如下：

|      |  IS   |  IX   |
| :--: | :---: | :---: |
|  IS  | 兼 容 | 兼 容 |
|  IX  | 兼 容 | 兼 容 |

　　虽然意向锁之间互相兼容，但是它与共享锁/排它锁互斥，其兼容互斥表如下:

|      |   S   |   X   |
| :--: | :---: | :---: |
|  IS  | 兼 容 | 互 斥 |
|  IX  | 互 斥 | 互 斥 |

　　排它锁是很强的锁，不与其他类型的锁兼容。这其实很好理解，修改和删除某一行的时候，必须获得强锁，禁止这一行上的其他并发，以保障数据的一致性。

### 3.3. 记录锁(Record Locks)

　　记录锁，它封锁索引记录，例如(其中id为pk)：

```
create table lock_example(id smallint(10),name varchar(20),primary key id)engine=innodb;
```

　　数据库隔离级别为RR，表中有如下数据：

```
10, zhangsan
20, lisi
30, wangwu
select * from t where id=1 for update;
```

　　其实这里是先获取该表的意向排他锁(IX)，再获取这行记录的排他锁(我的理解是因为这里直接命中索引了)，以阻止其他事务插入，更新，删除id=1的这一行。

### 3.4. 间隙锁(Gap Locks)

　　间隙锁，它封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。依然是上面的例子，InnoDB，RR：

```
select * from lock_example
    where id between 8 and 15 
    for update;
```

　　这个SQL语句会封锁区间(8,15)，以阻止其他事务插入id位于该区间的记录。

　　间隙锁的主要目的，就是为了防止其他事务在间隔中插入数据，以导致“不可重复读”。**如果把事务的隔离级别降级为读提交(Read Committed, RC)，间隙锁则会自动失效。**

### 3.5. 临键锁(Next-key Locks)

　　临键锁，是记录锁与间隙锁的组合，它的封锁范围，既包含索引记录，又包含索引区间。

　　默认情况下，innodb使用next-key locks来锁定记录。但当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围。

　　举个例子，依然是如上的表lock_example，但是id降级为普通索引(key)，也就是说即使这里声明了要加锁(for update)，而且命中的是索引，但是因为索引在这里没有UK约束，所以innodb会使用next-key locks，数据库隔离级别RR：

```
事务A执行如下语句，未提交：
select * from lock_example where id = 20 for update;
事务B开始，执行如下语句，会阻塞：
insert into lock_example values('zhang',15);
insert into lock_example values('zhang',15);
```

　　如上的例子，事务A执行查询语句之后，默认给id=20这条记录加上了next-key lock，所以事务B插入10(包括)到30(不包括)之间的记录都会阻塞。临键锁的主要目的，也是为了避免幻读(Phantom Read)。如果把事务的隔离级别**降级为RC，临键锁则也会失效**。

### 3.6. 插入意向锁(Insert Intention Locks)

　　对已有数据行的修改与删除，必须加强互斥锁(X锁)，那么对于数据的插入，是否还需要加这么强的锁，来实施互斥呢？插入意向锁，孕育而生。

　　插入意向锁，是间隙锁(Gap Locks)的一种（所以，也是实施在索引上的），它是专门针对insert操作的。多个事务，在同一个索引，同一个范围区间插入记录时，如果插入的位置不冲突，不会阻塞彼此。

> Insert Intention Lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap.

　　举个例子(表依然是如上的例子lock_example，数据依然是如上)，事务A先执行，在10与20两条记录中插入了一行，还未提交：

```
insert into t values(11, xxx);
```

　　事务B后执行，也在10与20两条记录中插入了一行：

```
insert into t values(12, ooo);
```

　　因为是插入操作，虽然是插入同一个区间，但是插入的记录并不冲突，所以使用的是插入意向锁，此处A事务并不会阻塞B事务。

### 3.7. 自增锁(Auto-inc Locks)

　　自增锁是一种特殊的表级别锁（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。

> AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.

　　举个例子(表依然是如上的例子lock_example)，但是id为AUTO_INCREMENT，数据库表中数据为：

```
1, zhangsan
2, lisi
3, wangwu
```

　　事务A先执行，还未提交： insert into t(name) values(xxx);

　　事务B后执行： insert into t(name) values(ooo);

　　此时事务B插入操作会阻塞，直到事务A提交。

## 4.总结

　　以上总结的7种锁，个人理解可以按两种方式来区分：

**1. 按锁的互斥程度来划分，可以分为共享、排他锁；**

- 共享锁(S锁、IS锁)，可以提高读读并发；
- 为了保证数据强一致，InnoDB使用强互斥锁(X锁、IX锁)，保证同一行记录修改与删除的串行性；

**2. 按锁的粒度来划分，可以分为：**

- 表锁：意向锁(IS锁、IX锁)、自增锁；
- 行锁：记录锁、间隙锁、临键锁、插入意向锁；

　　其中

1. InnoDB的细粒度锁(即行锁)，是实现在索引记录上的(我的理解是如果未命中索引则会失效)；　　
2. 记录锁锁定索引记录；间隙锁锁定间隔，防止间隔中被其他事务插入；临键锁锁定索引记录+间隔，防止幻读；
3. InnoDB使用插入意向锁，可以提高插入并发；
4. 间隙锁(gap lock)与临键锁(next-key lock)**只在RR以上的级别生效，RC下会失效**；



原文链接：https://zhuanlan.zhihu.com/p/356422542

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)