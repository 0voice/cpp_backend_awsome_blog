# 【NO.548】MYSQL---服务器配置相关问题

## 1.相关面试问题

1. 请分析一个Group By 语句的异常原因
2. 如何比较系统运行配置和配置文件中配置是否一致？
3. 举几个mysql 中的关键性能参数

## 2. 请分析一个Group By 语句的异常原因


可能这个问题会被认为是sql语句的原因， 刚开始我也是这么认为的。不过不急，可以思考思考在看看下面解释

假设有一个这样的表

![在这里插入图片描述](https://img-blog.csdnimg.cn/9acbac0309bc4cce8c4710cd7290165e.png)

sql 语句 ：
select prodcut_id, warehouse_id, sum(count) as cnt from stock group by product_id;

结果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/f558e972ef4145b48f8a3aeb56287ce1.png)

在mysql 中的能执行成功，在其他的数据库可能出现语法错误

为什么能在mysql 中执行成功，是因为在mysql中的一个配置起了作用, 它就是 SQL MODE。

### 2.1 SQL MODE 的作用及设置

SQL_MODE值 ： 会影响mysql 执行sql语句的结果。

配置mysql 处理SQL 的方式
set [session/global/persist] sql_mode = ‘xxxxxx’ (persist是在mysql 8.0 中的)
[mysqld] sql_mode = xxxxxxx

### 2.2 常用的SQL Mode

SQL_MODE= ‘ANSI’ 

![在这里插入图片描述](https://img-blog.csdnimg.cn/8c609b6b2345484e9219ca9465ea7f16.png)SQL_MODE = ‘TRADITIONAL’

![在这里插入图片描述](https://img-blog.csdnimg.cn/05d87f00bb0d48fdbabf3f605283acee.png)

### 2.3 演示 only_full_group_by

1.刚开始查看sql_mode

![在这里插入图片描述](https://img-blog.csdnimg.cn/d4f1db7f25f941a5b2afa145c810561d.png)

2.执行sql语句， 可以的看到是执行成功的

![在这里插入图片描述](https://img-blog.csdnimg.cn/3c777b3e4ff24d09a887cbb3144d8951.png)

3.修改sql_mode

![在这里插入图片描述](https://img-blog.csdnimg.cn/0136ee82d3b24bccac1728aa8d4211b0.png)

4.再次执行上面的sql语句，会报错

![在这里插入图片描述](https://img-blog.csdnimg.cn/2c7abcb14df54fa5a8146616945c822d.png)

5.此时必须在group by 后面写完整

![在这里插入图片描述](https://img-blog.csdnimg.cn/95675139bc554415871e59d68a44ac33.png)

### 2.4 演示 ansi_quotes

使用之后只能用单引号引字符串

![在这里插入图片描述](https://img-blog.csdnimg.cn/6ed2b6d18446476081b8010f2199399a.png)

### 2.5 演示 strict_trans_table

用普通模式，字符串插入int 类型 会成功
用严格模式，则会进行检查，字符串不能插入int 类型成功



## 3.比较系统运行配置和配置文件中配置

### 3.1 知识点

### 3.2 使用set 命令 配置动态参数

使用pt-config-diff 工具比较配置文件 （检查在运行中的配置和系统配置）
使用set 命令 配置动态参数
set [session | @@session.] system_var_name = expr
set [global | @@global .] system_var_name = expr
set [persist | @@persist .] system_var_name = expr (mysql 8.0 中增加)

### 3.3 检查在运行中的配置和系统配置（mysql 5.x）

pt-config-diff u=root, p=, h = localhost /etc/my.cnf

![在这里插入图片描述](https://img-blog.csdnimg.cn/24e7155427734dc5a0cc8d1636b0d341.png)

## 4.举几个mysql 中的关键性能参数

### 4.1 常用的性能参数

![在这里插入图片描述](https://img-blog.csdnimg.cn/6b2bb2968aea45ce8edb6a9638db3a89.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/0178572f3afb49ce8fdd1ab55b51f1f2.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/214d16326f104c66b000a92c69a25ab9.png)版权声明：本文为CSDN博主「_刘小雨」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_39486027/article/details/125210884