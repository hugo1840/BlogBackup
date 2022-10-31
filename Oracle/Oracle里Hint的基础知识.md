---
tags: [oracle, SQL优化]
title: Oracle里Hint的基础知识
created: '2022-10-20T07:24:53.999Z'
modified: '2022-10-31T03:14:43.247Z'
---

Oracle里Hint的基础知识

Hint是Oracle数据库里SQL优化的终极手段，通常用于直接指定目标SQL的执行计划。

# Hint是什么
Hint实际上是一种特殊的注释，它以一种固定的格式和位置出现在SQL语句文本中，它可以影响优化器对于执行计划的选择，但这种影响**不是强制性的**，优化器在某些情况下可能会忽略目标SQL中的Hint，即使这个Hint在语法和语义上都是有效的。

## Hint对优化器的影响
如果目标SQL文本中出现了Hint，则优化器在面临各种可能的、不同的执行路径时，其判断标准将不再是其原先的判断标准，而是会将Hint一并考虑进来。如果优化器判断这个Hint给出的建议是合适的，可以应用，就会直接遵循这个Hint给出的建议来选择执行计划；反之，优化器就会忽略该Hint，仍然采用原来的标准来选择执行计划（比如CBO优化器会选择成本值最小的执行路径）。

也就是说，Hint能够被优化器遵循的前提条件是，Hint所给出的建议是优化器在面临各种不同执行路径的选择时确实能够用上的。如果Hint建议的执行路径实际上是不存在的，就会被优化器忽略。

针对优化器的Hint实际上是可以直接影响优化器解析目标SQL产生执行计划这个过程中的每一步的，具体来说就是：
- Hint可以影响目标SQL是否能够被查询改写，比如`NO_QUERY_TRANSFORMATION`、`MERGE`、`UNNEST`、`USE_CONCAT`等；
- Hint可以影响优化器对于执行路径的选择，比如`FULL`、`INDEX`等；
- Hint可以影响优化器对于表连接方法的选择，比如`USE_HASH`、`USE_NL`等；
- Hint可以影响优化器对于执行计划中执行步骤返回结果集cardinality的判断，比如`DYNAMIC_SAMPLING`、`CARDINALITY`等。

对于大多数Hint，在其开头加上`NO_`后就能形成含义完全相反的Hint，比如与`INDEX`对应的反义词Hint就是`NO_INDEX`，与`UNNEST`对应的反义词Hint就是`NO_UNNEST`。

如果在目标SQL中使用了Hint，就意味着启用了CBO优化器，即Oracle会以CBO来解析包含Hint的目标SQL。仅有的两个例外是**RULE Hint**和**DRIVING_SITE Hint**，它们可以在RBO优化器下使用，同时不启用CBO。

>:warning:在Oracle 10g及其以后的版本中，参数`OPTIMIZER_MODE`的默认值是`ALL_ROWS`，这表示默认情况下已开启CBO优化器。


## 与优化器无关的Hint
Oracle数据库里的Hint并不都是针对优化器的。**不是针对优化器的Hint**包括以下几类：
- APPEND Hint：用于控制INSERT语句是否能以直接路径插入的方式插入数据。
- STATEMENT_QUEUING Hint：用于控制目标SQL在执行时是否采用并行执行排队，在参数`PARALLEL_DEGREE_POLICY`没有被设置为AUTO的情况下，该Hint依然可以让目标SQL以并行执行排队的方式执行。
- CACHE Hint：用于控制目标SQL在执行时是否将全表扫描目标表时的数据块放到Buffer Cache的LRU链表的热端（most recently used end）。
- MONITOR Hint：用于控制被执行的目标SQL是否会被SQL Monitor监控。
- GATHER_PLAN_STATISTICS Hint：用于在目标SQL执行时收集一些额外的统计信息，比如**每一个具体执行步骤的**实际返回结果的cardinality、实际执行时间、以及实际消耗的逻辑读和物理读，等等。

其中，GATHER_PLAN_STATISTICS Hint在SQL优化中是非常有用的手段。


# Hint的用法
## 一般格式与位置
Hint实际上是一种特殊的注释，以一种固定的格式和位置出现在SQL文本中。

Hint必须以如下格式出现在SQL文本中：
```sql
{SELECT | INSERT | UPDATE | DELETE | MERGE} /*+ 具体的Hint内容 */
```
固定位置是指Hint在SQL文本中必须紧随关键字SELECT、INSERT、UPDATE、DELETE或者MERGE之后。

此外Hint还可以以如下格式出现在SQL文本中：
```sql
--+ 具体的Hint内容
SQL文本
```
但是这种格式要求在**Hint后面的SQL文本必须另起一行**，不是很方便，所以不太常用。

关于第一种Hint格式，有以下几点需要注意：
- Hint中第一个星号和加号之间**不能有空格**；
- Hint的加号和具体的Hint内容之间可以有空格，也可以没有空格；
- Hint中的具体内容可以是单个Hint，也可以是**多个Hint的组合**。如果是后者，则各个Hint之间至少需要一个空格来彼此分隔。

## Hint中指定具体对象
在Hint中指定具体对象（比如指定表名或者索引名称）时，**不能带上该对象所在SCHEMA的名称**，即使该SQL语句文本中已经有对应的SCHEMA名称。如果Hint中带上了对象所在SCHEMA的名称，会导致Hint被忽略。

```sql
--正确的写法
> select /*+ full(emp) */* from scott.emp where empno=7369;
--错误的写法，Hint会失效
> select /*+ full(scott.emp) */* from scott.emp where empno=7369;
```

在Hint中指定具体表名时，如果该表在对应SQL文本中有别名（alias）时，**应该使用表的别名**。如果Hint中没有使用表的别名，会导致Hint被忽略。

```sql
--正确的写法
> select /*+ full(t1) */* from scott.emp t1 where empno=7369;
--错误的写法，Hint会失效
> select /*+ full(emp) */* from scott.emp t1 where empno=7369;
```

## Hint的生效范围
Oracle数据库中Hint的生效范围**仅限于它本身所在的Query Block**。

Oracle数据库中的**Query Block**是指**一个语义上完整的查询语句**。它可以是一个子查询所对应的SQL语句，也可以是一个视图所对应的视图定义语句。目标SQL可以包含一个或多个Query Block。当目标SQL只包含一个Query Block时，这个Query Block就是目标SQL本身。

如果想将Hint生效的范围扩展到它所在的Query Block之外，而又没有在该Hint中指定其生效的Query Block名称的话，则Oracle会忽略该Hint。

```sql
--正确的写法
> select /*+ full(t1) */t1.ename,t1.deptno from emp t1 
where t1.deptno in
(select /*+ full(t2) */t2.deptno from dept t2 
where t2.loc='VALINTINE'); 

--错误的写法，full(t2)跨了Query Block会失效
> select /*+ full(t1) full(t2) */t1.ename,t1.deptno from emp t1 
where t1.deptno in
(select t2.deptno from dept t2 where t2.loc='VALINTINE'); 
```

对于目标SQL中的Query Block，Oracle会对其中没有被我们自定义的所有Query Block自动命名。自定命名的规则为`类型$数字`，如果同类型的Query Block不止一个，自动命名里的数字就会递增。查询语句里的Query Block类型对应的关键字就是**SEL**。对于上面例子中的SQL，外部查询对应Query Block的自动命名就是`SEL$1`，子查询对应Query Block的自动命名就是`SEL$2`。

除了自动命名之外，还可以使用**QB_NAME Hint**来为一个Query Block指定自定义的名称，其格式为
```sql
/*+ QB_NAME(自定义的Query Block名称) */
```

通过指定Query Block的名称，可以使得Hint在跨Query Block时仍然生效，从而将目标SQL的所有Hint都写在一起。在Hint中出现的Query Block格式必须是`@Query Block名称`。可以通过以下两种方式指定Query Block：

- 在Hint的目标对象**前**额外加上该Hint生效的Query Block名称，它们之间用空格分隔。
```sql
--使用QB自动命名，且QB名称在对象前
> select /*+ full(@sel$1 t1) full(@sel$2 t2) */t1.ename,t1.deptno 
from emp t1 where t1.deptno in
(select t2.deptno from dept t2 where t2.loc='VALINTINE'); 

--使用QB自定义命名，且QB名称在对象前
> select /*+ full(@sel$1 t1) full(@blackwater t2) */t1.ename,t1.deptno 
from emp t1 where t1.deptno in
(select /*+ qb_name(blackwater) */t2.deptno 
from dept t2 where t2.loc='VALINTINE'); 
```

- 在Hint的目标对象**后**额外加上该Hint生效的Query Block名称，它们之间是连在一起的，不做任何分隔。
```sql
--使用QB自动命名，且QB名称在对象后
> select /*+ full(t1@sel$1) full(t2@sel$2) */t1.ename,t1.deptno 
from emp t1 where t1.deptno in
(select t2.deptno from dept t2 where t2.loc='VALINTINE'); 

--使用QB自定义命名，且QB名称在对象后
> select /*+ full(t1@sel$1) full(t2@blackwater) */t1.ename,t1.deptno 
from emp t1 where t1.deptno in
(select /*+ qb_name(blackwater) */t2.deptno 
from dept t2 where t2.loc='VALINTINE'); 
```

# Hint被忽略的常见情形
Oracle在10g及其以后的版本中引入了一个隐含参数`_OPTIMIZER_IGNORE_HINTS`，它可以在系统或者session级别设置，用来控制是否忽略SQL文本中的Hint，其默认值为FALSE。如果把它设置为TRUE，则Oracle就会忽略SQL文本中的所有Hint，即相当于此时所有的Hint都失效了。

除了该隐含参数以外，还存在各种原因可能会导致Hint被忽略或者失效。需要注意的是，Hint被忽略后Oracle并不会给出任何提示或者警告，更不会报错，目标SQL依然可以正常执行。

## 使用的Hint有语法或者拼写错误
一旦使用的Hint中有语法错误或者拼写错误，Oracle就会忽略该Hint。
```sql
--错误示例
> select /*+ ind(emp pk_emp) */* from emp;   --//Hint关键字拼写错误
> select /*+ index(emp pk_emp */* from emp;   --//漏了右括号
> select /* + index(emp pk_emp) */* from emp;   --//星号和加号之间有空格
> select */*+ index(emp pk_emp) */ from emp;   --//Hint没有紧跟select关键字
> select /*+ index(scott.emp pk_emp) */* from emp;   --//Hint里不能带对象的schema名称
> select /*+ index(emp pk_emp) */* from emp e;   --//没有使用表的别名
> select /*+ index(emp emp_pk) */* from emp;   --//索引名称写错
> select /*+ full(t2) */t1.ename,t1.deptno 
from emp t1 where t1.deptno in
(select t2.deptno from dept t2 where t2.loc='VALINTINE'); 
--//跨Query Block导致Hint无效
```

## 使用的Hint无效
即使语法是正确的，但如果由于某种原因导致Oracle认为该Hint无效，则Oracle还是会忽略该Hint。

1. 单键值B树索引与NULL值

```sql
> desc dept;
Name   Type         Nullable
----   -----------  --------
DEPTNO NUMBER(2)
DNAME  VARCHAR(14)      Y
LOC    VARCHAR(13)
> set autotrace traceonly
--查询条件中包含列LOC
> select /*+ index(dept idx_dept_loc) */deptno,dname from dept where loc='Rhodes';
执行计划
INDEX RANGE SCAN | IDX_DEPT_LOC

/*
LOC列为NOT NULL，即表dept的所有行记录的rowid都可以通过idx_dept_loc
扫描回表获得，故仍然可以走idx_dept_loc索引，即使查询条件中不包含列LOC
*/
> select /*+ index(dept idx_dept_loc) */deptno,dname from dept where deptno=30;
执行计划
INDEX FULL SCAN | IDX_DEPT_LOC

/*
LOC列为NULL，即通过idx_dept_loc扫描回表不能保证获得表dept的所有行
记录的rowid，故不能再走idx_dept_loc索引
*/
> alter table dept modify (loc null);
> select /*+ index(dept idx_dept_loc) */deptno,dname from dept where deptno=30;
执行计划
INDEX UNIQUE SCAN | PK_DEPT
```
在Oracle数据库中，**NULL值是不存储在单键值B树索引中的**，所以在LOC列属性为允许NULL时，对单键值B树索引的索引全扫描并不能保证扫描结果中会包含表DEPT所有行记录对应的ROWID，因此`/*+ index(dept idx_dept_loc) */`就没有意义了，会被Oracle忽略。

2. 指定对象不存在的情况

当Hint指定的索引不存在时，该Hint也会被Oracle忽略。
```sql
> drop index idx_dept_loc;
> select /*+ index(dept idx_dept_loc) */deptno,dname from dept where loc='Rhodes';
执行计划
TABLE ACCESS FULL | DEPT
```

3. 无法并行的情况

对于**非分区索引**而言，索引范围扫描和索引全扫描都**不能并行**执行。
```sql
--全表扫描可以并行执行
> select /*+ full(dept) parallel(dept 2) */deptno from dept;
执行计划
...
PX BLOCK ITERATOR
 TABLE ACCESS FULL | DEPT

--非分区索引扫描不能并行，导致Hint被忽略
> select /*+ index(dept pk_dept) parallel(dept 2) */deptno from dept;
执行计划
NDEX FULL SCAN | PK_DEPT
```

4. 哈希连接的被驱动表

哈希连接的Hint关键字`USE_HASH(目标表)`中目标表是指哈希连接中的**被驱动表**，而哈希连接的被驱动表应该是**数据量更多的那个表**。

```sql
--表emp的数据量更大，目标表应该是t1，故哈希连接Hint被忽略
> select /*+ use_hash(t2) */t1.empno,t1.ename,t2.loc from emp t1, dept t2
where t1.deptno = t2.deptno and t2.loc='Rhodes';
执行计划
MERGE JOIN                        |
 TABLE ACCESS FULL BY INDEX ROWID | DEPT
  INDEX FULL SCAN                 | PK_DEPT
 SORT JOIN                        |
  TABLE ACCESS FULL               | EMP
```

5. 哈希连接与不等值连接

**哈希连接只适用于等值连接条件**，不等值的连接条件对于哈希连接而言是没有意义的。
```sql
--不等值连接条件导致哈希连接Hint被忽略
> select /*+ use_hash(t1) */t1.empno,t1.ename,t2.loc from emp t1, dept t2
where t1.deptno >= t2.deptno and t2.loc='Rhodes';
执行计划
NESTED LOOPS       |
 TABLE ACCESS FULL | DEPT
 TABLE ACCESS FULL | EMP
```

## 使用的Hint自相矛盾
如果使用的Hint组合是自相矛盾的，那么这些互相矛盾的Hint就会被Oracle忽略，而其他有效Hint不会被忽略。

```sql
--全表扫描和索引快速全扫描两个Hint互相矛盾，最终走的是索引全扫描
> select /*+ full(dept) index_ffs(dept pk_dept) */deptno from dept;
执行计划
Operation       | Name    | Rows
INDEX FULL SCAN | PK_DEPT | 4

--Hint被忽略以后，重复添加也不会生效
> select /*+ full(dept) index_ffs(dept pk_dept) full(dept) */deptno from dept;
执行计划
Operation       | Name    | Rows
INDEX FULL SCAN | PK_DEPT | 4

--cardinality与前面的Hint不矛盾，所以生效了
> select /*+ full(dept) index_ffs(dept pk_dept) cardinality(dept 100) */deptno from dept;
执行计划
Operation       | Name    | Rows
INDEX FULL SCAN | PK_DEPT | 100
```

## 使用的Hint受到了查询转换的干扰

```sql
> create table jobs as select empno,job from emp;

/*
ordered表示表连接顺序应该与表在SQL文本中出现的顺序一致，即emp->jobs->视图v
cardinality用于直接指定emp扫描结果集的势
merge表示对内嵌视图v做视图合并
*/
> select /*+ ordered cardinality(e 100) */
e.ename, j.job, e.sal, v.avg_sal
from emp e, jobs j, 
(select /*+ merge */e.deptno, avg(e.sal) avg_sal
from emp e, dept d
where d.loc='Rhodes' and d.deptno=e.deptno
group by e.edeptno) v
where e.empno = j.empno
and e.deptno = v.deptno
and e.sal > v.avg_sal
order by e.ename;
执行计划
SELECT STATEMENT                  |        |
 FILTER                           |        |
  SORT GROUP BY                   |        |
   HASH JOIN                      |        |
    TABLE ACCESS FULL             | DEPT   |
    HASH JOIN                     |        |
     MERGE JOIN                   |        |
      TABLE ACCESS BY INDEX ROWID | EMP    | 100
       INDEX FULL SCAN            | PK_EMP |
      SORT JOIN                   |        |
       TABLE ACCESS FULL          | JOBS   |
     TABLE ACCESS FULL            | EMP    |
/*
cardinality 100和merge视图合并的Hint都已生效，但是表连接顺序不是
emp->jobs->视图v，而是最先选择了表dept，即ordered被忽略了
*/

--//no_merge表示对内嵌视图v不做视图合并
> select /*+ ordered cardinality(e 100) */
e.ename, j.job, e.sal, v.avg_sal
from emp e, jobs j, 
(select /*+ no_merge */e.deptno, avg(e.sal) avg_sal
from emp e, dept d
where d.loc='Rhodes' and d.deptno=e.deptno
group by e.edeptno) v
where e.empno = j.empno
and e.deptno = v.deptno
and e.sal > v.avg_sal
order by e.ename;
/*
取消视图合并后，表连接顺序变成了emp->jobs->视图v，即ordered生效了
*/
```

## 使用的Hint受到了保留关键字的干扰
Oracle在解析目标SQL时，是按照**从左到右**的顺序进行的，**如果遇到的词是Oracle的保留关键字，则Oracle将忽略这个词以及之后的所有词**；**如果遇到的词既不是保留关键字，也不是Hint，就忽略该词**；如果遇到的词是有效的Hint，那么就会保留该Hint。

Oracle的保留关键字可以从视图`V$RESERVED_WORDS`中查到，比如**逗号**、COMMENT和IS都是保留关键字。
```sql
> select keyword,length from v$reserved_words where keyword 
in (',','THIS','IS','COMMENT');
```

组合Hint中各个Hint应该使用空格来连接，而不是逗号。
```sql
--逗号之后的Hint被忽略
> select /*+ use_hash(t1), index(t2 pk_dept) */t1.empno,t1.ename,t2.loc 
from emp t1, dept t2 where t1.deptno=t2.deptno;
执行计划
HASH JOIN          |
 TABLE ACCESS FULL | DEPT
 TABLE ACCESS FULL | EMP
```

保留关键字之后的所有Hint都会被忽略。
```sql
--保留关键字comment后所有单个Hint都被忽略
> select /*+ comment use_hash(t1) full(t2) */t1.empno,t1.ename,t2.loc 
from emp t1, dept t2 where t1.deptno=t2.deptno;
执行计划
MERGE JOIN                   |
 TABLE ACCESS BY INDEX ROWID | DEPT
  INDEX FULL SCAN            | PK_DEPT
 SORT JOIN                   |
  TABLE ACCESS FULL          | EMP
```

非保留关键字对Hint无影响。
```sql
> select /*+ this_is_example use_hash(t1), index(t2 pk_dept) */t1.empno,
t1.ename,t2.loc from emp t1, dept t2 where t1.deptno=t2.deptno;
执行计划
HASH JOIN                    |
 TABLE ACCESS BY INDEX ROWID | DEPT
  INDEX FULL SCAN            | PK_DEPT
 TABLE ACCESS FULL           | EMP
```

尽量不要在组合Hint中同时混写有效的Hint和自定义的注释。如果非要同时写，最好把自定义的注释写到最右边末尾。

```sql
> select /*+ use_hash(t1), index(t2 pk_dept), this is comment */t1.empno,
t1.ename,t2.loc from emp t1, dept t2 where t1.deptno=t2.deptno;
```




