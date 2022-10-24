---
tags: [oracle, SQL优化]
title: Oracle查询转换之连接谓词推入与连接因式分解
created: '2022-10-22T03:15:44.710Z'
modified: '2022-10-24T13:32:54.515Z'
---

Oracle查询转换之连接谓词推入与连接因式分解

# 连接谓词推入
连接谓词推入（Join Predicate Pushdown）是优化器处理带视图的目标SQL的另一种优化手段，它是指虽然优化器还是会把该SQL中视图的定义语句当作一个独立的处理单元来单独执行，但是此时优化器会把原本处于该视图外部查询中和该视图之间的**连接条件**推入到该视图的定义语句内部，这样是为了能用上该视图内部相关**基表**上的索引，进而能走出**基于索引的嵌套循环连接**。

连接谓词推入所带来的基于索引的嵌套循环连接并不一定能走出更高效的执行计划。因为当做了连接谓词推入后，对于外部查询所在的结果集中的每一条记录，视图定义的SQL语句都得单独执行一次。一旦**外部查询所在结果集的cardinality**比较大的话，即使在执行视图定义的SQL语句时能用上索引，整个SQL的执行效率也不一定会比不做连接谓词推入时的哈希连接或排序合并连接更高。因此，Oracle在做连接谓词推入时会考虑成本，**只有当经过连接谓词推入后走嵌套循环连接的等价改写SQL的成本值小于原SQL**时，Oracle才会对目标SQL使用连接谓词推入。

Oracle是否能做连接谓词推入与目标视图的类型、该视图与外部查询之间的连接类型以及连接方法有关。目前Oracle仅支持对如下类型的试图做连接谓词推入：

- 视图定义SQL语句中包含**UNION ALL**或**UNION**；
- 视图定义SQL语句中包含**DISTINCT**；
- 视图定义SQL语句中包含**GROUP BY**；
- 和外部查询之间的连接类型是**外连接**的视图；
- 和外部查询之间的连接类型是**反连接**的视图；
- 和外部查询之间的连接类型是**半连接**的视图。

```sql
--创建测试表
> create table emp1 as select * from emp;
> create table emp2 as select * from emp;
--创建索引
> create index idx_emp1 on emp1(empno);
> create index idx_emp2 on emp2(empno);

--创建视图（不含union all）
> create or replace view emp_view as
select emp1.empno as empno1 from emp1;
--创建视图（含union all）
> create or replace view emp_view_union as
select emp1.empno as empno1 from emp1
union all
select emp2.empno as empno1 from emp2;

--测试一：外连接，视图基表在连接列上有索引
--//使用SQL Hint禁用视图合并
> select /*+ no_merge(emp_view) */emp.empno   
from emp, emp_view
where emp.empno = emp_view.empno1(+)
and emp.ename='Ford';
执行计划
Id | Operation               | Name     |
 0 | SELECT STATEMENT        |          |
 1 |  NESTED LOOPS OUTER     |          |
*2 |   TABLE ACCESS FULL     | EMP      |
 3 |   VIEW PUSHED PREDICATE | EMP_VIEW |
*4 |    INDEX RANGE SCAN     | IDX_EMP1 |

Predicate Information (identified by operation id):
2 - filter ("EMP"."ENAME"="FORD")
4 - access ("EMP1"."EMPNO"="EMP"."EMPNO")
```

可以看到，上面的SQL语句在使用Hint禁用视图合并后走了嵌套循环连接，并且用到了视图定义语句中基表EMP1上的索引`idx_emp1`。外部查询和视图`emp_view`之间的连接条件`emp.empno = emp_view.empno1(+)`被推入到了视图`emp_view`的定义语句内部。注意到连接谓词推入在执行计划中的关键字为`VIEW PUSHED PREDICATE`。

```sql
--测试二：外连接，视图基表在连接列上有索引
--//使用SQL Hint禁用视图合并和连接谓词推入
> select /*+ no_merge(emp_view) no_push_pred(emp_view) */emp.empno   
from emp, emp_view
where emp.empno = emp_view.empno1(+)
and emp.ename='Ford';
执行计划
Id | Operation                     | Name     |
 0 | SELECT STATEMENT              |          |
 1 |  MERGE JOIN OUTER             |          |
*2 |   TABLE ACCESS BY INDEX ROWID | EMP      |
 3 |    INDEX FULL SCAN            | PK_EMP   |
*4 |   SORT JOIN                   |          |
 5 |    VIEW                       | EMP_VIEW |
 6 |     TABLE ACCESS FULL         | EMP1     |

Predicate Information (identified by operation id):
2 - filter ("EMP"."ENAME"="FORD")
4 - access ("EMP"."EMPNO"="EMP_VIEW"."EMPNO1"(+))
    filter ("EMP"."EMPNO"="EMP_VIEW"."EMPNO1"(+))
```

从上面可以看到，在禁用视图合并和连接谓词推入后，原SQL走了排序合并连接的执行计划，并且在访问视图定义语句中的基表时走了全表扫描（因为走索引的话需要回表）。

```sql
--测试三：内连接，视图基表在连接列上有索引，视图定义中包含UNION ALL
> select emp.empno   
from emp, emp_view_union
where emp.empno = emp_view_union.empno1
and emp.ename='Ford';
执行计划
Id | Operation                     | Name           |
 0 | SELECT STATEMENT              |                |
 1 |  NESTED LOOPS                 |                |
*2 |   TABLE ACCESS FULL           | EMP            |
 3 |   VIEW                        | EMP_VIEW_UNION |
 4 |    UNION ALL PUSHED PREDICATE |                |
*5 |     INDEX RANGE SCAN          | IDX_EMP1       |
*6 |     INDEX RANGE SCAN          | IDX_EMP2       |

Predicate Information (identified by operation id):
2 - filter ("EMP"."ENAME"="FORD")
5 - access ("EMP1"."EMPNO"="EMP"."EMPNO")
6 - access ("EMP2"."EMPNO"="EMP"."EMPNO")
```

可以看到，上面的SQL语句走了嵌套循环连接，并且用到了视图定义语句中基表上的索引`idx_emp1`和`idx_emp2`。外部查询和视图`emp_view_union`之间的连接条件`emp.empno = emp_view_union.empno1`被推入到了视图的定义语句内部。注意到此处连接谓词推入在执行计划中的关键字为`UNION ALL PUSHED PREDICATE`。

最后看一个走不了连接谓词推入的情况。
```sql
> select /*+ no_merge(emp_view) use_nl(emp_view) push_pred(emp_view) */emp.empno
from emp, emp_view
where emp.empno = emp_view.empno1
and emp.ename = 'FORD';
执行计划
Id | Operation               | Name     |
 0 | SELECT STATEMENT        |          |
 1 |  NESTED LOOPS           |          |
*2 |   TABLE ACCESS FULL     | EMP      |
 3 |   VIEW                  | EMP_VIEW |
*4 |    TABLE ACCESS FULL    | EMP1     |

Predicate Information (identified by operation id):
2 - filter ("EMP"."ENAME"="FORD")
4 - filter ("EMP"."EMPNO"="EMP_VIEW"."EMPNO1")
```
可以看到，即使加上了使用嵌套循环连接和使用连接谓词推入的SQL Hint，上面的SQL执行计划中也没有走连接谓词推入，而是走了全表扫描。因为上面的SQL不符合前面提到的那几种情形之一。


# 连接因式分解
连接因式分解（Join Factorization）是优化器处理带**UNION ALL**的目标SQL的一种优化手段。它是指优化器在处理以UNION ALL连接的目标SQL的各个分支时，不再原封不动地分别重复执行每个分支，而是会把各个分支中公共的部分提出来作为一个单独的结果集，然后再和原UNION ALL中剩下的部分做表连接。

连接因式分解最早在Oracle 11gR2中被引入。如果不把UNION ALL中公共部分提取出来，意味着这部分所包含的表会在UNION ALL的各个分支中被重复访问。连接因式分解能够在最大程度上避免这种重复访问的现象，从而提升执行效率，尤其是当UNION ALL的公共部分所包含的表数据量很大时。

```sql
> select t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id
and t3.cust_gender = 'MALE'
union all
select t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id
and t3.cust_gender = 'FEMALE';

执行计划（Oracle 11gR2）
Id | Operation                         | Name
 0 | SELECT STATEMENT                  |
*1 |  HASH JOIN                        |
 2 |   VIEW                            | VIEW_JF_SET$0F531EB5
 3 |    UNION ALL                      |
*4 |     VIEW                          | index$_join$_002
*5 |      HASH JOIN                    |
 6 |       BITMAP CONVERSION TO ROWID  |
*7 |        BITMAP INDEX SINGLE VALUE  | CUSTOMERS_GENDER_BIX
 8 |       INDEX FAST FULL SCAN        | CUSTOMERS_PK
*9 |     VIEW                          | index$_join$_004
*10|      HASH JOIN                    |
 11|       BITMAP CONVERSION TO ROWID  |
*12|        BITMAP INDEX SINGLE VALUE  | CUSTOMERS_GENDER_BIX
 13|       INDEX FAST FULL SCAN        | CUSTOMERS_PK
 14|   PARTITION RANGE ALL             |
 15|    TABLE ACCESS FULL              | SALES

Predicate Information (identified by operation id):
 1 - access ("T2"."CUST_ID"="ITEM_1") 
 4 - filter ("T3"."CUST_GENDER"='MALE')
 5 - access (ROWID=ROWID)
 7 - access ("T3"."CUST_GENDER"='MALE')
 9 - filter ("T3"."CUST_GENDER"='FEMALE')
10 - access (ROWID=ROWID)
12 - access ("T3"."CUST_GENDER"='FEMALE')
```

在Oracle 10g中，由于没有引入连接因式分解，上面的SQL执行计划中会在每个UNION ALL的分支中都对表SALES进行一次全表扫描。但是在Oracle 11R2中，上面的执行计划中只在步骤15中对表SALES进行了一次全表扫描。步骤2中对应的视图`VIEW_JF_SET$0F531EB5`名称中的**JF**是Join Factorization的缩写。通过连接因式分解，Oracle把UNION ALL中的公共部分表SALES提取了出来，然后和UNION ALL剩下部分所形成的内嵌视图`VIEW_JF_SET$0F531EB5`做了一个哈希连接。

相当于Oracle把上面的目标SQL改写成了下面的等价形式：
```sql
> select t2.prod_id as prod_id
from sales t2,
( select t3.cust_id as ITEM_1 
from customers t3
where t3.cust_gender = 'MALE'
union all
select t3.cust_id as ITEM_1 
from customers t3
where t3.cust_gender = 'FEMALE') VIEW_JF_SET$0F531EB5
where t2.cust_id = VIEW_JF_SET$0F531EB5.ITEM_1;
```

在Oracle 11gR2及其后续的版本中，即使由于在视图定义语句中使用了UNION ALL关键字而导致不能做视图合并，Oracle也不一定会把视图定义语句当作一个整体来单独执行，因为此时Oracle还可能会对其做连接因式分解。

Oracle对包含UNION ALL的目标SQL做连接因式分解的前提条件是，连接因式分解后的等价改写SQL必须和原SQL在语义上完全等价。
```sql
--//一个不能做连接因式分解的例子
> select distinct t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id
and t3.cust_gender = 'MALE'
union all
select distinct t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id
and t3.cust_gender = 'FEMALE';

--//与上面的SQL在语义上不完全等价
> select distinct t2.prod_id as prod_id
from sales t2,
( select t3.cust_id as ITEM_1 
from customers t3
where t3.cust_gender = 'MALE'
union all
select t3.cust_id as ITEM_1 
from customers t3
where t3.cust_gender = 'FEMALE') VIEW_JF_SET$0F531EB5
where t2.cust_id = VIEW_JF_SET$0F531EB5.ITEM_1;
```



