@[TOC](TiDB体系结构之TiDB Server)

# TiDB Server
TiDB Server的主要作用如下：

- 处理客户端连接
- SQL语句的解析和编译
- 关系型数据与KV的转化
- SQL语句的执行
- 在线DDL的执行
- 垃圾回收（*Garbage Collection*）


## TiDB Server主要组成模块
TiDB Server由多个不同的功能模块组成。

以下三个模块负责解析和编译SQL语句，并生成SQL语句的执行计划：

- Protocol Layer
- Parse
- Compile

以下四个模块负责SQL的执行：

- Executor：负责执行SQL语句；
- DistSQL：负责涉及范围查询、分组查询、表连接的SQL执行；
- Transaction：负责事务相关的内容；
- KV：负责点查询SQL执行；

以下两个模块负责与PD和TiKV交互：

- PD Client：负责与PD节点交互；
- TiKV Client：负责与TiKV节点交互；

以下三个模块负责online DDL操作：

- schema load
- worker
- start job

以下两个模块负责垃圾回收与数据缓存：

- GC：负责垃圾回收；
- memBuffer：用户缓存读取的数据、元数据、登录认证信息、统计信息等内容。


## SQL语句的解析和编译
TiDB Server中SQL语句的解析和编译的流程大致如下：

首先，Parse模块对SQL语句依次进行**词法分析**与**语法分析**，并生成抽象语法树（*abstract syntax tree*, **AST**）。

然后，Compile模块会对Parse模块生成的AST依次进行

- **合法性验证**，比如验证SQL语句中的表是否存在；
- **逻辑优化**，SQL语句层面的优化，比如外连接变内连接、子查询优化、谓词下推；
- **物理优化**，结合统计信息，考虑全表扫描或走索引、以及索引选择；

并最终生成SQL的执行计划。该过程中需要的元数据和统计信息会缓存在memBuffer中。

## 行数据与KV的转化
对于聚簇表，TiDB会将**表的编号加上主键**作为Key，其余列作为Value；

对于非聚簇表，TiDB会自动为每行数据生成一个RowID。

## SQL读写相关模块
Compile模块生成的执行计划会被发送给Executor模块。

涉及表连接、子查询、范围查询的复杂SQL会被Executor模块发送给**DistSQL模块**。DistSQL模块会将这些复杂SQL**转化为对单表的操作的组合**，然后通过TiKV Client发送给TiKV节点。

如果是涉及**唯一索引的等值查询**（只获取零行或者一行数据），则会由**KV模块**来处理。

如果涉及到事务，**Transaction模块**会在事务开始和事务提交时，通过PD Client向PD节点获取TSO时间戳。

## 在线DDL相关模块
Online DDL操作不会阻塞数据库的读写。

TiDB中的Online DDL操作需要TiDB Server节点上的`start job`、`worker`、`schema load`模块以及TiKV节点上的`job queue`和`history queue`协同完成。

TiDB数据库中，同一时刻只能有一个TiDB Server做Online DDL操作。我们把当前时刻可以执行Online DDL任务的TiDB Server称为**Owner**。

TiDB中Online DDL操作的调度流程大致如下：

1. 不同用户连接到不同的TiDB Server，发出的DDL操作由**start job模块**接收，并由start job模块生成一个对应的job放到TiKV节点上的job queue队列中。

2. 承担Owner角色的TiDB Server中的**worker模块**会从TiKV节点上的job queue中获取Online DDL的job来执行。执行完的job会被移动到history queue中，然后Owner接着从job queue中获取下一个job来执行。

3. 当前的Owner任期结束时，会通过选举选出一个新的TiDB Server，作为新的Owner来执行Online DDL任务。同时，新的Owner中的**schema load模块**会将最新的所有表schema的信息同步到TiDB Server的缓存中。


## TiDB的垃圾回收
TiDB Server中的GC模块负责**清理MVCC中过期的数据版本**。

被选举为**GC Leader**的TiDB Server负责自己任期内的垃圾回收。GC Leader会记录一个名为**Safe Point**的时间戳，Safe Point之前的过期数据会被回收，以减轻数据库空间的压力。

GC每隔一段时间会定期触发一次，称之为**GC Life Time**，默认为10分钟。

## TiDB Server的缓存
TiDB Server的缓存由以下部分组成：

- SQL结果：如果SQL返回的结果集的数据分布在多个TiKV节点，汇总后的结果集会在TiDB Server的缓存中；
- 线程缓存：前面提到的不同模块线程所使用的缓存；
- 元数据、统计信息：解析和编译SQL使用到的元数据和统计信息，以及用户的认证信息等。


TiDB Server的缓存管理主要涉及到以下两个重要参数：

- `tidb_mem_quota_query`：决定了每条SQL能够使用的最大缓存；
- `oom-action`：决定了当单条SQL使用的内存超过`tidb_mem_quota_query`时，是中断SQL执行并报错、还是只记录日志。



