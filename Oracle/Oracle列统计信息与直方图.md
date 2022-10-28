---
tags: [oracle, SQL优化]
title: Oracle列统计信息与直方图
created: '2022-10-26T08:15:10.136Z'
modified: '2022-10-28T11:13:02.440Z'
---

Oracle列统计信息与直方图

# 列统计信息的种类和含义
列统计信息用于描述Oracle数据库里列的详细信息，包含了列的distinct值的数量、列的NULL值的数量、列的最小值和最大值等一些典型的维度。这些统计信息实际上存储在数据字典基表`HIST_HEAD$`中，可以通过数据字典`DBA_TAB_COL_STATISTICS`、`DBA_PART_COL_STATISTICS`、`DBA_SUBPART_COL_STATISTICS`来分别查看列、分区表的分区、以及分区表的子分区的列统计信息。

1. `NUM_DISTINCT`

上述数据字典中的字段`NUM_DISTINCT`存储的就是目标列的distinct值的数量。CBO优化器使用该字段的值来评估对目标列做**等值查询的可选择率**（Selectivity）。在目标列上没有直方图且没有NULL值的情况下，用目标列做等值查询的可选择率可以通过下面的公式计算：
```
selectivity_without_null = (1 / num_distinct)
```

2. `NUM_NULLS`

上述数据字典中的字段`NUM_NULLS`存储的就是目标列的NULL值的数量。CBO优化器使用该字段的值来评估对目标列施加`IS NULL`或者`IS NOT NULL`条件后返回的结果集的Cardinality。CBO还会使用`NUM_NULLS`的值来调整对有NULL值的目标列做等值查询的可选择率。在目标列上没有直方图且有NULL值的情况下，用目标列做等值查询的可选择率可以通过下面的公式计算：
```sql
--//当目标列上没有直方图时
selectivity_with_null = (1 / num_distinct) * ((num_rows - num_nulls)/num_rows)
```

3. `LOW_VALUE`和`HIGH_VALUE`

上述数据字典中的字段`LOW_VALUE`和`HIGH_VALUE`存储的分别就是目标列的最小值和最大值。CBO优化器使用这两个字段的值来评估对目标列做**范围查询时的可选择率**。当目标列上没有直方图时，用目标列做范围查询的可选择率可以通过下面的公式计算：

- 目标列大于指定VAL，且VAL处于`LOW_VALUE`和`HIGH_VALUE`之间：
```
selectivity = (high_value - val) / (high_value - low_value) * null_adjust
null_adjust = (num_rows - num_nulls) / num_rows
```

- 目标列小于指定VAL，且VAL处于`LOW_VALUE`和`HIGH_VALUE`之间：
```
selectivity = (val - low_value) / (high_value - low_value) * null_adjust
null_adjust = (num_rows - num_nulls) / num_rows
```

- 目标列大于或等于指定VAL，且VAL处于`LOW_VALUE`和`HIGH_VALUE`之间：
```
selectivity = ((high_value - val) / (high_value - low_value) + 1/num_distinct) * null_adjust
null_adjust = (num_rows - num_nulls) / num_rows
```

- 目标列小于或等于指定VAL，且VAL处于`LOW_VALUE`和`HIGH_VALUE`之间：
```
selectivity = ((val - low_value) / (high_value - low_value) + 1/num_distinct) * null_adjust
null_adjust = (num_rows - num_nulls) / num_rows
```

- 目标列在指定VAL1和VAL2之间，且VVAL1和VAL2均处于`LOW_VALUE`和`HIGH_VALUE`之间：
```
selectivity = ((val2 - val1) / (high_value - low_value) + 2/num_distinct) * null_adjust
null_adjust = (num_rows - num_nulls) / num_rows
```


4. `DENSITY`和`NUM_BUCKETS`

上述数据字典中的字段`DENSITY`和`NUM_BUCKETS`存储的分别就是目标列的密度和所用桶的数量。这两个维度仅与直方图有关。

>:fish:**谓词越界**
>
>如果对目标列指定的where查询条件不在该列（统计信息）的最大值和最小值之间，CBO优化器就无法判断出针对该列的查询条件的可选择率，所以只能用一个估算值来作为针对该目标列的查询条件的可选择率。如果这个估算的可选择率率与实际情况严重不符的话，就有可能导致CBO评估出来的cardinality出现严重偏差，进而使得CBO选错执行计划。


# 直方图
## 直方图的含义
在Oracle数据库中，CBO优化器会默认目标列的最小值和最大值（`LOW_VALUE`和`HIGH_VALUE`）之间的数据是均匀分布的，并且会据此来计算对目标列施加查询条件后的可选择率和结果集的cardinality，进而来计算成本值并选择执行计划。如果目标列的数据不是均匀分布的，那么CBO选择的执行计划就可能是不合理的。

```sql
--创建一个列B数据分布不均衡的测试表
> select b, count(*) from t1 group by b;
B    COUNT(*)
---  --------
1      10000
2        1

--创建索引并收集列统计信息
> create index idx_t1_b on t1(b);
> exec dbms_stats.gather_table_stats('CAIPRA', 'T1', estimate_percent=>100,
method_opt=>'for all columns size 1');

--测试SQL
> select * from t1 where b=2;
执行计划
Id | Operation          | Name | Rows |
 0 | SELECT STATEMENT   |      |      |
*1 |  TABLE ACCESS FULL |  T1  | 5001 |
```

测试表上`b=2`的数据只有一行，由于CBO默认列B的数据是均匀分布的，所以评估出来对列B施加等值查询的可选择率为`1/2`，导致最终计算出来的cardinality为5001，进而选择了走全表扫描。

为了解决上述问题，Oracle引入了直方图（Histogram）。直方图是一种特殊的列统计信息，它详细描述了目标列的数据分布情况。直方图实际上存储在数据字典基表`HISTGRM$`中，可以通过数据字典`DBA_TAB_HISTOGRAMS`、`DBA_PART_HISTOGRAMS`和`DBA_SUBPART_HISTOGRAMS`来分别查询表、分区表的分区和分区表的子分区的直方图统计信息。

如果对目标列收集了直方图，则意味着CBO将不再认为该目标列上的数据是均匀分布的了，CBO就会用该目标列上的直方图统计信息来计算对该列施加查询条件后的可选择率、以及返回结果集的cardinality，进而据此计算成本并选择执行计划。

## 直方图的类型
Oracle里的直方图使用了桶（Bucket）的方式来描述目标列的数据分布，类似于哈希算法中的哈希桶。每个桶就是一组，里面会存储一个或多个目标列中的数据。Oracle会使用`ENDPOINT_NUMBER`和`ENDPOINT_VALUE`两个维度来描述一个桶，并将这两个维度的值存储在数据字典基表`HISTGRM$`中。`ENDPOINT_NUMBER`和`ENDPOINT_VALUE`分别对应数据字典`DBA_TAB_HISTOGRAMS`、`DBA_PART_HISTOGRAMS`和`DBA_SUBPART_HISTOGRAMS`中的字段`ENDPOINT_NUMBER`（或者`BUCKET_NUMBER`）和`ENDPOINT_VALUE`。此外，还可以通过数据字典`DBA_TAB_COL_STATISTICS`、`DBA_PART_COL_STATISTICS`和`DBA_SUBPART_COL_STATISTICS`中的字段`NUM_BUCKETS`来查看目标列对应直方图的Bucket的总数。

在Oracle 12c之前，Oracle数据库中的直方图可以分为Frequency和Height Balacned两种类型（Oracle 12c中新增了Top-Frequency和Hybrid两种新的类型）。在Oracle 12c之前，如果存储在数据字典里描述**目标列直方图的Bucket的数量等于目标列distinct值的数量**，对应的就是Frequency类型的直方图；如果目标列直方图的Bucket的数量**小于**目标列distinct值的数量，对应的就是Height Balanced类型的直方图。

### Frequency类型的直方图
对于Frequency类型的直方图而言，目标列直方图的Bucket的数量等于目标列distinct值的数量。目标列有多少个distinct值，Oracle在数据字典`DBA_TAB_HISTOGRAMS`、`DBA_PART_HISTOGRAMS`和`DBA_SUBPART_HISTOGRAMS`中就会存储多少行记录，并且每一条记录就对应了其中的一个Bucket。上述数据字典中，字段`ENDPOINT_VALUE`记录了distinct的列值，而`ENDPOINT_NUMBER`是一个累加值，记录了到此distinct列值为止总共有多少条记录。用当前行记录的`ENDPOINT_NUMBER`减去上一条行记录的`ENDPOINT_NUMBER`，就得到了当前行记录本身对应的`ENDPOINT_VALUE`值的数量。

在Oracle 12c之前，使用Frequency类型的直方图有如下限制：**Frequency类型的直方图所对应的Bucket数量不能超过254个**。即Frequency类型的直方图只适用于那些目标列的distinct值数量小于或等于254的情形。

#### 一个例子

使用`DBMS_STATS.GATHER_TABLE_STATS`方法对表收集统计信息时，对输入参数`METHOD_OPT`指定值`for column size auto X`就表示对目标表的列X收集直方图统计信息，**auto**的含义表示让Oracle自行决定是否对列X收集直方图以及使用哪种类型的直方图。

```sql
--对表h1的列X收集直方图统计信息
> exec dbms_stats.gather_table_stats(ownname=>'IPRA',tabname=>'H1',
method_opt=>'for column size auto X',cascade=>true);

--查看数据字典中列X的统计信息
> select table_name,column_name,num_distinct,density,num_buckets,histogram
from dba_tab_col_statistics where table_name='H1';
TABLE_NAME  COLUMN_NAME  NUM_DIST  DENSITY  NUM_BUCKETS  HISTOGRAM
----------  -----------  --------  -------  -----------  ---------
    H1           X          10       0.1         1          NONE
--//列X未被使用过，所以没有自动收集直方图统计信息

> select object_id from dba_objects where object_name='H1';
OBJECT_ID
---------
  82463
> select name,intcol# from sys.col$ where obj#=82463;
NAME     INTCOL#
----     -------
 X          1
 > select obj#,intcol#,equality_preds from sys.col_usage$ where obj#=82463;
 OBJ#   INTCOL#   EQUALITY_PREDS
 ----   -------   --------------
--//列X未被使用过，sys.col_usage$中没有该列的使用记录

--使用目标列后重新收集直方图统计信息
> select count(*) from h1 where x=10;
> exec dbms_stats.gather_table_stats(ownname=>'IPRA',tabname=>'H1',
method_opt=>'for column size auto X',cascade=>true);

--sys.col_usage$中有了列X的使用记录
> select obj#,intcol#,equality_preds from sys.col_usage$ where obj#=82463;
 OBJ#    INTCOL#   EQUALITY_PREDS
 -----   -------   --------------
 82463      1             1

--列X上有了frequency类型的直方图统计信息
> select table_name,column_name,num_distinct,density,num_buckets,histogram
from dba_tab_col_statistics where table_name='H1';
TABLE_NAME  COLUMN_NAME  NUM_DIST  DENSITY  NUM_BUCKETS  HISTOGRAM
----------  -----------  --------  -------  -----------  ---------
    H1           X          10     1.25E-5      10       FREQUENCY

--查看数据字典中列X的直方图统计信息内容
> select table_name,column_name,endpoint_number,endpoint_value 
from dba_tab_histograms where table_name='H1';
TABLE_NAME  COLUMN_NAME  ENDPOINT_NUMBER  ENDPOINT_VALUE
----------  -----------  ---------------  --------------
    H1           X             3296              1
    H1           X             3396              3
    H1           X             4194              5
   ...
```

Oracle在自动收集直方图统计信息时会遵循一个原则，就是**只对那些使用过的**（在where条件中出现过的）列收集直方图统计信息。Oracle会在表`SYS.COL_USAGE$`中记录各表中各列的使用情况，在自动收集直方图统计信息时Oracle会去查询表`SYS.COL_USAGE$`。

#### 对文本型字段收集直方图统计信息的坑
在Oracle数据库里，如果针对文本型字段收集直方图统计信息，则Oracle只会将该文本型字段的文本值的前32个字节给取出来（实际上只取头15个字节），并将其转换为一个浮点数，然后将这个浮点数作为其直方图统计信息存储在数据字典里。

这种处理机制的缺陷在于，对于那些超过32个字节的文本型字段，如果对应记录文本值的头32个字节相同，Oracle在收集直方图统计信息时就会认为这些记录在该字段的值是相等的，即使实际上它们不相同。这种先天性的缺陷会直接影响CBO优化器对相关文本型字段的可选择率以及返回结果集cardinality的评估。

### Height Balanced类型的直方图
在Oracle 12c之前，如果列的distinct值的数量超过了254个，Oracle就会对目标列收集Height Balanced类型的直方图统计信息。

Oracle首先会根据目标列对目标表的所有记录按从小到大的顺序排列，然后用目标表的总记录数除以需要使用的Bucket数量，来决定每个Bucket里需要描述的已排好序的记录数。假设目标表总记录数为M，需要用到的Bucket数量为N，则每个Bucket里需要描述的已排好序的记录数`O = M/N`。

然后Oracle会用`DBA_TAB_HISTOGRAMS`、`DBA_PART_HISTOGRAMS`和`DBA_SUBPART_HISTOGRAMS`中每一条记录的`ENDPOINT_NUMBER`来记录Bucket编号（从0开始一直到N）。其中0号桶存储的`ENDPOINT_VALUE`是目标列的最小值，其余的Bucket中`ENDPOINT_VALUE`存储的则是到此记录的Bucket之前所有桶描述的记录里目标列的最大值。

```sql
--//计算1至N号桶里存储的ENDPOINT_VALUE
select max(目标列) from (select 目标列 from 目标表 order by 目标列) where rownum<=O;  --1号桶
select max(目标列) from (select 目标列 from 目标表 order by 目标列) where rownum<=O*2;  --2号桶
select max(目标列) from (select 目标列 from 目标表 order by 目标列) where rownum<=O*3;  --3号桶
...
select max(目标列) from (select 目标列 from 目标表 order by 目标列) where rownum<=O*N;  --N号桶
```

为了节省存储空间，Oracle会对那些相邻的、仅`ENDPOINT_NUMBER`不同、但是`ENDPOINT_VALUE`相同的记录合并存储，并且只在数据字典中存储合并后的记录。也就是说，对于下面两条记录
```
ENDPOINT_NUMBER  ENDPOINT_VALUE
---------------  --------------
      2                 P
      3                 P
```
Oracle只会存储`ENDPOINT_NUMBER=3`的那条记录。即数据字典`DBA_TAB_HISTOGRAMS`、`DBA_PART_HISTOGRAMS`和`DBA_SUBPART_HISTOGRAMS`中的`ENDPOINT_NUMBER`字段可能是不连续的。这种记录在数据字典里的合并后的记录所在的`ENDPOINT_VALUE`被称之为**popular value**。可以看出，popular value所在记录的`ENDPOINT_NUMBER`值和它上一条记录的`ENDPOINT_NUMBER`值之间的**差值越大**，则意味着该popular value在目标表中所占的比例也就越大，它所对应的cardinality也就越大。

## 直方图的收集方法
在Oracle数据里收集直方图统计信息，通常是在调用`DBMS_STATS`包中的存储过程收集统计信息时通过指定输入参数`METHOD_OPT`来实现的。`METHOD_OPT`可以接受如下输入值：
```
FOR ALL [INDEXED | HIDDEN] COLUMNS [size_clause]
FOR COLUMNS [size_clause] column|attribute [size_clause] [,column|attribute [size_clause]...]
```
其中`size_clause`子句中各选项含义如下：
- **Integer**：直方图的bucket数量，必须是在1到254之间，**1表示删除**该目标列上的直方图统计信息。
- **REPEAT**：只对已经有直方图统计信息的列收集直方图统计信息。
- **AUTO**：让Oracle自行决定是否收集直方图统计信息、以及使用哪种类型的直方图。
- **SKEWONLY**：只对数据分布不均衡的列收集直方图统计信息。

```sql
--对表emp上所有有索引的列以自动收集方式收集直方图统计信息
> exec dbms_stats.gather_table_stats(ownname=>'SCOTT',tabname=>'EMP',
method_opt=>'for all indexed columns size auto');

--对表emp上的列empno和deptno以自动收集方式收集直方图统计信息
> exec dbms_stats.gather_table_stats(ownname=>'SCOTT',tabname=>'EMP',
method_opt=>'for columns size auto EMPNO DEPTNO');

--对表emp上的列empno和deptno收集直方图统计信息，同时指定桶的数量为10
> exec dbms_stats.gather_table_stats(ownname=>'SCOTT',tabname=>'EMP',
method_opt=>'for columns size 10 EMPNO DEPTNO');

--对表emp上的列empno和deptno收集直方图统计信息，分别指定桶的数量为10和5
> exec dbms_stats.gather_table_stats(ownname=>'SCOTT',tabname=>'EMP',
method_opt=>'for columns EMPNO size 10 DEPTNO size 5');

--只删除表emp上empno列的直方图统计信息
> exec dbms_stats.gather_table_stats(ownname=>'SCOTT',tabname=>'EMP',
method_opt=>'for columns EMPNO size 1');

--删除表emp上所有列的直方图统计信息
> exec dbms_stats.gather_table_stats(ownname=>'SCOTT',tabname=>'EMP',
method_opt=>'for all columns size 1');
```

## 直方图对CBO的影响
### 直方图对游标共享的影响
当某个列上面有了直方图统计信息后，CBO优化器就会认为对该列施加等值查询条件是一个不安全的谓词条件。当我们把常规游标共享参数`CURSOR_SHARING`设置为SIMILAR后，如果目标列上有直方图统计信息，此时Oracle就会对目标列传入的每一个distinct值都产生一个child cursor，这实际上还是相当于硬解析，共享shared cursor、降低系统硬解析数量的目的就没有实现。

### 直方图对可选择率的影响
直方图对CBO优化器估算可选择率、计算成本以及选择执行计划有着直接影响。

数据字典`DBA_TAB_COL_STATISTICS`、`DBA_PART_COL_STATISTICS`和`DBA_SUBPART_COL_STATISTICS`中的字段**DENSITY**实际上可以看作是对目标列施加等值查询后的可选择率，在没有直方图统计信息时，DENSITY的值就等于`1/NUM_DISTINCT`。

#### Frequency类型直方图与可选择率
当目标列上有Frequency类型的直方图时，如果对目标列施加等值查询条件，且查询条件的输入值在目标列的最小值和最大值之间，那么其结果集的cardinality的计算公式如下：

- 如果查询条件的输入值等于目标列的某个Bucket的`ENDPOINT_VALUE`：
```
cardinality = NUM_ROWS * selectivity
selectivity = BucketSize / NUM_ROWS
BucketSize = current_ENDPOINT_NUMBER - previous_ENDPOINT_NUMBER
```

- 如果查询条件的输入值不等于目标列的任意一个Bucket的`ENDPOINT_VALUE`：
```
cardinality = NUM_ROWS * selectivity
selectivity = MIN(BucketSize) / (2 * NUM_ROWS)
BucketSize = current_ENDPOINT_NUMBER - previous_ENDPOINT_NUMBER
```

#### Height Balanced类型直方图与可选择率
当目标列上有Height Balanced类型的直方图时，如果对目标列施加等值查询条件，且查询条件的输入值在目标列的最小值和最大值之间，那么其结果集的cardinality的计算公式如下：

- 如果查询条件的输入值是popular value：
```sql
cardinality = NUM_ROWS * selectivity
selectivity = (Buckets_this_popular_value / Buckets_total) * Null_Adjust
Null_Adjust = (NUM_ROWS - NUM_NULLS) / NUM_ROWS 
--//Buckets_this_popular_value表示此popular value占用的桶数量
--//Buckets_total表示桶的总数
```

- 如果查询条件的输入值不是popular value，且Oracle版本是10.2.0.4及其以上的版本：
```sql
cardinality = NUM_ROWS * selectivity
selectivity = NewDensity * Null_Adjust
Null_Adjust = (NUM_ROWS - NUM_NULLS) / NUM_ROWS 
NewDensity = (Buckets_total - Buckets_all_popular_values) / Buckets_total / (NDV - popular_values_count)
NDV = NUM_DISTINCT
--//Buckets_all_popular_values表示的是所有popular values占用的桶数量
--//popular_values_count表示的是popular value的个数
```

## 使用直方图的注意事项

1. 直方图是专门为了准确评估分布不均匀的目标列的可选择率、结果集cardinality而被引入的。如果目标列的数据是均匀分布的，比如**主键列、有唯一索引的列**，则根本不需要对这些列收集直方图统计信息。

2. 对于那些从来**没有在SQL语句的where条件中出现过的列**，不管其数据分布是否均匀，都无需对其收集直方图统计信息。

3. 直方图统计信息可能会影响到shared cursor的共享，特别是在与有绑定变量的SQL联合使用时。必要时可以删除目标列上的直方图统计信息。

4. 在配置Oracle 10g引入的自动统计信息收集作业时，需要特别注意对直方图统计信息的收集策略。


