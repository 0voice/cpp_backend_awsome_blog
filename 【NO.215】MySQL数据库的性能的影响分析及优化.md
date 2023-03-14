# 【NO.215】MySQL数据库的性能的影响分析及优化

**MySQL数据库的性能的影响**

**一. 服务器的硬件的限制**

**二. 服务器所使用的操作系统**

**三. 服务器的所配置的参数设置不同**

四. 数据库存储引擎的选择

**五. 数据库的参数配置的不同**

六. (重点)数据库的结构的设计和SQL语句

## 1. 服务器的配置和设置(cpu和可用的内存的大小)

```
1.网络和I/O资源 
 2.cpu的主频和核心的数量的选择
 (对于密集型的应用应该优先考虑主频高的cpu)
 (对于并发量大的应用优先考虑的多核的cpu)
 3.磁盘的配置和选择
 (使用传统的机械硬盘:
    特点：读写较慢、存储空间大、最常见、使用最多、价格低;
    工作过程：移动磁头到磁盘表面上的正确位置;
             等待磁盘的旋转，使得所得所需的数据在磁头之下;
             等待磁盘旋转过去，所有所需的数据都被磁头读出
    选择因素：存储容量、传输速度、访问时间、主轴转速、物理尺寸)
 (使用RAID增强传统的机器硬盘的性能：
    特点：利用小的磁盘组成大的磁盘并提供数据的冗余保证数据的完整性的技术
    数据库中所使用的RAID的级别：
        RAID0级别、RAID1级别、RAID5级别[分布式奇偶校验磁盘阵列]、RAID10[分片的镜像(数据库最好的方式)]
    RAID级别选择：如下图)
 (使用固态存储的SSD和PCI-E卡:
    特点：相对于机械盘固态磁盘有更好的随机读写性能;
         相对于机械固态磁盘能更好的支持并发;
         相对于机械固态磁盘更容易损坏
    SSD:使用SATA接口可以替换传统的磁盘而不需要任何的改变[受到接口的速度的限制];
        SATA接口的SSD同样支持RAID技术
    PCI-E卡(Fusion-IO卡)：无法使用在SATA接口[需要使用独特的驱动和配置];
                         价格贵,使用了cpu的资源和内存
    使用的场景：适用于存在大量的随机I/O的场景;
              适用于解决单线程负载的I/O瓶颈)
 (使用网络存储NAS和SAN：
    SAN[光纤接入服务器]:大量顺序读写操作、读写I/O、缓存、I/O合并、随机读写慢（不如本地的RAID）
    NAS设备使用网络连接，基于文件的协议如NFS或者SMB来访问
    适合场景：数据库的备份、)
```

![img](https://pic1.zhimg.com/80/v2-f47b12fb0984a8948a5dce8b331f00fc_720w.webp)

![img](https://pic4.zhimg.com/80/v2-3e8d2315b134c440ea629b55ba10430b_720w.webp)

![img](https://pic2.zhimg.com/80/v2-37201284feb41d5712aa3af90920afb1_720w.webp)

![img](https://pic2.zhimg.com/80/v2-d3a0e6fbb6ae23bb4cdb088bb8943631_720w.webp)

## 2.不同REAID级别的对比：

![img](https://pic4.zhimg.com/80/v2-98cbee262c189a88ad72cb0c58b299cf_720w.webp)

注意事项：

```
1.64位数据库的版本使用32位的服务器的版本
 2.内存的主频的选择主板所能支持的最大内存的频率
```

## 3.总结：

```
对于cpu：
        1.64位的cpu一定能够要工作在64位的系统下
        2.对于并发比较高的场景cpu的数量比频率重要
        3.对于cpu密集型的场景和复杂SQL则频率越高越好
    对于内存：
        1.选择主板所能使用的最高频率的内存
        2.内存的大小对性能很重要，所以尽可能的大
    I/O子系统：
        1.PCIe -> SSD -> RAID10 -> 磁盘 -> SAN
```

## 4. 操作系统对性能的影响

```
Windows、FreeBSD、Solaris、Linux
centos的参数优化的设置：
    (1)内核相关的参数（/etc/sysctl.conf）
        net.core.somaxconn = 65535
        net.core.netdev_max_backlog = 65535
        net.ipv4.tcp_max_syn_backlog = 65535
        net.ipv4.tcp_fin_timeout = 10
        net.ipv4.tcp_tw_reuse = 1
        net.ipv4.tcp_tw_recycle = 1
        net.core.wmem_defaullt = 87380
        net.core.wmem_max = 16777216
        net.core.rmem_defaullt = 87380
        net.core.rmem_max = 16777216
        net.ipv4.tcp_keepalive_time = 120
        net.ipv4.tcp_keepalive_intvl = 30
        net.ipv4.tcp_keepalive_probes = 3
        kernel.shmmax = 4294967295
        vm.swappiness = 0
    (2)增加资源限制（/etc/security/limit.conf）
        * soft nofile 65535
        * hard nofile 65535
            * 表示对所有的用户有效
            soft 指的是当前系统的生效的设置
            hard 表明系统中所能设定的最大值
            nofile 表示所限制的资源是打开文件的最大数目
            65535 就是限制的数量
    (3).磁盘调度策略(/sys/block/devname/queue/scheduler)
        noop(电梯式调度策略）、deadline（截止时间调度策略）、anticipatory(预料I/O调度策略)
        cat /sys/block/sda/queue/scheduler
        noop anticipatory deadline [cfq]
            echo deadline > /sys/block/sda/queue/scheduler
```

## 5.MySQl的数据库的体系

![img](https://pic2.zhimg.com/80/v2-c8de4414bb8e3f61f9611dcf90dab3a9_720w.webp)

## 6.MySQl的数据库的存储引擎

```
(1).Mysql之存储引擎MyISAM
    组成的结构：表为MYD和MYI、frm的文件组成
    特性：并发性和锁级别
         MyISAM表支持索引类型
         MyISAM表支持数据的压缩(命令行：myisampack)
             myisampack -b -f myIsam.MYI;
            压缩后的表不能进行写操作，只能进行读操作
    修复:对数据库中的表进行检查并修复:
        check table mytable;
        repair table mytable;
        myisamchk工具,修复时数据库服务必须停止
    限制：使用MySQL5.0之前时默认表的大小4G(存储大表修改MAX_Rows和AVG_ROW_LENGTH)
         使用MySQL5.0之后的版本默认支持256TB
    适用的场景:非事务型的应用
              只读类的应用
              空间类的应用(GPS的数据)
(2).Mysql之存储引擎InnoDB
    mysql5.5.8之后版本默认使用的存储引擎
    组成结构：通过设置innodb_file_per_table参数存储的位置不同
                ON:独立表空间：tablename.ibd
                OFF:系统表空间:ibdataX
    建议：对于mysql中建议使用InnoDB的独立表空间
    特性：事务性存储引擎
         完全支持事务的存储引擎
         Redo log(存储已经提交的事务)和Undo log(存储未提交的事务)
         InnoDB支持行级别锁
         最大程序的支持并发
         行级别的锁是由存储引擎层实现的
    锁：共享锁(读锁)、独占锁(写锁)
         表级锁、行级锁
        阻塞：确保事务并发的正常的执行
        死锁：两个或者两个以上的事务执行过程中相互等待对方的资源而产生的一种异常
    InnoDB状态检查：
        show engine innodb status;    
    适用场景：InooDB适用于大多数OLTP应用
(3).Mysql之存储引擎CSV
    特点：数据以文本的方式存储在文件中
        .CSV文件存储表的内容
        .CSM文件存储表的元数据如表的状态和数据量
        .frm文件存储表的结构的信息
        以CSV格式进行数据的存储
        所有的列必须不能为NULL的
        不支持索引(不适合大表，不适合在线处理)
        可以对数据文件直接进行编辑
    适用的场景：适合作为数据交换的中间表
              mysql数据目录->csv文件->其他web程序
              excel电子表格 -> csv文件 -> mysql数据目录
(4).Mysql之存储引擎Archive
    特点：以zlib对表数据进行压缩，磁盘I/O更少
         数据存储在ARZ为后缀的文件中
         只支持insert和select操作
         只支持在自增的ID列上加索引
    适用场景：
         日志和数据采集类的应用
(4).Mysql之存储引擎Memory
    特点：数据只保存在内存中
         Memory存储引擎的I/O效率特别高
         支持HASH索引和BTree索引
         所有的字段为固定长度
         不支持BLOG和TEXT等大字段
         Memory存储引擎使用表级锁
         表中存储数据的最大值由max_heap_table_size参数决定
    适用场景：用于查找或者映射表，例如邮编和地区
             用于保存数据分析产生的中间表
             用于缓存周期性聚合数据的结果表
```

## 7.MySQl的数据库的服务器参数

```
(1).Mysql配置参数作用域
    全局参数 
        set global 参数名=参数值；
        set @@global.参数名：=参数值；
    会话参数
        set[session] 参数名=参数值；
        set @@session.参数名：=参数值；
(2).内存配置相关的参数
        确定可以使用的内存的上限
        确定MySQL的每个连接使用的内存
            sort_buffer_size join_buffer_size
            read_buffer_size read_rnd_buffer_size
        确定需要为操作系统保留多少内存
        如何为缓存池分配内存
            Innodb_buffer_pool_size
            总内存-（每个线程锁需要的内存*连接数）- 系统的保留内存
            key_buffer_size
(3).I/O相关配置参数
        InnoDb存储引擎的I/O参数设置：
        Innodb_log_file_size
        Innodb_log_file_in_group
        Innodb_log_buffer_size
        Innodb_flush_log_at_trx_commit
        Innodb_flush_method = O_DIRECT
        Innodb_file_per_table = 1
        Innodb_doublewrite = 1
      MySIAM存储引擎的I/O参数设置：
        delay_key_write
            OFF:每次操作后刷新键缓冲中的脏块到磁盘
            ON:只对在键表时指定了delay_key_write选项的表使用延迟刷新
            ALL:对所有MYSIAM表都使用延迟键写入
(4).安全相关配置参数
        expire_logs_days 指定自动清理binlog的天数
        max_allowed_packet 控制MySQL可以接受的包的大小（32M）
        skip_name_resolve 禁用DNS查找
        sysdate_is_now 确保sysdate()返回确定性的日期
        read_only 禁止非super权限的用户写权限
        skip_slave_start 禁止Slave自动恢复
        sql_mode 设置MySQL所使用的SQL模式
            strict_trans_tables
            no_engine_subtitutoion
            no_zero_date
            no_zero_in_date
            only_full_group_by
(5).其他相关配置参数
        sync_binlog = 1控制MySQL如何向磁盘刷新binlog
        tmp_table_size和max_heap_table_size 控制内存临时表的大小(设置一致)
        max_connections = 2000 控制允许的最大连接数
```

## 8.MySQl的数据库的结构设计和SQL的优化

```
(1).过分的反范式化为表的建立太多的列
(2).过分的范式化造成太多的表关联
(3).在OLTP环境中使用不恰当的分区表
(4).使用外键保证数据的完整性
```

## 9.性能优化的顺序

- 数据库结构设计和SQL语句优化
- 数据库的存储引擎的选择和参数的配置
- 系统的选择及其优化
- 硬件升级

原文链接：https://zhuanlan.zhihu.com/p/371524818

作者：Hu先生的Linux