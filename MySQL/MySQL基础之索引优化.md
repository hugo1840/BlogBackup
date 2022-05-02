@[TOC](MySQL基础之索引优化)

# 覆盖索引
创建一个表如下

```sql
mysql> CREATE TABLE T (
ID int PRIMARY KEY,
k int NOT NULL DEFAULT 0, 
s varchar(16) NOT NULL DEFAULT '',
index k(k))
engine=InnoDB;

mysql> INSERT INTO T VALUES(100,1,'aa'),(200,2,'bb'),
(300,3,'cc'),(500,5,'ee'),(600,6,'ff'),(700,7,'gg');
```

如果执行的语句是 `SELECT ID FROM T WHERE k BETWEEN 3 AND 5`，这时只需要查 ID 的值，而 ID 的值已经在 k 索引树上了，因此可以直接提供查询结果，不需要回表。索引 k 已经覆盖了查询需求，这种情况被称为**索引覆盖**。将需要查询的字段放到索引中去，比如建立**联合索引**，也可以避免回表，提高查询效率。

```sql
mysql> CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `ismale` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `id_card` (`id_card`),
  KEY `name_age` (`name`,`age`)  #联合索引
) ENGINE=InnoDB
```

> 关于Key/Primary Key/Index的区别，请参考：https://www.cnblogs.com/zjfjava/p/6922494.html

# 最左前缀原则
B+ 树这种索引结构，可以利用索引的“最左前缀”来定位记录。索引项是按照索引定义里面出现的字段顺序排序的。以联合索引 (name, age) 为例，其排序如图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210316085634247.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

如果你要查询所有名字第一个字是“张”的人，你的 SQL 语句的条件是 `WHERE name LIKE '张%'`。这时，你也能够用上这个联合索引 (name, age)，查找到第一个符合条件的记录是 ID3，然后向后遍历，直到不满足条件为止。

也就是说，不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索。这个最左前缀可以是联合索引的最左 N 个字段，也可以是字符串索引的最左 M 个字符。所以，当已经有了 (a,b) 这个联合索引后，一般就不需要单独在 a 上建立索引了。

因此，在建立联合索引时需要考虑的一个重要问题就是：如何安排索引内的字段顺序。第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。

如果如果既需要联合查询，又需要基于 a、b 各自的查询呢？此时，要考虑的原则就是空间了。比如 name 字段是比 age 字段大的 ，那就应该创建一个 (name, age) 的联合索引和一个 (age) 的单字段索引。
 
# 索引下推
MySQL 5.6 引入的**索引下推优化**（index condition pushdown)， 可以在索引遍历过程中，==对索引中包含的字段先做判断==，直接过滤掉不满足条件的记录，减少回表次数。以下面的语句为例

```sql
mysql> SELECT * FROM tuser WHERE name LIKE '张%' AND age=10 AND ismale=1;
```
无索引下推时，需要回表4次。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210316092338745.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


有索引下推时，仅需回表2次。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210316092355338.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)



References:
[1\] 联合索引，https://www.cnblogs.com/gudi/p/4058411.html
[2\] Key vs Primary key vs Index，https://www.cnblogs.com/zjfjava/p/6922494.html
[3\] MySQL Course, https://time.geekbang.org/column/intro/139

