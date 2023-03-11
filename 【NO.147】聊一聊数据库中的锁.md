# 【NO.147】聊一聊数据库中的锁

## 0.背景

数据库中有一张叫`后宫佳丽`的表,每天都有几百万新的小姐姐插到表中,光阴荏苒,夜以继日,日久生情,时间长了,表中就有了几十亿的`小姐姐`数据,看到几十亿的小姐姐,每到晚上,我可愁死了,这么多小姐姐,我翻张牌呢?
办法当然是精兵简政,删除那些`age>18`的,给年轻的小姐姐们留位置...
于是我在数据库中添加了一个定时执行的小程序,每到周日,就自动运行如下的脚本

```sql
Copydelete from `后宫佳丽` where age>18
```

一开始还自我感觉良好,后面我就发现不对了,每到周日,这个脚本一执行就是一整天,运行的时间有点长是小事,重点是这大好周日,我再想读这张表的数据,怎么也读不出来了,怎是一句空虚了得,我好难啊!

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/%E9%9A%BE.gif)

## 1.为什么

编不下去了,真实背景是公司中遇到的一张有海量数据表,每次一旦执行历史数据的清理,我们的程序就因为读不到这张表的数据,疯狂地报错,后面一查了解到,原来是因为定时删除的语句设计不合理,导致数据库中数据由行锁(`Row lock`)升级为表锁(`Table lock`)了😂.
解决这个问题的过程中把数据库锁相关的学习了一下,这里把学习成果,分享给大家,希望对大家有所帮助.
我将讨论SQL Server锁机制以及如何使用SQL Server标准动态管理视图监视SQL Server 中的锁,相信其他数据的锁也大同小异,具有一定参考意义.

## 2.铺垫知识

在我开始解释SQL Server锁定体系结构之前，让我们花点时间来描述ACID（原子性，一致性，隔离性和持久性）是什么。ACID是指数据库管理系统（DBMS）在写入或更新资料的过程中，为保证事务（transaction）是正确可靠的，所必须具备的四个特性：原子性（atomicity，或称不可分割性）、一致性（consistency）、隔离性（isolation，又称独立性）、持久性（durability）。

### 2.1 ACID

#### 2.1.1 原子性(Atomicity)

一个事务（transaction）中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。即，事务不可分割、不可约简。

#### 2.1.2一致性(Consistency)

在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设约束、触发器、级联回滚等。

#### 2.1.3 隔离性(Isolation)

数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括未提交读（Read uncommitted）、提交读（read committed）、可重复读（repeatable read）和串行化（Serializable）。

#### 2.1.4 持久性(Durability)

事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

来源:维基百科 https://zh.wikipedia.org/wiki/ACID

### 2.2 事务 (Transaction:)

事务是进程中最小的堆栈，不能分成更小的部分。此外，某些事务处理组可以按顺序执行，但正如我们在原子性原则中所解释的那样，即使其中一个事务失败，所有事务块也将失败。

### 2.3 锁定 (Lock)

锁定是一种确保数据一致性的机制。SQL Server在事务启动时锁定对象。事务完成后，SQL Server将释放锁定的对象。可以根据SQL Server进程类型和隔离级别更改此锁定模式。这些锁定模式是：

#### 2.3.1 锁定层次结构

SQL Server具有锁定层次结构，用于获取此层次结构中的锁定对象。数据库位于层次结构的顶部，行位于底部。下图说明了SQL Server的锁层次结构。

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/hierarchy.png)

#### 2.3.2 共享（S）锁 (Shared (S) Locks)

当需要读取对象时，会发生此锁定类型。这种锁定类型不会造成太大问题。

#### 2.3.3 独占（X）锁定 (Exclusive (X) Locks)

发生此锁定类型时，会发生以防止其他事务修改或访问锁定对象。

#### 2.3.4 更新（U）锁 (Update (U) Locks)

此锁类型与独占锁类似，但它有一些差异。我们可以将更新操作划分为不同的阶段：读取阶段和写入阶段。在读取阶段，SQL Server不希望其他事务有权访问此对象以进行更改,因此，SQL Server使用更新锁。

#### 2.3.5 意图锁定 (Intent Locks)

当SQL Server想要在锁定层次结构中较低的某些资源上获取共享（S）锁定或独占（X）锁定时，会发生意图锁定。实际上，当SQL Server获取页面或行上的锁时，表中需要设置意图锁。

### 2.4 SQL Server locking

了解了这些背景知识后，我们尝试再SQL Server找到这些锁。SQL Server提供了许多动态管理视图来访问指标。要识别SQL Server锁，我们可以使用sys.dm_tran_locks视图。在此视图中，我们可以找到有关当前活动锁管理的大量信息。

在第一个示例中，我们将创建一个不包含任何索引的演示表，并尝试更新此演示表。

```sql
CopyCREATE TABLE TestBlock
(Id INT ,
Nm VARCHAR(100))

INSERT INTO TestBlock
values(1,'CodingSight')
In this step, we will create an open transaction and analyze the locked resources.
BEGIN TRAN
UPDATE TestBlock SET   Nm='NewValue_CodingSight' where Id=1
select @@SPID
```

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/update-demo-table.png)

再获取到了SPID后，我们来看看`sys.dm_tran_lock`视图里有什么。

```sql
Copyselect * from sys.dm_tran_locks  WHERE request_session_id=74
```

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/sys.dm_tran_lock-view-1.png)

此视图返回有关活动锁资源的大量信息,但是是一些我们难以理解的一些数据。因此，我们必须将`sys.dm_tran_locks` join 一些其他表。

```sql
CopySELECT dm_tran_locks.request_session_id,
       dm_tran_locks.resource_database_id,
       DB_NAME(dm_tran_locks.resource_database_id) AS dbname,
       CASE
           WHEN resource_type = 'OBJECT'
               THEN OBJECT_NAME(dm_tran_locks.resource_associated_entity_id)
           ELSE OBJECT_NAME(partitions.OBJECT_ID)
       END AS ObjectName,
       partitions.index_id,
       indexes.name AS index_name,
       dm_tran_locks.resource_type,
       dm_tran_locks.resource_description,
       dm_tran_locks.resource_associated_entity_id,
       dm_tran_locks.request_mode,
       dm_tran_locks.request_status
FROM sys.dm_tran_locks
LEFT JOIN sys.partitions ON partitions.hobt_id = dm_tran_locks.resource_associated_entity_id
LEFT JOIN sys.indexes ON indexes.OBJECT_ID = partitions.OBJECT_ID AND indexes.index_id = partitions.index_id
WHERE resource_associated_entity_id > 0
  AND resource_database_id = DB_ID()
 and request_session_id=74
ORDER BY request_session_id, resource_associated_entity_id
```

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/join-sys.dm_tran_locks-view-to-other-views-1.png)

在上图中，您可以看到锁定的资源。SQL Server获取该行中的独占锁。（RID：用于锁定堆中单个行的行标识符）同时，SQL Server获取页中的独占锁和TestBlock表意向锁。这意味着在SQL Server释放锁之前，任何其他进程都无法读取此资源,这是SQL Server中的基本锁定机制。

现在，我们将在测试表上填充一些合成数据。

```sql
CopyTRUNCATE TABLE 	  TestBlock
DECLARE @K AS INT=0
WHILE @K <8000
BEGIN
INSERT TestBlock VALUES(@K, CAST(@K AS varchar(10)) + ' Value' )
SET @K=@K+1
 END
--After completing this step, we will run two queries and check the sys.dm_tran_locks view.
BEGIN TRAN
 UPDATE TestBlock  set Nm ='New_Value' where Id<5000
```

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/check-sys.dm_tran_locks-view-2.png)

在上面的查询中，SQL Server获取每一行的独占锁。现在，我们将运行另一个查询。

```sql
CopyBEGIN TRAN
 UPDATE TestBlock  set Nm ='New_Value' where Id<7000
```

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/run-another-query-1.png)

在上面的查询中，SQL Server在表上创建了独占锁，因为SQL Server尝试为这些将要更新的行获取大量RID锁,这种情况会导致数据库引擎中的大量资源消耗,因此，SQL Server会自动将此独占锁定移动到锁定层次结构中的上级对象(Table)。我们将此机制定义为Lock Escalation, 这就是我开篇所说的锁升级,它由行锁升级成了表锁。

根据官方文档的描述存在以下任一条件，则会触发锁定升级：

- 单个Transact-SQL语句在单个非分区表或索引上获取至少5,000个锁。
- 单个Transact-SQL语句在分区表的单个分区上获取至少5,000个锁，并且ALTER TABLE SET LOCK_ESCALATION选项设置为AUTO。
- 数据库引擎实例中的锁数超过了内存或配置阈值。

https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008-r2/ms184286(v=sql.105)

### 2.5 如何避免锁升级

防止锁升级的最简单，最安全的方法是保持事务的简短，并减少昂贵查询的锁占用空间，以便不超过锁升级阈值,有几种方法可以实现这一目标.

#### 2.5.1 将大批量操作分解为几个较小的操作

例如，在我开篇所说的在几十亿条数据中删除小姐姐的数据：

```sql
Copydelete from `后宫佳丽` where age>18
```

我们可以不要这么心急,一次只删除500个，可以显着减少每个事务累积的锁定数量并防止锁定升级。例如：

```sql
CopySET ROWCOUNT 500
delete_more:
     delete from `后宫佳丽` where age>18
IF @@ROWCOUNT > 0 GOTO delete_more
SET ROWCOUNT 0
```

#### 2.5.2 创建索引使查询尽可能高效来减少查询的锁定占用空间

如果没有索引会造成表扫描可能会增加锁定升级的可能性, 更可怕的是，它增加了死锁的可能性，并且通常会对并发性和性能产生负面影响。
根据查询条件创建合适的索引,最大化提升索引查找的效率,此优化的一个目标是使索引查找返回尽可能少的行，以最小化查询的的成本。

#### 2.5.3 如果其他SPID当前持有不兼容的表锁，则不会发生锁升级

锁定升级始总是升级成表锁，而不会升级到页面锁定。如果另一个SPID持有与升级的表锁冲突的IX（intent exclusive）锁定，则它会获取更细粒度的级别（行，key或页面）锁定，定期进行额外的升级尝试。表级别的IX（intent exclusive）锁定不会锁定任何行或页面，但它仍然与升级的S（共享）或X（独占）TAB锁定不兼容。
如下所示,如果有个操作始终在不到一小时内完成，您可以创建包含以下代码的sql，并安排在操作的前执行

```sql
CopyBEGIN TRAN
SELECT * FROM mytable (UPDLOCK, HOLDLOCK) WHERE 1=0
WAITFOR DELAY '1:00:00'
COMMIT TRAN
```

此查询在mytable上获取并保持IX锁定一小时，这可防止在此期间对表进行锁定升级。

## 3.Happy Ending

好了,不说了,小姐姐们因为不想离我开又打起来了(死锁).

![Alt text](https://raw.githubusercontent.com/liuzhenyulive/GitDisk/blogs/pic/TalkAboutLockInDatabase/ending.png)

参考文献:
SQL Server Transaction Locking and Row Versioning Guide https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-guides/jj856598(v=sql.110)
SQL Server, Locks Object https://docs.microsoft.com/en-us/sql/relational-databases/performance-monitor/sql-server-locks-object?view=sql-server-2017
How to resolve blocking problems that are caused by lock escalation in SQL Server https://support.microsoft.com/es-ve/help/323630/how-to-resolve-blocking-problems-that-are-caused-by-lock-escalation-in
Main concept of SQL Server locking https://codingsight.com/main-concept-of-sql-server-locking/

原文作者：码农阿宇

原文链接：https://www.cnblogs.com/CoderAyu/p/11375088.html