---
tags: [oracle, SQL优化]
title: Oracle优化器基础之访问数据的方法
created: '2022-10-01T07:54:00.525Z'
modified: '2022-10-02T15:34:14.542Z'
---

Oracle优化器基础之访问数据的方法

# 优化器的模式
优化器的模式是指Oracle数据库中解析目标SQL时所用的优化器类型，决定了使用CBO计算成本值的侧重点。优化器模式由参数`OPTIMIZER_MODE`控制，取值可以为RULE、CHOOSE、FIRST_ROWS_n（n=1,10,100,1000）、FIRST_ROWS或者ALL_ROWS。

1. `RULE`

表示使用RBO来解析SQL。RBO优化器不会使用到目标SQL中涉及的各个对象的统计信息。

2. `CHOOSE`

只要目标SQL中所涉及的表对象中有一个有统计信息，就使用CBO优化器解析SQL；如果目标SQL涉及的所有表对象都没有统计信息，就会使用RBO优化器。

3. `FIRST_ROWS_n`

使用CBO优化器。CBO在计算目标SQL的各个执行路径的成本值时，会把那些以最快响应速度返回头n条记录（n=1,10,100,1000）的执行步骤的成本值改成一个很小的值，远小于默认情况下CBO对同样执行步骤计算出来的成本值。因此，包含这些执行步骤的执行路径在计算成本值时更加有优势。

4. `FIRST_ROWS`

Oracle会联合使用RBO和CBO来解析SQL。在大多数情况下，会使用CBO优化器，此时类似于`FIRST_ROWS_n`；在某些特定情况下，会转而使用RBO而不会考虑成本。比如，如果Oracle发现能够使用索引来避免排序，就会选择基于该索引的执行路径，而不再考虑成本。

5. `ALL_ROWS`

Oracle 10g以后数据库版本中参数`OPTIMIZER_MODE`的默认模式。表示使用CBO来解析目标SQL，在计算执行路径成本值时的侧重点在于最佳的吞吐量，即最小的系统I/O和CPU资源消耗。


# 结果集
结果集（Row Source）是指包含SQL执行结果的集合。结果集和目标SQL执行计划的执行步骤相对应。一个执行步骤所产生的执行结果就是它对应的输出结果集。

对于RBO而言，执行计划中看不到对相关步骤的结果集的描述；对于CBO而言，对应执行计划中的**Rows**就是相关执行步骤对应的输出结果集的记录数（cardinality）的估算值。


# 访问数据的方法
Oracle访问表数据一般有两种方法，一是直接访问表，二是先访问索引再回表（如果要访问的数据刚好都在索引中，则无需回表，即索引覆盖）。

## 访问表的方法
Oracle数据库中可以通过全表扫描或者ROWID扫描两种方式直接访问表数据。

1. 全表扫描

Oracle通过全表扫描方式访问表数据时，会从表所占用的第一个区（Extent）的第一个数据块（Data block）开始扫描，一直扫描到该表的高水位线，这段范围内所有的数据块。

如果表的数据量比较小，全表扫描的效率很高；但是随着表数据量的增长，全表扫描的执行时间也会随之增加。因此，走全表扫描的SQL的执行时间是不稳定、不可控的。另一方面，由于高水位线的特点，即使Delete了表中的所有数据，HWM的位置也不会下降，此时走全表扫描还是会扫描HWM以下所有的数据块。

>高水位线
>
>Oracle存储中，高水位线（HWM, High Water Mark）以上的数据块尚未格式化且从未被使用，HWM以下则是已经分配可供使用的数据块。HWM以下的数据块可以有三种状态：
>（1）已分配但尚未格式化；
>（2）已经格式化且存有数据；
>（3）已经格式化且存储的数据已被删除。
>具体请参考：https://blog.csdn.net/Sebastien23/article/details/118252286


2. ROWID扫描

ROWID表示的是Oracle中的数据行记录所在的物理存储地址。ROWID扫描可以指根据SQL中指定的ROWID直接访问数据块中对应的行记录，也可以是先去访问相关索引、通过索引获取到ROWID后再回表去访问行记录。

对于Oracle中的堆表，可以通过访问Oracle内置的ROWID伪列得到对应行记录的ROWID，并通过DBMS_ROWID包中的相关方法，将ROWID翻译为实际的物理存储地址。

```sql
> select empno, ename, rowid, dbms_rowid.rowid_relative_fno(rowid) || '_' ||
 dbms_rowid.rowid_block_number(rowid) || '_' || dbms_rowid.rowid_row_number(rowid) 
 location from emp;
 --------------------------------------------------
 EMPNO     ENAME     ROWID                  LOCATION
 -----     -----     ------------------     --------
 7369      LIWEI     AAARR3sAAEAAAXCAAB     4_151_12
 ...
 --empno=7369的行记录对应的物理存储地址位于4号文件的第151个数据块的第12行记录
```

>堆表 vs 索引组织表
>
>与MySQL InnoDB中的索引组织表（Index organized table, IOT）不同，Oracle同时支持索引组织表和堆表，但默认是堆表（Heap table）。二者的主要区别在于：
> - 索引组织表中，主键索引所在的B+树的叶子节点中存储了行记录，而二级索引所在的B+树的叶子节点中存储了该二级索引和主键索引的值。
> - 堆表中，行记录和索引是分开存储的。存储在堆中的行记录数据是无序的。堆表中的主键索引和普通索引一样，都存储了指向堆表中行数据的指针。


## 访问索引

### B树索引
Oracle数据库中最常用的索引是B树索引。B树索引中的数据块可以分为分支节点和叶子节点。

B树索引的分支节点的数据块中存储了索引键列值、以及指向下一级分支节点或叶子节点的指针。每个分支节点都会存储两种类型的指针，一种是lmc指针（即Left Most Child），另一种是分支节点的索引行记录中的指针。lmc指针指向的子节点中所有索引键值列的最大值，一定小于父节点的所有索引键值列中的最小值；父节点的索引行记录中的指针指向的子节点中，所有索引键列值中的最小值，一定大于或等于该行记录本身索引键值列的值。

B树索引的叶子节点的数据块中存储了被索引键值、以及用于定位该索引键值所在的数据行在表中实际物理存储位置的ROWID。对于唯一B树索引，ROWID存储在索引行的行首，无需额外存储该ROWID的长度；对于非唯一B树索引而言，ROWID则被当作额外的列与被索引的键值列一起存储，此时需要额外存储ROWID的长度。因此在同等条件下，唯一B树索引比非唯一B树索引更加节省叶子节点的存储空间。在非唯一B树索引中，Oracle会按照被索引键值和对应ROWID来联合排序。Oracle中的叶子节点是左右互联的，相当于有一个双向指针链表，把B树索引中所有叶子节点左右互相连接在了一起。

Oracle数据库中的B树索引具有以下优势：

- 所有叶子节点都在同一层，即它们距离根节点的深度是相同的。因此访问叶子节点中任何一个索引键值所花费的时间几乎相等。
- Oracle会保证所有的B树索引都是**自平衡**的，即不可能出现不同的叶子节点不位于同一层的现象。
- 通过B树索引访问表里行记录的效率不会随着表的数据量的递增而显著降低，即通过走索引访问数据的时间是基本稳定的、可控的。

B树索引的数据结构决定了在Oracle中通过索引访问数据时，需要先访问B树索引，得到ROWID后再回表去访问对应的行记录（不考虑索引覆盖的情况）。因此访问索引的成本也由两部分构成，一是访问B树索引的成本，即从根节点扫描定位到到目标叶子节点的成本；二是回表的成本。

### 访问索引的方法

1. INDEX UNIQUE SCAN

索引唯一性扫描是针对唯一索引的扫描，扫描结果最多返回一条记录。仅适用于WHERE条件中是等值查询的目标SQL。

2. INDEX RANGE SCAN

索引范围扫描适用于所有类型的B树索引。当扫描对象是唯一索引时，目标SQL的WHERE条件中一定是范围查询，谓词条件一般为BETWEEN、<、>等；扫描非唯一索引时，WHERE条件中既可以是范围查询，也可以是等值查询。索引范围扫描一般会返回多条记录。

在同等条件下，当目标索引的索引行树大于1时，索引范围扫描至少会比相应的索引唯一性扫描多一次逻辑读。即使索引范围扫描最终返回的记录也只有一行，在扫描索引的过程中，为了确定范围扫描的终点，就需要一直扫描到一个不在范围内的索引才停止。

```sql
--创建测试表
> create table emp_temp as select * from emp;
> select count(empno) from emp_temp;
COUNT(EMPNO)
------------
     13

--创建唯一B树索引
> create unique index idx_emp_temp on emp_temp(empno);
--收集表的统计信息
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT', tabname => 'EMP_TEMP', estimate_percent => 100, cascade => true, method_opt=>'for all columns size 1');

--清空buffer cache和数据字典缓存以避免它们对逻辑读统计结果的影响
> alter system flush shared_pool;  //请勿随意在生产环境执行
> alter system flush buffer_cache;  //请勿随意在生产环境执行

--测试一
> set autotrace traceonly
> select * from emp_temp where empno=7369;
---------------------执行计划-----------------------
...
INDEX UNIQUE SCAN  ...   //索引唯一性扫描
...
---------------------统计信息-----------------------
...
73 consistent gets      //耗费的逻辑读为73
35 physical reads
...

--创建非唯一索引
> drop index idx_emp_temp;
> create index idx_emp_temp on emp_temp(empno);
--收集表的统计信息
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT', tabname => 'EMP_TEMP', estimate_percent => 100, cascade => true, method_opt=>'for all columns size 1');

--清空buffer cache和数据字典缓存以避免它们对逻辑读统计结果的影响
> alter system flush shared_pool;  //请勿随意在生产环境执行
> alter system flush buffer_cache;  //请勿随意在生产环境执行

--测试二
> select * from emp_temp where empno=7369;
---------------------执行计划-----------------------
...
INDEX RANGE SCAN  ...   //索引范围扫描
...
---------------------统计信息-----------------------
...
74 consistent gets      //耗费的逻辑读比Index Unique Scan多一次
35 physical reads
...
```

3. INDEX FULL SCAN

索引全扫描会扫描目标索引的所有叶子节点的所有索引行，适用于所有类型的B树索引。默认情况下，Oracle在做索引全扫描时，只需要通过访问必要的分支节点定位到索引树最左边的叶子节点的第一行索引，就可以利用索引树叶子节点之间的**双向指针链表**，从左至右依次顺序扫描所有叶子节点的索引行。

由于索引是有序的，因此索引全扫描的结果集也是有序的，按照索引的索引键值列来排序。因此走索引全扫描可以在避免排序操作的同时得到有序的结果集。

```sql
> set autotrace on
> select empno from emp;
---------------------执行计划-----------------------
...
INDEX FULL SCAN  ...   //索引全扫描
...
---------------------统计信息-----------------------
...
0 sorts (memory)    //内存中没有排序操作
0 sorts (disk)      //磁盘中也没有排序操作
...
```

默认情况下，索引全扫描的有序性决定了它不能够并行执行，通常情况下使用的是单块读。通常情况下，索引全扫描不需要回表，适用于目标SQL查询列全部是目标索引的索引键值列的情况。由于Oracle数据库中，当所有索引键值列全为NULL时，不会为这些NULL值建立索引，因此使用索引全扫描的前提条件是：**目标索引至少有一个索引键值列的属性是NOT NULL**。

4. INDEX FAST FULL SCAN

索引快速全扫描和索引全扫描非常相似，主要区别有以下三点：

- 索引快速全扫描只适用于CBO优化器；
- 索引快速全扫描可以使用多块读，也可以并行执行；
- 索引快速全扫描的结果集不一定是有序的。这是因为索引快速全扫描是根据索引行在磁盘上的物理存储顺序来扫描的，而不是根据索引行的逻辑顺序来扫描的。对于单个叶子节点的数据块中的索引行而言，其物理存储顺序与逻辑存储顺序一致；但是对于两个物理存储位置相邻的叶子节点而言，它们数据块内的索引行不一定在逻辑上是有序的。

```sql
> create table emp_test (empno number, col1 char(2000), col2 char(2000), col3 char(2000));
--创建测试用的复合主键索引
> alter table emp_test add constraint pk_emp_test(empno,col1,col2,col3);
--插入数据
> insert into emp_test select empno,ename,job,'A' from emp;
> insert into emp_test select empno,ename,job,'B' from emp;
...
> select count(*) from emp_test;
COUNT(*)
---------
   91

--收集表的统计信息
> exec dbms_stats.gather_table_stats(ownname => 'SCOTT', tabname => 'EMP_TEST', estimate_percent => 100, cascade => true, method_opt=>'for all columns size 1');

--使用Hint来走主键复合索引
> select /*+ index_ffs(emp_test pk_emp_test) */ empno from emp_test;
  EMPNO
-------
7369
7499
7521
...        
7934     //返回的结果集未按照索引键值列empno来排序
7521
...
---------------------执行计划-----------------------
...
INDEX FAST FULL SCAN  ...   //索引快速全扫描
...
```

5. INDEX SKIP SCAN

索引跳跃式扫描适用于所有类型的B树索引，它使得那些在WHERE条件中没有对联合索引的最左列指定查询条件、但同时对联合索引的其他列执行了查询条件的目标SQL依然可以用上索引。在走索引跳跃式扫描时，Oracle会对目标索引的最左列的所有distinct值做遍历。

```sql
> create table employee (gender varchar2(1), employee_id number);
--将employee_id列设置为非空
> alter table employee modify(employee_id not null);
--创建联合索引
> create index idx_employee on employee(gender,employee_id);

--使用PL/SQL插入数据
begin
for i in 1..5000 loop
insert into employee values ('F',i);
end loop;
commit;
end;
/

begin
for i in 5001..10000 loop
insert into employee values ('M',i);
end loop;
commit;
end;
/

--查看执行计划
> set autotrace traceonly
> select * from employee where employee_id=100;
---------------------执行计划-----------------------
...
INDEX SKIP SCAN | IDX_EMPLOYEE  //索引跳跃式扫描使用了索引idx_employee
...
```

索引跳跃式扫描对索引最左列做遍历，相当于对目标SQL做以下的等价改写：

```sql
select * from employee where gender='F' and employee_id=100
union all
select * from employee where gender='M' and employee_id=100;
```

可以看出，索引跳跃式扫描仅适用于目标索引中最左列（WHERE条件中未使用的前导列）的distinct值数量较少、其他索引列的可选择性又非常好的场景。索引跳跃式扫描的执行效率一定会随着目标索引前导列distinct值的数量的增加而降低。






