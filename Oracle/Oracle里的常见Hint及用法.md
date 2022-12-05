---
tags: [oracle, SQL优化]
title: Oracle里的常见Hint及用法
created: '2022-10-29T12:39:28.567Z'
modified: '2022-12-05T08:40:04.053Z'
---

Oracle里的常见Hint及用法

Oracle数据库各个版本中的Hint都不尽相同，这里描述的是Oracle 11gR2中的常见Hint。

# 与优化器模式相关的Hint

1. `ALL_ROWS`

针对整个目标SQL的Hint，它的含义是让优化器启用CBO，而且在得到目标SQL的执行计划时会选择**吞吐量**最佳（**I/O、CPU**等硬件资源消耗最小）的执行路径。

从Oracle 10g开始，默认情况下启用的优化器就是CBO。
```sql
SQL> show parameter optimizer_mode;
```

2. `FIRST_ROWS(n)`

针对整个目标SQL的Hint，它的含义是让优化器启用CBO，而且在得到目标SQL的执行计划时会选择能**以最快的响应时间返回前n条记录**的执行路径。

`FIRST_ROWS(n)` Hint和优化器模式`FIRST_ROWS_n`不是一一对应的。优化器模式`FIRST_ROWS_n`中的n只能是**1、10、100和1000**。

如果在**UPDATE、DELETE**或者包含以下内容的查询语句中使用了`FIRST_ROWS(n)` Hint，则该`FIRST_ROWS(n)` Hint会被Oracle忽略。

- **集合运算**，如UNION、INTERSECT、MINUS、UNION ALL等；
- **GROUP BY**；
- **FOR UPDATE**；
- **聚合函数**，比如SUM等；
- **DISTINCT**；
- **ORDER BY**（对应的排序列上没有索引）。

这些情况下`FIRST_ROWS(n)` Hint会被忽略是因为对于上述类型的SQL而言，Oracle必须访问所有的行记录后才能返回满足条件的前n行记录。也即是说，在这些情况下使用`FIRST_ROWS(n)`是没有意义的。

3. `RULE`

针对整个目标SQL的Hint，它的含义是对目标SQL启用RBO（基于规则的优化器）。

RULE不能与除`DRIVING_SITE`以外的其他Hint联合使用，否则可能会导致其他Hint失效。而且当RULE与`DRIVING_SITE`联用时，它自身可能会失效。因此，RULE最好单独使用。

因为很多执行路径RBO根本就不支持，因此不推荐使用RULE Hint。

# 与表访问相关的Hint

1. `FULL`

针对单个目标表的Hint，其含义是让优化器对目标表执行**全表扫描**。

```sql
--不能带上目标表的schema名称；如果有别名，应该使用表的别名
/*+ full(目标表) */
```

2. `ROWID`

针对单个目标表的Hint，其含义是让优化器对目标表执行**ROWID扫描**。只有目标SQL中使用了**包含ROWID的where条件**时，ROWID Hint才有意义。

```sql
/*+ rowid(目标表) */
```


# 与索引访问相关的Hint

1. `INDEX`

针对单个目标表的Hint，其含义是让优化器对目标表上的目标索引执行索引扫描操作。INDEX Hint中的目标索引几乎可以是Oracle数据库中的所有类型的索引，包括B树索引、位图索引、函数索引等。

```sql
--格式1
/*+ INDEX(目标表 目标索引) */
--格式2
/*+ INDEX(目标表 目标索引1 目标索引2 ... 目标索引n) */
--格式3
/*+ INDEX(目标表 (目标索引1的索引列) (目标索引2的索引列) ... (目标索引n的索引列)) */
--格式4
/*+ INDEX(目标表) */
```

格式3中，是用指定目标索引的索引列名来代替对应的目标索引名。如果目标索引是复合索引，则在用于指定索引列名的括号内也可以指定该目标索引的多个索引列，各个索引列之间用**空格**分隔就行。

格式4中，表示指定了目标表上所有已存在的索引。此时优化器只会考虑对目标表上所有的索引执行索引扫描操作，而不会去考虑全表扫描。

格式2、3和4中，优化器在考虑目标表上的索引时，可能是分别计算出单独扫描各个索引列的成本后，再选择其中成本值最低的索引；也可能是先分别扫描这些索引中的两个或多个索引，然后再对扫描结果执行合并操作（前提条件是优化器计算出来这样做的成本值是最低的）。

2. `NO_INDEX`

是`INDEX`的反义Hint，其含义是让优化器**不**对目标表上的目标索引执行索引扫描操作。

```sql
--格式1
/*+ NO_INDEX(目标表 目标索引) */
--格式2
/*+ NO_INDEX(目标表 目标索引1 目标索引2 ... 目标索引n) */
--格式3
/*+ NO_INDEX(目标表) */
```

格式3表示指定了目标表上的所有索引，此时优化器不会考虑对目标表上所有已存在的索引执行索引扫描操作，相当于对目标表指定了全表扫描。

3. `INDEX_DESC`

针对单个目标表的Hint，其含义是让优化器对目标表上的目标索引执行**索引降序扫描**操作。如果目标索引是升序的，则`INDEX_DESC` Hint会使Oracle以降序的方式扫描该索引；如果目标索引是降序的，则`INDEX_DESC` Hint会使Oracle以升序的方式扫描该索引。

```sql
--格式1
/*+ INDEX_DESC(目标表 目标索引) */
--格式2
/*+ INDEX_DESC(目标表 目标索引1 目标索引2 ... 目标索引n) */
--格式3
/*+ INDEX_DESC(目标表) */
```
这三种格式的含义与INDEX Hint中对应格式的含义相同。

4. `INDEX_COMBINE`

针对单个目标表的Hint，其含义是让优化器对目标表上的多个目标索引执行**位图布尔运算**。Oracle数据库里有一个映射函数，可以实现B树索引中的ROWID和对应位图索引中的位图之间的互相转换（对应执行计划中的关键字为`BITMAP CONVERSION FROM/TO ROWIDS`），所以`INDEX_COMBINE`并不局限于位图索引，它的作用对象也可以是B树索引。

```sql
--格式1
/*+ INDEX_COMBINE(目标表 目标索引1 目标索引2 ... 目标索引n) */
--格式2
/*+ INDEX_COMBINE(目标表) */
```

格式1指定了目标表上的n个目标索引，此时优化器会考虑对这n个目标索引中的两个或者多个执行位图布尔运算。

格式2指定了目标表上的所有索引，此时优化器会考虑对目标表上所有存在的索引中的两个或者多个执行位图布尔运算。


5. `INDEX_FFS`

针对单个目标表的Hint，其含义是让优化器对目标表上的目标索引执行**索引快速全扫描**操作。索引快速全扫描能成立的前提条件是：SELECT语句中**所有的查询列都存在于目标索引中**，即通过扫描目标索引就可以得到所有的查询列，而不用回表。

```sql
--格式1
/*+ INDEX_FFS(目标表 目标索引) */
--格式2
/*+ INDEX_FFS(目标表 目标索引1 目标索引2 ... 目标索引n) */
--格式3
/*+ INDEX_FFS(目标表) */
```
这三种格式的含义与INDEX Hint中对应格式的含义相同。

6. `INDEX_JOIN`

针对单个目标表的Hint，其含义是让优化器对目标表上的多个目标索引执行INDEX JOIN操作。INDEX JOIN能够成立的前提条件是：SELECT语句中**所有的查询列都存在于目标表上的多个目标索引中**，即通过扫描这些索引就可以得到所有的查询列，而不用回表。

```sql
--格式1
/*+ INDEX_JOIN(目标表 目标索引1 目标索引2 ... 目标索引n) */
--格式2
/*+ INDEX_JOIN(目标表) */
```
这几种格式的含义与INDEX_COMBINE Hint中对应格式的含义相同。

7. `AND_EQUAL`

针对单个目标表的Hint，其含义是让优化器对目标表上的多个目标索引执行**INDEX MERGE**操作。INDEX MERGE能够成立的前提条件是：目标SQL的**where条件中出现了多个针对不同单列的等值条件，并且这些列上都有单键值的索引**。另外，Oracle数据库中能够做INDEX MERGE的索引数量的最大值是**5**。

```sql
/*+ AND_EQUAL(目标表 目标索引1 目标索引2 ... 目标索引n) */
```
其格式的含义与INDEX_COMBINE Hint中对应格式的含义相同。

# 与表连接顺序相关的Hint

1. `ORDERED`

针对多个目标表的Hint，它的含义是让优化器对多个目标表执行表连接操作时，按照它们在目标SQL的**where条件中出现的顺序从左到右依次进行连接**。

```sql
/*+ ordered */
```

2. `LEADING`

针对多个目标表的Hint，它的含义是让优化器将指定的多个表的连接结果作为目标SQL表连接过程中的**驱动结果集**，并且将LEADING Hint中从左至右出现的第一个目标表作为整个表连接过程中的**首个驱动表**。

```sql
/*+ LEADING(目标表1 目标表2 ... 目标表n) */
```

LEADING比ORDERED要更加温和一些，因为它只是指定了首个驱动表和驱动结果集，但并没有像ORDERED一样完全指定了表连接顺序。


# 与表连接方法相关的Hint
以下是针对多个目标表的一些Hint。

1. `USE_MERGE`

针对多个目标表的Hint，它的含义是让优化器将指定的多个表作为**被驱动表**与其他表或结果集做**排序合并连接**。

在USE_MERGE Hint中指定的目标表应该是排序合并连接中的被驱动表，如果指定的表并不能作为排序合并连接的被驱动表，则Oracle要么会忽略USE_MERGE Hint，要么会忽略指定的表。

```sql
/*+ use_merge(目标表1 目标表2 ... 目标表n) */
/*+ use_merge(目标表1,目标表2,...,标表n) */
```

在Oracle里Hint的基础知识里提到过，**组合Hint中各个Hint之间不要使用逗号来彼此分隔**（应该使用空格）。但是对单个Hint而言，它所指定的多个目标表之间既可以使用空格来分隔，也可以使用逗号来分隔。

由于Oracle可能会忽略USE_MERGE Hint或该Hint中指定的被驱动表，所以我们通常会用LEADING Hint（或ORDERED Hint）配合USE_MERGE Hint一起使用。

2. `NO_USE_MERGE`

针对多个目标表的Hint，USE_MERGE的反义Hint，它的含义是不让优化器将指定的多个表作为被驱动表与其他表或结果集做排序合并连接。

在`NO_USE_MERGE` Hint中指定的目标表应该是原先在排序合并连接中的被驱动表，否则Oracle要么会忽略`NO_USE_MERGE` Hint，要么会忽略指定的表。正是由于Oracle可能会忽略`NO_USE_MERGE` Hint或该Hint中指定的被驱动表，所以我们通常会用LEADING Hint（或ORDERED Hint）配合`NO_USE_MERGE` Hint一起使用。

```sql
/*+ no_use_merge(目标表1 目标表2 ... 目标表n) */
/*+ no_use_merge(目标表1,目标表2,...,标表n) */
```


3. `USE_NL`

针对多个目标表的Hint，它的含义是让优化器将指定的多个表作为**被驱动表**与其他表或者结果集做**嵌套循环连接**。

在`USE_NL` Hint中指定的目标表应该是嵌套循环连接中的被驱动表，否则Oracle要么会忽略`USE_NL` Hint，要么会忽略指定的表。正是由于Oracle可能会忽略`USE_NL` Hint或该Hint中指定的被驱动表，所以我们通常会用LEADING Hint（或ORDERED Hint）配合`USE_NL` Hint一起使用。

```sql
/*+ use_nl(目标表1 目标表2 ... 目标表n) */
/*+ use_nl(目标表1,目标表2,...,标表n) */
```

4. `NO_USE_NL`

针对多个目标表的Hint，USE_NL的反义Hint，它的含义是**不**让优化器将指定的多个表作为**被驱动表**与其他表或者结果集做**嵌套循环连接**。

在`NO_USE_NL` Hint中指定的目标表应该是原先在嵌套循环连接中的被驱动表，否则Oracle要么会忽略`NO_USE_NL` Hint，要么会忽略指定的表。正是由于Oracle可能会忽略`NO_USE_NL` Hint或该Hint中指定的被驱动表，所以我们通常会用LEADING Hint（或ORDERED Hint）配合`NO_USE_NL` Hint一起使用。

```sql
/*+ no_use_nl(目标表1 目标表2 ... 目标表n) */
/*+ no_use_nl(目标表1,目标表2,...,标表n) */
```

5. `USE_HASH`

针对多个目标表的Hint，它的含义是让优化器将指定的多个表作为**被驱动表**与其他表或结果集做哈希连接。

在`USE_HASH` Hint中指定的目标表应该是哈希连接中的被驱动表，否则Oracle要么会忽略`USE_HASH` Hint，要么会忽略指定的表。正是由于Oracle可能会忽略`USE_HASH` Hint或该Hint中指定的被驱动表，所以我们通常会用LEADING Hint（或ORDERED Hint）配合`USE_HASH` Hint一起使用。

```sql
/*+ use_hash(目标表1 目标表2 ... 目标表n) */
/*+ use_hash(目标表1,目标表2,...,标表n) */
```

6. `NO_USE_HASH`

针对多个目标表的Hint，是`USE_HASH`的反义Hint，它的含义是**不**让优化器将指定的多个表作为**被驱动表**与其他表或结果集做哈希连接。

在`NO_USE_HASH` Hint中指定的目标表应该是原先在哈希连接中的被驱动表，否则Oracle要么会忽略`NO_USE_HASH` Hint，要么会忽略指定的表。正是由于Oracle可能会忽略`NO_USE_HASH` Hint或该Hint中指定的被驱动表，所以我们通常会用LEADING Hint（或ORDERED Hint）配合`NO_USE_HASH` Hint一起使用。

```sql
/*+ no_use_hash(目标表1 目标表2 ... 目标表n) */
/*+ no_use_hash(目标表1,目标表2,...,标表n) */
```

****
以下是针对子查询的一些Hint。

7. `MERGE_AJ`

针对**子查询**的Hint，它的含义是让优化器对相关目标表执行**排序合并反连接**。

```sql
/*+ merge_aj */
```

也可以将`QB_NAME` Hint与`MERGE_AJ` Hint联合使用，这样就可以将`MERGE_AJ` Hint从子查询中挪出来。例如以下两个SQL就是等价的：
```sql
SQL> select * from emp
where deptno not in (select /*+ merge_aj */deptno
from dept where loc='DETROIT');

SQL> select /*+ merge_aj(@qb) */* from emp
where deptno not in (select /*+ qb_name(qb) */deptno
from dept where loc='DETROIT');
```

8. `NL_AJ`

针对**子查询**的Hint，它的含义是让优化器对相关目标表执行**嵌套循环反连接**。

```sql
/*+ nl_aj */
```

9. `HASH_AJ`

针对**子查询**的Hint，它的含义是让优化器对相关目标表执行**哈希反连接**。

```sql
/*+ hash_aj */
```

10. `MERGE_SJ`

针对**子查询**的Hint，它的含义是让优化器对相关目标表执行**排序合并半连接**。

```sql
/*+ merge_sj */
```

11. `NL_SJ`

针对**子查询**的Hint，它的含义是让优化器对相关目标表执行**嵌套循环半连接**。

```sql
/*+ nl_sj */
```

12. `HASH_SJ`

针对**子查询**的Hint，它的含义是让优化器对相关目标表执行**哈希半连接**。

```sql
/*+ hash_sj */
```

# 与查询转换相关的Hint

1. `USE_CONCAT`

针对整个目标SQL的Hint，其含义是让优化器对目标SQL使用**IN-List扩展**或者**OR展开**。

```sql
/*+ use_concat */
```

2. `NO_EXPAND`

针对整个目标SQL的Hint，它是`USE_CONCAT`的反义Hint，其含义是不让优化器对目标SQL使用**IN-List扩展**或者**OR展开**。

```sql
/*+ no_expand */
```

3. `MERGE`

针对单个目标视图的Hint，其含义是让优化器对目标视图执行**视图合并**。

```sql
/*+ merge(目标视图) */
```

如果目标视图是一个**内嵌视图**，则MERGE Hint也可以出现在其视图定义语句所在的Query Block中，此时MERGE Hint中就不应该再带上该内嵌视图的名称，其使用格式应该变为：
```sql
/*+ merge */
```

示例：
```sql
SQL> select /*+ merge(dept_view) */empno, ename, dname
from emp, dept_view
where emp.deptno = dept_view.deptno;

SQL> select empno, ename, dname
from emp,
(select /*+ merge */ from dept
where loc='DETROIT') dept_view_inline
where emp.deptno = dept_view_inline.deptno;
```

4. `NO_MERGE`

针对单个目标视图的Hint，它是`USE_MERGE`的反义Hint，其含义是不让优化器对目标视图执行**视图合并**。

```sql
/*+ no_merge(目标视图) */
```

如果目标视图是一个内嵌视图，则NO_MERGE Hint也可以出现在其视图定义语句所在的Query Block中，此时NO_MERGE Hint中就不应该再带上该内嵌视图的名称，其使用格式应该变为：
```sql
/*+ no_merge */
```

5. `UNNEST`

针对子查询的Hint，其含义是让优化器对目标SQL中的子查询执行**子查询展开**。

```sql
/*+ unnest */
```

也可以将`QB_NAME` Hint与`UNNEST` Hint联合使用，这样就可以将`UNNEST` Hint从子查询中挪出来。具体可以参考前面`MERGE_AJ` Hint与`QB_NAME` Hint联用的示例。

6. `NO_UNNEST`

针对子查询的Hint，它是`UNNEST`的反义Hint，其含义是不让优化器对目标SQL中的子查询执行**子查询展开**。

```sql
/*+ no_unnest */
```

也可以将`QB_NAME` Hint与`NO_UNNEST` Hint联合使用，这样就可以将`NO_UNNEST` Hint从子查询中挪出来。具体可以参考前面`MERGE_AJ` Hint与`QB_NAME` Hint联用的示例。

7. `EXPAND_TABLE`

针对单个目标表的Hint，其含义是让优化器在**不考虑成本**的情况下，对目标SQL中的目标表执行**表扩展**。

```sql
/*+ expand_table(目标表) */
```

8. `NO_EXPAND_TABLE`

针对单个目标表的Hint，它是`EXPAND_TABLE`的反义Hint，其含义是不让优化器对目标SQL中的目标表执行**表扩展**。

```sql
/*+ no_expand_table(目标表) */
```


# 与并行相关的Hint

1. `PARALLEL`

在Oracle 11gR2之前，PARALLEL是针对单个目标表的Hint，其含义是让优化器以指定的或者系统计算出来的并行度去并行访问目标表。

从11gR2开始，Oracle引入了自动并行后，PARALLEL Hint的用法和作用范围也发生了变化。Oracle 11gR2中的PRALLEL Hint是针对整个目标SQL的Hint，其含义是让优化器**以指定的或者系统计算出来的并行度**，去并行执行目标SQL的执行计划中**所有可以被并行执行的执行步骤**。

新的针对整个目标SQL的PARALLEL Hint格式有以下四种：
```sql
--格式1：目标SQL总是会以并行方式执行，Oracle会计算出一个并行度（大于或等于2）
/*+ PARALLEL */

--格式2：Oracle会计算出一个并行度（大于或等于1，等于1时会以串行方式执行）
/*+ PARALLEL(AUTO) */

--格式3：目标SQL能否并行执行取决于相关对象（目标表、目标索引）的并行度设置
/*+ PARALLEL(MANUAL) */

--格式4：目标SQL总是会以指定的并行度去并行执行
/*+ PARALLEL(指定的并行度) */
```

旧的针对单个表的PARALLEL Hint依然可以使用，但是其优先级会比新的针对整个目标SQL的PARALLEL Hint低。其格式有如下两种：
```sql
--格式1：目标SQL总是会以指定的并行度去并行访问目标表
/*+ PARALLEL(目标表 指定的并行度) */
/*+ PARALLEL(目标表,指定的并行度) */

--格式2：目标SQL总是会以根据相关系统参数计算出来的默认并行度去并行访问目标表
/*+ PARALLEL(目标表 DEFAULT) */
/*+ PARALLEL(目标表,DEFAULT) */
```

在Oracle 11gR2中，并行Hint也可以用于临时表。

2. `NO_PARALLEL`

是PARALLEL的反义Hint。从11gR2开始，`NO_PARALLEL` Hint的用法和作用范围也发生了变化。Oracle 11gR2中的`NO_PRALLEL` Hint是针对整个目标SQL的Hint，其含义是**不**让优化器去并行执行目标SQL的执行计划中**所有可以被并行执行的执行步骤**。

新的针对整个目标SQL的`NO_PARALLEL` Hint格式为：
```sql
/*+ NO_PARALLEL */
```

旧的针对单个表的`NO_PARALLEL` Hint依然可以使用，其用法格式为：
```sql
/*+ NO_PARALLEL(目标表) */
```

3. `PARALLEL_INDEX`

针对单个目标表的Hint，其含义是让优化器以指定的或者系统计算出来的并行度去对目标表上的目标**分区索引**执行**并行索引扫描**操作。

```sql
--格式1：目标SQL总是会以指定的并行度去访问目标表上的目标分区索引
/*+ PARALLEL_INDEX(目标表 目标分区索引 指定的并行度) */
/*+ PARALLEL_INDEX(目标表,目标分区索引,指定的并行度) */

--格式2：目标SQL总是会以根据相关系统参数计算出来的默认并行度（可能会做一定调整）去访问目标表上的目标分区索引
/*+ PARALLEL_INDEX(目标表 目标分区索引 DEFAULT) */
/*+ PARALLEL_INDEX(目标表,目标分区索引,DEFAULT) */

--格式3：指定多个目标索引，并分别指定它们的并行度
/*+ PARALLEL_INDEX(目标表 目标分区索引1 目标分区索引2 ... 目标分区索引n 目标分区索引1的并行度 目标分区索引2的并行度 ... 目标分区索引n的并行度) */
/*+ PARALLEL_INDEX(目标表,目标分区索引1,目标分区索引2,...,目标分区索引n,目标分区索引1的并行度,目标分区索引2的并行度,...,目标分区索引n的并行度) */

--格式4：指定多个目标索引，并统一采用系统计算出来的默认并行度
/*+ PARALLEL_INDEX(目标表 目标分区索引1 目标分区索引2 ... 目标分区索引n DEFAULT DEFAULT ... DEFAULT) */
/*+ PARALLEL_INDEX(目标表,目标分区索引1,目标分区索引2,...,目标分区索引n,DEFAULT,DEFAULT,...,DEFAULT) */

--格式5：同时指定目标表上所有已存在的索引，Oracle会分别计算各个索引做并行扫描的成本，并从中选择成本值最低的一个目标索引
/*+ PARALLEL_INDEX(目标表) */
```

4. `NO_PARALLEL_INDEX`

针对单个目标表的Hint，`PARALLEL_INDEX`的反义Hint，其含义是**不**让优化器对指定的目标表上的目标分区索引执行并行索引扫描操作。

```sql
--格式1
/*+ NO_PARALLEL_INDEX(目标表 目标分区索引) */
/*+ NO_PARALLEL_INDEX(目标表,目标分区索引) */

--格式2
/*+ NO_PARALLEL_INDEX(目标表 目标分区索引1 目标分区索引2 ... 目标分区索引n) */
/*+ NO_PARALLEL_INDEX(目标表,目标分区索引1,目标分区索引2,...,目标分区索引n) */

--格式3
/*+ NO_PARALLEL_INDEX(目标表) */
```


# 其他常见Hint

1. `DRIVING_SITE`

针对单个目标表的Hint，其含义是让优化器在指定的目标表所在的节点上执行目标SQL。

`DRIVING_SITE` Hint适用于带dblink的分布式查询语句。对于这种带dblink的目标SQL，Oracle要么选择在本地执行，要么选择在dblink所在的远程节点执行，并将执行结果通过网络传回本地。执行地点的选择完全是由优化器根据一定条件来决定的（比如计算出来的成本值）。`DRIVING_SITE` Hint既适用于CBO，也适用于RBO。

```sql
/*+ driving_site(目标表) */
```

`DRIVING_SITE` Hint选取执行节点的一个很重要的原则就是：要想办法减少本地节点和远程节点之间通过网络传输的数据量。最后，`DRIVING_SITE` Hint**只适用于带dblink的分布式查询语句，不**能用于分布式DML或者DDL语句。

2. `APPEND`

针对整个目标SQL的Hint，其含义是让优化器在执行**带子查询的INSERT语句**时绕开Buffer Cache，使用**直接路径插入**（Direct Path Insert）。

带子查询的INSERT语句一般形如：
```sql
SQL> insert into table_name select * from xxx;
```

默认情况下，Oracle在执行INSERT语句时使用的是常规插入，相关的数据会缓存在Buffer Cache中。对于直接路径插入，Oracle在插入数据时会直接在目标表的高水位线（HWM）以上插入数据（最后再升高高HWM），而不是向常规插入那样先去目标表的HWM以下寻找是否存在能插入数据的数据块，这样就节省了寻找合适的数据块的时间。另外由于直接路径插入不需要在Buffer Cache中缓存相关数据块，所以也省去了缓存数据块的开销。最后，在非归档模式、或者在归档模式但表是NOLOGGING时，使用APPEND Hint在目标表上插入数据时不会产生redo（对表的数据块的数据部分的更改不会有redo产生），这样就省去了产生redo的开销。基于这三方面的原因，使用APPEND Hint的直接路径插入通常会比常规插入的速度要快很多。

```sql
/*+ append */
```

当我们使用了有效的APPEND Hint以直接路径插入对目标表INSERT数据时，Oracle会在目标表上加`LMODE=6`的排他锁（TM Enqueue），这意味着在我们插入数据时其他人无法对目标表做任何DML操作。因此，能否对目标表做直接路径插入需要结合业务场景和系统特点来考虑，否则可能会导致严重的并发问题。。

3. `APPEND_VALUES`

针对整个目标SQL的Hint，其含义是让优化器在执行**带VALUES子句的INSERT语句**时绕开Buffer Cache，使用**直接路径插入**。

```sql
/*+ append_values */
```

`APPEND_VALUES` Hint应该用于一次插入一批数据的场景（比如PL/SQL的FORALL子句或者OCI的批量绑定），不要将其应用于单个的INSERT语句，否则可能会导致严重的**存储空间浪费**。

和APPEND Hint一样，使用`APPEND_VALUES` Hint以直接路径插入对目标表INSERT数据时，Oracle会在目标表上加`LMODE=6`的排他锁（TM Enqueue）。

4. `PUSH_PRED`

针对单个目标视图的Hint，其含义是让优化器对目标视图执行**连接谓词推入**，即将目标SQL中处于该视图定义语句外部的谓词连接条件推入到该视图定义语句内部。

```sql
/*+ push_pred(目标视图) */
```

如果目标视图是一个内嵌视图，那么`PUSH_PRED` Hint也可以出现在其视图定义语句所在的Query Block中，此时`PUSH_PRED` Hint中就不应该再带上该内嵌式图的名称。
```sql
/*+ push_pred */
```

5. `NO_PUSH_PRED`

针对单个目标视图的Hint，`PUSH_PRED`的反义Hint，其含义是不让优化器对目标视图执行连接谓词推入。

```sql
/*+ no_push_pred(目标视图) */
```

如果目标视图是一个内嵌视图，那么`NO_PUSH_PRED` Hint也可以出现在其视图定义语句所在的Query Block中，此时`NO_PUSH_PRED` Hint中就不应该再带上该内嵌式图的名称。
```sql
/*+ no_push_pred */
```

6. `PUSH_SUBQ`

针对子查询的Hint，其含义是让优化器**尽可能早地执行目标SQL中不能做子查询展开的子查询**。

通常情况下，目标SQL中不能做子查询展开的子查询总是在其执行计划的最后一步才被执行，但如果执行这个子查询后能够显著减少返回的结果集的数量，则先执行这个子查询就有可能提高该SQL的执行效率。

```sql
/*+ push_subq */
```
我们也可以将`PUSH_SUBQ` Hint与`QB_NAME` Hint联合使用，这样就可以将`PUSH_SUBQ` Hint从子查询中挪出来。

7. `NO_PUSH_SUBQ`

针对子查询的Hint，是`PUSH_SUBQ`的反义Hint，其含义是让优化器最后执行目标SQL中不能做子查询展开的子查询。

```sql
/*+ no_push_subq */
```
我们也可以将`NO_PUSH_SUBQ` Hint与`QB_NAME` Hint联合使用，这样就可以将`NO_PUSH_SUBQ` Hint从子查询中挪出来。

8. `OPT_PARAM`

针对整个目标SQL的Hint，可以用来修改针对目标SQL的与优化器相关的一些参数。`OPT_PARAM` Hint的好处在于它所修改的参数仅**对它所在的目标SQL有效**，是比在系统级和Session级修改相关参数更细的一个粒度。

`OPT_PARAM` Hint能够修改的与优化器相关的参数有：
- `OPTIMIZER_DYNAMIC_SAMPLING`
- `OPTIMIZER_INDEX_CACHING`
- `OPTIMIZER_INDEX_COST_ADJ`
- `OPTIMIZER_USE_PENDING_STATISTICS`
- `STAR_TRANSFORMATION_ENABLED`
- `PARALLEL_DEGREE_POLICY`
- `PARALLEL_DEGREE_LIMIT`
- 一些与修改器相关的隐含参数，比如`_HASH_JOIN_ENABLED`、`_B_TREE_BITMAP_PLANS`、`_COMPLEX_VIEW_MERGING`等。

`OPT_PARAM` Hint的使用格式为：
```sql
/*+ opt_param(参数名称 参数值) */
```

9. `OPTIMIZER_FEATURES_ENABLE`

针对整个目标SQL的Hint，可以用来修改针对目标SQL的**优化器版本**。该Hint相当于针对目标SQL修改参数`OPTIMIZER_FEATURES_ENABLE`的值，其好处是它只对所在的目标SQL有效，这是比在系统级和Session级修改`OPTIMIZER_FEATURES_ENABLE`参数更细的一个粒度。

```sql
/*+ optimizer_features_enable('优化器的版本号') */
```

我们可以使用该Hint将解析目标SQL的优化器回退到之前的版本，这样就可以保证该SQL执行计划的稳定性，从而规避由于数据库升级后新版本优化器的一些新特性、新功能所可能带来的对目标SQL执行计划的不利影响。

10. `QB_NAME`

针对整个目标SQL的Hint，可以用来对一个Query Block指定自定义的名称，并且指定的名称在整个目标SQL内都是可用的。所以`QB_NAME` Hint可以将一些本应该在子查询或者内嵌视图中使用的Hint挪到外部使用。

```sql
/*+ qb_name(自定义的名称) */
```

如果在目标SQL中使用`QB_NAME` Hint针对不同的Query Block指定了相同的名称，或者在目标SQL中对同一个Query Block自定义了不同的名称，那么Oracle就会**忽略**这些自相矛盾的`QB_NAME` Hint。

11. `CARDINALITY`

针对单个目标表的Hint，可以用来设置对目标表执行扫描操作后返回结果集的Cardinality的值，它所能够设置的扫描类型包括对目标表的**全表扫描、索引范围扫描、索引全扫描和索引快速全扫描**。

```sql
/*+ cardinality(目标表 扫描结果集的势) */
```

需要注意的是，**CARDINALITY Hint对于索引唯一扫描而言是无效的**。

12. `SWAP_JOIN_INPUTS`

针对哈希连接的Hint，其含义是让优化器**交换原始哈希连接的驱动表和被驱动表的顺序**，即在依然走哈希连接的情况下让原来哈希连接的驱动表变为被驱动表，让原来的被驱动表变为驱动表。

```sql
/*+ swap_join_inputs(原被驱动表) */
```

在`SWAP_JOIN_INPUTS` Hint中指定的目标表应该是原哈希连接中的**被驱动表**，否则Oracle会忽略该Hint。



