@[TOC](【MySQL基础篇】启动选项、系统变量、状态变量)

# 启动选项
执行mysqld初始化数据库时，可以指定服务器启动选项（**server options**）。**MySQL 5.7**中，启动项可以分为与安全相关的启动项、SSL相关的启动项、Binlog相关的启动项、复制相关的启动项、插件加载相关的启动项、以及存储引擎相关的启动项。

## 安全相关的启动项
与安全相关的启动项参见：

> [https://dev.mysql.com/doc/refman/5.7/en/security-options.html](https://dev.mysql.com/doc/refman/5.7/en/security-options.html)

其中，`secure_file_priv`、`skip-grant-tables`、`skip_networking`三个启动项着重介绍一下。

**secure_file_priv**
该启动项会影响`LOAD DATA`、`SELECT ... INTO OUTFILE`语句以及`LOAD_FILE()`函数的使用。

 - 该启动项的值为空时，无任何影响；
 - 该启动项的值为路径名时，文件的导入和导出操作**只能在该路径下**进行；
 - 该启动项的值为NULL时，MySQL Server会**禁止**文件的导入和导出操作。

**skip-grant-tables**
该启动项默认关闭（OFF），如果开启，会有如下后果：

 - MySQL服务在启动时不会读取系统数据库中的权限表（grant tables），也就会导致任何用户对所有数据库都有**无限制的访问权限**（因此，该启动项可以在我们忘记数据库管理员账号密码时用来修改密码）；
 - MySQL服务在启动时不会加载系统数据库中注册的某些对象，比如通过`INSTALL PLUGIN`安装的插件、通过`CREATE EVENT`创建的定时事件、通过`CREATE FUNCTION`创建的函数；
 - 系统变量`disabled_storage_engines`不会生效。

**skip_networking**
该启动项控制数据库是否允许TCP/IP连接，默认是关闭的（即允许TCP连接）。在我们忘记数据库管理员账号密码时，如果想要通过开启`skip-grant-tables`来修改密码，出于安全考虑，最好一并开启`skip_networking`。

## SSL相关的启动项
与SSL相关的启动项参见：
> [https://dev.mysql.com/doc/refman/5.7/en/connection-options.html#encrypted-connection-options](https://dev.mysql.com/doc/refman/5.7/en/connection-options.html#encrypted-connection-options)


## Binlog相关的启动项
与binlog相关的启动项参见：
>[https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html)

其中，比较重要的有下面几个。

**log-bin**
二进制日志的文件名前缀。MySQL会用此文件名加上数字后缀来创建后续的binlog，例如`log_bin = /mysql/log/mysql-bin`。

**binlog_format**
决定了二进制日志的内容格式，取值可以为`STATEMENT`、`ROW`或者`MIXED`。MySQL 5.7.7之前默认为STATEMENT格式，5.7.7以及之后的版本默认为ROW格式；NDB集群中默认为MIXED格式。

**log-bin-index**
二进制文件的index文件的名字。binlog index文件中包含了所有binlog日志文件的文件名。

## 复制相关的启动项
与复制相关的启动项参见：
>[https://dev.mysql.com/doc/refman/5.7/en/replication-options-reference.html](https://dev.mysql.com/doc/refman/5.7/en/replication-options-reference.html)


**server_id**
在主从复制架构中，主库和每个副本都必须设定一个不同的server-id以区分彼此。server-id的值在1到$2^{32} − 1$之间。

**relay_log**
Relay日志的文件名前缀，例如`relay-log = /mysql/log/relay-log`。


**rpl_semi_sync_master_enabled**
主库复制架构中，在主库开启配置：`rpl_semi_sync_master_enabled=1`。需要先在主库安装半同步复制插件才能开启生效。

```sql
mysql> install plugin rpl_semi_sync_master SONAME 'semisync_master.so';
```

**rpl_semi_sync_slave_enabled**
主库复制架构中，在备库开启配置：`rpl_semi_sync_slave_enabled=1`。需要先在备库安装半同步复制插件才能开启生效。

```sql
mysql> install plugin rpl_semi_sync_slave SONAME 'semisync_slave.so';
```

**expire_logs_days**
单位为天，自动清除（purge）设定天数前生成的binlog日志文件。可设定的值为0到99天。设置`expire_logs_days=0`表示不自动清除binlog日志文件。

```sql
PURGE BINARY LOGS TO 'mysql-bin.010';
PURGE BINARY LOGS BEFORE '2019-04-02 22:46:26';
```

## 插件加载相关的启动项
与插件加载相关的内容参见：

>[https://dev.mysql.com/doc/refman/5.7/en/plugin-loading.html](https://dev.mysql.com/doc/refman/5.7/en/plugin-loading.html)

## 存储引擎相关的启动项
下面仅介绍InnoDB相关的几个启动项，其余内容参见：
>[https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html)


**innodb_flush_log_at_trx_commit**
决定了InnoDB日志（即**redo log**和**undo log**，记录在`ib_logfile`中）写入文件系统和持久到磁盘的方式。可取值为0、1、2，默认为1。

 - `innodb_flush_log_at_trx_commit=0`：**每隔1秒钟**将日志写入文件系统缓存并刷到磁盘。也就是说，服务异常的情况下可能丢失1秒以内的事务数据；
 - `innodb_flush_log_at_trx_commit=1`：在**每一次事务提交时**，都把日志写入文件系统缓存并刷到磁盘。对IO性能的要求非常高；
 - `innodb_flush_log_at_trx_commit=2`：在每一次事务提交时，写入日志到文件系统缓存，同时每隔1秒钟刷到磁盘。此时如果只是数据库服务挂了，文件系统没有问题，就不会丢失数据；如果服务器宕机或者掉电了，还是可能损失1秒以内的数据。


**innodb_file_per_table** 
- 设置为ON时，数据库中的表默认会在**file-per-table表空间**中创建；
- 设置为OFF时，数据库中的表默认会在系统表空间中创建。

默认值为ON。假设我们在product数据库下创建了一个fruit表，该表的数据和索引都将存储在 `/mysql/data/product/fruit.ibd`数据文件中（`/mysql/data`为数据目录）。

**innodb_lock_wait_timeout** 
InnoDB事务等待行锁的超时时间，默认为50秒。如果等待时间达到了该值，就会报下面的错误，并回滚要执行的语句。

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

**innodb_buffer_pool_size** 
InnoDB在内存中缓存表数据和索引的缓冲池大小，单位为bytes。默认值为134217728 bytes，即128MB。

**innodb_log_buffer_size** 
InnoDB写日志文件用到的缓存区大小，单位为bytes。`innodb_page_size`为32K或64K时，默认值为16M。如果日志缓存区设置得太小，在进行大事务时，log buffer被占满就会导致刷盘，从而增加了磁盘IO开销。

# 系统变量
MySQL Server的系统变量（**system variables**）会影响MySQL服务的运行。系统变量可以在服务启动时设定（==系统变量某种意义上来说也是启动项==）。许多系统变量也可以在服务运行时通过SET命令修改（==需要有SUPER权限==），从而动态地影响服务配置。

```sql
mysql> show variables like 'rpl_semi_sync_master%';
...
mysql> set global rpl_semi_sync_master_timeout = 20000;
```

系统变量分为全局级别（GLOBAL）和会话级别（SESSION）。前者对整个数据库服务生效，后者仅对当前会话生效。

比较重要的系统变量举例如下。

**slow_query_log**
默认为ON，即开启慢查询日志记录功能。

**slow_query_log_file**
慢查询日志的文件名，例如`slow_query_log_file = /mysql/log/mysqld_slow.log`。

**long_query_time** 
单位为秒，取值在0到10秒之间。如果一条SQL查询的执行时间超过了该值，则会被记录为慢查询。

**max_connections**
MySQL同时允许的最大客户端连接数，默认为151，取值应在1到100000之间。

**log_error**
错误日志的文件名，例如`log_error = /mysql/log/mysqld_err.log`。

**sql_log_bin**
全局变量级别的sql_log_bin即将被废弃。目前sql_log_bin更多地被作为一个session级别的系统变量来使用，默认为ON。如果在一个会话中将其设置为OFF，则会关闭对当前会话的binlog记录功能。MySQL不支持在事务或者子查询中设置sql_log_bin。

**max_binlog_size**
如果一个写日志的操作会使得当前binlog的大小超过max_binlog_size，MySQL不会写入当前的日志文件，而是新建一个日志文件来写入（称之为**log rotation**）。max_binlog_size的最小值为4096 bytes，最大为1G（默认值）。
由于**一个事务的日志不能拆开写到两个日志文件中**，如果遇到特别大的事务，一个binlog文件的大小有可能超过max_binlog_size。

**binlog_cache_size**
用于存放事务中binlog更新的缓存大小，其值必须是4096的倍数。MySQL会为每个客户端都分配一片Binlog缓存。

**sync_binlog**
控制MySQL以哪种方式将binlog日志刷到磁盘。其值可以是0、1、数字N。

 - **sync_binlog=0**：MySQL完全依赖操作系统将binlog刷到磁盘，自己不会主动持久化日志文件。如果发生操作系统crash或者断电，很可能会发生已提交事务的日志文件丢失。
 - **sync_binlog=1**：MySQL在每次提交事务前都会将对应的binlog日志持久化到磁盘。这种情况最安全，但是由于频繁地写盘可能会影响数据库性能。
 - **sync_binlog=N**：MySQL在N次事务提交后，才会主动将binlog日志缓存刷到磁盘。

在使用InnoDB存储引擎且存在主从复制关系的架构中，MySQL推荐双一配置：`sync_binlog=1` 且 `innodb_flush_log_at_trx_commit=1`。


# 状态变量
MySQL Server的状态变量（**status variables**）提供了服务运行的状态信息，监控这些数据可以了解数据库服务的运行性能。

执行下面的命令查看全局或者会话级别的状态变量信息：

```sql
mysql> SHOW GLOBAL STATUS;
mysql> SHOW SESSION STATUS;
```

其他详细内容参见：
>[https://dev.mysql.com/doc/refman/5.7/en/server-status-variable-reference.html](https://dev.mysql.com/doc/refman/5.7/en/server-status-variable-reference.html)



**References**
【1】https://dev.mysql.com/doc/refman/5.7/en/mysqld.html
【2】https://dev.mysql.com/doc/refman/5.7/en/mysqld-server.html
【3】https://dev.mysql.com/doc/refman/5.7/en/server-configuration.html
【4】https://dev.mysql.com/doc/refman/5.7/en/option-files.html
【5】https://dev.mysql.com/doc/refman/8.0/en/server-option-variable-reference.html
【6】https://dev.mysql.com/doc/refman/5.7/en/binary-log.html
【7】https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html
【8】https://dev.mysql.com/doc/refman/5.7/en/server-options.html



