---
tags: [oracle, SQL优化]
title: Oracle里的绑定变量
created: '2022-10-15T13:22:34.685Z'
modified: '2022-10-19T13:15:00.727Z'
---

Oracle里的绑定变量

绑定变量（Bind Variable）是一种特殊的变量，又被称为占位符（Placeholder）。绑定变量通常出现在目标SQL文本中，用于替换where条件或者INSERT语句的values子句中的具体输入值。Oracle数据库中绑定变量的使用语法是`:variable_name`。这里自定义的变量名称`variable_name`可以是字母、数字或字母与数字的组合。

# 绑定变量的作用
当Oracle在解析和执行目标SQL时，会根据SQL文本的哈希值去库缓存中查找匹配的父游标。只要目标SQL的文本稍有不同，据此计算出来的哈希值就极有可能不同（除非发生哈希碰撞）。因此这些SQL文本稍有不同的SQL语句之间是没有办法重用解析树和执行计划的。

在OLTP系统中，同一类型的SQL可能会并发地被不同用户反复执行，每次执行时只有输入的变量值不相同。在不使用绑定变量的情形下，这些语句计算出来的哈希值不相同，因此彼此之间也不能重用解析树和执行计划，每次执行时都需要硬解析，会严重影响OLTP系统的性能和可扩展性。在使用绑定变量后，这些目标SQL的文本变得完全相同，计算出来的哈希值也一样，就可以重用解析树和执行计划了。

```sql
--不使用绑定变量
> select ename from emp where empno = 7370;
> select ename from emp where empno = 7989;

--使用绑定变量
> select ename from emp where empno = :x;
```

# 绑定变量的用法

## 在SQL语句中使用绑定变量
绑定变量的名称可以是字母、**数字**、或者字母与数字的组合。在SQL语句中定义绑定变量的关键字是**var**，给绑定变量赋值的关键字是**exec**。

```sql
--绑定变量定义
var x number;
var 1 number;   --//绑定变量名称可以是纯数字
var xyz123 number;

--绑定变量赋值
exec :x := 7369;   --//使用绑定变量时前面带上冒号
exec :1 := 7369;   --//使用:=符号给绑定变量赋值
exec :xyz123 := 7369;
```

在SQL语句中使用绑定变量：

```sql
> select ename from emp where empno = :x;
> select ename from emp where empno = :1;
> select ename from emp where empno = :xyz123;
```

## 在PL/SQL代码中使用绑定变量

在PL/SQL代码中**SELECT**语句使用绑定变量的标准语法如下：
```sql
execute immediate [带绑定变量的SQL语句] using [绑定变量的具体输入值]
```

示例：
```sql
declare 
  vc_name varchar2(10);
begin
  execute immediate 'select ename from emp where empno = :1' into vc_name using 7369;
  dbms_output.put_line(vc_name);
end;
/
```

在PL/SQL代码中**DML**语句使用绑定变量的一般方法如下：

```sql
declare
  vc_sql_1 varchar2(4000);
  vc_sql_2 varchar2(4000);
  n_temp_1 number;
  n_temp_2 number;
begin
  vc_sql_1 := 'insert into emp(empno,ename,job) values(:1,:2,:3)';
  execute immediate vc_sql_1 using 7370,'Monica','Cook';
  n_temp_1 := sql%rowcount;

  vc_sql_2 := 'insert into emp(empno,ename,job) values(:1,:1,:1)';
  execute immediate vc_sql_2 using 7371,'Joey','Actor';
  n_temp_2 := sql%rowcount;
  dbms_output.put_line(to_char(n_temp_1+n_temp_2));
  commit;
end;
/
```

与在SQL语句中不同，在PL/SQL代码中给绑定变量赋值的关键字是**using**，并且using后传入的绑定变量具体输入值**只与对应绑定变量在目标SQL所处的位置有关**，而与其名称无关。事实上，上面PL/SQL代码的输出值为2，说明`vc_sql_2`中三个处于不同位置的绑定变量可以是同名的。

下面的PL/SQL代码表示的是删除EMP表里empno列值为7369的行、并将对应行的ename列打印出来。
```sql
declare
  vc_column varchar2(10);
  vc_sql varchar2(4000);
  n_temp number;
  vc_ename varchar2(10);
begin
  vc_column := 'empno';
  vc_sql := 'delete from emp where ' || vc_column || ' = :1 returning ename into :2';
  execute immediate vc_sql using 7369 returning into vc_ename;
  dbms_output.put_line(vc_ename);
  commit;
end;
/
```

从上面的PL/SQL代码可以看出，动态SQL（SQL文本会随vc_column变化）也可以使用绑定变量。关键字**returning**可以和带绑定变量的目标SQL连用，用于提取受DML语句影响的行记录中指定字段的值。

## 在PL/SQL代码中使用批量绑定
PL/SQL中的批量绑定是一种优化后的使用绑定变量的方式。其核心在于：

- PL/SQL中的批量绑定还是会以前面介绍过的常规方法来使用绑定变量；
- PL/SQL中的批量绑定的优势在于一次性处理一批数据，而不是一次只处理一条数据，所以能够有效减少PL/SQL引擎和SQL引擎上下文切换的次数。

可以简单地将PL/SQL引擎看作是Oracle数据库中专门用来处理PL/SQL代码中除SQL语句之外的所有剩余部分，如变量、赋值、循环、数组等。SQL引擎则是专门用来处理SQL语句的子系统。PL/SQL引擎和SQL引擎**上下文切换**就是指两种引擎之间的交互。

理论上在PL/SQL代码中只要执行SQL语句，就会发生引擎的上下文切换，但是对PL/SQL代码性能有影响的交互主要发生在如下两处：

- **显示游标或者参考游标需要循环执行Fetch操作时**。循环操作需要PL/SQL引擎处理，而Fetch操作由SQL引擎处理。在没有优化的情况下，每Fetch一条行记录，就会发生一次引擎切换。
- **显示游标或者参考游标的循环内部需要执行SQL语句时**。在没有优化的情况下，循环内部每执行一次SQL语句，就会发生一次引擎切换。

PL/SQL代码中批量Fetch的语法如下：
```sql
fetch CURSORNAME bulk collect into [自定义数组] <limit CN_BATCH_SIZE>
```
其中，可选关键字`limit CN_BATCH_SIZE`表示一次只批量fetch常量`CN_BATCH_SIZE`所限制的记录数（建议值为1000）。当CURSORNAME对应结果集的记录数很大时，如果一次性把所有的记录全部fetch，可能会给PGA带来很大压力，进而造成无节制占用PGA，甚至撑爆paging space。

PL/SQL代码中一次执行一批SQL语句的语法如下：
```sql
forall i in 1..[自定义数组的长度]
  execute immediate [带绑定变量的目标SQL] using [绑定变量具体输入值]
```
关键字**forall**表示一次执行一批SQL语句，它可以和INSERT、UPDATE和DELETE语句联用。

参考以下的使用范例：
```sql
declare
  cur_emp sys_refcursor;     --//定义参考游标
  vc_sql varchar2(4000);
  type namelist is table of varchar2(10);   --//定义集合类型namelist
  enames namelist;
  CN_BATCH_SIZE constant pls_integer := 1000;
begin
  vc_sql := 'select ename from emp where empno > :1';
  open cur_emp for vc_sql using 7900;
  loop
    fetch cur_emp bulk collect into enames limit CN_BATCH_SIZE;
    
    for i in 1..enames.count loop
      dbms_output.put_line(enames(i));
    end loop;
    
    exit when enames.count < CN_BATCH_SIZE;
  end loop;
  close cur_emp;
end;
/
```

绑定变量和批量绑定同样可以用于应用系统开发，比如Java和.NET语言。

# 绑定变量使用的最佳实践
对于绑定变量，推荐的使用原则为：依据应用系统的类型来决定是否使用绑定变量。

- 对于OLAP/DSS类型的应用系统，可以不使用绑定变量； 
- 对于OLTP类型的应用系统，在SQL语句中**一定要使用绑定变量**，并且最好使用批量绑定，尽可能在前台代码和后台PL/SQL代码中都使用绑定变量；
- 对于OLAP和OLTP混合型的应用系统，如果有循环，不管这个循环是在前台代码还是在后台PL/SQL代码中，**循环内部的SQL语句一定要使用绑定变量**，并且最好使用批量绑定；至于循环外部的SQL语句，可以不使用绑定变量。

# 绑定变量窥探
随着具体输入值的不同，目标SQL的where条件的可选择率（selectivity）和结果集的势（cardinality）也会随之发生变化。Selectivity和Cardinality的值又会直接影响CBO优化器对执行成本的估算，进而影响CBO对执行计划的选择。

对于使用了绑定变量的SQL而言，Oracle可以通过以下两种方法来决定执行计划：
- 使用绑定变量窥探；
- 如果不使用绑定变量窥探，则对于那些可选择率可能会随着具体输入值的不同而变化的谓词条件使用默认的可选择率（例如5%）。

**绑定变量窥探**（Bind Peeking）是在Oracle 9i中引入的，是否启用绑定变量窥探受隐含参数`_OPTIM_PEEK_USER_BINDS`的控制，其默认值是**TRUE**。

启用绑定变量窥探以后，每当Oracle以**硬解析**的方式解析包含绑定变量的目标SQL时，Oracle都会实际窥探一下对应绑定变量的具体输入值，并以这些具体输入值为标准，来决定目标SQL的where条件的selectivity和cardinality，并据此来选择执行计划。

>:apple:**注意**：绑定变量窥探只会发生在**硬解析**时。当目标SQL再次执行（对应软解析或软软解析）时，**即使绑定变量的具体输入值发生了变化**，也会沿用硬解析时的解析树和执行计划，而不会再次执行绑定变量窥探。

绑定变量窥探这种不管后续传入的具体输入值是什么、而一直沿用之前硬解析时所产生的解析树和执行计划的特性一直饱受诟病（这种情况一直持续到Oracle 11g中引入自适应游标共享后才有所缓解）。因为它可能使得CBO优化器在某些情况下所选择的执行计划并不是当前的最优解，而且它可能会带来目标SQL执行计划的突然改变，进而直接影响应用系统性能。

```sql
--创建测试表并收集统计信息
> create table t1 as select * from dba_objects;
> create index idx_t1 on t1(object_id);
> select count(*) from t1;
COUNT(*)
--------
  72389
> select count(distinct(object_id)) from t1;
COUNT(DISTINCT(OBJECT_ID))
--------------------------
        72387
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT',tabname => 'T1', estimate_percent => 100,cascade => true,method_opt =>'for all columns size 1',no_invalidate => false);

--执行目标SQL（不使用绑定变量）
> select count(*) from t1 where object_id between 999 and 1000;
COUNT(*)
--------
   2
> select count(*) from t1 where object_id between 999 and 60000;
COUNT(*)
--------
  58180

--查看SQL_ID和游标数量
> select sql_text,sql_id,version_count from v$sqlarea where sql_text like 'select count(*) from t1%';
SQL_TEXT                                            SQL_ID        VERSION_COUNT
--------------------------------------------------  ------------- -------------
select count(*) from t1 ... between 999 and 1000;   7mp5tw7cfn51h       1
select count(*) from t1 ... between 999 and 60000;  dn00m91bywub3       1
```
其中，`verison_count`表示某个父游标所拥有的子游标的数量。可以看出，上述两条目标SQL在执行时都使用了硬解析，各自生成了一个父游标和一个子游标。

```sql
--查看执行计划
> select * from table(dbms_xplan.display_cursor('7mp5tw7cfn51h',0,'advanced'));
...
INDEX RANGE SCAN | IDX_T1 | 3     --走的索引范围扫描
> select * from table(dbms_xplan.display_cursor('dn00m91bywub3',0,'advanced'));
...
INDEX FAST FULL SCAN | IDX_T1 | 49843     --走的索引快速全扫描
```

假设我们使用绑定变量来执行第一条目标SQL：
```sql
> var x number;
> var y number;
> exec :x := 999;
> exec :y := 1000;

--执行目标SQL（使用绑定变量）
> select count(*) from t1 where object_id between :x and :y;
COUNT(*)
--------
   2

--查看SQL_ID和游标数量
> select sql_text,sql_id,version_count from v$sqlarea where sql_text like 'select count(*) from t1%';
SQL_TEXT                                            SQL_ID        VERSION_COUNT
--------------------------------------------------  ------------- -------------
select count(*) from t1 ... between 999 and 1000;   7mp5tw7cfn51h       1
select count(*) from t1 ... between 999 and 60000;  dn00m91bywub3       1
select count(*) from t1 ... between :x and :y;      btkxfccx5rkdc       1

--查看执行计划
> select * from table(dbms_xplan.display_cursor('btkxfccx5rkdc',0,'advanced'));
...
INDEX RANGE SCAN | IDX_T1 | 3     --走的索引范围扫描
...
Peeked Binds (identified by position):
 1 - :x (NUMBER): 999
 2 - :y (NUMBER): 1000
```

修改绑定变量的值后，使用绑定变量来执行第二条目标SQL：
```sql
> exec :y := 60000;
> select count(*) from t1 where object_id between :x and :y;
COUNT(*)
--------
  58180

--查看SQL_ID和游标数量
> select sql_text,sql_id,version_count from v$sqlarea where sql_text like 'select count(*) from t1%';
SQL_TEXT                                            SQL_ID        VERSION_COUNT EXECUTIONS
--------------------------------------------------  ------------- ------------- ----------
select count(*) from t1 ... between 999 and 1000;   7mp5tw7cfn51h       1           1
select count(*) from t1 ... between 999 and 60000;  dn00m91bywub3       1           1
select count(*) from t1 ... between :x and :y;      btkxfccx5rkdc       1           2
--//version_count还是1，说明没有生成新的子游标（执行计划）

--查看执行计划
> select * from table(dbms_xplan.display_cursor('btkxfccx5rkdc',0,'advanced'));
...
INDEX RANGE SCAN | IDX_T1 | 3     --变成了索引范围扫描
...
Peeked Binds (identified by position):
 1 - :x (NUMBER): 999
 2 - :y (NUMBER): 1000       --值没有变，本次执行SQL时没有进行绑定变量窥探
```

如果想让目标SQL再次执行时使用硬解析，可以使用`DBMS_SHARED_POOL.PURGE`从库缓存中删除掉指定的shared cursor。
```sql
> select sql_text,sql_id,address,hash_value from v$sqlarea where sql_text like 'select count(*) from t1%';
SQL_TEXT                                            SQL_ID        ADDRESS HASH_VALUE
--------------------------------------------------  ------------- ------- ----------
select count(*) from t1 ... between 999 and 1000;   7mp5tw7cfn51h 2EC75CC 362678438
select count(*) from t1 ... between 999 and 60000;  dn00m91bywub3 358C7B4 954837836
select count(*) from t1 ... between :x and :y;      btkxfccx5rkdc 2ED21DB 756623893

--通过address和hash_value删除指定shared cursor（字母c表示删除）
> exec sys.dbms_shared_pool.purge('2ED21DB,756623893','c');
```


# 绑定变量分级
绑定变量分级（Bind Graduation）是指Oracle在PL/SQL代码中会根据**文本型**绑定变量的定义长度而将这些文本型绑定变量分为四个等级。

- 定义长度在**32**字节（byte）以内的文本型绑定变量被划分在第一个等级；
- 定义长度在33到**128**字节之间的文本型绑定变量被划分在第二个等级；
- 定义长度在129到**2000**字节之间的文本型绑定变量被划分在第三个等级；
- 定义长度在2000字节以上的文本型绑定变量被划分在第四个等级。

在执行目标SQL时，session cursor会根据文本型绑定变量的等级为其在PGA中分配对应大小的内存空间。

- 第一等级的文本型绑定变量会被分配32字节内存空间；
- 第二等级的文本型绑定变量会被分配128字节内存空间；
- 第三等级的文本型绑定变量会被分配2000字节内存空间；
- 第四等级的文本型绑定变量会被分配的内存空间大小，取决于对应绑定变量所传入的实际绑定变量值的大小。如果实际传入的绑定变量值不超过2000字节，就会分配2000字节内存空间；如果超过了2000字节，Oracle就会为其分配4000字节内存空间。

>:horse:绑定变量分级仅适用于文本型的绑定变量。Oracle数据库中数值型的绑定变量最大只能占用22字节，Oracle统一为其分配了22字节的内存空间。

对于PL/SQL代码中那些使用了文本型绑定变量的目标SQL，只要文本型绑定变量的**定义长度**发生了变化，Oracle为这些文本型绑定变量所分配的内存空间也可能随之发生变化，从而导致存储在child cursor中的解析树和执行计划不能被重用了。其原因在于，**child cursor中除了解析树和执行计划之外，还会存储目标SQL使用的绑定变量的类型和长度**。这意味着即使SQL文本没有发生任何改变，只要SQL文本中文本型绑定变量的定义长度发生了变化，那么目标SQL再次执行时还可能需要做硬解析。

# 绑定变量个数不宜太多
目标SQL的SQL文本中绑定变量的个数不宜太多，否则可能会导致目标SQL总的执行时间大幅度增长。增长的时间主要耗费在执行目标SQL时对每一个绑定变量都用其实际的值来替换。绑定变量个数越多，这个替换的过程所耗费的时间就越长，目标SQL总的执行时间也就越长。

# 如何获取已执行SQL中绑定变量的值
如果想要获取已执行过的目标SQL中绑定变量的实际输入值，可以查询视图`V$SQL_BIND_CAPTURE`。如果该视图中查不到，那么有可能对应的shared cursor已经被age out出shared pool了。此时需要去AWR Repository相关的数据字典表`DBA_HIST_SQLSTAT`或`DBA_HIST_SQLBIND`中查询。

当Oracle解析和执行含有绑定变量的目标SQL时，如果满足以下两个条件之一，则该SQL中的绑定变量的具体输入值就会被Oracle捕获，并可以通过视图`V$SQL_BIND_CAPTURE`查询。

- 当含有绑定变量的目标SQL以**硬解析**的方式执行时。
- 当含有绑定变量的目标SQL以软解析或软软解析的方式重复执行时，该SQL中的绑定变量的具体输入值也可能会被捕获，只不过默认情况下这种捕获操作Oracle**至少得间隔15分钟**才会做一次（以避免影响系统性能）。

**Oracle只会捕获位于目标SQL的where条件中的绑定变量的具体输入值**。对于使用了绑定变量的INSERT语句，无论什么情况下，Oracle都不会捕获其values子句中对应绑定变量的具体输入值。

```sql
--执行包含INSERT语句的存储过程
declare
 ...     --//此处省略部分代码
 execute immediate 'insert into t values (:n, :v)' using n,v;
 commit;
end;
/

--查看SQL_ID
> select sql_text,sql_id,version_count,executions 
from v$sqlarea where sql_text like 'insert into t%';
SQL_TEXT                       SQL_ID        VERSION_COUNT  EXECUTIONS
-----------------------------  ------------  -------------  ----------
insert into t values (:n, :v)  79w509gxt352        4             6

--查询视图V$SQL_BIND_CAPTURE
> select sql_id,name,position,datatype_string,last_captured,value_string 
from v$sql_bind_capture where sql_id='79w509gxt352';
--//Oracle不会捕获insert语句的绑定变量输入值，所以value_string的值都为空

--查询数据字典DBA_HIST_SQLSTAT
> select snap_id,
dbms_sql_tune.extract_bind(bind_data,1).value_string bind1,
dbms_sql_tune.extract_bind(bind_data,2).value_string bind2
from dba_hist_sqlstat where sql_id='79w509gxt352' order by snap_id;
--//bind1和bind2的值都为空

--查询数据字典DBA_HIST_SQLBIND
> select snap_id,name,position,value_string,last_captured,was_captured
from dba_hist_sqlbind  where sql_id='79w509gxt352' order by snap_id;
--//value_string为空，last_captured为NO，was_captured为空
```


