# 【NO.247】MySQL WriteSet并行复制分析

> 以下内容首发于腾讯数据库技术（公众号）

## 0.背景

在mysql支持基于LOGICAL CLOCK的复制后，主从延迟得到了很大的改善，但是LOGICAL CLOCK一定程度上会受到master的并发度的影响。当master的并发度较低，每次组提交的事务数较少的时候，binlog在slave上的回放的并发度也会因此而降低，即使这些事务之间并没有任何冲突。示例：

```text
Trx1 -----L----C---------------------------------->
Trx2 ---------------L----C------------------------>
Trx3 ------------------------L----C--------------->
Trx4 --------------------------------L----C------->
```

假定这些事务之间并没有任何冲突，由于它们的LOGICAL CLOCK周期没有重叠，在slave上也只能串行回放。为了解决这个问题，mysql8.0.1引入了基于WriteSet的复制。

## 1. 原理

LOGICAL CLOCK由两部分组成，分别是commit parent和sequence number。commit parent表示当前事务所依赖的事务的序号，只有依赖的事务完成后当前事务才能进行。WriteSet是一种更细粒度的事务冲突检测手段，它是在LOGICAL CLOCK的基础上，对事务的commit parent进行处理。例如，有如下两个事务：

```text
Trx1 -----L----C------------->
Trx2 ---------------L----C--->
```

假定这两个事务没有任何冲突，但是他们的LOGICAL CLOCK区间没有重叠，原本不能进行并行回放，经过WriteSet处理后，这两个事务的commit parent可能会被修改，让LOGICAL CLOCK区间有可能重叠，使并行回放成为可能。

WriteSet冲突检测的原理：

- 全局有一个数据结构(实际上使用std::map)维护了一定数量的行的hash值与修改行的事务的sequence number之间的映射。
- 一个事务会记录所修改行的hash值，在事务提交写入binlog的时候，遍历该事务修改的行的hash值，在全局的map中进行查找，如果有相同的hash值表明有两个事务修改了同一行，记录有冲突的sequence number，取最大的sequence number作为该事务的commit parent（如果没有任何冲突，则commit parent设置为map中最小的sequence number）。

更详细的示例：

![img](https://pic4.zhimg.com/80/v2-a5c0725a008ddc3d4aaeaa11de6fff63_720w.webp)

图中每一个方块代表一个事务，方块对应的区域代表事务影响的范围，如果有重叠则表示事务有冲突，每一个step代表一次组提交，T1-T8代表事务的执行顺序。如果不使用WriteSet，在slave上回放的时候的顺序：

```text
<T1,T2>, <T3>, <T4,T5>, <T6>, <T7,T8>
```

使用WriteSet后，在slave上的回放效果：

![img](https://pic4.zhimg.com/80/v2-1a70899f13293cef694e3755f859e543_720w.webp)

回放顺序：

```text
<T1,T2,T3>, <T4,T5,T6,T7>, <T8>
```

这里有一个地方需要注意，就是对于T2和T3，他们对应于同一个连接，然而在回放的时候却是并行的，可能导致事务的提交顺序不一致。解决这个问题有两种方法：

- slave_preserve_commit_order设置为on
- 使用writeset_session模式

writeset_session模式下同一个session的事务不能并发执行。它的原理很简单，在writeset的基础上，将事务的commit parent与当前session的last sequence number进行比较，取较大值作为新的commit parent。

## 2.源码分析

### **2.1 事务writeset更新**

MySQL 5.7.6引入了事务的写集合，在计算事务依赖的时候可以直接使用。在开启了gtid，且binlog_format为row格式，且transaction_write_set_extraction不为OFF的实例上，写binlog的时候会将事务所修改的行的hash值添加到事务的写集合中，对应的函数是binlog_log_row和add_pke(pke是primary key equivalent的缩写)。函数add_pke所做的事情：

- 对所修改的表的主键和唯一键的key各计算一个hash值（hash字符串由更新行的index，db，table，value按照一定规则拼接，这里不深究），放入到事务的写集合中（利用主键和unique key的唯一性来检测冲突和依赖）。
- 记录表的外键信息，如果表是某一些表的父表，调用Rpl_transaction_write_set_ctx::set_has_related_foreign_keys进行标记；如果表是某一些表的子表，外键列不为NULL，且foreign_key_checks不为0，也会对这样的外键列计算一个hash值。
- 如果没有添加任何hash值到写集合中，调用Rpl_transaction_write_set_ctx::set_has_missing_keys进行标记，说明记录因为某些原因没有计算hash值（比如有的表没有主键）。

需要注意的是，如果表没有人为定义主键（不包括innodb内部自动生成的主键）也没有定义非空唯一键，则不会计算任何hash值，即使表有其它索引。

### **2.2 事务依赖计算**

在函数MYSQL_BIN_LOG::write_transaction入口处会调用Transaction_dependency_tracker::get_dependency来获取事务的依赖（获取sequence number和commit parent）。函数Transaction_dependency_tracker::get_dependency会根据变量binlog_transaction_dependency_tracking来决定使用哪种方式计算事务的依赖：

```text
void Transaction_dependency_tracker::get_dependency(THD *thd,
                                                    int64 &sequence_number,
                                                    int64 &commit_parent) {
  sequence_number = commit_parent = 0;
  switch (m_opt_tracking_mode) {
    // 根据提交时的timestamp来决定依赖，COMMIT_ORDER是5.7引入的，这里不再深究
    case DEPENDENCY_TRACKING_COMMIT_ORDER:
      m_commit_order.get_dependency(thd, sequence_number, commit_parent);
      break;
    // 根据事务的写集合来决定依赖
    case DEPENDENCY_TRACKING_WRITESET:
      m_commit_order.get_dependency(thd, sequence_number, commit_parent);
      // writeset是在COMMIT_ORDER的基础上进行优化
      m_writeset.get_dependency(thd, sequence_number, commit_parent);
      break;
    // writeset_session，同一个session的事务不能并发执行
    case DEPENDENCY_TRACKING_WRITESET_SESSION:
      m_commit_order.get_dependency(thd, sequence_number, commit_parent);
      m_writeset.get_dependency(thd, sequence_number, commit_parent);
      // 在writeset的基础上对同一个session的事务做限制
      m_writeset_session.get_dependency(thd, sequence_number, commit_parent);
      break;
    default:
      assert(0);  // blow up on debug
      /*
        Fallback to commit order on production builds.
       */
      m_commit_order.get_dependency(thd, sequence_number, commit_parent);
  }
}
```

函数Writeset_trx_dependency_tracker::get_dependency根据写集合对事务的commit parent进行优化，关键代码：

```text
void Writeset_trx_dependency_tracker::get_dependency(THD *thd,
                                                     int64 &sequence_number,
                                                     int64 &commit_parent) {
  Rpl_transaction_write_set_ctx *write_set_ctx =
      thd->get_transaction()->get_transaction_write_set_ctx();
  std::vector<uint64> *writeset = write_set_ctx->get_write_set();
  // 检查是否能使用writeset
  bool can_use_writesets =
      // empty writeset implies DDL or similar, except if there are missing keys
      (writeset->size() != 0 || write_set_ctx->get_has_missing_keys() ||
       /*
         The empty transactions do not need to clear the writeset history, since
         they can be executed in parallel.
       */
       is_empty_transaction_in_binlog_cache(thd)) &&
      // hashing algorithm for the session must be the same as used by other
      // rows in history
      (global_system_variables.transaction_write_set_extraction ==
       thd->variables.transaction_write_set_extraction) &&
      // must not use foreign keys
      !write_set_ctx->get_has_related_foreign_keys() &&
      // it did not broke past the capacity already
      !write_set_ctx->was_write_set_limit_reached();
  bool exceeds_capacity = false;
  if (can_use_writesets) {
    /*
     Check if adding this transaction exceeds the capacity of the writeset
     history. If that happens, m_writeset_history will be cleared only after
     using its information for current transaction.
    */
    exceeds_capacity =
        m_writeset_history.size() + writeset->size() > m_opt_max_history_size;
    // 起始的parent值，history为空时为0，不为空时为当前history中最小的sequence
    // number
    int64 last_parent = m_writeset_history_start;
    // 遍历一个事务所有修改的行的hash值
    for (std::vector<uint64>::iterator it = writeset->begin();
         it != writeset->end(); ++it) {
      // 对每一个hash值都去history中寻找是否有对应的hash
      Writeset_history::iterator hst = m_writeset_history.find(*it);
      if (hst != m_writeset_history.end()) {
        // 如果一个行的hash存在于history中且对应的事务先于当前事务
        if (hst->second > last_parent && hst->second < sequence_number)
          // 修改当前事务所依赖的事务的sequence number
          last_parent = hst->second;
        // 标记该行由当前事务修改
        hst->second = sequence_number;
      } else {
        // 将hash值和事务的sequence number插入到history中
        if (!exceeds_capacity)
          m_writeset_history.insert(
              std::pair<uint64, int64>(*it, sequence_number));
      }
    }
    // 同时没有主键和非空唯一键的表不能使用writeset
    if (!write_set_ctx->get_has_missing_keys()) {
      // 当前事务所操作的table都有主键的前提下
      // 取last parent和commit parent中的较小值
      // 作为当前事务的commit parent (last_committed)
      commit_parent = std::min(last_parent, commit_parent);
    }
    if (exceeds_capacity || !can_use_writesets) {
      // 超过history最大值或者当前事务不能使用writeset则清空当前history
      m_writeset_history_start = sequence_number;
      m_writeset_history.clear();
    }
  }
}
```

使用writeset有一定的限制：

- DDL不可以使用writeset
- 当前session 的 hash 算法和 writeset history 必须相同
- 事务所更新的列不能被其它表引用
- 事务的写集合存放的hash值数量不能超过binlog_transaction_dependency_history_size设定的最大值
- 没有主键或者非空唯一键的表，事务的依赖获取会退化到COMMIT_ORDER的方式

函数Writeset_session_trx_dependency_tracker::get_dependency对同一个session的事务的commit parent做出限制：

```text
void Writeset_session_trx_dependency_tracker::get_dependency(
    THD *thd, int64 &sequence_number, int64 &commit_parent) {
  // 获取当前session提交的上一个事务的sequence number
  int64 session_parent = thd->rpl_thd_ctx.dependency_tracker_ctx()
                             .get_last_session_sequence_number();
  // 在commit parent和session parent中取较大值作为事务的commit parent
  if (session_parent != 0 && session_parent < sequence_number)
    commit_parent = std::max(commit_parent, session_parent);
  // 更新当前session的last sequence number
  thd->rpl_thd_ctx.dependency_tracker_ctx().set_last_session_sequence_number(
      sequence_number);
}
```

## 3.简易测试

测试数据由sysbench生成：10张表，每张表10万行。使用sysbench进行300s的read-write测试，然后让slave回放压测产生的binlog，slave回放并行度为8，记录回放时间。

从机每秒回放事务数对比图：

![img](https://pic4.zhimg.com/80/v2-6ce8174375a4a6221e0825c6ee7566c3_720w.webp)

可以看到，基于writeset的复制比基于commit_order的复制会快很多；当master并发度较低的时候，基于writeset_session的复制也会较慢，但比commit_order快；当并发度较高的时候writeset_session和writeset的复制速度比较接近，总体上writeset_session的速度要低于writeset。

## 4.总结

在LOGICAL CLOCK的基础上，根据事务的写集合将事务的依赖进一步细化，让事务在从机上的回放的并发度进一步提高。WriteSet主要适用于master上并发度不高的情况，如果主并发度较高或者主从没有延迟则不需要使用，因为WriteSet会带来额外的内存与CPU的消耗，在一些小的实例上可能会造成资源紧张。

## 5.参考

**MySQL :: WL#9556: Writeset-based MTS dependency tracking on master：***[https://dev.mysql.com/worklog/task/?id=9556](https://link.zhihu.com/?target=https%3A//dev.mysql.com/worklog/task/%3Fid%3D9556)*

**Improving the Parallel Applier with Writeset-based Dependency Tracking | MySQL High Availability：***[https://mysqlhighavailability.com/improving-the-parallel-applier-with-writeset-based-dependency-tracking](https://link.zhihu.com/?target=https%3A//mysqlhighavailability.com/improving-the-parallel-applier-with-writeset-based-dependency-tracking)*



**欢迎点赞分享，关注**[@鹅厂架构师](https://www.zhihu.com/org/e-han-jia-gou-shi)**，一起探索更多业界领先产品技术。**

原文作者：鹅厂架构师

原文链接：https://zhuanlan.zhihu.com/p/410720981