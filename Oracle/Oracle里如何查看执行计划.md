---
tags: [oracle, SQL优化]
title: Oracle里如何查看执行计划
created: '2022-10-04T08:54:08.330Z'
modified: '2022-10-07T15:13:18.185Z'
---

Oracle里如何查看执行计划

# 什么是执行计划
```sql
> select * from table(dbms_xplan.display_cursor('6fc6zasdtltr7',0,'advanced'));
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------
SQL_ID  6fc6zasdtltr7, child number 0
--------------------------------------------------------------------------
select /*+ real_exp_exmaple1 */t1.col1,t1.col2,t2.col3 from t1,t2 where t1.col2=t2.col2
Plan hash value: 282751716
--------------------------------------------------------------------------
| ID | Operation           | Name | Rows | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------
|  0 | SELECT STATEMENT    |      |      |       | 31 (100)   |          |
|* 1 |  HASH JOIN          |      | 4    | 80    | 31 (4)     | 00:00:01 |
|  2 |   TABLE ACCESS FULL | T2   | 4    | 20    | 15 (0)     | 00:00:01 |
|  3 |   TABLE ACCESS FULL | T1   | 5    | 75    | 15 (0)     | 00:00:01 |
--------------------------------------------------------------------------
Query Block Name / Object Alias (identified by operation id):
--------------------------------------------------------------------------
   1 - SEL$1
   2 - SEL$1 / T2@SEL$1
   3 - SEL$1 / T1@SEL$1
Outline Data
------------
  /*+
      BEGIN OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLED('10.2.0.5')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      FULL(@"SEL$1" "T2"@"SEL$1")
      FULL(@"SEL$1" "T1"@"SEL$1")
      LEADING(@"SEL$1" "T2"@"SEL$1" "T1"@"SEL$1")
      USE_HASH(@"SEL$1" "T1"@"SEL$1")
      END_OUTLINE_DATA
  */

Predicate Information (identified by operation id):
--------------------------------------------------------------------------
  1 - access("T1"."COL2"="T2"."COL2")
Column Projection Information (indentified by operation id):
--------------------------------------------------------------------------
  1 - (#keys=1) "T1"."COL2"[VARCHAR2,1], "T2"."COL3"[VARCHAR2,2], 
      "T1"."COL1"[NUMBER,22]
  2 - "T2"."COL2"[VARCHAR2,1], "T2"."COL3"[VARCHAR2,2]
  3 - "T1"."COL1"[NUMBER,22], "T1"."COL2"[VARCHAR2,1]
Note
--------------------------------------------------------------------------
  - dynamic sampling used for this statement
56 rows selected
```

上面的执行计划是在执行目标SQL：`select /*+ real_exp_exmaple1 */t1.col1,t1.col2,t2.col3 from t1,t2 where t1.col2=t2.col2`后使用DBMS_XPLAN包中的DISPLAY_CURSOR方法得到的，这是目标SQL的真实执行计划。可以看出，SQL执行计划可以分为如下三部分：

1. 目标SQL正文、SQL_ID和对应的Plan Hash Value。

2. 执行计划主体部分。

执行计划中的Query Block Name和Outline Data分别是CBO优化器在执行SQL时所用到的Query Block名称和用于固定执行计划的内部Hint组合。实际上可以将Outline Data的部分摘出来加到SQL中以固定执行计划。

```sql
> select /*+
      BEGIN OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLED('10.2.0.5')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      FULL(@"SEL$1" "T2"@"SEL$1")
      FULL(@"SEL$1" "T1"@"SEL$1")
      LEADING(@"SEL$1" "T2"@"SEL$1" "T1"@"SEL$1")
      USE_HASH(@"SEL$1" "T1"@"SEL$1")
      END_OUTLINE_DATA
  */
t1.col1,t1.col2,t2.col3 
from t1,t2 where t1.col2=t2.col2;
```

上述执行计划中，Id=1的Hash Join步骤前面有一个星号，表示该步骤有对应的驱动或过滤查询条件。这个星号对应的具体驱动或过滤查询条件可以在Predicate Information (identified by operation id)中找到。
```
 1 - access("T1"."COL2"="T2"."COL2")
```
其中，1表示执行步骤ID=1，acess关键字表示驱动查询条件，后面的T1.col2=T2.col2即为Hash Join对应的谓词连接条件。

3. 执行计划的额外补充信息。

对应上面执行计划中的Note部分。一般用于补充说明在执行目标SQL时是否使用了一些额外的技术手段，比如动态采样（dynamic sampling）、Cardinality Feedback、SQL Profile，等等。

# 如何查看执行计划

在Oracle数据库中可以使用下面的这些方法来获取目标SQL的执行计划（包括但不限于）：

- explain plan命令
- DBMS_XPLAN包
- SQLPlus中的AUTOTRACE开关
- 10046事件
- 10053事件
- AWR报告或者Statspack报告

## explain plan命令
使用explain plan命令查看执行计划需要依次执行以下两条语句：
```sql
explain plan for 目标SQL;
select * from table(dbms_xplan.display);
```

示例：
```sql
> explain plan for select empno,ename,dname from scott.emp,scott.dept where emp.deptno=dept.deptno;
> select * from table(dbms_xplan.display);
```

在对目标SQL执行explain plan命令时，Oracle就会将解析SQL所产生的执行计划的具体步骤写入PLAN_TABLE\$，随后执行的`select * from table(dbms_xplan.display)`语句只是从PLAN_TABLE\$中将执行计划以格式化的方式显示出来。各个session往PLAN_TABLE\$写入执行计划的过程互不干扰。

## DBMS_XPLAN包
使用DBMS_XPLAN包查看SQL执行计划可以分为四种情况：

1. 与explain plan for搭配使用（参见上文）：

```sql
select * from table(dbms_xplan.display);
```

2. 在SQLPlus中查看刚刚执行过的SQL的执行计划：

```sql
select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
```

3. 查看指定SQL的执行计划：

这里针对DISPLAY_CURSOR需要传入两个参数，第一个是目标SQL的SQL_ID或者SQL Hash Value，第二个参数是要查看的执行计划所在的Child Cursor Number。
```sql
select * from table(dbms_xplan.display_cursor('sql_id/hash_value',child_cursor_number,'advanced'));
```

只要目标SQL所对应的Child Cursor还在Library Cache中，就可以从`v$sql`中查到它的Child Cursor信息。
```sql
> select sql_text,sql_id,hash_value,child_number from v$sql where sql_text like 'select empno,ename%';
```
其中，CHILD_NUMBER就是对应的Child Cursor Number。

4. 查看指定SQL的所有历史执行计划：

```sql
select * from table(dbms_xplan.display_awr('sql_id'));
```

方法2和方法3中使用DISPLAY_CURSOR能够显示目标SQL执行计划的前提是，对应的执行计划还在Shared Pool中，没有被aged out。如果执行计划已经被置换出Shared Pool，只要该SQL的执行计划被Oracle采集到AWR Repository中，就可以使用方法4来查看执行计划。

## AUTOTRACE开关
在SQLPlus中将AUTOTRACE打开也可以得到目标SQL的执行计划，还能得到目标SQL执行时的资源消耗量。

AUTOTRACE命令的使用方法如下：
```sql
SET AUTOTRACE {OFF|ON|TRACEONLY} [EXPLAIN] [STATISTICS]
```

1. `SET AUTOTRACE ON`

AUTOTRACE默认是OFF关闭的，执行`set autotrace on`可以在当前session中开启autotrace。开启autotrace后，当前session中随后所有执行的SQL不仅会显示执行结果，还会显示SQL对应的执行计划和资源消耗情况。执行`set autotrace off`可以关闭autotrace。

2. `SET AUTOTRACE TRACEONLY`

与SET AUTOTRACE ON相比，省略了SQL执行结果的具体内容，只会显示执行结果的行数、SQL执行计划和资源消耗情况。

3. `SET AUTOTRACE TRACEONLY EXPLAIN`

只显示SQL的执行计划，不会显示SQL执行结果和资源消耗情况。

4. `SET AUTOTRACE TRACEONLY STATISTICS`

只显示SQL执行时的资源消耗情况、以及执行结果的行数，不会显示SQL执行结果的具体内容，也不会显示执行计划。


## 10046事件与tkprof命令
通过10046事件查看执行计划与前几种方法的不同之处在于，可以额外明确地获取目标SQL实际执行计划中每一个步骤所消耗的逻辑读、物理读和花费的时间。

通过10046事件查看执行计划的步骤如下：

1. 在当前session中激活10046事件
2. 在当前session中执行目标SQL
3. 当前session中关闭10046事件

执行完上述步骤后，Oracle就会将目标SQL的执行计划和资源消耗明细写入当前session对应的trace文件中。Oracle会在**USER_DUMP_DEST**对应的目录下生成该trace文件，命名格式为`实例名_ora_当前会话的spid.trc`。

在Oracle中可以通过下面两种方法来激活10046事件：

- 在当前session中执行
```sql
alter session set events '10046 trace name context forover,level 12'
```
- 在当前session中执行
```sql
oradebug event 10046 trace name context forover,level 12
```

其中，level 12表示在产生的trace文件中除了有目标SQL的执行计划和资源消耗明细之外，还会包含目标SQL使用的绑定变量的值、以及经历的等待事件。第二种方法还支持在激活10046事件后通过执行`oradebug tracefile_name`命令来得到当前session对应的trace文件的具体路径和名称。

对应地，在Oracle中关闭10046事件的两种方法为：

- 在当前session中执行
```sql
alter session set events '10046 trace name context off'
```
- 在当前session中执行
```sql
oradebug event 10046 trace name context off
```

10046事件产生的trace文件为裸trace文件，需要使用Oracle自带的tkprof命令翻译后才能看懂。

```sql
> oradebug setmypid     --表示准备对当前session使用oradebug命令
> oradebug event 10046 trace name context forover,level 12

--执行目标SQL
> select empno,ename,dname from scott.emp,scott.dept where emp.deptno=dept.deptno;
...

> oradebug tracefile_name     --查看生成的trace文件路径
/oracle/app/almas/diag/rdbms/almas1/almas1/trace/almas1_ora_4233.trc
> oradebug event 10046 trace name context off
```

翻译trace文件：
```bash
tkprof /oracle/app/almas/diag/rdbms/almas1/almas1/trace/almas1_ora_4233.trc
```
其中，**cr**代表逻辑读（consistent reads），**pr**代表物理读（physical reads），**card**代表结果集的行数（cardinality）。


# 如何得到真实的执行计划
前面介绍的四种查看执行计划的方法中，除了第四种（10046事件），其他三种方法得到的执行计划都有可能是不准确的。

1. explan plan

对于explain plan命令，由于目标SQL并**没有被实际执行**，所以得到的执行计划可能是不准的，尤其是当SQL中包含绑定变量时。在默认开启绑定变量窥探（Bind Peeking）的情况下，在对包含绑定变量的SQL使用explain plan命令时得到只是一个初步的执行计划，Oracle随后通过绑定变量窥探得到这些绑定变量具体的值后，可能会对执行计划做进一步调整。

2. DBMS_XPLAN包

```sql
select * from table(dbms_xplan.display);
select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
select * from table(dbms_xplan.display_cursor('sql_id/hash_value',child_cursor_number,'advanced'));
select * from table(dbms_xplan.display_awr('sql_id'));
```
上面使用DBMS_XPLAN包的四种方式中，第一种方式要与explain plan命令配合使用，因此可能是不准的。后面三种方式得到的执行计划都是准的，因为目标SQL都已经被**实际执行过**了。

3. AUTOTRACE

```sql
set autotrace on
set autotrace traceonly
set autotrace traceonly explain
```
上面使用AUTOTRACE的三种方式中，使用`set autotrace on`或`set autotrace traceonly`时，目标SQL都会**被实际执行**。使用`set autotrace traceonly explain`时，**如果是SELECT语句，就不会被实际执行；但如果是DML语句，则会被实际执行**。

```sql
> select count(*) from emp where ename='Chiwawa';
> select sql_text,executions from v$sqlarea where sql_text like 'select count(*) from emp%';
--EXECUTIONS的值为1，表示对应SQL执行了一次

> alter system flush shared_pool;   --清空共享池
> select sql_text,executions from v$sqlarea where sql_text like 'select count(*) from emp%';
--结果为空

> set autotrace traceonly explain
> select count(*) from emp where ename='Chiwawa';
> select sql_text,executions from v$sqlarea where sql_text like 'select count(*) from emp%';
--EXECUTIONS的值为0，表示未被实际执行

> delete from emp where ename='Chiwawa';
1 row deleted.
> set autotrace off
> select sql_text,executions from v$sqlarea where sql_text like 'select count(*) from emp%';
--EXECUTIONS的值为1
```

虽然使用部分AUTOTRACE命令时，目标SQL会被实际执行，但是所有使用SET AUTOTRACE命令得到的执行都有可能不准确，包括`set autotrace on`和`set autotrace traceonly`。因为**使用AUTOTRACE得到的执行计划都是来源于调用explain plan命令**。

```sql
--创建测试表
> create table t1 as select * from dba_objects;
> insert into t1 select * from t1;
61553 rows inserted.
> commit;
> select count(*) from t1;
COUNT(*)
--------
 123106

--创建索引并收集统计信息
> create index idx_t1 on t1(object_id);
> exec dbms_stats.gather_stats(ownname => 'SCOTT', tabname => 'T1', estimate_percent => 100, cascade => true);

--创建绑定变量并赋值
> var x number;
> var y number;
> exec :x := 0;
> exec :y := 100000;

--通过explain plan查看执行计划
> explain plan for select count(*) from t1 where pbject_id between :x and :y;
> select * from table(dbms_xplan.display);
...
Plan hash value: 3858015043
...
INDEX RANGE SCAN | IDX_T1     --//走索引范围扫描

--查看真实的执行计划
> select count(*) from t1 where pbject_id between :x and :y;
> select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
...
Plan hash value: 3581975568
...
INDEX FAST FULL SCAN | IDX_T1     --//走索引快速全扫描

--通过AUTOTRACE查看执行计划
> set autotrace traceonly
> select count(*) from t1 where pbject_id between :x and :y;
...
Plan hash value: 3858015043     --//执行计划与explain plan方法的相同
...
INDEX RANGE SCAN | IDX_T1     --//走索引范围扫描

> set autotrace on
> select count(*) from t1 where pbject_id between :x and :y;
...
Plan hash value: 3858015043     --//执行计划与explain plan方法的相同
...
INDEX RANGE SCAN | IDX_T1     --//走索引范围扫描
```

通过测试可以看到，使用AUTOTRACE得到的执行计划与explain plan方法得到的是相同的，都是不准确的。

# 执行计划的执行顺序

在使用上面介绍的方法得到执行计划的具体步骤后，按照如下顺序阅读：先从最左一直连续往右看，看到最右边并列的地方；**对于并列的步骤，上面的先执行；对于不并列的步骤，靠右的先执行**。

```sql
Id | Operation                        | Name
0  | SELECT STATEMENT                 |
1  |  SORT AGGREGATE                  |
2  |   CONCATENATION                  |
3  |    MERGE JOIN CARTESIAN          |
4  |     TABLE ACCESS FULL            | OBIWAN
5  |     BUFFER SORT                  |
6  |      TABLE ACCESS FULL           | LUKE
7  |    FILTER                        |
8  |     HASH JOIN                    |
9  |      TABLE ACCESS FULL           | OBIWAN
10 |      INDEX FAST FULL SCAN        | IDX_LUKE_TK
11 |     SORT AGGREGATE               |
12 |      TABLE ACCESS BY INDEX ROWID | LUKE
13 |       INDEX RABGE SCAN           | IDX_LUKE_TK
```

上面步骤的执行顺序为：
```
4-6-5-3-9-10-8-13-12-11-7-2-1-0
```



