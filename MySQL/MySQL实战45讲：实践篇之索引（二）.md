@[TOC](MySQL实战45讲：实践篇之索引（二）)

# 基本概念
# 普通索引和唯一索引怎么选择
# MySQL为什么有时会选错索引
##  实验环境
**创建表**

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `b` (`b`)
) ENGINE=InnoDB;
```

**插入数据**

```sql
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000)do
    insert into t values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```

**评估语句**
使用`EXPLAIN`命令可以查看SQL语句执行的具体情况。输出中，key表示选择的索引，rows表示预计扫描的行数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032710323393.png#pic_center)
可以看到，对于语句 `select * from where between 10000 and 20000`，MySQL选择了索引a，扫描行数预计为10001。假设我们在已经插入了十万行数据的表上执行如下操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327104738161.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)
可以看到，session B 把表 t 中的数据都删除后，又调用了 idata 这个存储过程，插入了 10 万行数据。此时，session B 的查询语句 `select * from t where a between 10000 and 20000` 就不会再选择索引 a 了。我们可以通过慢查询日志（slow log）来查看具体的执行情况。

```sql
#将慢查询日志的阈值设置为 0，表示这个线程接下来的语句都会被记录入慢查询日志中
set long_query_time=0;

select * from t where a between 10000 and 20000; /*Q1*/
select * from t force index(a) where a between 10000 and 20000;/*Q2*/
```

其中，`force index(a)` 表示强制使用索引 a。慢日志的输出如下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327105025665.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

可以看到，Q1 扫描了 10 万行（**Rows_examined**），显然是走了全表扫描，执行时间是 40 毫秒。Q2 扫描了 10001 行，执行了 21 毫秒。也就是说，我们在没有使用 force index 的时候，MySQL 用错了索引，导致了更长的执行时间。

## 优化器的逻辑
我们提到过，选择索引是优化器的工作。而优化器选择索引的目的，是找到一个最优的执行方案，并用最小的代价去执行语句。在数据库里面，**扫描行数**是影响执行代价的因素之一。扫描的行数越少，意味着访问磁盘数据的次数越少，消耗的 CPU 资源越少。同时，优化器还会结合是否使用临时表、是否排序等因素进行综合判断。

**统计信息**
但是，MySQL 在真正开始执行语句之前，并不能精确地知道满足这个条件的记录有多少条，而只能根据**统计信息**来估算记录数。这个统计信息就是索引的“区分度”。显然，一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，我们称之为**基数**（cardinality）。也就是说，这个基数越大，索引的区分度越好。

我们可以使用 `SHOW  INDEX FROM tablename` 方法，看到一个表中索引的基数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327103258835.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


MySQL通过采样统计来获得一个索引的基数。采样统计的时候，InnoDB 默认会选择 `N` 个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。由于数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过 `1/M` 的时候，会自动触发重新做一次索引统计。

在 MySQL 中，有两种存储索引统计的方式，可以通过设置参数 `innodb_stats_persistent` 的值来选择：

 - 设置为 on 的时候，表示统计信息会持久化存储。这时，默认的 N 是 **20**，M 是 **10**；
 - 设置为 off 的时候，表示统计信息只存储在内存中。这时，默认的 N 是 **8**，M 是 **16**。


我们使用 EXPLAIN 命令查看一下优化器对 Q1 和 Q2 语句预计的扫描行数。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327105526328.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70)
可以发现，与慢日志的输出相比，Q1 的结果还是符合预期的，rows 的值是 104620，接近十万行；但是 Q2 的 rows 值是 37116，跟实际扫描行数 10001 的偏差很大。

首先，为什么MySQL选择了全表扫描而不是使用索引 a？这是因为，如果使用索引 a，每次从索引 a 上拿到一个值，都要回到主键索引上查出整行数据，这个代价优化器也要算进去的。而如果选择扫描 10 万行，是直接在主键索引上扫描的，没有额外的代价。优化器会估算这两个选择的代价，从结果看来，优化器认为直接扫描主键索引更快。当然，从执行时间看来，这个选择并不是最优的。

但是，在执行 session A 和 session B 的操作之前，由于预计的扫描行数是准确的，所以选择了索引 a。在执行了 session B 中的删数据并重新插入数据的操作后，表中的统计信息不准确了，导致预计扫描行数出现了较大偏差。

**重新统计索引信息**
使用 `ANALYZE TABLE tablename` 命令可以重新统计表中的索引信息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327110748684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

可以看到，重新统计索引信息后，MySQL选择了正确的索引。


## 索引选择异常和处理
**索引选择错误的另一种情形**

```sql
mysql> select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 1;
```

对于上面这条语句，如果使用索引 a 进行查询，那么就是扫描索引 a 的前 1000 个值，然后取到对应的 id，再到主键索引上去查出每一行，然后根据字段 b 来过滤。显然这样需要扫描 1000 行；反之，如果使用索引 b 进行查询，那么需要扫描 50001 行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210327114611142.png#pic_center)
可以看到，优化器又选错了索引。优化器选择使用索引 b，是因为它认为使用索引 b 可以避免排序（b 本身是索引，已经是有序的了，如果选择索引 b 的话，不需要再做排序，只需要遍历），所以即使扫描行数多，也判定为代价更小。

**索引选择错误处理**
第一种方法是，采用 `FORCE INDEX` 命令强行选择一个索引。

第二种方法就是，可以考虑修改语句，引导 MySQL 使用我们期望的索引。比如，在上面的例子里，把 `order by b limit 1` 改成 `order by b,a limit 1`。

第三种方法使用了子查询。

```sql
mysql> select * from  (select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 100)alias limit 1;
```
这里我们用 limit 100 让优化器意识到，使用 b 索引代价是很高的。其实是我们根据数据特征诱导了一下优化器，也不具备通用性。

# 怎么给字符串字段加索引
见下一篇。



References：
[1\] https://time.geekbang.org/column/article/70848
[2\] https://time.geekbang.org/column/article/71173
[3\] https://time.geekbang.org/column/article/71492
[4\] https://www.cnblogs.com/zjfjava/p/6922494.html
[5\] https://dev.mysql.com/doc/refman/8.0/en/create-table.html
[6\] https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html
[7\] https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html
[8\] https://dev.mysql.com/doc/refman/8.0/en/innodb-system-tablespace.html
[9\] https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html
[10\] https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html
