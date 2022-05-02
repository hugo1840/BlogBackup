@[TOC](为什么这些SQL语句逻辑相同，性能却差异巨大？)

# 案例一：条件字段函数操作
假设你现在维护了一个交易系统，其中交易记录表 tradelog 包含交易流水号（tradeid）、交易员 id（operator）、交易时间（t_modified）等字段。这个表的建表语句如下：

```sql
mysql> CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

假设，现在已经记录了从 2016 年初到 2018 年底的所有数据，运营部门有一个需求是，要统计发生在所有年份中 7 月份的交易记录总数。你的 SQL 语句可能会这么写：

```sql
mysql> select count(*) from tradelog where month(t_modified)=7;
```

由于 t_modified 字段上有索引，于是你就很放心地在生产库中执行了这条语句，但却发现执行了特别久，才返回了结果。实际上：**如果对字段做了函数计算，就用不上索引了**，这是 MySQL 的规定。

为什么条件是 `where t_modified='2018-7-1'` 的时候可以用上索引，而改成 `where month(t_modified)=7` 的时候就不行了？实际上，B+ 树提供的这个快速定位能力，来源于同一层兄弟节点的有序性。==对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能==。

但是，优化器并没有放弃使用索引。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/a00de6ff8bcb80c5391a84080abd36a5.png#pic_center)
`key="t_modified"` 表示的是使用了 t_modified 这个索引；我在测试表数据中插入了 10 万行数据，`rows=100335`，说明这条语句扫描了整个索引的所有值；Extra 字段的 `Using index`，表示的是使用了覆盖索引。

也就是说，由于在 t_modified 字段加了 `month()` 函数操作，导致了**全索引扫描**。为了能够用上索引的快速定位能力，我们就要把 SQL 语句改成基于字段本身的范围查询。

```sql
mysql> select count(*) from tradelog where
    -> (t_modified >= '2016-7-1' and t_modified<'2016-8-1') or
    -> (t_modified >= '2017-7-1' and t_modified<'2017-8-1') or 
    -> (t_modified >= '2018-7-1' and t_modified<'2018-8-1');
```

优化器在个问题上确实有“偷懒”行为，即使是对于不改变有序性的函数，也不会考虑使用索引。比如，对于 `select * from tradelog where id + 1 = 10000` 这个 SQL 语句，这个加 1 操作并不会改变有序性，但是 MySQL 优化器还是不能用 id 索引快速定位到 9999 这一行。所以，需要你在写 SQL 语句的时候，手动改写成 `where id = 10000 - 1` 才可以。


# 案例二：隐式类型转换

```sql
mysql> select * from tradelog where tradeid=110717;
```

交易编号 tradeid 这个字段上，本来就有索引，但是 explain 的结果却显示，这条语句需要走全表扫描。这是因为，tradeid 的字段类型是 varchar(32)，而输入的参数却是整型，所以需要做**类型转换**。

在 MySQL 中，字符串和数字做比较的话，是**将字符串转换成数字**。对于优化器来说，上面的查询语句相当于：

```sql
mysql> select * from tradelog where CAST(tradid AS signed int) = 110717;
```
也就是说，这条语句触发了我们上面说到的规则：对索引字段做函数操作，优化器会放弃走树搜索功能。

# 案例三：隐式字符编码转换
假设系统里还有另外一个表 trade_detail，用于记录交易的操作细节。为了便于量化分析和复现，我往交易日志表 tradelog 和交易详情表 trade_detail 这两个表里插入一些数据。

```sql
mysql> CREATE TABLE `trade_detail` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `trade_step` int(11) DEFAULT NULL, /*操作步骤*/
  `step_info` varchar(32) DEFAULT NULL, /*步骤信息*/
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into tradelog values(1, 'aaaaaaaa', 1000, now());
insert into tradelog values(2, 'aaaaaaab', 1000, now());
insert into tradelog values(3, 'aaaaaaac', 1000, now());

insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');
insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');
insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');
insert into trade_detail values(4, 'aaaaaaab', 1, 'add');
insert into trade_detail values(5, 'aaaaaaab', 2, 'update');
insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');
insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');
insert into trade_detail values(8, 'aaaaaaac', 1, 'add');
insert into trade_detail values(9, 'aaaaaaac', 2, 'update');
insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');
insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
```

如果要查询 id=2 的交易的所有操作步骤信息，SQL 语句可以这么写：

```sql
mysql> select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2; /*语句Q1*/
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210407160425592.PNG#pic_center)

EXPLAIN 的输出说明：

 - 第一行显示优化器会先在交易记录表 tradelog 上查到 id=2 的行，这个步骤用上了主键索引，rows=1 表示只扫描一行；
 - 第二行 key=NULL，表示没有用上交易详情表 trade_detail 上的 tradeid 索引，进行了全表扫描。

在这个执行计划里，是从 tradelog 表中取 tradeid 字段，再去 trade_detail 表里查询匹配字段。因此，我们把 tradelog 称为**驱动表**，把 trade_detail 称为**被驱动表**，把 tradeid 称为**关联字段**。

我们本来是希望通过使用 tradeid 索引能够快速定位到等值的行。但，这里并没有。因为这两个表的字符集不同，一个是 **utf8**，一个是 **utf8mb4**，所以做表连接查询的时候用不上关联字段的索引。

这里问题出现在根据 tradeid 值到 trade_detail 表中查找条件匹配的行时

```sql
mysql> select * from trade_detail where tradeid=$L2.tradeid.value; 
```

其中，`$L2.tradeid.value` 的字符集是 utf8mb4。

字符集 utf8mb4 是 utf8 的超集，所以当这两个类型的字符串在做比较的时候，MySQL 内部的操作是，==先把 utf8 字符串转成 utf8mb4 字符集，再做比较==。也就是说，实际上这个语句等同于下面这个写法：

```sql
select * from trade_detail  where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value; 
```

这就再次触发了我们上面说到的原则：对索引字段做函数操作，优化器会放弃走树搜索功能。

反过来，对于查询语句

```sql
mysql>select l.operator from tradelog l, trade_detail d where d.tradeid=l.tradeid and d.id=4; /*语句Q2*/
```

做表连接时，在 tradelog 上的操作类似

```sql
select operator from tradelog  where traideid =$R4.tradeid.value; 
```

等价于下面的语句

```sql
select operator from tradelog  where traideid =CONVERT($R4.tradeid.value USING utf8mb4); 
```

所以能够使用上 tradeid 索引。

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/a37b176d36a7605df37fa3c0438ae7b9.png#pic_center)

最后，如果要优化语句

```sql
select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2; /*语句Q1*/
```

的执行过程，有两种做法：

 - 比较常见的优化方法是，把 trade_detail 表上的 tradeid 字段的字符集也改成 utf8mb4。
 `alter table trade_detail modify tradeid varchar(32) CHARACTER SET utf8mb4 default null;`
 - 如果数据量比较大， 或者业务上暂时不能做这个 DDL 的话，那就只能采用修改 SQL 语句的方法了。
 `select d.* from tradelog l, trade_detail d where d.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2; 
`
这里主动把 `l.tradeid` 转成 utf8，就避免了被驱动表上的字符编码转换。
