---
tags: [oracle, SQL优化]
title: Oracle中常见的执行计划
created: '2022-10-07T11:22:20.562Z'
modified: '2022-10-09T12:37:12.511Z'
---

Oracle中常见的执行计划

# 常见的执行计划

## 与表访问相关的执行计划
Oracle数据库中访问表数据有两种方法：全表扫描和ROWID扫描。在执行计划中，与全表扫描对应的关键字是**TABLE ACCESS FULL**，与ROWID扫描对应的关键字则是**TABLE ACCESS BY USER ROWID**或者**TABLE ACCESS BY INDEX ROWID**。

通过ROWID访问表数据时，如果ROWID来源于用户手工指定（谓词条件如`where rowid='xxxxx'`），则对应的执行计划关键字为**TABLE ACCESS BY USER ROWID**；如果ROWID来源于索引，则对应的关键字为**TABLE ACCESS BY INDEX ROWID**。

## 与B树索引相关的执行计划
Oracle种常见的索引扫描方式以及执行计划关键字的对应关系如下：

| 索引扫描方式 | 执行计划关键字 |
| :--: | :--: |
| 索引唯一性扫描 | INDEX UNIQUE SCAN |
| 索引范围扫描 | INDEX RANGE SCAN |
| 索引全扫描 | INDEX FULL SCAN |
| 索引快速全扫描 | INDEX FAST FULL SCAN |
| 索引跳跃式扫描 | INDEX SKIP SCAN |

## 与位图索引相关的执行计划
位图索引（Bitmap Index）是Oracle数据库中除了B树索引之外的另外一种索引类型，主要用于数据仓库或者决策支持系统（DSS）。位图索引中实现了快捷的按位运算，在某些类型的SQL中，效率要高于B树索引。

### 位图索引的物理存储结构
位图索引的物理存储结构和普通的B树索引类似，也是按照被索引的键值有序排列；但是，位图索引中和索引键值一起存储的不再仅仅是索引键值对应的ROWID，而是变成了三部分的组合，分别对应ROWID的下限、ROWID的上限、以及被压缩存储的位图段（Bitmap Segment）。

```
位图索引的物理存储结构=<被索引的键值, 对应ROWID下限, 对应ROWID上限, 位图段>
```

位图索引中的位图段是被压缩存储的，解压缩后就是一连串0和1的二进制位图序列，其中1表示被索引键值的一个有效rowid。Oracle通过转换函数（mapping function）将解压缩后的位图段中的1结合对应rowid的上下限，转换为被索引键值所对应的有效rowid。

### 位图索引的优缺点
位图索引的物理存储结构决定了Oracle中位图索引的锁的粒度是在索引行的位图段上。Oracle数据库中的位图索引没有行锁的概念，因为要锁就锁索引行的整个位图段，而**多个数据行可能对应同一个索引行的位图段**。这种锁的粒度决定了**位图索引不适用于高并发并且频繁更新的OLTP业务**。如果在高并发的OLTP系统中使用了位图索引，很可能导致严重的并发问题，甚至产生死锁。

与B树索引相比，位图索引的优势主要体现在如下三个方面：

1. 由于位图索引的位图段是压缩后存储的，如果被索引的列distinct的值较少，那么位图索引与相同列上的B树索引比起来，会显著节省存储空间。

2. 如果需要在多个列上创建索引，那么位图索引与同等条件下的B树索引比起来，往往会显著节省存储空间。

3. 位图索引能够快速处理一些包含了各种**AND**或者**OR**查询条件的SQL，这主要是因为位图索引能够实现快捷的按位运算。

### 位图索引的访问方法
Oracle数据库里常见的与位图索引访问相关的方法包括：位图索引单键值扫描、位图索引范围扫描、位图索引快速全扫描、位图索引按位与、位图索引按位或、位图索引按位减等。它们对应的执行计划中的关键字如下：

| 索引扫描方式 | 执行计划关键字 |
| :--: | :--: |
| 位图索引单键值扫描 | BITMAP INDEX SINGLE VALUE |
| 位图索引范围扫描 | BITMAP INDEX RANGE SCAN |
| 位图索引全扫描 | BITMAP INDEX FULL SCAN |
| 位图索引快速全扫描 | BITMAP INDEX FAST FULL SCAN |
| 位图索引按位与 | BITMAP ADN |
| 位图索引按位或 | BITMAP OR |
| 位图索引按位减 | BITMAP MINUS |

Oracle在使用完位图索引后会将最后的位图运算结果转化为ROWID，这一转换过程在执行计划中对应的关键字为**BITMAP CONVERSION TO ROWIDS**。

```sql
--创建测试表
> create table customer (
  customer# number,
  marital_status varchar2(10),
  region varchar2(10),
  gender varchar2(10),
  income_level varchar2(10)
);

> insert into customer values(101, 'single', 'east', 'male', 'bracket_1');
> insert into customer values(102, 'married', 'central', 'female', 'bracket_4');
> insert into customer values(103, 'married', 'west', 'female', 'bracket_2');
> insert into customer values(104, 'divorced', 'west', 'male', 'bracket_4');
> insert into customer values(105, 'single', 'central', 'female', 'bracket_2');
> insert into customer values(106, 'married', 'central', 'female', 'bracket_3');
> commit;

--创建位图索引
> create bitmap index idx_b_region on customer(region);
> create bitmap index idx_b_maritalstatus on customer(marital_status);
--收集统计信息
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT', tabname => 'CUSTOMER', estimate_percent => 100, cascade => true);

--测试1
> select /*+ index(customer IDX_B_REGION) */customer# from customer where region='east';
> select * from table(dbms_xplan.display_cursor(null,null,'ALL'));
...
SELECT STATEMENT               |
 TABLE ACCESS BY INDEX ROWID   | CUSTOMER
   BITMAP CONVERSION TO ROWIDS |
    BITMAP INDEX SINGLE VALUE  | IDX_B_REGION   --//位图索引单键值扫描

--测试2
> select /*+ index(customer IDX_B_REGION) */customer# from customer where region between 'east' and 'west';
> select * from table(dbms_xplan.display_cursor(null,null,'ALL'));
...
    BITMAP INDEX RANGE SCAN  | IDX_B_REGION   --//位图索引范围扫描

--测试3
> select /*+ index(customer IDX_B_REGION) */region from customer;
> select * from table(dbms_xplan.display_cursor(null,null,'ALL'));
...
    BITMAP INDEX FULL SCAN  | IDX_B_REGION   --//位图索引全扫描

--测试4
> select count(*) from customer where region='east';
> select * from table(dbms_xplan.display_cursor(null,null,'ALL'));
...
    BITMAP INDEX FAST FULL SCAN  | IDX_B_REGION   --//位图索引快速全扫描

--测试5
> select count(*) from customer where marital_status='married' and region in ('central','west');
> select * from table(dbms_xplan.display_cursor(null,null,'ALL'));
...
SELECT STATEMENT                |
 SORT AGGREGATE                 |
  BITMAP CONVERSION COUNT       |
   BITMAP AND                   |                       --//位图索引按位与
    BITMAP INDEX SINGLE VALUE   | IDX_B_MARITALSTATUS   --//位图索引单键值扫描
    BITMAP OR                   |                       --//位图索引按位或
     BITMAP INDEX SINGLE VALUE  | IDX_B_REGION  
     BITMAP INDEX SINGLE VALUE  | IDX_B_REGION   

--测试6
> select /*+ index(customer IDX_BMARITALSTATUS) index(cusotmer IIDX_B_REGION) */customer# from customer where marital_status='married' and region != 'central';
> select * from table(dbms_xplan.display_cursor(null,null,'ALL'));
...
 0 | SELECT STATEMENT                |
 1 |  TABLE ACCESS BY INDEX ROWID    | CUSOTMER
 2 |   BITMAP CONVERSION TO ROWIDS   |
 3 |    BITMAP MINUS                 |                      --//位图索引按位减   
 4 |     BITMAP MINUS                |                      --//位图索引按位减 
*5 |      BITMAP INDEX SINGLE VALUE  | IDX_B_MARITALSTATUS  --//位图索引单键值扫描
*6 |      BITMAP INDEX SINGLE VALUE  | IDX_B_REGION   
*7 |     BITMAP INDEX SINGLE VALUE   | IDX_B_REGION
```

上面最后的测试中，先通过步骤5筛选出满足` marital_status='married'`的结果集1，然后通过步骤6获取`region='central'`的结果集2，最后在步骤4对结果集1和结果集2进行按位减运算。随后的步骤7和步骤3则是为了通过按位减运算排除掉`region is null`的记录。

## 与表连接相关的执行计划
Oracle种常见的表连接方法以及执行计划关键字的对应关系如下：

| 表连接方法 | 执行计划关键字 |
| :--: | :--: |
| 排序合并连接 | SORT JOIN、MERGE JOIN |
| 嵌套循环连接 | NESTED LOOPS |
| 哈希连接 | HASH JOIN |
| 反连接 | [HASH JOIN \| MERGE JOIN \| NESTED LOOPS] ANTI |
| 半连接 | [HASH JOIN \| MERGE JOIN \| NESTED LOOPS] SEMI |


## 其他典型执行计划

1. **AND-EQUAL (INDEX MERGE)**

AND-EQUAL又叫做INDEX MERGE，是指如果WHERE条件中出现了多个针对不同单列的等值条件，并且这些列上都有单键值的索引时，那么Oracle就可能会以单个等值条件去分别扫描这些索引，然后合并扫描单个索引所得到的ROWID集合。如果能从这些集合中找到值相同的ROWID，那么这些相同的ROWID就是目标SQL最终执行结果对应的ROWID。通过利用这些ROWID回表就能得到最终的结果集。

2. **INDEX JOIN**

INDEX JOIN是指针对**单表**上的不同索引之间的连接。以表EMP_TEMP为例，假设我们已经在该表的列MGR和列DEPTNO上分别创建了两个单键值的B树索引`IDX_MGR`和`IDX_DEPTNO`，如果此时执行SQL语句：
```sql
> select mgr,deptno from emp_temp;
```
如果Oracle采用如下方法：先分别扫描索引`IDX_MGR`和`IDX_DEPTNO`，得到的结果集分别记为结果集1和结果集2，然后将结果集1和结果集2做一个连接，连接条件为ROWID相等，这样最终得到的结果（不用回表）就是目标SQL的最终执行结果。这种执行方法就是INDEX JOIN。它的执行效率比不上直接在列MGR和列DEPTNO上建立一个联合索引，然后直接扫描该联合索引。INDEX JOIN只是为CBO优化器提供了一种可选的执行路径。

INDEX JOIN在执行计划中对应的关键字和普通的表连接是一样的，只不过参与连接的对象是索引而不是表。走Index Join方法时，在执行计划的**Outline Data**部分可以看到`INDEX_JOIN`关键字。

3. **VIEW**

Oracle中在处理包含视图的SQL中，根据该视图能否做**视图合并**（View Merging），对应的执行计划有以下两种形式：

- 如果可以做视图合并，Oracle在执行SQL时可以直接针对该视图的**基表**，此时执行计划中很可能不会出现VIEW关键字；
- 如果不能做视图合并，Oracle将把该视图看作一个整体并独立执行它，此时执行计划中将会出现关键字VIEW。

4. **FILTER**

FILTER是一种特殊的执行计划，对应的执行过程为以下三个步骤：

- 得到一个驱动结果集；
- 根据一定的条件从驱动结果集中过滤掉不满足条件的记录；
- 结果集中剩下的记录就会返回给最终用户或者继续参与下一个执行步骤。

```sql
--定义视图
> create or replace view emp_mgr_view as select * from emp_temp where job='MANAGER' and rownum<10;

--测试1
> select empno,ename from emp where empno in (select empno from emp_mgr_view);
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
...
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  NESTED LOOPS                 |
 2 |   VIEW                        | VM_NSO_1
 3 |    HASH UNIQUE                |                 --//去重
 4 |     VIEW                      | EMP_MGR_VIEW
*5 |      COUNT STOPKEY            |                 --//对应rownum<10
*6 |       TABLE ACCESS FULL       | EMP_TEMP
 7 |   TABLE ACCESS BY INDEX ROWID | EMP
*8 |    INDEX UNIQUE SCAN          | PK_EMP
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
5 - filter(ROWNUM<10)
6 - filter("JOB"-'MANAGER')
8 - access("EMPNO"="$nso_col_1")
```
上面的执行计划走的是嵌套循环连接，而没有走FILTER类型的执行计划。这是由于Oracle做了**子查询展开**（Subquery Unnesting），把子查询`select empno from emp_mgr_view`和它外部的SQL做了合并，转换成了视图`VM_NSO_1`和表EMP的嵌套循环连接。

使用Hint禁用子查询展开后重新执行SQL，可以看到这次执行计划走了FILTER类型。
```sql
--测试2
> select empno,ename from emp where empno in (select /*+ NO_UNNEST */empno from emp_mgr_view);
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
...
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
*1 |  FILTER                       |
 2 |   TABLE ACCESS FULL           | EMP
*3 |   VIEW                        | EMP_MGR_VIEW            
*4 |    COUNT STOPKEY              | 
*5 |     TABLE ACCESS FULL         | EMP_TEMP    
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
1 - filter(IS NOT NULL)
3 - filter("EMPNO"=:B1)
4 - filter(ROWNUM<10)
5 - filter("JOB"-'MANAGER')
```
FILTER类型的执行计划实际上是一种改良的嵌套循环连接，它并不像嵌套循环连接那样，驱动结果集中有多少次记录就得访问多少次被驱动表。

5. **SORT**

执行计划中的排序操作通常会以组合形式出现，包括但不限于：SORT AGGREGATE、SORT UNIQUE、SORT JOIN、SORT GROUP BY、SORT ORDER BY、BUFFER SORT等。即使执行计划中出现了SORT关键字，也不一定意味着就需要排序，比如SORT AGGREGATE和BUFFER SORT就不一定需要排序。

```sql
--测试1
> set autotrace traceonly
> select sum(sal) from emp_temp where job='MANAGER';
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  SORT AGGREGATE               |               --//查看统计信息实际未发生排序
*2 |   TABLE ACCESS FULL           | EMP_TEMP
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
2 - filter("JOB"='MANAGER')
统计信息
...
0 sorts (memory)           --//内存中未排序
0 sorts (disk)             --//磁盘中未排序

--测试2
> set autotrace off
> select distinct ename from emp_temp where job='MANAGER' order by ename;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  SORT UNIQUE                  |               --//排序加去重
*2 |   TABLE ACCESS FULL           | EMP_TEMP
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
2 - filter("JOB"='MANAGER')

--测试3
> select /*+ use_merge(t1 t2) */t1.empno,t1.ename,t2.sal from emp t1, emp_temp t2 where t1.empno=t2.empno;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  MERGE JOIN                   |                --//排序合并连接      
 2 |   TABLE ACCESS BY INDEX ROWID | EMP
 3 |    INDEX FULL SCAN            | PK_EMP
*4 |   SORT JOIN                   |                --//排序合并连接
 5 |    TABLE ACCESS FULL          | EMP_TEMP
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
4 - access("T1"."EMPNO"="T2"."EMPNO")
    filter("T1"."EMPNO"="T2"."EMPNO")

--测试4
> select ename from emp_temp where job='MANGER' order by ename;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  SORT ORDER BY                |                --//单纯的排序      
*2 |   TABLE ACCESS FULL           | EMP_TEMP
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
2 - filter("JOB"='MANGER')

--测试5
> select ename from emp_temp where job='MANGER' group by ename order by ename;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  SORT GROUP BY                |                --//排序加分组      
*2 |   TABLE ACCESS FULL           | EMP_TEMP
-------------------------------------------------
...
Predicate Information (identified by operation id):
----------------------------------------------------
2 - filter("JOB"='MANGER')
```

>:rabbit:看一个SQL执行是否有排序，最直观的方法就是看统计信息中`sorts(memory)`和`sorts(disk)`的值是否都为0。但是这两个指标对BUFFER SORT而言是**不准**的。

>:wolf:执行计划为BUFFER SORT时，要通过目标SQL真实执行计划中**Column Projeciton Information**部分中`#keys`的值来判断是否发生了排序。`#keys`表示参与实际排序的列数量，它的值大于0就表示发生了排序。

```sql
--测试6
> set autotrace traceonly
> select t1.ename,t2.loc from emp t1, dept t2;
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  MERGE JOIN CARTESIAN         |                    
 2 |   TABLE ACCESS FULL           | DEPT
 3 |   BUFFER SORT                 | 
 4 |    TABLE ACCESS FULL          | EMP 
-------------------------------------------------
统计信息
-------------------------------------------------
...
1 sorts (memory)
0 sorts (disk)

--查看真实的执行计划
> select sql_text,sql_id,child_number from v$sql where sql_text like 'select t1.ename%';
SQL_TEXT                                     SQL_ID         CHILD_NUMBER
-------------------------------------------  -------------  ------------
select t1.ename,t2.loc from emp t1, dept t2  3rbk2gjy2fyjk       0

> select * from table(dbms_xplan.display_cursor('3rbk2gjy2fyjk',0,'advanced'));
执行计划
-------------------------------------------------
ID | Operation                     | Name
-------------------------------------------------
 0 | SELECT STATEMENT              | 
 1 |  MERGE JOIN CARTESIAN         |                    
 2 |   TABLE ACCESS FULL           | DEPT
 3 |   BUFFER SORT                 | 
 4 |    TABLE ACCESS FULL          | EMP 
-------------------------------------------------
...
Column Projection Information (identified by operation id):
-----------------------------------------------------------
1 - "T2"."LOC"[VARCHAR2,13], "T1"."ENAME"[VARCHAR2,10]
2 - "T2"."LOC"[VARCHAR2,13]
3 - (#keys=0) "T1"."ENAME"[VARCHAR2,10]     --//#keys=0表示步骤3没有列参与排序
4 - "T1"."ENAME"[VARCHAR2,10]
```

BUFFER SORT表示Oracle会借用PGA把扫描结果load进去，这样可以节省对应的缓存在SGA中所带来的种种开销，如持有、释放相关latch等。

6. **UNION | UNION ALL**

UNION和UNION ALL表示对两个结果集进行合并。两者的区别在于：UNION ALL仅仅是将两个结果集简单合并，不做额外处理；而UNION除了合并之外，还会对合并后的结果集做**排序和去重**，相当于在进行UNION ALL之后又进行了SORT UNIQUE操作。

```sql
> select empno, ename from emp union all select empno,ename from emp_temp;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
执行计划
0 | SELECT STATEMENT
1 |  UNION ALL
2 |   TABLE ACCESS FULL
3 |   TABLE ACCESS FULL

> select empno, ename from emp union select empno,ename from emp_temp;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
执行计划
0 | SELECT STATEMENT
1 |  SORT UNIQUE
2 |   UNION ALL
3 |    TABLE ACCESS FULL
4 |    TABLE ACCESS FULL
```

7. **CONCAT**

CONCAT就是IN-List扩展或者OR扩展，在执行计划中对应的关键字是**CONCATENATION**。

8. **CONNECT BY**

CONNECT BY是Oracle数据库中层次查询（Hierarchical Queries）对应的关键字。层次查询一般用于行记录之间存在父子、上下级等级关系的表的查询。



