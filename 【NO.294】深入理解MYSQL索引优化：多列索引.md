# 【NO.294】深入理解MYSQL索引优化：多列索引

## 1.索引是什么

是存储引擎用于找到数据的一种数据结构。

## 2.索引的性能

在数据量小的时候，一个坏的索引往往作用没有那么明显，但是在数据量比较大的时候一个坏的索引和好的索引有巨大的区别。

> 在查询优化的时候应该首先考虑索引优化。这个是最简单的，也是效果最好。

## 3.索引的执行流程

索引 => 索引值 => 数据行

```
mysql> explain select first_name from actor where actor_id = 5;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | actor | NULL       | const | PRIMARY       | PRIMARY | 2       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

通过explian可以看到where条件后面是使用了主键索引。

## 4.多列索引

如果是多列组成的索引，那么在使用索引的时候要考虑索引的顺序。
多列索引有如下原则

- 最左匹配原则，如果缺失了索引的最左侧列，那么索引不生效
- 中间不可中断原则，如果中间定义的索引没有在where条件里面体现，那么后面的字段过滤将不会用到索引
- 范围中断原则，只要使用了范围查询，那么其右侧的索引都无法正常使用

## 5.多列索引的验证

### 5.1.建表语句

```
create table person
(
    A INT(10) not null,
    B INT(10) not null,
    C INT(10) not null,
    version VARCHAR(20) not null
);
create index A
    on person (A, B, C);
```

### 5.2.填充数据的存储过程

```
delimiter //
//
CREATE DEFINER=`dev`@`%` PROCEDURE `insert_person`(IN item int)
BEGIN
DECLARE counter INT;
SET counter = item;
WHILE counter >= 1000 DO
INSERT INTO person VALUES(left(counter, 2),left(counter,3), counter,counter );
SET counter = counter - 1;
END WHILE;
END;
//
delimiter ;
call insert_person(100000);
```

### 5.3.数据详情

```
mysql> select count(*) from person;
+----------+
| count(*) |
+----------+
|   100000 |
+----------+
1 row in set (0.04 sec)
```

## 6.场景分析

### 6.1.索引生效

#### 6.1.1. 符合多列索引生效的三个原则，全部用上了索引

(a) 只对A做等值查询

```
mysql> explain select * from person  where A = 12;
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | person | NULL       | ref  | A             | A    | 4       | const | 1100 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

> 可以看到是用到了A的索引的。

(b)只对A做范围查询,并且结果集覆盖索引的时候。

```
mysql> explain select A,B,C from person  where A > 13;
+----+-------------+--------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
|  1 | SIMPLE      | person | NULL       | range | A             | A    | 4       | NULL | 49555 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

（c）对A,B做等值查询

```
mysql> explain select A,B,C from person  where A = 13 and b = 129;
+----+-------------+--------+------------+------+---------------+------+---------+-------------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref         | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ref  | A             | A    | 8       | const,const |    1 |   100.00 | Using index |
+----+-------------+--------+------------+------+---------------+------+---------+-------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

(d) 对A做等值，B做范围

```
mysql> explain select A,B,C from person  where A = 13 and b > 129;
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | person | NULL       | range | A             | A    | 8       | NULL | 1100 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
```

(e) ABC等值查询

```
mysql> explain select A,B,C from person  where A = 12 and b = 129 and c = 1294;
+----+-------------+--------+------------+------+---------------+------+---------+-------------------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref               | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+-------------------+------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ref  | A             | A    | 12      | const,const,const |    1 |   100.00 | Using index |
+----+-------------+--------+------------+------+---------------+------+---------+-------------------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

(f) 对AB做等值，C范围查询

```
mysql> explain select A,B,C from person  where A = 12 and b = 129 and c > 1294;
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | person | NULL       | range | A             | A    | 12      | NULL |  105 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

## 7.索引部分生效

满足最左原则，但是不满足中间不中断或者范围中断。
(a) 对A做等值，C做等值或者是范围

```
mysql> explain select * from person where  A = 12 and c = 1230;
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | person | NULL       | ref  | A             | A    | 4       | const | 1100 |    10.00 | Using index condition |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from person where  A = 12 and c >1230;
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | person | NULL       | ref  | A             | A    | 4       | const | 1100 |    33.33 | Using index condition |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

(b)对A做等值，B做范围，C做等值或者范围，B做了范围查询之后，无法对C正常使用索引

```
mysql> explain select * from person where  A = 12 and b > 123 and c >1230;
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | person | NULL       | range | A             | A    | 8       | NULL |  660 |    33.33 | Using index condition |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from person where  A = 12 and b = 123 and c >1230;
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | person | NULL       | range | A             | A    | 12      | NULL |  109 |   100.00 | Using index condition |
+----+-------------+--------+------------+-------+---------------+------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

(c) 对A做范围查询，BC做等值或者范围，那么BC将不会使用索引

```
mysql> explain select A,B,C  from person where  A > 12 and b > 123 and c >1230;
+----+-------------+--------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
|  1 | SIMPLE      | person | NULL       | range | A             | A    | 4       | NULL | 49555 |    11.11 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+------+---------+------+-------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

> 需要注意的是这个地方ABC只是普通的索引如果是select * from,那么将不会使用索引。因为还有一个字段不再索引持有的数据上。

## 8.多列索引失效：

（a）查询条件中不包含最左列

```
mysql> explain select * from person where  b = 123 and c >1230;
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |     3.33 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from person where  b = 123
    -> ;
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from person where  b >123;
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |    33.33 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from person where  c =123;
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |    10.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select * from person where  c >123;
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |    33.33 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```

(b) 含有最左列，但是索引的列带条件

```
mysql> explain select * from person where  a + 1 =12;
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |   100.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
## 从哪里带条件哪里就不开始使用索引
mysql> explain select * from person where  a =12 and b + 1 = 123;
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | person | NULL       | ref  | A             | A    | 4       | const | 1100 |   100.00 | Using index condition |
+----+-------------+--------+------------+------+---------------+------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

## 9.order by使用索引

(a) order by 字段的顺序和table中多列索引定义的顺序一样。

```
mysql> explain select * from person1 order by a,b, c;
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+----------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra          |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+----------------+
|  1 | SIMPLE      | person1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 98999 |   100.00 | Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select a,b,c from person1 order by a,b, c;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person1 | NULL       | index | NULL          | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## 10.order by 不使用索引

(a) order by多个字段，每个字段都是独立的索引

```
mysql> create table person2 select * from person1;
Query OK, 99001 rows affected (0.68 sec)
Records: 99001  Duplicates: 0  Warnings: 0
mysql> alter table add key(a);
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'add key(a)' at line 1
mysql> alter table add index_a key(a);
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'add index_a key(a)' at line 1
mysql> alter table add index  key(a);
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'add index  key(a)' at line 1
mysql> alter table add index index_a key(a);
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'add index index_a key(a)' at line 1
mysql> alter table add index index_a(a);
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'add index index_a(a)' at line 1
mysql> alter table person2 add index index_a(a);
Query OK, 0 rows affected (0.15 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> alter table person2 add index index_b(b);
Query OK, 0 rows affected (0.15 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> explain select * from person2 order by a,b;
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+----------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra          |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+----------------+
|  1 | SIMPLE      | person2 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 99110 |   100.00 | Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+----------------+
1 row in set, 1 warning (0.00 sec)
```

(b) order by语句中同时含有desc和asc

```
mysql> explain select a,b,c from person1 order by a desc ;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person1 | NULL       | index | NULL          | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
mysql> explain select a,b,c from person1 order by a desc ,b asc;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra                       |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
|  1 | SIMPLE      | person1 | NULL       | index | NULL          | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index; Using filesort |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```

(c) order by 里面有计算字段

```
mysql> explain select a,b,c from person1 order by a desc ,b  + 1
    -> ;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra                       |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
|  1 | SIMPLE      | person1 | NULL       | index | NULL          | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index; Using filesort |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```

(d) orderby 里面顺序和索引的顺序不同

```
mysql> explain select a,b,c from person1 order by b,a;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra                       |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
|  1 | SIMPLE      | person1 | NULL       | index | NULL          | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index; Using filesort |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```

(e) group by 和orderby 的顺序不同

```
mysql> explain select a,b,c from person1 group by a,b,c order by a,b;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
|  1 | SIMPLE      | person1 | NULL       | index | index_A_B_C   | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
mysql> explain select a,b,c from person1 group by a,b,c order by b,a;
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+----------------------------------------------+
| id | select_type | table   | partitions | type  | possible_keys | key         | key_len | ref  | rows  | filtered | Extra                                        |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+----------------------------------------------+
|  1 | SIMPLE      | person1 | NULL       | index | index_A_B_C   | index_A_B_C | 21      | NULL | 98999 |   100.00 | Using index; Using temporary; Using filesort |
+----+-------------+---------+------------+-------+---------------+-------------+---------+------+-------+----------+----------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

> 上面的查询顺序一样，没有filesort，下面的不一样，就用了

(f) join查询的时候order只能按照主键的来排序，否则就会失效

```
mysql> explain select *  from person3 t1 left join person4 t2 on t1.version = t2.version where t1.version = 1200  order by t1.version;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | index  | PRIMARY       | PRIMARY | 22      | NULL           |  101 |    10.00 | Using where |
|  1 | SIMPLE      | t2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 22      | dev.t1.version |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------------+
2 rows in set, 3 warnings (0.00 sec)
mysql> explain select *  from person3 t1 left join person4 t2 on t1.version = t2.version where t1.version = 1200  order by t2.version;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+----------------------------------------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra                                        |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL           |  101 |    10.00 | Using where; Using temporary; Using filesort |
|  1 | SIMPLE      | t2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 22      | dev.t1.version |    1 |   100.00 | NULL                                         |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+----------------------------------------------+
2 rows in set, 3 warnings (0.00 sec)
mysql> show create table person3;
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                                                                |
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| person3 | CREATE TABLE `person3` (
  `A` varchar(10) DEFAULT NULL,
  `B` int(50) NOT NULL,
  `C` int(50) NOT NULL,
  `version` varchar(20) NOT NULL,
  PRIMARY KEY (`version`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
mysql> show create table person4;
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                                                                |
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| person4 | CREATE TABLE `person4` (
  `A` varchar(10) DEFAULT NULL,
  `B` int(50) NOT NULL,
  `C` int(50) NOT NULL,
  `version` varchar(20) NOT NULL,
  PRIMARY KEY (`version`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
mysql> explain select *  from person3 t1 left join person4 t2 on t1.version = t2.version order by t1.version;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------+
|  1 | SIMPLE      | t1    | NULL       | index  | NULL          | PRIMARY | 22      | NULL           |  101 |   100.00 | NULL  |
|  1 | SIMPLE      | t2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 22      | dev.t1.version |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
mysql> explain select *  from person3 t1 left join person4 t2 on t1.version = t2.version order by t2.version;
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+---------------------------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra                           |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+---------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL    | NULL          | NULL    | NULL    | NULL           |  101 |   100.00 | Using temporary; Using filesort |
|  1 | SIMPLE      | t2    | NULL       | eq_ref | PRIMARY       | PRIMARY | 22      | dev.t1.version |    1 |   100.00 | NULL                            |
+----+-------------+-------+------------+--------+---------------+---------+---------+----------------+------+----------+---------------------------------+
2 rows in set, 1 warning (0.00 sec)
## 不加where也是一样的。
```

优化此种情况

```
mysql> explain select * from person3 t1 left join person4 t2 on t1.version = t2.version join(select version from person3 order by a desc) a_order on t1.version = a_order.version;
+----+-------------+---------+------------+--------+---------------+---------+---------+----------------+------+----------+-------+
| id | select_type | table   | partitions | type   | possible_keys | key     | key_len | ref            | rows | filtered | Extra |
+----+-------------+---------+------------+--------+---------------+---------+---------+----------------+------+----------+-------+
|  1 | SIMPLE      | t1      | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL           |  101 |   100.00 | NULL  |
|  1 | SIMPLE      | person3 | NULL       | eq_ref | PRIMARY       | PRIMARY | 22      | dev.t1.version |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | t2      | NULL       | eq_ref | PRIMARY       | PRIMARY | 22      | dev.t1.version |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+--------+---------------+---------+---------+----------------+------+----------+-------+
3 rows in set, 1 warning (0.00 sec)
```

原文链接：https://zhuanlan.zhihu.com/p/384216746

作者：[Hu先生的Linux](https://www.zhihu.com/people/huhu520-10)