---
tags: [mysql, SQL优化]
title: MySQL执行计划中访问表数据的方法
created: '2022-09-28T12:22:18.170Z'
modified: '2022-12-07T08:40:23.686Z'
---

MySQL执行计划中访问表数据的方法

创建测试用的表：
```sql
mysql> create table discosongs ( 
  id INT NOT NULL AUTO_INCREMENT, 
  name varchar(100), 
  ISNcode INT, 
  singer varchar(50), 
  info_n1 varchar(100), 
  info_n2 varchar(100), 
  info_n3 varchar(100), 
  ratings varchar(50), 
  primary key (id), 
  key idx_name(name), 
  unique key uk_ISN(ISNcode), 
  key idx_singer(singer), 
  key idx_info(info_n1,info_n2,info_n3) 
  ) engine=InnoDB charset=UTF8;
```
其中，id列为主键，name列上存在二级索引`idx_name`，ISNcode列上存在唯一二级索引`uk_ISN`，singer列上存在二级索引`idx_singer`，info_n1、info_n2和info_n3三列上存在二级联合索引`idx_info`。

下面介绍MySQL在执行SQL时访问表数据的几种常见方法（对应explain执行计划中的**Type**）。

# system
当目标表中**只有一行记录**，并且该表使用的存储引擎的统计数据是精确的（比如MyISAM、MEMORY，不包括InnoDB），MySQL访问数据的方法就是**system**。

```sql
mysql> create table st(i INT) engine=MyISAM;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into st values(1);
Query OK, 1 row affected (0.00 sec)

mysql> explain select * from st;
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | st    | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

**注**：当目标表中只有一行记录，并且该表使用的存储引擎是InnoDB时，会通过全表扫描访问表数据。

# const
**const**是指通过**主键或唯一二级索引与常数的等值比较**来定位一条行记录。


```sql
mysql> insert into discosongs (name,ISNcode,singer,
    info_n1,info_n2,info_n3,ratings) values
    ('Whirling in Rags',23,'Sea Power',
    'Disco Elysium','Hary Du Bois','Revacho','Not Bad');
Query OK, 1 row affected (0.00 sec)

--通过主键与常数的等值比较条件来唯一定位一条行记录
mysql> explain select * from discosongs where id=1;
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | discosongs | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

--通过唯一二级索引与常数的等值比较条件来唯一定位一条行记录
mysql> explain select * from discosongs where ISNcode=23;
+----+-------------+------------+------------+-------+---------------+--------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key    | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+--------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | discosongs | NULL       | const | uk_ISN        | uk_ISN | 5       | const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+--------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

**注**：此处等值条件里的常数不能是NULL。

# eq_ref
在做表连接时，如果**被驱动表**是通过**主键或者唯一二级索引上的等值查询**进行访问的，那么对该被驱动表的访问方式就是**eq_ref**。

```sql
mysql> create table disco_replica select * from discosongs;
Query OK, 9 rows affected (0.01 sec)
Records: 9  Duplicates: 0  Warnings: 0

mysql> explain select * from discosongs t1 inner join disco_replica t2 on t1.id=t2.id;
+----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref           | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-------+
|  1 | SIMPLE      | t2    | NULL       | ALL    | NULL          | NULL    | NULL    | NULL          |    9 |   100.00 | NULL  |
|  1 | SIMPLE      | t1    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | appgame.t2.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+---------------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)
```

# ref
**ref**是指通过**普通二级索引与常数的等值比较**、或者**所有二级索引与NULL的比较**来定位一条或多条行记录。

由于普通二级索引和唯一二级索引都不会限制NULL值的数量，因此在`key is NULL`条件下都有可能匹配到多行记录。

```sql
mysql> select * from discosongs;
+----+------------------+---------+---------------+------------------+-------------------+---------+---------+
| id | name             | ISNcode | singer        | info_n1          | info_n2           | info_n3 | ratings |
+----+------------------+---------+---------------+------------------+-------------------+---------+---------+
|  1 | Whirling in Rags |      23 | Sea Power     | Disco Elysium    | Hary Du Bois      | Revacho | Not Bad |
|  2 | Slide            |     233 | Calvin Harris | Funk Wav Bounces | Frank Ocean       | UK      | Good    |
|  3 | Rollin           |     234 | Calvin Harris | Funk Wav Bounces | Future & Khalid   | UK      | Nice    |
|  4 | Feels            |     235 | Calvin Harris | Funk Wav Bounces | Pharrell Williams | UK      | Chiling |
|  5 | NNN              |    NULL | Unknown       | XXXXX            | XXX               | XXX     | Unknown |
|  6 | NXXX             |    NULL | Unknown       | XXXXX            | XXX               | XXX     | Unknown |
|  7 | XXXXX            |    NULL | Unknown       | XNNN             | ZZZ               | XXX     | Unknown |
|  8 | The Business     |     443 | Tiesto        | The Business     | Tiesto            | Europe  | Regular |
+----+------------------+---------+---------------+------------------+-------------------+---------+---------+
8 rows in set (0.00 sec)

--通过普通二级索引与常数的等值比较来定位多条行记录
mysql> explain select * from discosongs where singer='Calvin Harris';
+----+-------------+------------+------------+------+---------------+------------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | discosongs | NULL       | ref  | idx_singer    | idx_singer | 153     | const |    3 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

--通过唯一二级索引与NULL的比较来定位多条行记录
mysql> explain select * from discosongs where ISNcode is NULL;
+----+-------------+------------+------------+------+---------------+--------+---------+-------+------+----------+-----------------------+
| id | select_type | table      | partitions | type | possible_keys | key    | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+------------+------------+------+---------------+--------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | discosongs | NULL       | ref  | uk_ISN        | uk_ISN | 5       | const |    3 |   100.00 | Using index condition |
+----+-------------+------------+------------+------+---------------+--------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

```

# ref_or_null
**ref_or_null**是指当我们同时需要访问某个二级索引列的值等于某个常数的记录，并且还需要访问该二级索引列的值为NULL的所有行记录。

```sql
mysql> select * from discosongs;
+----+------------------+---------+---------------+------------------+-------------------+---------+---------+
| id | name             | ISNcode | singer        | info_n1          | info_n2           | info_n3 | ratings |
+----+------------------+---------+---------------+------------------+-------------------+---------+---------+
|  1 | Whirling in Rags |      23 | Sea Power     | Disco Elysium    | Hary Du Bois      | Revacho | Not Bad |
|  2 | Slide            |     233 | Calvin Harris | Funk Wav Bounces | Frank Ocean       | UK      | Good    |
|  3 | Rollin           |     234 | Calvin Harris | Funk Wav Bounces | Future & Khalid   | UK      | Nice    |
|  4 | Feels            |     235 | Calvin Harris | Funk Wav Bounces | Pharrell Williams | UK      | Chiling |
|  5 | NNN              |    NULL | Unknown       | XXXXX            | XXX               | XXX     | Unknown |
|  6 | NXXX             |    NULL | Unknown       | XXXXX            | XXX               | XXX     | Unknown |
|  7 | XXXXX            |    NULL | Unknown       | XNNN             | ZZZ               | XXX     | Unknown |
|  8 | The Business     |     443 | Tiesto        | The Business     | Tiesto            | Europe  | Regular |
|  9 | TheXX            |     669 | NULL          | TheXX            | CCC               | Europe  | Regular |
+----+------------------+---------+---------------+------------------+-------------------+---------+---------+
9 rows in set (0.00 sec)

mysql>
mysql> explain select * from discosongs where singer='Calvin Harris' or singer is NULL;
+----+-------------+------------+------------+-------------+---------------+------------+---------+-------+------+----------+-----------------------+
| id | select_type | table      | partitions | type        | possible_keys | key        | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+------------+------------+-------------+---------------+------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | discosongs | NULL       | ref_or_null | idx_singer    | idx_singer | 153     | const |    4 |   100.00 | Using index condition |
+----+-------------+------------+------------+-------------+---------------+------------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

```

# index_merge
索引合并（**index_merge**）可以分为Intersect索引合并、Union索引合并、以及Sort-Union索引合并三种情况。

考虑下面的表：
```sql
mysql> select * from discosongs;
+----+-------------------+---------+---------------+-------------------+-------------------+---------+---------+
| id | name              | ISNcode | singer        | info_n1           | info_n2           | info_n3 | ratings |
+----+-------------------+---------+---------------+-------------------+-------------------+---------+---------+
|  1 | Whirling in Rags  |      23 | Sea Power     | Disco Elysium     | Hary Du Bois      | Revacho | Not Bad |
|  2 | Slide             |     233 | Calvin Harris | Funk Wav Bounces  | Frank Ocean       | UK      | Good    |
|  3 | Rollin            |     234 | Calvin Harris | Funk Wav Bounces  | Future & Khalid   | UK      | Nice    |
|  4 | Feels             |     235 | Calvin Harris | Funk Wav Bounces  | Pharrell Williams | UK      | Chiling |
|  5 | NNN               |    NULL | Unknown       | XXXXX             | XXX               | XXX     | Unknown |
|  6 | NXXX              |    NULL | Unknown       | XXXXX             | XXX               | XXX     | Unknown |
|  7 | XXXXX             |    NULL | Unknown       | XNNN              | ZZZ               | XXX     | Unknown |
|  8 | The Business      |     443 | Tiesto        | The Business      | Tiesto            | Europe  | Regular |
|  9 | TheXX             |     669 | NULL          | TheXX             | CCC               | Europe  | Regular |
| 10 | Bohemian Rhapsody |     128 | Queen         | Bohemian Rhapsody | Freddie Mercury   | UK      | Rock    |
| 12 | Feels             |     404 | Calvin Harris | Funk Wav Bounces  | The Weeknd        | Canada  | Classic |
+----+-------------------+---------+---------------+-------------------+-------------------+---------+---------+
11 rows in set (0.00 sec)

```

## Intersect索引合并
对于目标SQL：
```sql
select * from discosongs where name='Feels' and singer='Calvin Harris';
```
由于name和singer两列上都有普通二级索引，可以有下面两种执行方案：
- 方案一：先扫描其中一个索引列，通过回表获取到符合条件的结果集，再对该结果集中的记录逐一判断另一个索引列条件是否成立。
- 方案二：同时扫描两个索引列（无需回表），得到两个包含主键信息的结果集；对这两个结果集取**交集**，找出id列值相同的记录，最后对这些id值相同的列执行回表操作。

方案二采用的表数据访问方法就是**Intersect索引合并**。与方案一相比，采用索引合并能够有效减少回表的次数。同时，二级索引的等值查询对应的扫描区间都是单点扫描区间，区间内的索引记录都是按照主键值进行排序的。对两个有序集合（`name='Feels'`和`singer='Calvin Harris'`）取交集的效率更高，并且根据已经排好序的id进行回表也可以提高执行效率。

```sql
mysql> explain select * from discosongs where name='Feels' and singer='Calvin Harris';
+----+-------------+------------+------------+-------------+---------------------+---------------------+---------+------+------+----------+---------------------------------------------------+
| id | select_type | table      | partitions | type        | possible_keys       | key                 | key_len | ref  | rows | filtered | Extra                                             |
+----+-------------+------------+------------+-------------+---------------------+---------------------+---------+------+------+----------+---------------------------------------------------+
|  1 | SIMPLE      | discosongs | NULL       | index_merge | idx_name,idx_singer | idx_name,idx_singer | 303,153 | NULL |    1 |   100.00 | Using intersect(idx_name,idx_singer); Using where |
+----+-------------+------------+------------+-------------+---------------------+---------------------+---------+------+------+----------+---------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

**反例**

下面两个SQL不会采用交集索引合并的方法访问表数据。
```sql
--name列的索引扫描区间不是单点扫描区间，结果集不是按照主键排序的
mysql> explain select * from discosongs where name>'Feels' and singer='Calvin Harris';
+----+-------------+------------+------------+------+---------------------+------------+---------+-------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys       | key        | key_len | ref   | rows | filtered | Extra       |
+----+-------------+------------+------------+------+---------------------+------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | discosongs | NULL       | ref  | idx_name,idx_singer | idx_singer | 153     | const |    4 |    72.73 | Using where |
+----+-------------+------------+------------+------+---------------------+------------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

--info_n1列位于联合索引idx_info中，联合索引从左至右依次按照各索引列进行排序
mysql> explain select * from discosongs where name='Feels' and info_n1='Funk Wav Bounces';
+----+-------------+------------+------------+------+-------------------+----------+---------+-------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys     | key      | key_len | ref   | rows | filtered | Extra       |
+----+-------------+------------+------------+------+-------------------+----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | discosongs | NULL       | ref  | idx_name,idx_info | idx_name | 303     | const |    2 |    36.36 | Using where |
+----+-------------+------------+------------+------+-------------------+----------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

## Union索引合并
对于目标SQL：
```sql
select * from discosongs where name='Feels' or singer='Queen';
```
我们也可以分别对两个二级索引`idx_name`和`idx_singer`分别执行索引扫描（无需回表），分别获取到符合条件的结果集（已经按照主键id排好序），再对两个结果集**合并去重**，最后根据主键id对去重后的并集做回表操作。这种访问表数据的方法就是**Union索引合并**。

```sql
mysql> explain select * from discosongs where name='Feels' or singer='Queen';
+----+-------------+------------+------------+-------------+---------------------+---------------------+---------+------+------+----------+-----------------------------------------------+
| id | select_type | table      | partitions | type        | possible_keys       | key                 | key_len | ref  | rows | filtered | Extra                                         |
+----+-------------+------------+------------+-------------+---------------------+---------------------+---------+------+------+----------+-----------------------------------------------+
|  1 | SIMPLE      | discosongs | NULL       | index_merge | idx_name,idx_singer | idx_name,idx_singer | 303,153 | NULL |    3 |   100.00 | Using union(idx_name,idx_singer); Using where |
+----+-------------+------------+------------+-------------+---------------------+---------------------+---------+------+------+----------+-----------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

**反例**

下面两个SQL也不会采用并集索引合并的方法访问表数据。
```sql
--name列的索引扫描区间不是单点扫描区间，结果集不是按照主键排序的
mysql> explain select * from discosongs where name>'Feels' or singer='Queen';
+----+-------------+------------+------------+------+---------------------+------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys       | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+------+---------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | discosongs | NULL       | ALL  | idx_name,idx_singer | NULL | NULL    | NULL |   11 |    40.00 | Using where |
+----+-------------+------------+------------+------+---------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

--info_n1列位于联合索引idx_info中，联合索引从左至右依次按照各索引列进行排序
mysql> explain select * from discosongs where name='Feels' or info_n1='Funk Wav Bounces';
+----+-------------+------------+------------+------+-------------------+------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys     | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+------+-------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | discosongs | NULL       | ALL  | idx_name,idx_info | NULL | NULL    | NULL |   11 |    19.00 | Using where |
+----+-------------+------------+------------+------+-------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

```

## Sort-Union索引合并
对于目标SQL：
```sql
select * from discosongs where name<'Feels' or ISNcode>600;
```
很明显无法直接采用Union索引合并的访问方法。但是如果我们分别对两个索引列进行扫描（无需回表），并对扫描后的结果分别按照主键id进行排序，就可以采用Union索引合并方法了。这种先将从各个索引中扫描到的行记录按照主键进行排序、再执行Union索引合并的表数据访问方法就是**Sort-Union索引合并**。

```sql
mysql> explain select * from discosongs where name<'Feels' or ISNcode>600;
+----+-------------+------------+------------+-------------+-----------------+-----------------+---------+------+------+----------+------------------------------------------------+
| id | select_type | table      | partitions | type        | possible_keys   | key             | key_len | ref  | rows | filtered | Extra                                          |
+----+-------------+------------+------------+-------------+-----------------+-----------------+---------+------+------+----------+------------------------------------------------+
|  1 | SIMPLE      | discosongs | NULL       | index_merge | uk_ISN,idx_name | idx_name,uk_ISN | 303,5   | NULL |    2 |   100.00 | Using sort_union(idx_name,uk_ISN); Using where |
+----+-------------+------------+------------+-------------+-----------------+-----------------+---------+------+------+----------+------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```


# unique_subquery
**unique_subquery**是针对包含**IN子查询**的SQL语句的执行计划。如果优化器决定将IN子查询转换为**EXISTS**子查询，并且子查询转换后可以使用**主键或者不允许存储NULL值的唯一二级索引**进行**等值匹配**，那么对应的就是**unique_subquery**方法。


# index_subquery
**index_subquery**与**unique_subquery**类似，只不过在访问子查询中的表时使用的是普通二级索引。


# range
**range**是指通过索引的范围扫描来获取要访问的行记录，一般是同时扫描多个单点值、或者扫描一个范围区间。

```sql
mysql> explain select * from discosongs where id>2 and id<5;
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | discosongs | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    2 |   100.00 | Using where |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql>
mysql> explain select * from discosongs where info_n1 in ('The Business','Disco Elysium');
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table      | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | discosongs | NULL       | range | idx_info      | idx_info | 303     | NULL |    2 |   100.00 | Using index condition |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from discosongs where info_n1='Funk Wav Bounces' and info_n2>'Frank Ocean';
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table      | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | discosongs | NULL       | range | idx_info      | idx_info | 606     | NULL |    2 |   100.00 | Using index condition |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)

```

# index
**index**是指通过扫描全部的二级索引记录来获取行记录。如果要访问的数据列都位于二级索引中，就可以通过扫描全部的二级索引记录来获取数据（**索引覆盖**），而且不用回表。

```sql
mysql> explain select info_n1,info_n2,info_n3 from discosongs where info_n3='UK';
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | discosongs | NULL       | index | NULL          | idx_info | 909     | NULL |    9 |    11.11 | Using where; Using index |
+----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

当通过全表扫描访问InnoDB表时，如果添加了`order by 主键`子句，那么在执行计划中也会被认为是index访问方法。

```sql
mysql> explain select name,singer,ratings from discosongs order by id;
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
|  1 | SIMPLE      | discosongs | NULL       | index | NULL          | PRIMARY | 4       | NULL |    9 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

# all
即全表扫描，通过扫描全部的主键索引来获取行记录。

```sql
mysql> explain select name,singer,ratings from discosongs;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | discosongs | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    9 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```



