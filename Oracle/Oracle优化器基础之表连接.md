---
tags: [oracle, SQL优化]
title: Oracle优化器基础之表连接
created: '2022-10-02T15:23:03.091Z'
modified: '2022-10-04T08:47:26.688Z'
---

Oracle优化器基础之表连接

当优化器解析包含表连接的目标SQL时，除了会根据SQL文本的写法来决定表连接类型，还必须考虑以下三点：

1. 表连接顺序

不管目标SQL中有多少表做连接，在实际执行过程中都只能**依次两两做表连接**，直到所有的表都连接完毕。当两张表做连接时，优化器需要考虑谁是驱动表（outer table），谁是被驱动表（inner table）。当多表做连接时，优化器需要决定两两表连接的顺序。

2. 表连接方法

Oracle数据库中，两表连接的方法包括排序合并连接、嵌套循环连接、哈希连接、笛卡尔连接四种。

3. 访问单表的方法

优化器在对目标SQL做表连接时，还必须决定如何获取存储在表里的不同维度的数据，即决定访问单表的方法。比如采用全表扫描还是走索引，以及采用什么索引访问方法。

# 表连接类型

## 内连接（Inner Join）
内连接是指表连接的连接结果只包含那些完全满足连接条件的记录。对于包含表连接的目标SQL而言，只要WHERE条件中没有出现外连接关键字（如left outer join、right outer join、full outer join）以及外连接符号`(+)`，那么连接类型就是内连接。

```sql
--没有outer关键字，默认为内连接
> select t1.col1, t1.col2, t2.col3
from t1, t2 
where t1.col2=t2.col2;

--标准SQL的内连接写法
> select t1.col1, t1.col2, t2.col3
from t1
join t2 on (t1.col2=t2.col2);

--标准SQL的另一种内连接写法（连接列col2前不能带上表名）
> select t1.col1, col2, t2.col3
from t1
join t2 using col2;
```

对于内连接而言，除了表连接条件之外的额外限制条件在目标SQL文本中所处的位置，并不会影响SQL的实际执行结果。例如下面两条SQL的执行结果是相同的：

```sql
> select t1.col1, t1.col2, t2.col3
from t1 join t2
on (t1.col2=t2.col2 and t1.col1=1);

> select t1.col1, t1.col2, t2.col3
from t1 join t2
on (t1.col2=t2.col2)
where t1.col1=1;
```

## 外连接（Outer Join）
标准SQL中的外连接分为左外连接（left outer join）、右外连接（right outer join）、全连接（full outer join）。外连接的连接结果中除了包含那些完全满足连接条件的记录之外，还会包含驱动表中所有不满足连接条件的记录。驱动表中所有不满足连接条件的记录所对应的被驱动表中的查询列，均会以NULL来填充。

```sql
--左连接关键字左边的表t1为驱动表
> t1 left outer join t2 on (...)
> t1 left outer join t2 using (...)

--右连接关键字右边的表t2为驱动表
> t1 right outer join t2 on (...)
> t1 right outer join t2 using (...)

--全连接
> t1 full outer join t2 on (...)
> t1 full outer join t2 using (...)
```

对于外连接而言，除了表连接条件之外的额外限制条件在目标SQL文本中所处的位置，**可能会影响**SQL的实际执行结果。例如下面两条SQL的执行结果是不同的：

```sql
> select t1.col1, t1.col2, t2.col3
from t1 right outer join t2
on (t1.col2=t2.col2 and t1.col1=1);

COL1  COL2  COL3
----  ----  ----
  1     A     A2
              D2

> select t1.col1, t1.col2, t2.col3
from t1 right outer join t2
on (t1.col2=t2.col2)
where t1.col1=1;

COL1  COL2  COL3
----  ----  ----
  1     A     A2     //少了col1为NULL的行
```

Oracle中使用`(+)`符号来表示外连接，该符号出现在哪一个表的连接列后面，就表示哪个表会以NULL填充不满足连接条件的查询列。也就是说，`(+)`符号前面列对应的表为外连接的**被驱动表**，另一张表就是驱动表。

```sql
> select t1.col1, t1.col2, t2.col3
from t1 left outer join t2
on (t1.col2=t2.col2);
--相当于
> select t1.col1, t1.col2, t2.col3
from t1, t2
where t1.col2 = t2.col2(+);
```

如果外连接条件中还包含额外的限制条件，那么额外的限制条件中对应的列后面也需要加上`(+)`符号。

```sql
> select t1.col1, t1.col2, t2.col3
from t1 right outer join t2
on (t1.col2=t2.col2 and t1.col1=1);
--相当于
> select t1.col1, t1.col2, t2.col3
from t1, t2
where t1.col2(+) = t2.col2
and t1.col1(+) = 1;             --//表示该限制条件在做表连接之前应用

> select t1.col1, t1.col2, t2.col3
from t1 right outer join t2
on (t1.col2=t2.col2)
where t1.col1=1;
--相当于
> select t1.col1, t1.col2, t2.col3
from t1, t2
where t1.col2(+) = t2.col2
and t1.col1 = 1;                --//表示该限制条件在做表连接之后应用
```


# 表连接方法
Oracle数据库中表连接包括排序合并连接、嵌套循环连接、哈希连接、笛卡尔连接四种方法。

## 排序合并连接（Sort Merge Join）
排序合并连接是一种在做表连接时通过排序操作和合并操作来得到连接结果集的表连接方法。具体可以分为以下步骤：

1. 首先以目标SQL中指定的谓词条件去访问T1表，并对访问结果按照T1中的连接列来排序，排好序的结果集记为Set_1；
2. 然后以目标SQL中指定的谓词条件去访问T2表，并对访问结果按照T2中的连接列来排序，排好序的结果集记为Set_2；
3. 最后对Set_1和Set_2执行合并操作，从中取出匹配记录来作为排序合并连接的最终执行结果。

通常情况下，Sort Merge Join的执行效率远不如哈希连接，但是适用范围更广。因为哈希连接通常只能用于等值连接条件，而排序合并连接还能用于范围连接条件（<、<=、>、>=）。

一般情况下，排序合并连接并不适合OLTP类型的系统。对于OLTP类型的系统而言，排序是代价非常昂贵的操作。但是如果能够避免排序（比如连接列上有索引），OLTP系统也可以走排序合并连接。

严格意义上讲，排序合并连接中并不存在驱动表的概念。

## 嵌套循环连接（Nested Loops Join）
循环嵌套连接是指在做表连接时依靠两层嵌套循环来得到连接结果集的表连接方法。具体可以分为以下步骤：

1. 首先，优化器会按照一定的规则来决定谁是驱动表、谁是被驱动表。驱动表用于外层循环，被驱动表用于内层循环。假设T1为驱动表，T2为被驱动表；

2. 以目标SQL中指定的谓词条件去访问驱动表T1，访问T1后得到的结果集记为驱动结果集Set_1；

3. 然后遍历驱动结果集Set_1，并同时遍历被驱动表T2。对于Set_1中的每一条记录，都需要遍历被驱动表T2，按照连接条件判断T2中是否存在匹配的记录。

如果驱动表对应的**驱动结果集的记录数比较少**，同时在被驱动表的连接列上又存在唯一性索引，或者在被驱动表的连接列上存在选择性很好的非唯一性索引，那么嵌套循环连接的效率就会很高。反之，如果驱动结果集中的记录数很多，即便在被驱动表的连接列上存在索引，嵌套循环连接的效率也不会高。

大表也可以作为嵌套循环连接的驱动表，关键看目标SQL中的谓词条件能够否将驱动结果集的数据量显著降下来。

相比于其他连接方法，嵌套循环连接可以实现**快速响应**，即可以第一时间先返回已经连接过且满足条件的记录，而不必等待所有的连接操作全部完成后才返回结果集。


## 哈希连接（Hash Join）
哈希连接是指在做表连接时主要依靠哈希运算来得到连接结果集的表连接方法。具体可以分为以下步骤：

1. 首先，Oracle会根据`HASH_AREA_SIZE`、`DB_BLOCK_SIZE`和`_HASH_MULTIBLOCK_IO_COUNT`的值来决定哈希分区（hash partition）的数量。哈希分区实际上是一组哈希桶（hash bucket）的集合，而所有哈希分区的集合组成了一个哈希表（hash table）。

2. 表t1和t2在应用了目标SQL中的谓词条件后，得到的结果集中数据量较少的一个会被选定为哈希连接的驱动结果集。假设t1对应的结果集为驱动结果集，记为S；t2对应的为被驱动结果集，记为M。

3. Oracle会遍历驱动结果集S，并对其中的每一条记录按照该记录在表t1中的连接列做哈希运算。这里会使用到两个内置哈希函数同时对连接列计算哈希值。假设计算出来的哈希值分别为hash_val1和hash_val2，对应的哈希函数分别记为hash_func1和hash_func2。

4. 然后按照hash_val1的值把相应的S中的对应记录存储在不同哈希分区的不同哈希桶里，同时和该记录存储在一起的还有该记录对应的hash_val2。存储在哈希桶里的记录并不是完整的行记录，只需要存储目标SQL中与目标表相关的查询列和连接列就行。我们把S对应的每一个哈希分区编号为Si。

5. 在创建Si的同时，Oracle会构建一个位图（Bitmap）用于标记Si所包含的每一个哈希桶是否有记录（即记录数是否大于0）。

6. 如果驱动结果集S的数据量很大，在构建对应的哈希表时，可能会出现PGA工作区（Work Area）被填满的情况。此时Oracle会把工作区中包含记录数最多的哈希分区写到磁盘上（TEMP表空间）。如果要构建的记录对应的哈希分区已经被写回磁盘，Oracle就会去磁盘上更新该哈希分区。

7. 持续构建S对应的哈希表，直到遍历完S中的所有记录为止。

8. 接下来Oracle会对所有的哈希分区Si按照它们包含的记录数来排序，然后把这些排好序的哈希分区按顺序依次且尽可能地全部放到内存中PGA的工作区中（放不下的部分会留在磁盘中）。其目的是尽可能多地把那些记录数较小的哈希分区保留在内存中。

9. 接下来Oracle开始遍历被驱动结果集M，对其中的每一条记录按照该记录在表t2中的连接列做哈希运算。同时也会使用hash_func1和hash_func2分别计算得到相应的hash_val1和hash_val2。

Oracle会按照该记录对应的hash_val1去Si里寻找匹配的哈希桶。如果找到了，Oracle还会遍历该哈希桶里的每一条记录，并校验存储在该哈希桶里的每一条记录的连接列，看S和M中的匹配记录所对应的连接列是否真的相等（考虑到有哈希碰撞的可能）。如果是真的匹配，则hash_val1所对应M中记录的位于目标SQL中的查询列和该哈希桶中的匹配记录就会被组合起来，一起作为满足连接条件的记录被返回。如果找不到匹配的哈希桶，Oracle就会访问步骤5中构建的位图。

如果位图显示该哈希桶在Si中对应的记录数大于0，则说明该哈希桶已经被写回磁盘。此时Oracle就会按照hash_val1的值，把相应M中的对应记录也以哈希分区的方式写回到磁盘上，同时和该记录存储在一起的还有该记录对应的hash_val2值。如果位图显示该哈希桶在Si中对应的记录数等于0，则说明该记录必然不满足连接条件，也就无须被写回磁盘了。这种根据位图来决定是否将hash_val1对应的M中的记录写回到磁盘的动作，被称为**位图过滤**。

我们把M对应的每一个哈希分区记为Mj。

10. 持续进行上面区Si中查找匹配哈希桶和构建Mj的过程，直到遍历完M中的所有记录为止。

11. 至此Oracle已经处理完所有位于内存中的Si和对应的Mj。只剩下磁盘中的记录还未处理。

12. 在构建Si和Mj时使用的是相同的哈希函数，因此在处理磁盘上的记录时，只有对应哈希分区编号相同的Si和Mj才有可能会产生满足连接条件的记录。我们用Sn和Mn分别来表示位于磁盘上且对应哈希分区编号相同的Si和Mj。

13. 对于每一对Sn和Mn，他它们之中记录数较少的会被当作驱动结果集，然后Oracle会用这个驱动结果集哈希桶里记录的hash_val2来构建新的哈希表。另一个记录数较多的则会被当做被驱动结果集，Oracle会利用被驱动结果集哈希桶里记录的hash_val2去新构建的哈希表中寻找匹配的记录。

14. 上一步骤中如果存在匹配记录，则该匹配记录也会作为满足目标SQL连接条件的记录被返回。

15. 上述处理Sn和Mn的过程会一直持续，直到遍历完所有的Sn和Mn为止。

哈希连接有以下特点：

- 哈希连接大多数情况下都不需要排序。
- 哈希连接的**驱动表对应的连接列的可选择性应该尽可能好**，因为会影响到对应哈希桶里的记录数，而哈希桶里的记录数会直接影响到从中查找匹配记录的效率。如果一个哈希桶里的记录数太多，可能会严重降低哈希连接的效率。典型的表现就是哈希连接执行了很长时间都没有结束，数据库服务器的CPU使用率很高但是目标SQL的逻辑读却很少。因为此时大部分时间都消耗在了遍历哈希桶里的记录上，这个动作发生在PGA的工作区，不耗费逻辑读。
- 哈希连接只适用于CBO优化器，也只能用于**等值连接条件**。
- 哈希连接很适合于小表和大表之间做表连接且连接结果集的记录数较多的情况，特别是在小表的连接列的可选择性非常好的情况下。此时哈希连接的执行时间可以近似看作是和全表扫描大表所耗费的时间相当。
- 如果在应用了谓词条件后，得到的数据量较小的结果集（即驱动结果集）所对应的哈希表能够完全被容纳在内存中（PGA工作区），此时的哈希连接的执行效率会非常高。


## 笛卡尔连接（Cross Join | Cartesian）
笛卡尔连接又被称为笛卡尔积（Cartesian Product），是一种在做表连接时没有任何连接条件的表连接方法。具体可以分为以下步骤：

1. 首先以目标SQL中的谓词条件访问表T1，得到的结果集记为Set_1。假设Set_1的记录数为m。

2. 然后以目标SQL中的谓词条件访问表T2，得到的结果集记为Set_2。假设Set_2的记录数为n。

3. 最后对Set_1和Set_2执行合并操作，从中取出匹配记录作为笛卡尔连接的最终执行结果。由于没有连接条件，对于Set_1中的任意一条记录，Set_2中的所有记录都满足条件，所以两者笛卡尔连接的结果集记录数为`m*n`。

笛卡尔连接实际上是一种特殊的合并连接，只不过在执行合并时没有连接条件。
```sql
> set autotrace on
> select t1.col1,t2.col2 from t1,t2;
...
-------------执行计划-------------
0 | SELECT STATEMENT       |    |
1 |  MERGE JOIN CARTESIAN  |    |   --//笛卡尔连接  
2 |   TABLE ACCESS FULL    | T2 |
3 |   BUFFER SORT          |    |
4 |    TABLE ACCESS FULL   | T1 |

--标准SQL中的笛卡尔连接写法
> select t1.col1,t2.col2 from t1 cross join t2;
```

笛卡尔连接具有以下特点：

- 笛卡尔连接的出现一般是由于漏写了连接条件，所以笛卡尔连接一般是不好的。
- 有时候笛卡尔连接是因为在目标SQL中使用了ORDERED Hint，但是SQL文本中位置相邻的两个表之间又没有直接关联的条件。
- 有时候出现笛卡尔连接是因为目标SQL中相关表的统计信息不准。


# 反连接（Anti Join）
反连接表示从驱动表结果集中移除满足反连接条件的记录。Oracle中并没有专门的关键字来表示使用反连接，但是在做子查询展开时，Oracle经常会把外部WHERE条件为`NOT EXISTS`、`NOT IN`或`<> ALL`的子查询转化为对应的反连接。

```sql
> set autotrace on
--如果T1,T2在各自的连接列col2上没有Null值，那么以下三条SQL是等价的
> select * from t1
where col2 not in (select col2 from t2);
---------------执行计划----------------
  0 | SELECT STATEMENT       |    |
* 1 |  HASH JOIN ANTI NA     |    |   --//反连接  
  2 |   TABLE ACCESS FULL    | T1 |
  3 |   TABLE ACCESS FULL    | T2 |

> select * from t1
where col2 <> all (select col2 from t2);
---------------执行计划----------------
  0 | SELECT STATEMENT       |    |
* 1 |  HASH JOIN ANTI NA     |    |   --//反连接  
  2 |   TABLE ACCESS FULL    | T1 |
  3 |   TABLE ACCESS FULL    | T2 |

> select * from t1
where not exists (select 1 from t2 where col2=t1.col2);
---------------执行计划----------------
  0 | SELECT STATEMENT       |    |
* 1 |  HASH JOIN ANTI        |    |   --//反连接  
  2 |   TABLE ACCESS FULL    | T1 |
  3 |   TABLE ACCESS FULL    | T2 |
```

上面执行计划中的**NA**表示Null-Aware，表示是否对NULL敏感。可以看出，`NOT IN`和`<> ALL`对NULL敏感。如果它们后面的子查询或者常量集合一旦有NULL值出现，整个SQL的执行结果就会变为NULL，即此时的执行结果将不包含任何记录。`NOT EXISTS`对NULL不敏感，因此其后面出现的NULL值对NOT EXISTS的执行结果不会有什么影响。

# 半连接（Semi Join）
半连接是一种特殊的连接类型，在做表连接时，只要被驱动表中存在一条满足连接条件的记录，就会立即停止遍历被驱动表，并返回第一条满足条件的记录（实际上起到了去重的作用）。Oracle中也没有专门的关键字来表示使用半连接，但是在做子查询展开时，Oracle经常会把外部WHERE条件为`EXISTS`、`IN`或`= ANY`的子查询转化为对应的半连接。

```sql
> set autotrace on
> select * from t1
where col2 in (select col2 from t2);
---------------执行计划----------------
  0 | SELECT STATEMENT       |    |
* 1 |  HASH JOIN SEMI        |    |   --//半连接  
  2 |   TABLE ACCESS FULL    | T1 |
  3 |   TABLE ACCESS FULL    | T2 |

> select * from t1
where col2 = any (select col2 from t2);
---------------执行计划----------------
  0 | SELECT STATEMENT       |    |
* 1 |  HASH JOIN SEMI        |    |   --//半连接  
  2 |   TABLE ACCESS FULL    | T1 |
  3 |   TABLE ACCESS FULL    | T2 |

> select * from t1
where exists (select 1 from t2 where col2=t1.col2);
---------------执行计划----------------
  0 | SELECT STATEMENT       |    |
* 1 |  HASH JOIN SEMI        |    |   --//半连接  
  2 |   TABLE ACCESS FULL    | T1 |
  3 |   TABLE ACCESS FULL    | T2 |
```

# 星型连接（Star Join）
星型连接通常应用于数据仓库类型的应用，它是一种单个事实表（fact table）与多个维度表（dimension table）之间的连接。星型连接的各维度表之间没有直接的关联条件，其事实表与各维度表之间是基于事实表的外键列和对应维度表的主键列之间的连接，并且通常在事实表的外键列上还会存在对应的位图索引。

严格来说，星型连接并不是一种额外的连接类型，也不是一种额外的连接方法。星型连接命名的来源是因为事实表和各维度表之间的表连接看起来就像一颗五角星。



