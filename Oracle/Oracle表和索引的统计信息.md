---
tags: [oracle, SQL优化]
title: Oracle表和索引的统计信息
created: '2022-10-20T07:24:36.604Z'
modified: '2022-10-27T04:46:16.099Z'
---

Oracle表和索引的统计信息

# Oracle里的统计信息

Oracle数据库里的统计信息存储在**数据字典**里，从多个维度描述了Oracle数据库中对象的详细信息。CBO优化器会利用统计信息来计算目标SQL各种可能的、不同的执行路径的成本，并从中选择一条成本值最小的执行路径来作为目标SQL的执行计划。

Oracle里的统计信息可以分为如下6种类型：
- **表**的统计信息：用于描述表的详细信息，包含了诸如记录数、表块（表里的数据块）的数量、平均行长度等信息维度；
- **索引**的统计信息：用于描述索引的详细信息，包含了索引的层级、叶子块的数量、聚簇因子等信息维度；
- **列**的统计信息：用于描述列的详细信息，包含了列的distinct值的数量、列的NULL值的数量、列的最小值和最大值、以及直方图等统计信息；
- **系统**统计信息：用于描述Oracle所在的数据库服务器的系统处理能力，包含了CPU和I/O两个维度。借助于系统统计信息，Oracle可以更清楚地知道目标数据库服务器的实际处理能力；
- **数据字典**统计信息：用于描述Oracle数据字典基表（如`TAB$`、`IND$`等）、数据字典基表上的索引、以及这些数据字典基表的列的详细信息，描述上述数据字典基表的统计信息与描述普通表、索引、列的统计信息没有本质区别；
- **内部对象**统计信息：用于描述Oracle数据库里的一些内部表（如`X$`系列表）的详细信息，它的维度和普通表的统计信息维度类似，只不过其表块数量为0，因为`X$`系列表实际上只是Oracle自定义的内存结构。


# 统计信息收集

Oracle数据库中可以使用ANALYZE命令或者`DBMS_STATS`包收集统计信息。前面提到的六种统计信息中，**表、索引、列和数据字典**的统计信息用ANALYZE命令或者`DBMS_STATS`包都可以收集，但是**系统统计信息和内部对象**统计信息只能用`DBMS_STATS`包收集。

## 使用ANALYZE命令收集统计信息
从Oracle 10g开始，在创建索引后，Oracle会**自动收集目标索引的统计信息**。

```sql
--创建测试表和索引
> create table t2 as select * from dba_objects;
> create index idx_t2 on t2(object_id);

--删除自动收集的索引统计信息
> analyze index idx_t2 delete statistics;

--以估算模式对表T2收集统计信息，采样比例为15%
> analyze table t2 estimate statistics sample 15 percent for table;

--以计算模式对表T2收集统计信息
> analyze table t2 compute statistics for table;

--以计算模式对表T2的列object_id和object_name收集统计信息
> analyze table t2 compute statistics for columns object_name,object_id;
--//会抹掉上面的analyze命令收集的表统计信息
```

对同一个对象而言，新执行的ANALYZE命令会抹掉之前ANALYZE命令收集的结果（主要是表统计信息和列统计信息，不包括索引的统计信息）。

执行下面的命令来以计算模式同时收集表的统计信息和列的统计信息：
```sql
--以计算模式同时收集表T2的统计信息、列object_id和object_name的统计信息
> analyze table t2 compute statistics for table for columns object_name,object_id;
```

手动收集索引的统计信息：
```sql
> analyze index idx_t2 compute statistics;
```

删除**表、表的所有列以及表上所有索引**的统计信息：
```sql
> analyze table t2 delete statistics;
```

一次性收集**表、表的所有列以及表上所有索引**的统计信息：
```sql
> analyze table t2 compute statistics;
```

## 使用DBMS_STATS包收集统计信息
使用`DBMS_STATS`包收集统计信息是Oracle官方推荐的方式。`DBMS_STATS`包里最常用的是下面4个存储过程：
- `GATHER_TABLE_STATS`：用于收集目标表、及其列和索引的统计信息；
- `GATHER_INDEX_STATS`：用于收集指定索引的统计信息；
- `GATHER_SCHEMA_STATS`：用于收集指定schema下所有对象的统计信息；
- `GATHER_DATABASE_STATS`：用于收集全库所有对象的统计信息。

```sql
--以估算模式对表T2收集统计信息，采样比例为15%
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT',tabname => 'T2',estimate_percent => 15,method_opt => 'FOR TABLE',cascade => false);
```

如果要以**计算模式**收集统计信息，只需要把`estimate_percent`设置为100或者NULL即可。
```sql
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT',tabname => 'T2',estimate_percent => 100,method_opt => 'FOR TABLE',cascade => false);

> exec dbms_stats.gather_table_stats(ownname => 'SCOTT',tabname => 'T2',estimate_percent => NULL,method_opt => 'FOR TABLE',cascade => false);
```

以计算模式收集列和索引的统计信息：
```sql
--以计算模式收集表T2中列object_name和object_id的统计信息
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT',tabname => 'T2',estimate_percent => 100,method_opt => 'for columns size 1 object_name object_id',cascade => false);
--//上面的命令会同时收集表T2的统计信息

----以计算模式收集索引idx_t2的统计信息
> exec dbms_stats.gather_index_stats(ownname => 'SCOTT',indname => 'IDX_T2',estimate_percent => 100);
```

删除**表、表的所有列、以及表的所有索引**的统计信息：
```sql
> exec dbms_stats.delete_table_stats(ownname => 'SCOTT',tabname => 'T2');
```

一次性收集**表、表的所有列、以及表的所有索引**的统计信息：
```sql
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT',tabname => 'T2',estimate_percent => 100,cascade => true);
```

>:moon: ANALYZE和DBMS_STATS包收集统计信息的区别：
> - ANALYZE命令不能正确地收集**分区表**的统计信息，而DBMS_STATS包却可以；
> - ANALYZE命令不能**并行**收集统计信息，而DBMS_STATS包却可以；
> - DBMS_STATS包只能收集与CBO优化器相关的统计信息，而ANALYZE命令还可以收集与CBO无关的统计信息（例如行迁移/行链接的数量、索引的结构信息等）。


# 表的统计信息
## 表统计信息的种类和含义
表统计信息用于描述Oracle数据库里表的详细信息，包含了记录数、表块的数量、平均行长度等典型维度。这些信息实际上存储在数据字典基表`TAB$`、`TABPART$`、`TABSUBPART$`等中，可以通过数据字典`DBA_TABLES`、`DBA_TAB_PARTITIONS`和`DBA_TAB_SUBPARTITIONS`来分别查看表、分区表的分区、以及分区表的子分区的统计信息。

1. `NUM_ROWS`

上述数据字典中的字段`NUM_ROWS`存储的就是目标表的记录数。目标表的记录数是计算结果集的cardinality的基础，而结果集的cardinality则往往直接决定了CBO计算的成本值。比如对于嵌套循环连接而言，驱动结果集的cardinality越大，则走嵌套循环连接的成本值就会越大。

2. `BLOCKS`

上述数据字典中的字段`BLOCKS`存储的就是目标表表块的数量，即目标表的数据所占用数据块的数量。目标表表块的数量会直接决定CBO计算出来的对目标表做全表扫描的成本，目标表表块的数量越大，则对目标表走全表扫描的成本值就越大。

3. `AVG_ROW_LEN`

上述数据字典中的字段`AVG_ROW_LEN`存储的就是目标表的平均行长度。平均行长度的计算方法是用目标表的所有行记录所占用的字节数（不算行头）除以目标表的总行数。它可能会被Oracle用来计算目标表对应的结果集所占用内存的大小。

## 表统计信息不准导致SQL性能问题
**在导入大量数据后，应及时收集统计信息后才进行相关的后续业务处理**（包括查询和修改），否则可能会由于实际数据量和统计信息里记录的数据量存在巨大差异而导致CBO选择错误的执行计划。

无论使用ANALYZE还是用`DBMS_STATS`包收集统计信息，它们**均会提交当前事务**。ANALYZE命令是DDL语句，DDL语句通常情况下就会隐式提交；而`DBMS_STATS`包的那些收集统计信息的存储过程内部本身就包括了COMMIT语句。所以如果应用对事务有强一致性要求，则在导入大量数据后就不能在当前事务中收集统计信息了，因为只要一收集统计信息，当前事务就被提交了，这意味着当前事务的一致性已被破坏。

如果应用对事务有强一致性要求，而且在当前事务中导入大量数据后又必须在这个事务中进行相关的后续业务处理，则可以在后续处理的相关SQL中加入Hint（或者使用SQL Profile/SPM来替换相关SQL的执行计划），以便走出理想的执行计划，而不再受统计信息正确与否的干扰。

# 索引的统计信息
## 索引统计信息的种类和含义
索引统计信息是用于描述Oracle数据库里索引的详细信息，包含了索引的层级、叶子块的数量、聚簇因子等典型的维度。这些维度信息实际上存储在数据字典基表`IND$`、`INDPART$`、`INDCOMPART$`、`INDSUBPART$`等中，可以通过数据字典`DBA_INDEXES`、`DBA_IND_PARTITIONS`和`DBA_IND_SUBPARTITIONS`来分别查看索引、分区索引的分区和局部分区索引的子分区的统计信息。

1. `BLEVEL`

上述数据字典中的字段`BLEVEL`存储的就是目标索引的层级，它表示的是从根节点到叶子块的深度。BLEVEL被CBO优化器用于计算访问索引叶子块的成本。BLEVEL的值越大，从根节点到叶子块所需要访问的数据块的数量就越多，耗费的I/O也就越多，访问索引的成本就会越大。BLEVEL的值从0开始算起，当BLEVEL值为0时，表示该B树索引只有一层，且根节点和叶子块就是同一个块。

在Oracle数据库里，如果要降低目标B树索引的层级（比如在删除大量行记录后），可以通过rebuild该索引的方式来实现。
```sql
--重建索引
> alter index idx_t1 rebuild;
--分析索引结构
> analyze index idx_t1 validate structure;
> select name,height,lf_rows,lf_blks,del_lf_rows from index_stats;
```

2. `LEAF_BLOCKS`

上述数据字典中的字段`LEAF_BLOCKS`存储的就是目标索引的叶子块的数量。它被CBO用于计算对目标索引**做索引全扫描和索引范围扫描的成本**。目标索引叶子块的数量越多，则对目标索引做索引全扫描和索引范围扫描的成本值就越大。

3. `CLUSTERING_FACTOR`

上述数据字典中的字段`CLUSTERING_FACTOR`存储的就是目标索引的聚簇因子。聚簇因子是指按照索引键值排序的索引行和存储于对应表中的数据行的存储顺序的相似程度。

4. `DISTINCT_KEYS`

上述数据字典中的字段`DISTINCT_KEYS`存储的就是目标索引的索引键值的distinct值的数量。对于唯一索引而言，在没有NULL值的情况下，`DISTINCT_KEYS`的值就等于行记录的数量。

5. `AVG_LEAF_BLOCKS_PER_KEY`

上述数据字典中的字段`AVG_LEAF_BLOCKS_PER_KEY`存储的就是目标索引的每个distinct索引键值所占用的叶子块数量的平均值，对于唯一索引而言，`AVG_LEAF_BLOCKS_PER_KEY`的值等于1。

6. `AVG_DATA_BLOCKS_PER_KEY`

上述数据字典中的字段`AVG_DATA_BLOCKS_PER_KEY`存储的就是目标索引的每个distinct索引键值所对应的表中的数据行所在数据块数量的平均值。

7. `NUM_ROWS`

上述数据字典中的字段`NUM_ROWS`存储的就是目标索引的索引行的数量。

索引统计信息维度中需要重点关注的是`BLEVEL`、`LEAF_BLOCKS`和`CLUSTERING_FACTOR`。

## 聚簇因子的含义和重要性
在Oracle数据库中，聚簇因子指的是按照索引键值排序的索引行和存储于对应表中的数据行的存储顺序的相似程度。Oracle按照如下的方法来计算聚簇因子。

>1. 聚簇因子的初始值为1。
>2. Oracle首先定位到目标索引处于最左边的叶子块。
>3. 从最左边的叶子块的第一个索引键值所在的索引行开始顺序扫描。在顺序扫描的过程中，Oracle会比较当前索引行的ROWID和它之前的那个索引行的ROWID，如果这两个ROWID并不是指向同一个表块，那么Oracle就将聚簇因子的当前值递增1；如果这两个ROWID是指向同一个表块，Oracle就不改变聚簇因子的值。这里Oracle在对比ROWID的时候不需要回表去访问相应的表块。
>4. 上述对比ROWID的过程会一直持续下去，直到顺序扫描完目标索引所有叶子块里的所有索引行。
>5. 上述顺序扫描操作完成后，聚簇因子的当前值就是索引统计信息中的`CLUSTERING_FACTOR`，Oracle会将其存储在数据字典中。

如果聚簇因子的值**接近对应表的表块数量**，则说明目标索引的索引行和存储于对应表中的数据行的**存储顺序的相似程度非常高**。这意味着Oracle走索引范围扫描后取得目标ROWID再回表去访问对应表块数据时，相邻的索引行所对应的ROWID极有可能处于同一个表块中，即Oracle在通过索引行记录的rowid回表第一次去读取对应的表块并将该表块缓存在buffer cache中后，当再通过相邻索引行记录的rowid回表第二次去读取对应表块时，就**不需要再产生物理IO**了，因为这次要访问的和上次已经访问的是同一个表块，而Oracle已经将其缓存在了buffer cache中了。

如果聚簇因子的值**接近对应表的行记录数量**，则说明目标索引的索引行和存储于对应表中的数据行的**存储顺序的相似程度非常低**。这意味着Oracle走索引范围扫描后取得目标ROWID再回表去访问对应表块数据时，相邻的索引行所对应的ROWID极有可能**不**处于同一个表块中，即Oracle在通过索引行记录的rowid回表第一次去读取对应的表块并将该表块缓存在buffer cache中后，当再通过相邻索引行记录的rowid回表第二次去读取对应表块时，**还需要产生物理IO**，因为这次要访问的和上次已经访问的并**不**是同一个表块。

>:dog:聚簇因子高的索引走**索引范围扫描**时，比相同条件下聚簇因子低的索引要耗费更多的物理IO，所以聚簇因子高的索引走索引范围扫描的成本会比相同条件下聚簇因子低的索引走索引范围扫描的成本高。

在Oracle数据库中，能够降低目标索引的聚簇因子的唯一方法就是对表中数据按照目标索引的索引键值排序后重新存储。但是，这种按某一个目标索引的索引键值排序后重新存储表中数据的方法可能会同时增加该表上存在的其他索引的聚簇因子的值。

在Oracle数据库里，CBO优化器在计算索引范围扫描（Index Range Scan）的成本时会使用如下公式：
```
IRS Cost = IO Cost + CPU Cost
IO Cost = Index Access IO Cost + Table Access IO Cost
Index Access IO Cost = BLEVEL + CEIL(#LEAF_BLOCKS * IX_SEL)
Table Access IO Cost = CEIL(CLUSTERING_FACTOR * IX_SEL_WITH_FILTERS)
```
从上面的计算公式可以看出，走索引范围扫描的成本可以近似看作是和聚簇因子成正比。因此，聚簇因子的大小实际上对CBO判断是否走相关的索引起着至关重要的作用。




