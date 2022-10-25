---
tags: [oracle, SQL优化]
title: Oracle查询转换之表扩展、表移除、IN运算符
created: '2022-10-24T13:32:35.240Z'
modified: '2022-10-25T04:35:08.582Z'
---

Oracle查询转换之表扩展、表移除、IN运算符

# 表扩展
表扩展（Table Expansion）是优化器针对分区表的目标SQL的一种优化手段，它是指当目标SQL中分区表的某个**局部分区索引**由于某种原因在某些分区上变得不可用（索引状态变为**UNUSABLE**）时，Oracle能将目标SQL等价改写成**按分区UNION ALL**的形式，这样除了那些不可用的分区所对应的UNION ALL分支之外，其他分区所对应的UNION ALL分支还是可以正常地使用该局部分区索引。

表扩展最早于Oracle 11gR2中才被引入。如果不能做表扩展，对于上述局部分区索引而言，只要在一个分区上它的状态为UNUSABLE，则整个目标SQL就不能使用该局部分区索引。

当查询的数据只是全部数据中的少部分时，使用索引确实能够极大地提升查询速度，但是索引的维护必然会减慢相关DML操作的执行速度，所以使用索引是有副作用的。有的业务系统需要尽可能地往目标表的当前分区中导入数据，其修改的数据也仅限于当前分区的活动数据，同时还会有针对目标表的查询，这个查询不限于当前分区。针对这种类型的系统，为了尽可能快地对当前分区执行DML操作，往往人为地把当前分区的局部分区索引设置为UNUSABLE状态，以省去对当前分区执行DML操作时同步维护局部分区索引的成本。但是在Oracle 11gR2之前，这样的操作会导致所有原本可以使用该局部分区索引的查询语句都无法正常使用该局部分区索引。

下面SQL中的表SALES是一个按照列`TIME_ID`来做RANGE分区的分区表，一共有28个分区，其中最后一个分区为`SALES_Q4_2003`，同时在列`CUST_ID`上存在一个局部分区位图索引`SALES_CUST_BIX`，在分区键`TIME_ID`上也存在一个局部分区位图索引`SALES_TIME_BIX`。

```sql
--//测试一：局部分区索引可用
> select * from sales
where cust_id = 3754
and time_id between
to_date('2000-01-01 00:00:00','SYYYY-MM-DD HH24:MI:SS') and
to_date('2004-01-01 00:00:00','SYYYY-MM-DD HH24:MI:SS');
执行计划
Id | Operation                           | Name           | Pstart | Pstop |
----------------------------------------------------------------------------
 0 | SELECT STATEMENT                    |                |        |       |      
 1 |  PARTITION RANGE ITERATOR           |                |   13   |  28   |
*2 |   TABLE ACCESS BY LOCAL INDEX ROWID | SALES          |   13   |       |
 3 |    BITMAP CONVERSION TO ROWIDS      |                |        |       |
*4 |     BITMAP INDEX SINGLE VALUE       | SALES_CUST_BIX |   13   |  28   |

Predicate Information (identified by operation id):
2 - filter ("TIME_ID"<=TO_DATE('2004-01-01 00:00:00','syyyy-mm-dd hh24:mi:ss'))
4 - access ("CUST_ID"=3754)
```

可以看到，上面的SQL走的是对局部分区索引`SALES_CUST_BIX`的单键值位图索引扫描，并且只扫描了SALES表的16个分区（分区裁剪）。假设我们把最后一个分区`SALES_Q4_2003`上的局部分区位图索引设置为不可用：

```sql
> alter index SALES_CUST_BIX modify partition SALES_Q4_2003 unusable;
> alter index SALES_TIME_BIX modify partition SALES_Q4_2003 unusable;

--//测试二：局部分区索引在某个分区不可用
> select * from sales
where cust_id = 3754
and time_id between
to_date('2000-01-01 00:00:00','SYYYY-MM-DD HH24:MI:SS') and
to_date('2004-01-01 00:00:00','SYYYY-MM-DD HH24:MI:SS');

执行计划（Oracle 10g）
Id | Operation                           | Name           | Pstart | Pstop |
----------------------------------------------------------------------------
 0 | SELECT STATEMENT                    |                |        |       |      
 1 |  PARTITION RANGE ITERATOR           |                |   13   |  28   |
*2 |   TABLE ACCESS FULL                 | SALES          |   13   |  28   |

Predicate Information (identified by operation id):
2 - filter ("CUST_ID"=3754 AND "TIME_ID"<=TO_DATE('2004-01-01 00:00:00','syyyy-mm-dd hh24:mi:ss'))

执行计划（Oracle 11gR2）
Id | Operation                             | Name           | Pstart | Pstop |
------------------------------------------------------------------------------
 0 | SELECT STATEMENT                      |                |        |       |      
 1 |  VIEW                                 | VM_TE_2        |        |       |
 2 |   UNION ALL                           |                |        |       |
 3 |    PARTITION RANGE ITERATOR           |                |   13   |  27   |
 4 |     TABLE ACCESS BY LOCAL INDEX ROWID | SALES          |   13   |  27   |
 5 |      BITMAP CONVERSION TO ROWIDS      |                |        |       |
*6 |       BITMAP INDEX SINGLE VALUE       | SALES_CUST_BIX |   13   |  27   |
 7 |    PARTITION RANGE SINGLE             |                |   28   |  28   |
*8 |     TABLE ACCESS FULL                 | SALES          |   28   |  28   |

Predicate Information (identified by operation id):
6 - filter ("CUST_ID"=3754)
8 - access ("CUST_ID"=3754)
```
可以看到，在Oracle 10g中，目标SQL的执行计划已经从对局部分区索引`SALES_CUST_BIX`的单键值位图索引扫描变成了全表扫描，即某个局部分区索引设置为不可用后，会导致所有使用该局部分区索引的查询语句都无法正常使用局部分区索引。

在Oracle 11gR2中，目标SQL的执行计划中，步骤1的关键字VIEW对应的视图名称为`VM_TE_2`，其中**TE**是Table Expansion的缩写。通过表扩展，把访问的16个分区分成了UNION ALL的两个分支。其中前15个分区走的仍然是对局部分区索引`SALES_CUST_BIX`的单键值位图索引扫描，只有最后一个分区为全表扫描。原SQL通过表扩展后，实际上被等价改写成为了下面的形式：

```sql
> select * from 
(select * from sales
where cust_id = 3754
and time_id between
to_date('2000-01-01 00:00:00','SYYYY-MM-DD HH24:MI:SS') and
to_date('2003-09-30 00:00:00','SYYYY-MM-DD HH24:MI:SS')
union all
select * from sales
where cust_id = 3754
and time_id between
to_date('2003-10-01 00:00:00','SYYYY-MM-DD HH24:MI:SS') and
to_date('2004-01-01 00:00:00','SYYYY-MM-DD HH24:MI:SS')
) VM_TE_2;
```

Oracle数据里的表扩展也是基于成本的。**只有当经过表扩展后得到的等价改写SQL的成本值小于原SQL时**，Oracle才会对目标SQL做表扩展。


# 表移除
表移除（Table Elimination）是优化器处理带**多表连接**的目标SQL的一种优化手段，它是指优化器把虽然在目标SQL中存在，但是其存在与否**对最终执行结果没有影响的表**从目标SQL中移除，以减少表连接的次数，从而提高SQL执行效率。

表移除在Oracle 10gR2中已经被引入，通常应用于那些表与表之间通过**外键关联**或者表与表之间是**外连接**的情形。

```sql
--测试一：外键关联
> select ename from emp,dept
where emp.deptno = dept.deptno;
执行计划
Id | Operation          | Name |
 0 | SELECT STATEMENT   |      |
 1 |  TABLE ACCESS FULL | EMP  |
--//执行计划中没有出现表dept

--查看表EMP的定义
> select dbms_metadata.get_ddl('TBALE','EMP','SCOTT') from dual;
...
CREATE TABLE "SCOTT"."EMP"
(
  ...
  "DEPTNO" NUMBER(2,0) NOT NULL ENABLE,
  ...
  CONSTRAINT "FK_DEPTNO" FOREIGN KEY ("DEPTNO")
  REFERENCES "SCOTT"."DEPT" ("DEPTNO") ENABLE
)
```

从上面可以看出，表EMP中的列DEPTNO的属性是NOT NULL，且在列DEPTNO和表DEPT的主键列DEPTNO之间存在一个名为`FK_DEPTNO`的外键约束。对于该外键而言，表EMP是子表，表DEPTNO是主表。也就是说对于子表EMP的列DEPTNO而言，在主表DEPT中一定存在对应的匹配记录，因此原SQL其实等价于`select ename from emp`。

```sql
--测试二：外连接
> select ename from emp, dept
where emp.deptno = dept.deptno (+);
执行计划
Id | Operation          | Name |
 0 | SELECT STATEMENT   |      |
 1 |  TABLE ACCESS FULL | EMP  |

--//上面的SQL等价于
> select ename from emp where deptno is not null
union all
select ename from emp where deptno is null;
```


# 对IN运算符的处理

在Oracle数据库里，IN和OR是等价的，优化器在处理带IN的目标SQL时实际上会将其转化为带OR的等价改写SQL。

```sql
> select * from emp where deptno in (10,20);
--//等价于
> select * from emp where deptno=10 or deptno=20;

> select * from emp where deptno=10 or mgr=7728;
--//等价于
> select * from emp where deptno in (10)
union all 
select * from emp where mgr in (7728) and lnnvl(deptno in (10));
--//lnnvl(TRUE)=FALSE, lnnvl(FALSE)=TRUE, lnnvl(NULL)=TRUE
```

优化器在处理带IN的目标SQL时，通常会采用如下四种方法：
- IN-List Iterator
- IN-List Expansion
- IN-List Filter
- 对IN做子查询展开，或者既做子查询展开又做视图合并。

## IN-List Iterator
IN-List Iterator是针对IN后面是**常量集合**的一种处理方法。此时优化器会遍历目标SQL中IN后面的常量集合中的每一个值，然后去做比较，看目标结果中是否存在和这个值匹配的记录。如果存在匹配的记录，则这个记录就会成为该SQL的最终返回结果集中的一员；如果不存在匹配记录，则优化器会继续遍历IN后面的常量集合中的下一个值，直到该常量集合遍历完毕。

IN-List Iterator在SQL执行计划中的关键字是**INLIST ITERATOR**。关于IN-List Iterator，需要注意以下几点：
- IN-List Iterator是Oracle针对目标SQL的IN后面是**常量集合的首选**处理方法，它的处理效率通常都会比IN-List Expansion高。
- Oracle能够用IN-List Iterator来处理IN的前提条件是**IN所在的列上一定要有索引**。
- 不能强制让Oracle走IN-List Iterator类型的执行计划，Oracle也没有强制走IN-List Iterator的Hint，但是可以通过联合设置10142和10157事件来禁用IN-List Iterator。

```sql
--测试一：IN所在列上有索引
> select * from emp where deptno in (10,20,30);
Id | Operation                     | Name         |
 0 | SELECT STATEMENT              |              |
 1 |  INLIST ITERATOR              |              |
 2 |   TABLE ACCESS BY INDEX ROWID | EMP          | 
*3 |    INDEX RANGE SCAN           | IDX_EMP_DEPT |

Predicate Information (identified by operation id):
3 - access ("DEPTNO"=10 OR "DEPTNO"=20 OR "DEPTNO"=30)

--测试一：IN所在列上没有索引
> drop index idx_emp_dept;
> select * from emp where deptno in (10,20,30);
Id | Operation                     | Name         |
 0 | SELECT STATEMENT              |              |
*1 |  TABLE ACCESS FULL            | EMP          |

Predicate Information (identified by operation id):
1 - filter ("DEPTNO"=10 OR "DEPTNO"=20 OR "DEPTNO"=30)
```

## IN-List Expansion
IN-List Expansion也被称为OR EXPANSION，是针对IN后面是**常量集合**的另一种处理方法。它是指优化器会把目标SQL中IN后面的常量集合拆开，把里面的每个常量都提出来形成一个分支，各分支之间用**UNION ALL**来连接。

```sql
--原SQL
> select * from emp where deptno in (10,20,30);

--使用IN-List Expansion后的等价改写形式
> select * from emp where deptno=10
union all
select * from emp where deptno=20
union all
select * from emp where deptno=30;
```

IN-List Expansion的好处是改写成以UNION ALL连接的分支后，各个分支就可以各自走索引、分区裁剪（Partition Pruning）、表连接等相关的执行计划而互不干扰。它的坏处是优化器需要对改写后的每一个分支都执行同样的解析、决定执行计划的工作。也就是说，改写后的SQL解析时间会随着UNION ALL分支的增加而递增。IN-List Expansion也是基于成本的，**只有当经过IN-List Expansion改写后的等价改写SQL的成本值小于原SQL**时，Oracle才会对目标SQL使用IN-List Expansion。

IN-List Expansion在SQL执行计划中的关键字是**CONCATENATION**。可以在SQL文本中加入强制走IN-List Expansion的`USE_CONCAT` Hint（不一定起作用，因为要考虑成本），也可以加入强制不走IN-List Expansion的`NO_EXPAND` Hint。

```sql
--强制走IN-List Expansion，会考虑成本
> select /*+ use_concat */* from emp where deptno in (10,20,30);
--强制不走IN-List Expansion
> select /*+ no_expand */* from emp where deptno in (10,20,30);
```

## IN-List Filter
IN-List Filter是针对IN后面是**子查询**的一种处理方法，优化器会把IN后面的子查询所对应的结果集当作过滤条件，并且走**FILTER**类型的执行计划。

IN后面是子查询，意味着IN后面是变量的集合；走的是FILTER类型的执行计划，意味着Oracle**没有对子查询做子查询展开**。因此，走IN-List Filter类型的执行计划意味着目标SQL要满足以下两个条件：
- 目标SQL的**IN后面是子查询**，而不是常量集合；
- Oracle没有对目标SQL的IN后面的子查询做子查询展开。

```sql
> select t1.ename,t1.deptno
from emp t1
where t1.deptno in 
(select /*+ no_unnest */t2.deptno     --//使用hint禁用子查询展开
from dept t2
where t2.loc='CHICAGO');
执行计划
Id | Operation                     | Name         |
 0 | SELECT STATEMENT              |              |
*1 |  FILTER                       |              |
 2 |   TABLE ACCESS FULL           | EMP          | 
*3 |   TABLE ACCESS BY INDEX ROWID | DEPT         |
*4 |    INDEX UNIQUE SCAN          | PK_DEPT      |
```

## IN-List子查询展开&视图合并
对IN-List做子查询展开（或者同时做视图合并）是针对IN后面是**子查询**的另一种处理方法。

能对IN做子查询展开和视图合并（假设子查询中包含视图），意味着目标SQL需要满足以下前提条件：
- 目标SQL的**IN后面是子查询**，而不是常量集合；
- Oracle能够对目标SQL的IN后面的子查询做子查询展开。

```sql
--创建测试表和视图
> create table emp_temp as select * from emp;
> create or replace view emp_mgr_view as select * from emp_temp where job='MANGER' and rownum<10;
--//视图定义语句中包含ROWNUM关键字，所以不能做视图合并

--测试一：子查询包含视图，但不能做视图合并
> select empno, ename from emp
where empno in (select empno from emp_mgr_view);

执行计划
Id | Operation                     | Name         |
 0 | SELECT STATEMENT              |              |
 1 |  NESTED LOOPS                 |              |
 2 |   VIEW                        | VIEW_NSO_1   | 
 3 |    HASH UNIQUE                |              |
 4 |     VIEW                      | EMP_MGR_VIEW |
*5 |      COUNT STOPKEY            |              |
*6 |       TABLE ACCESS FULL       | EMP_TEMP     |
 7 |   TABLE ACCESS BY INDEX ROWID | EMP          |
*8 |    INDEX UNIQUE SCAN          | PK_EMP       |
```

从上面的执行计划可以看到，目标SQL走的是嵌套循环连接。步骤4中关键字VIEW对应的视图名称是`EMP_MGR_VIEW`，说明没有做视图合并，但同时Oracle已经对子查询做了子查询展开。

```sql
--测试二：子查询包含视图，可以做视图合并
> create or replace view emp_mgr_view as select * from emp_temp where job='MANGER';
> select empno, ename from emp
where empno in (select empno from emp_mgr_view);

执行计划
Id | Operation                     | Name         |
 0 | SELECT STATEMENT              |              |
 1 |  NESTED LOOPS                 |              |
 2 |   NESTED LOOPS                |              | 
 3 |    SORT UNIQUE                |              |
 4 |     TABLE ACCESS FULL         | EMP_TEMP     |
*5 |    INDEX UNIQUE SCAN          | PK_EMP       |
*6 |   TABLE ACCESS BY INDEX ROWID | EMP          |

Outline Data
------------
...
UNNEST(@"SEL$335DD26A")
MERGE(@"SEL$3")
...
```

从上面的执行计划和Outline Data可以看到，目标SQL走的是嵌套循环连接。执行计划中没有出现视图`EMP_MGR_VIEW`，说明做了视图合并。


