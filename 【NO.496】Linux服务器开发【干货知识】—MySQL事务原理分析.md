# 【NO.496】Linux服务器开发【干货知识】—MySQL事务原理分析

前言

今天的目标是学习MySQL事务原理分析，但是却似乎总是非常不顺利，概念和实操实在多到令人发指，故干脆轻松学完一节课，等到时机到了再重新刷一遍吧！

### 1.事务是什么？

将数据库从一致性状态转化成另一种一致性状态。

单条语句是隐含得事务，多条语句需要手动开启事务。

### 2.ACID特性是什么？

原子性依靠undolog(共享表空间)实现，记录进行操作，然后进行反向操作。

一致性最难理解，换句话说在事务执行前后，数据库完整性约束不能被破坏。

### 3.隔离级别

mvcc提供了一种快照读的方式提升读的并发性能，通过读历史版本的方式。

个人理解：各个级别其实就像是一种胆大策略，没有对错之分。只要不担心错读数据，就可以选用读不加锁而写加排他锁，依次类推。串行化读取虽然保险，但是效率也是最低，也许是我天生性格的原因，似乎对这种方式情有独钟。人生路慢慢，欲速则不达也。

- 快照读就是不加任何参数。
- 如果是当前读，需要加上lock in share mode 或 for update 加读写锁。
- 查看锁信息select * from information_schema.innodb_locks;

myisam不会发生并发读异常、并发死锁，因为是表锁。所以说当出现问题，试着换一下引擎也许是一个很好的选择。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165913_23211.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165914_56724.jpg)

删除、更改需要增加排他锁，而增加插入需要将插入位置加入位置意向锁，然后间接的加入增加排他锁。三个事项加在一起是完整的写操作。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165914_88571.jpg)

表级别的对写锁才会和意向锁冲突，行级别咱就都不考虑啦。

### 4.Record Lock

记录单个行上的锁。

### 5.Gap Lock

间隙锁，锁定一个范围。只有在可重复读上面才会有间隙锁。

插入意向锁容易造成死锁，如果没有这个锁要一个一个的插入，有了它是为了提升效率。

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165915_27939.jpg)

死锁80%都是上图第二行造成的！

### 6.小计

更改mysql5.7的字符集

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165916_17067.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165917_86738.jpg)

![img](https://linuxcpp.0voice.com/zb_users/upload/2022/12/16/20221216165917_95247.jpg)

原文链接：https://zhuanlan.zhihu.com/p/507239547

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)