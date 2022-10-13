---
tags: [oracle, SQL优化]
title: Oracle里的Shared Cursor
created: '2022-10-11T07:16:15.941Z'
modified: '2022-10-13T16:00:46.205Z'
---

# Oracle里的Shared Cursor

Cursor即**游标**，是Oracle数据库中SQL解析和执行的载体。Oracle数据库是用C语言写的，本质上可以将游标理解为C语言中的一种结构（Structure）。

## Library Cache
Oracle数据库中的**库缓存**（Library Cache）是SGA（系统全局区）中Shared Pool中的一块内存区域。库缓存的主要作用是缓存刚刚执行过的SQL语句和PL/SQL语句（如存储过程、函数、包、触发器）对应的执行计划、解析树（Parse Tree）、Pcode、Mcode等对象。当同样的SQL语句和PL/SQL语句再次被执行时，就可以利用已经缓存在Library Cache中的执行计划等对象，而无需再次从头开始解析，这样就提高了语句在重复执行时的效率。

### 库缓存对象句柄
缓存在Library Cache中的对象被称为库缓存对象（Library Cache Object）。所有的库缓存对象都是以库缓存对象**句柄**（Handle）的结构存储在库缓存中的。Oracle通过访问相关的句柄来访问对应的库缓存对象。实际上，库缓存对象句柄是以**哈希表**的方式存储在库缓存中的。

整个库缓存可以看作是由一组Hash Bucket（哈希桶）所组成。每一个哈希桶都对应不同的哈希值。对于单个哈希桶而言，里面存储的就是哈希值相同的所有库缓存对象句柄。同一个哈希桶的不同库缓存对象句柄之间会通过指针连接起来。同一个哈希桶的不同库缓存对象句柄之间实际上组成了一个库缓存对象句柄**链表**。

当Oracle要执行目标SQL时，首先会对目标SQL文本进行哈希运算，然后根据得到的哈希值去相关的哈希桶中遍历对应的库缓存对象句柄链表。如果找到了对应的句柄，就可以直接访问到该SQL的执行计划、解析树等对象并进行重用。如果找不到对应的库缓存对象句柄，就必须从头开始解析SQL，并把解析后得到的执行计划、解析树等对象以库缓存对象句柄的方式链接在相关的哈希桶中的句柄链表中。

作为一种复杂C语言结构，库缓存对象句柄有很多属性，其中Name、Namespace和Heap 0 Pointer三个属性尤为重要。

1. **Name**

Name是库缓存对象句柄对应的库缓存对象的名称。如果是SQL语句的库缓存对象句柄，则Name的值就是该语句的SQL文本；如果是表对应的库缓存对象句柄，Name的值就是该表的表名。

2. **Namespace**

Namespace是库缓存对象句柄对应的库缓存对象所在的分组名。不同类型的库缓存对象可能属于同一个分组，即不同类型的库缓存对象所对应的句柄的Namespace值可能相同。

Oracle数据库中常见的Namespace及其对应的库缓存对象句柄如下：
| Namespace | 库缓存对象句柄 |
| :--: | :--: |
| CRSR | SQL语句和匿名PL/SQL语句对应的库缓存对象句柄 | 
| TABL/PRCD/TYPE | 表、视图、Sequence、同义词、存储过程、函数、Type和Package的定义所对应的库缓存对象句柄 | 
| BODY/TYBD | Type和Package的具体实现（body）所对应的库缓存对象句柄 | 
| TRGR | Trigger对应的库缓存对象句柄 | 
| INDX | 索引所对应的库缓存对象句柄 | 
| CLST | Cluster所对应的库缓存对象句柄 | 

3. **Heap 0 Pointer**

Heap 0 Pointer是库缓存对象句柄指向子结构Heap 0的指针。Heap 0也是一种复杂的结构，具有很多属性。

Heap 0的属性**Tables**里记录的是与该Heap 0所在的库缓存对象有关联关系的库缓存对象句柄地址的集合。Tables可以细分为很多类，其中叫做**Child table**的记录了从属于该Heap 0所在的库缓存对象的子库缓存对象的句柄地址的集合。Heap 0里的Tables实际上记录的就是各个库缓存对象之间的关联关系。Oracle可以通过这些关联关系直接访问到对应的库缓存对象。

对于每一个库缓存对象而言，都或多或少需要往库缓存中存储一些它所特有的动态运行时（runtime）数据，比如SQL语句就需要在库缓存中缓存它所对应的编译好的二进制格式的执行计划。Oracle会用**Data Heap**来存储这些动态运行时数据。Data Heap是动态分配的，大小不固定。

每一个库缓存对象都可能会拥有多个Data Heap（Heap 1、Heap 2、...、Heap n），它们之间是独立的，没有关联关系。Oracle会在Heap 0的属性**Data Blocks Pointer**中存储指向这些Data Heap的指针。即通过访问Heap 0就可以访问其所在的库缓存对象所拥有的所有Data Heap。

## Shared Cursor
Shared Curosr是缓存在库缓存里的SQL语句和匿名PL/SQL语句所对应的库缓存对象，它所对应的Namespace为CRSR。Shared Curosr里会存储目标SQL的SQL文本、解析树、SQL涉及的对象定义、SQL使用的绑定变量类型和长度、以及SQL执行计划等信息。

### 父游标 vs 子游标
Oracle数据库中的Shared Cursor又可以细分为父游标（Parent Cursor）和子游标（Child Cursor）。可以分别通过查询视图`v$sqlarea`和`v$sql`来查看当前缓存在库缓存中的父游标和子游标。父游标和子游标的结构是一样的，二者的区别在于：

- 目标SQL的**SQL文本**会缓存在**父游标**对应的库缓存对象句柄的Name属性中，而子游标对应句柄的Name属性为空；
- 目标SQL的**解析树和执行计划**会存储在**子游标**对应的库缓存对象句柄的Heap 6中；
- 父游标的Heap 0的Child table中会存储从属于该父游标的所有子游标的库缓存对象句柄地址。

这种父游标和子游标的结构就决定了在Oracle数据库里，任意一个目标SQL一定会同时对应两个Shared Cursor，其中一个是父游标，另一个是子游标。父游标存储SQL文本，子游标存储可以被重用的解析树和执行计划。

>Shared Cursor为什么要分父游标和子游标？
Oracle是根据SQL文本的哈希值去相应哈希桶里的库缓存对象句柄链表里寻找匹配的库缓存对象句柄的。不同的SQL文本的哈希值可能相同（即哈希碰撞），而且同一条SQL也可能有多个不同的解析树和执行计划。如果这些库缓存对象句柄都存储到同一个哈希桶里的句柄链表中，就会使得该链表的长度增加，同时从头到尾检索该链表所需要耗费的时间也会变长。为了能够尽量减少对应哈希桶里库缓存对象句柄链表的长度，Oracle才设计了这种嵌套的父游标和子游标并存的结构。

```sql
> conn scott/taylorTS1989
--测试一
> select empno,ename from emp;
...
> select sql_text,sql_id,version_count from v$sqlarea where sql_text like 'slect empno,ename%';
SQL_TEXT                      SQL_ID        VERSION_COUNT
----------------------------  ------------  -------------
select empno,ename from emp;  fhxd98jlu79a        1
--verison_count表示某个父游标所拥有的子游标的数量
--上面只返回了一条记录，说明只产生了一个父游标和一个子游标

> select plan_hash_value,child_number from v$sql where sql_id='fhxd98jlu79a';
PLAN_HASH_VALUE   CHILD_NUMBER
---------------   ------------
3956160932              0
--上面只返回了一条记录，说明只产生了一个子游标号为0的child cursor

--测试二：将表名改为大写（哈希运算对大小写敏感）
> select empno,ename from EMP;
...
> select sql_text,sql_id,version_count from v$sqlarea where sql_text like 'slect empno,ename%';
SQL_TEXT                      SQL_ID        VERSION_COUNT
----------------------------  ------------  -------------
select empno,ename from EMP;  2f0r5dmnas0z        1
select empno,ename from emp;  fhxd98jlu79a        1

> select plan_hash_value,child_number from v$sql where sql_id='2f0r5dmnas0z';
PLAN_HASH_VALUE   CHILD_NUMBER
---------------   ------------
3956160932              0
--针对包含大写表名的SQL，生成了一对新的父游标和子游标              

--测试三：不同用户下的同名表
> conn jobs/macintosh886
> create table emp as select * from scott.emp;
> select empno,ename from emp;
...
> select sql_text,sql_id,version_count from v$sqlarea where sql_text like 'slect empno,ename%';
SQL_TEXT                      SQL_ID        VERSION_COUNT
----------------------------  ------------  -------------
select empno,ename from EMP;  2f0r5dmnas0z        1
select empno,ename from emp;  fhxd98jlu79a        2
--version_count的值变成了2，说明对应SQL执行时产生了一个父游标和两个子游标

> select plan_hash_value,child_number from v$sql where sql_id='fhxd98jlu79a';
PLAN_HASH_VALUE   CHILD_NUMBER
---------------   ------------
3956160932              0
3956160932              1
--返回了两条匹配记录，对应子游标号分别为0和1，说明生成了两个child cursor
```

Oracle在解析目标SQL去库缓存中查找匹配的Shared Cursor的过程大致可以分为如下步骤：

1. 根据库缓存对象句柄的Name和Namespace属性（即SQL文本和CRSR）的哈希值，去库缓存中寻找匹配的哈希桶。
2. 在匹配到的哈希桶里的库缓存对象句柄链表中查找匹配的父游标（需要对比SQL文本，以避免哈希碰撞的影响）。
3. 如果找到了匹配的父游标，则遍历从属于该父游标的所有子游标，找到匹配的子游标。
4. 步骤2中如果没有找到匹配的父游标，就会从头开始解析目标SQL，新生成一个父游标和子游标，并把它们存放到对应的哈希桶句柄链表中。 
5. 步骤3中如果找到了匹配的子游标，就可以直接把该子游标中的解析树和执行计划拿过来重用。
6. 步骤3中如果没有找到匹配的子游标，就会从头开始解析目标SQL，新生成一个子游标，并把它挂到对应的父游标下。 


## 硬解析
硬解析（Hard Parse）是指Oracle在生成执行计划时，在库缓存中找不到可以重用的解析树和执行计划，不得不从头开始解析目标SQL，并生成相应的父游标和子游标（也有可能只需生成子游标）的过程。

硬解析不好的地方主要体现在以下方面：

1. 硬解析可能会导致**Shared Pool Latch争用**。

硬解析需要生成新的子游标，并将其存储在库缓存中。这意味着Oracle需要在Shared Pool中分配出一块内存区域用于存储新生成的子游标。在shared pool中分配内存是需要持有shared pool latch的，如果硬解析的并发量较大，就可能会导致shared pool latch的争用。一旦发生大量的shared pool latch争用，系统的性能和可扩展性是会受到严重影响的，通常表现为CPU使用率居高不下，接近100%。

2. 硬解析可能会导致**库缓存相关Latch和Mutex的争用**。

发生硬解析时，需要扫描相关哈希桶里的库缓存对象句柄链表，这个动作需要持有**Library Cache Latch**。Oracle数据库中Latch的另外一个作用就是用于共享SGA内存结构的并发访问控制。如果并发硬解析较高，就可能会导致Library Cache Latch争用。从11gR1开始，Oracle用Mutex替换了库缓存相关Latch，所以在Oracle 11gR1及其之后的版本中，取而代之的是Mutex的争用。与之相关的等待事件比如Cursor: pin S、Cursor: pin X、Cursor: pin S wait on X、Cursor: mutex S、Cursor: mutex X、Library cache: mutex X等。

>关于硬解析时Library Cache Latch和Shared Pool Latch的详细持有过程，可以参考博客文章：https://www.laoxiong.net/shared-pool-latch-and-library-cache-latch.html

对于高并发的OLTP系统而言，硬解析会严重影响系统的性能和可扩展性。但是对于OLAP/DSS系统而言，并发的数量很少，目标SQL也很少被重复执行，而且在执行SQL时硬解析所耗费的时间和资源相对于SQL总的执行时间和资源消耗来说是微不足道的，这种情况下硬解析一般是可以接受的。

## 软解析
软解析（Soft Parse）是指Oracle在执行目标SQL时，在Library Cache中找到了匹配的父游标和子游标，并将存储在子游标中的解析树和执行计划直接拿过来重用、而无需从头开始解析的过程。

和硬解析相比，软解析的优点主要体现在以下两个方面：

1. 软解析不会导致Shared Pool Latch的争用。

软解析不需要生成新的父游标和子游标，因此无需在Shared Pool中申请分配内存区域，因此不需要持有shared pool latch。

2. 软解析在发生库缓存相关Latch和Mutex争用时的影响更小。

软解析虽然也可能会导致库缓存相关Latch和Mutex的争用，但是软解析持有库缓存相关latch的次数要少，而且软解析对library cache latch的持有时间会比硬解析更短。这意味着即使产生了库缓存相关latch的争用，程度也不会有硬解析那么严重，带来的系统性能和可扩展性问题会少很多。





