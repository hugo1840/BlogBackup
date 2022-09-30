---
tags: [oracle, SQL优化]
title: Oracle数据库中的优化器
created: '2022-09-28T12:18:53.629Z'
modified: '2022-09-30T03:45:26.685Z'
---

Oracle数据库中的优化器

优化器（Optimizer）存在的目的是为了获取目标SQL在当前情形下最高效的执行路径。Oracle数据库中的优化器可以分为基于规则的优化器（Rule-based Optimizer, **RBO**）、以及基于成本的优化器（Cost-based Optimizer, **CBO**）。

# 基于规则的优化器

## RBO存在的缺陷
从Oracle 10g开始，Oracle已不再支持RBO。因为和CBO相比，RBO存在较为明显的缺陷。

- 通过硬编码在数据库代码中的一系列固定规则来决定目标SQL的执行计划，没有考虑到目标SQL中所涉及的实际数据量、数据分布情况等；
- 目标SQL的写法，甚至是目标SQL中所涉及的各个对象在该SQL文本中出现的先后顺序，都可能会影响到RBO对执行计划的选择；
- 在使用RBO时，执行计划一旦出了问题，很难对其进行调整；
- Oracle数据库中很多很好的特性、功能，都不被RBO支持。

在使用RBO时，如果RBO选择的执行计划不是最优解，很难人为地对执行计划做出调整。如果我们试图通过使用Hint来调整执行计划，就会自动启用CBO，即Oracle会以CBO的方法来解析包含Hint的目标SQL。

## RBO模式下如何优化SQL
RBO模式下无法使用Hint来调整执行计划。但是我们可以通过以下方法来使执行计划发生变化。

1. 等价改写目标SQL
比如在目标SQL的where条件语句中，**对NUMBER或DATE类型的列加上0；如果是VARCHAR2或者CHAR类型，可以加上一个空字符**，例如`||`、`''`。如此就可以使得原来可以走的索引现在走不了了。

```sql
--创建测试表
> create table emp_test as select * from emp;
> create index idx_mgr_test on emp_test(mgr);
> create index idx_deptno_test on emp_test(deptno);

-- 修改优化器模式为RBO
> alter session set optimizer_mode='RULE';
> set autotrace traceonly explain
> select * from emp_test where mgr>100 and deptno>100;
---------------------------------------------------
...
INDEX RABGE SCAN      IDX_DEPTNO_TEST       
...

--等价改写SQL
> select * from emp_test where mgr>100 and deptno+0>100;
---------------------------------------------------
...
INDEX RABGE SCAN      IDX_MGR_TEST       
...
```

2. 调整对象的先后顺序
RBO模式下，如果出现了**两条或者两条以上优先级相同的执行路径**，RBO则会依据目标SQL中所涉及的相关对象在数据字典缓存中的缓存顺序、以及目标SQL中所涉及的各个对象在SQL文本中出现的先后顺序来综合判断。因此，我们也可以通过**调整相关对象在数据字典缓存中的缓存顺序、以及改变目标SQL文本中的各个对象出现的先后顺序**来影响执行计划的选择。

在上面的例子上，两个索引`IDX_MGR_TEST`和`IDX_DEPTNO_TEST`的优先级是一样的。因此，通过改变两者在数据字典缓存中的顺序，也能影响执行计划的选择。根据索引创建的顺序，数据字典中先缓存了`IDX_MGR_TEST`，然后缓存了`IDX_DEPTNO_TEST`。通过删除并重建索引`IDX_MGR_TEST`，就能颠倒两者的缓存顺序。

```sql
--重建索引
> drop index idx_mgr_test;
> create index idx_mgr_test on emp_test(mgr);

> select * from emp_test where mgr>100 and deptno>100;
---------------------------------------------------
...
INDEX RABGE SCAN      IDX_MGR_TEST       
...
```
可以发现，在没有改写SQL的情况下，这次RBO优化器选择了走不同的索引。

在涉及到**多表连接**的场景中，如果目标SQL中出现了两条或两条以上优先级相同的执行路径，RBO优化器会按照**从右到左**的顺序来决定谁是驱动表、谁是被驱动表，进而据此来选择执行计划。

```sql
--创建第二张测试表
> create table emp_test1 as select * from emp;

> select t0.mgr, t1.deptno
from emp_test t0, emp_test1 t1
where t0.empno=t1.empno;
---------------------------------------------------
...
0 | SELECT STATEMENT     |           |
1 |   MERGE JOIN         |           |
2 |    SORT JOIN         |           |
3 |     TABLE ACCESS FULL| EMP_TEST1 |
4 |    SORT JOIN         |           |
5 |     TABLE ACCESS FULL| EMP_TEST  |
...
```

假设我们改变上面例子中FROM子句中两张表的顺序：
```sql
> select t0.mgr, t1.deptno
from emp_test1 t1, emp_test t0 
where t0.empno=t1.empno;
---------------------------------------------------
...
0 | SELECT STATEMENT     |           |
1 |   MERGE JOIN         |           |
2 |    SORT JOIN         |           |
3 |     TABLE ACCESS FULL| EMP_TEST |
4 |    SORT JOIN         |           |
5 |     TABLE ACCESS FULL| EMP_TEST1  |
...
```

比较发现，驱动表从EMP_TEST1变成了EMP_TEST。最后，通过调整对象在SQL文本中出现的先后顺序来改变执行计划，只有在两条执行路径的优先级相等的情况下才会起作用。

## 强制使用CBO的情形
虽然在Oracle 11g中，仍然可以通过修改优化器模式、或者使用Rule Hint来使用RBO，但是在遇到以下情形时，Oracle数据库会强制使用CBO:

- 目标SQL中涉及的对象有IOT（Index organized table）；
- 目标SQL中涉及的对象有分区表；
- 使用了并行查询或者并行DML；
- 使用了星型连接；
- 使用了哈希连接；
- 使用了索引快速全扫描；
- 使用了函数索引；
- 等等。


# 基于成本的优化器
CBO优化器会从目标SQL所有可能的执行路径中选择一条成本最小的。该成本是根据目标SQL语句所涉及的表、索引、列等相关对象的**统计信息**计算出来的，可以看作是对目标SQL执行路径的I/O、CPU和网络资源的消耗量的一个估算值。

CBO中有cardinality、selectivity和transitivity三个比较基本的概念。

## Cardinality：集的势
Cardinality是指特定集合所包含的记录的数目，其实就是结果集的行数。Cardinality表示对目标SQL某个具体执行步骤的执行结果所包含记录数的估算。某个执行步骤对应的cardinality值越大，它所对应的成本也就越大。

## Selectivity：可选择率
可选择率是指添加特定谓词条件后，返回结果集的记录数占未添加任何谓词条件的原始结果集记录数的比率。

可选择率的取值范围在0到1之间，它的值越小，表明可选择性越好。可选择率的值越大，意味着返回结果集的cardinality越大，即成本的估算值也就越大。

假设$Card_{computed}$表示添加谓词谓词条件后返回结果集的记录数，$Card_{origin}$表示未添加任何谓词条件的原始结果集记录数，CBO用来估算cardinality的公式可以简单地写成下面的形式：
$$Card_{computed} = Card_{origin} * selectivity$$

如果是对目标列做等值查询，目标列上没有直方图且没有NULL值，可选择率可以按如下方式计算：
$$selectivity = \frac{1}{目标列的distinct值的数目}$$

>**谓词条件**
Oracle数据库中，谓词条件（**Predicate**）是WHERE语句后使用的返回值为True或False的特殊函数，包括LIKE、BETWEEN、IS NULL、IS NOT NULL、IN、EXISTS等。


## Transitivity：可传递性
Oracle中的优化器在使用RBO或者CBO对SQL进行优化之前，会先对目标SQL进行**查询转换**。在查询转换的过程中，CBO优化器会利用可传递性对目标SQL进行简单的等价改写，在原目标SQL中加上根据该SQL现有的谓词条件推算出来的新的谓词条件。其目的是提供更多执行路径给CBO选择，以提高得到更高执行效率的执行计划的可能性。

1. 简单谓词传递
```sql
--改写前
where t1.c1=t2.c1 and t1.c1=10;
--改写后
where t1.c1=t2.c1 and t1.c1=10 and t2.c1=10;
```

2. 连接谓词传递
```sql
--改写前
where t1.c1=t2.c1 and t2.c1=t3.c1;
--改写后
where t1.c1=t2.c1 and t2.c1=t3.c1 and t1.c1=t3.c1;
```

3. 外连接谓词传递
```sql
--改写前
where  t1.c1=t2.c1(+) and t1.c1=10;
--改写后
where  t1.c1=t2.c1(+) and t1.c1=10 and t2.c1(+)=10;
```

>外连接符号(+)
Oracle数据库中，符号(+)表示LEFT/RIGHT OUTER JOIN。如果加号写在右表的列上，则表示左连接，左表全部显示；反之加号写在左表的列上，则表示右连接，右表全部显示。


## CBO的局限性
CBO主要有以下几个局限性：

1. CBO会默认目标SQL语句中where条件中出现的各个列之间是相互独立的，没有关联关系。

实际应用中，目标SQL的各列之列有可能存在关联关系，进而影响到CBO对返回结果集cardinality的估算，导致选错执行计划。使用动态采样或者多列统计信息可以在一定程度上缓解该问题。但是动态采样的准确性取决于采样数据的数量和质量，而多列统计信息不适用于多表之间有关联关系的情形。

2. CBO会假设所有目标SQL都是单独执行的，并且互不干扰。

执行目标SQL时所需要访问的索引叶子块、数据块，可能由于之前执行的SQL已经被缓存到Buffer Cache中了，所以本次执行时并不需要耗费I/O去访问物理存储。如果CBO还是按照目标SQL单独执行，而不考虑缓存的话，对成本的估算就可能出现较大偏差。

3. CBO对直方图统计信息有较多限制。

Oracle数据库中对文本类型的字段收集统计信息时，只会将文本型字段的前32字节取出来，并将其转化为一个浮点数作为统计信息存储到数据字典里。对于那些超过32字节的文本型字段，只要前32字节相同，在收集统计信息时就会认为它们是相同的。这种缺陷会直接影响到CBO对文本型字段的可选择率和结果集cardinality的估算。

4. CBO在解析多表关联的目标SQL时，可能会漏选正确的执行计划。

假设目标SQL所包含的表的数量为n，则各表之间的连接顺序就有$n!$种。考虑到SQL解析的时间，CBO不可能遍历所有的连接顺序。在Oracle 11gR2中，所考虑的各表连接顺序的总和会受到隐含参数`_OPTIMIZER_MAX_PERMUTATIONS`的限制，即Oracle最多会考虑的连接顺序的总数不会超过该值。如果最优的执行计划不在该值的范围内，就肯定选不到最优的执行路径。


**References**
【1】基于Oracle的SQL优化，崔华，电子工业出版社



