@[TOC](MySQL基础入门：架构、日志、事务隔离)

# 基础架构
以目前广泛使用的MySQL 5.7为例，MySQL的架构可以分为**服务器层**（Server）和**存储引擎层**（Storage Engine）。其中，服务器层由连接器（Connectors）、查询缓存（Query Cache）、分析器（Parser）、优化器（Optimizer）、执行器（Query Execution）构成。


![在这里插入图片描述](https://img-blog.csdnimg.cn/2021031311233290.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

**连接器**
连接器负责跟客户端建立连接、获取权限、维持和管理连接。连接命令的一般格式为

```bash
$ mysql -h{$IP} -P{$PORT} -u{$USERNAME} -p
```

客户端建立连接后如果长时间无操作，那么在经历一段时间后连接会自动断开。这个时间由参数 `wait_timeout` 控制，默认为8小时。

实际应用中，一般建议使用**长连接**，即连接成功后如果客户端持续有请求，则一直使用同一个连接，以尽量减避免因超时而反复建立连接。mysql在执行过程中临时使用的内存是管理在连接对象中的，直到连接断开时才会释放。如果长期积累下来，可能会导致内存占用过大，从而被系统强行杀掉（**OOM kill**），使得mysql异常重启。为了避免出现此种情况，建议每次在执行一个数据量较大的操作以后，通过 `mysql_reset_connection` 来重新初始化连接资源。使用此命令不需要重新连接和验证权限。

**查询缓存**
查询缓存中保留了历史的查询记录。如果客户端的查询能够在查询缓存中找到，则会跳过后面的分析器、优化器等步骤，将查询结果直接返回给客户端，因此效率非常高。

但是查询缓存也存在明显的弊端，即只要一个表的数据发生了更新，则此表上的所有查询缓存都会被清空。因此，对于数据更新较为频繁的数据库来说，查询缓存的命中率会非常低。出于这方面的考虑，从 **MySQL 8.0** 开始不再支持查询缓存功能。对于之前的版本，可以设置参数 `query_cache_type=DEMAND`，然后在查询时使用关键字 `SQL_CACHE` 显示指定使用查询缓存。

```sql
mysql> SELECT SQL_CACHE * FROM mytable WHERE col=10;
```

**分析器**
分析器会对查询语句进行**词法分析**，比如识别出SELECT关键字、以及FROM后接的表名、WHERE后接的列名，等等。然后，分析器会进行**语法分析**，比如你把SELECT写成了ELECT，分析器就会报 Syntax Error。一般语法报错只会提示第一个出错的位置。

**优化器**
优化器用于生成执行计划。例如，在表里存在多个索引时，决定使用哪一个索引；又或者，当一条语句中使用JOIN关联多个表时，决定各个表的连接顺序。

**执行器**
执行器是真正执行语句的阶段。开始执行前，会判断是否对表有执行查询的权限。如果客户端有操作权限，就会打开表执行SQL语句。执行器会根据表的引擎定义，调用对应的存储引擎来读取表。

**存储引擎**
存储引擎以插件（plugin）的形式存在。目前主流也是默认的存储引擎为 **InnoDB**。其他的存储引擎还有 MyISAM、Memory等。

# 日志系统
## 重做日志：redo log
redo log是InnoDB引擎特有的日志。当数据库中有一条记录要更新时，InnoDB引擎会先把记录写道redo log里，并更新内存，就算更新完成了。然后，InnoDB会在适当的时候将内存中的操作记录固化到磁盘中。

redo log的大小是固定的，并且采用**循环写**的模式。假设有ib-logfile-0、ib-logfile-1、ib-logfile-2、ib-logfile-3四个redo log日志文件。当重做日志写到3号文件末尾时，又会回到0号文件开头写。因此，需要定期地将重做日志固化到磁盘，以防止没有足够的日志空间。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313152416567.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

redo log使得InnoDB引擎能够保证即使数据库发生异常重启，之前提交的记录也不会丢失。这种能力被称为 **crash-safe**。

## 归档日志：binlog
binlog是Server层所有的日志。binlog与redo log的不同之处包括：

 - redo log是InnoDB引擎所特有的，而binlog则是服务器层实现的，所有存储引擎都可以使用；
 - redo log是**物理日志**，记录的是在某个数据页上做了什么修改，而binlog是**逻辑日志**，记录的是SQL语句的原始逻辑，比如“给表中ID=10的某一行的某age变量的值增加1”；
 - redo log是**循环写**的，空间固定，而binlog可以**追加写**入，即一个binlog日志文件写到一定大小后，会切换到一个新的文件继续写。


## 两阶段提交
当我们执行

```sql
mysql> CREATE TABLE mytable(ID int primary key, price int);
mysql> UPDATE mytable SET price=price+1 WHERE ID=9;
```
由于ID是主键，InnoDB存储引擎直接使用树搜索找到这一行，将price的值加1，然后将新数据更新到内存中，同时将更新的操作记录写入redo log。此时redo log处于 "**Prepare**" 状态。然后，执行器生成该更新操作的binlog，并将其写入磁盘。最后，执行器调用InnoDB的事务提交接口，将redo log的状态改为 "**Commit**"。

将redo log的写入分为prepare和commit两个阶段，这就是**两阶段提交**。这样如果binlog写入失败，redo log也不会提交事务，而是会回滚事务。两阶段提交能够使得redo log和binlog中的事务提交状态保持逻辑上的一致。

与日志有关的两个重要参数，设置 `innodb_flush_log_at_trx_commit=1`，表示每次事务的redo log都直接持久化到磁盘，可以保证mysql异常重启之后不丢失数据。设置 `sync_binlog=1` 表示每次事务的binlog都直接持久化到磁盘，可以保证mysql异常重启之后不丢失binlog。

# 事务隔离
在MySQL中，事务是在**存储引擎**层实现的。原生的MyISAM不支持事务，因此逐渐被InnoDB取代。事务具有ACID特性，即原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）。

## 隔离性与隔离级别
当同时有多个事务执行时，就可能出现**脏读**（dirty read）、**不可重复读**（non-repeatable read）、**幻读**（phantom read）的问题。为了有针对性地解决这些问题，就有了隔离级别的概念。

SQL标准的隔离级别包括：**未提交读**（read uncommitted）、**提交读**（read committed）、**可重复读**（repeatable read）和**串行化**（serializable）。

 - 未提交读：一个事务还未提交时，它所做的修改就能被其他事务看到；
 - 提交读：一个事务提交后，它所做的修改才能被其他事务看到；
 - 可重复读：一个事务在执行过程中看到的数据，总是与它在启动时所看到的数据保持一致。同时，未提交的修改对其他事务也是不可见的；
 - 串行化：对于同一行记录，读的时候会加“读锁”，写的时候会加“写锁”。当出现读写锁冲突时，后访问的事务必须等前一个事务执行完成以后，才能继续执行。

## 事务隔离的实现
实际上，在读写数据时，数据库会创建一个**视图**（view），访问的时候以视图的逻辑结果为准。未提交读时，直接返回最新的值，没有视图创建；提交读时，视图会在每条SQL语句开始执行的时候创建；可重复读时，视图是在事务启动时创建的；串行化时，通过加锁来避免并行访问。

配置参数 `transaction-isolation=READ-COMMITTED` 可以将数据库隔离级别设置为提交读。使用下面的命令来查看事务隔离级别

```sql
mysql> SHOW VARIABLES LIKE 'transaction_isolation'; 
```

以可重复读为例，在 MySQL 中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。记录回滚操作的日志文件被称为 **undo log**，即**回滚日志**。当没有事务要用到这些回滚日志时，它们就会被删除。

由于事务随时可能访问数据库里面的任何数据，所以事务提交之前，数据库里面它可能用到的回滚记录都必须保留，这就会导致大量占用存储空间。因此要避免使用长事务。此外，在 MySQL 5.5 及以前的版本，回滚日志是跟数据字典一起放在 ibdata 文件里的，即使长事务最终提交，回滚段被清理，文件也不会变小。

## 事务的启动方式
事务有两种启动方式，一种是使用begin或start命令显式启动事务

```sql
mysql> BEGIN TRANSACTION;
#执行查询或更新操作
mysql> COMMIT;  #提交事务

mysql> START TRANSACTION;
#执行查询或更新操作
mysql> ROLLBACK;  #回滚事务
```

另一种方法是使用命令 `set autocommit=0`，该命令会将当前线程的自动提交关闭。当你执行一个SELECT语句时，事务就启动了，但是并不会自动提交，直到你执行 commit 或者 rollback、或者断开连接。如果忘了手动提交事务，这种方法容易导致长事务的产生，因此更推荐前一种方法。

可以在 information_schema 库的 innodb_trx 这个表中查询长事务，比如下面这个语句，用于查找持续时间超过 60s 的事务。

```sql
mysql> SELECT * FROM information_schema.innodb_trx WHERE TIME_TO_SEC(timediff(now(),trx_started))>60
```


References: 
[1\] https://time.geekbang.org/column/intro/139
[2\] https://dev.mysql.com/doc/refman/8.0/en/pluggable-storage-overview.html
