---
tags: [mysql]
title: MySQL Shell拷贝一个库到一个新库
created: '2023-12-02T03:27:30.244Z'
modified: '2023-12-02T03:36:43.521Z'
---

MySQL Shell拷贝一个库到一个新库

>:boat:场景：从同一台MySQL服务器的testdb中导出所有表。新建一个库testdb23，将导出的备份导入新建的testdb23库。


# dump-schemas还是dump-tables

默认情况下，MySQL Shell导出和导入的数据库（SCHEMA）是同一个。如果要导入到不同的数据库，需要通过`--schema`关键字指定目标库。而在MySQL 8.0.28版本中，该关键字只能用于导入**dump-tables**导出的备份。

```bash
#mysqlsh   Ver 8.0.28 for Linux on x86_64 - for MySQL 8.0.28 (MySQL Community Server (GPL))

mysqlsh -- util load-dump --help
...
--schema=<str>
        Load the dump into the given schema. This option can only be used
        when loading dumps created by the util.dumpTables() function.
        Default: not set.
```
因此我们选择dump-tables，而不是dump-schemas。需要注意的是，dump-tables只会导出**表和视图**，不会导出函数和存储过程。  

官方文档中关于使用`--schema`关键字导入视图定义时可能存在的问题：
```
From MySQL Shell 8.0.31, if the schema does not exist, it is created, 
and the dump is loaded to that new schema. 
If the new schema name differs from the schema name in the dump, the dump is loaded 
to the new schema, but no changes are made to the loaded data. 
That is, any reference to the old schema name remains in the data. 
All stored procedures, views, and so on, refer to the original schema, not the new one.
```

也就是说，导入的视图定义中引用的表会指向原始库中的表。但是根据实际情况来看，貌似MySQL 8.0.30中已经会自动更新视图定义中的schema名字了。
```sql
--Server version: 8.0.30 MySQL Community Server - GPL

SQL> show create view oradb2023.v_inf_asset;

| CREATE ... DEFINER VIEW `oradb2023`.`v_inf_asset` AS select ... from `oradb2023`.`v_gp_asset_rpt` `t` where...

SQL> show create view oradb.v_inf_asset;

| CREATE ... DEFINER VIEW `oradb`.`v_inf_asset` AS select ... from `oradb`.`v_gp_asset_rpt` `t` where ...
```

> :snake: dump-schemas和dump-tables的可用参数差异：
> 
> - 只适用于dump-instance和dump-schemas的参数:
>   - includeTables、excludeTables：过滤导出的表；
>   - includeRoutines、excludeRoutines：过滤导出的函数和存储过程；
>   - routines：是否导出函数和存储过程，默认开启。
> 
> - 只适用于dump-tables的参数:
>   - all: 导出指定SCHEMA下得所有表和视图。 


# 导入是否skipBinlog

导入数据前，使用`--dryRun`参数检查导入信息，但不进行实际的导入操作：
```bash
mysqlsh root:@127.0.0.1 -- util load-dump /mydata/backup/testdmp --threads=4 --schema=testdb23 --dryRun
```

使用`--skipBinlog`参数控制是否在导入数据时写binlog，默认关闭该参数。

在导入数据到生产主库，并且希望导入的数据同步到备库时，需要关闭该参数：
```bash
mysqlsh root:@127.0.0.1 -- util load-dump /mydata/backup/testdmp --threads=4 --schema=testdb23
```

如果不希望导入的数据同步到备库，或者仅仅在测试的单机库导入数据时，为节省日志空间可以开启该参数：
```bash
mysqlsh root:@127.0.0.1 -- util load-dump /mydata/backup/testdmp --threads=4 --schema=testdb23 --skipBinlog
```


# 实操：导出和导入

要复制的源库为testdb：
```sql
root@testdb> show tables;
+------------------+
| Tables_in_testdb |
+------------------+
| t1               |
| t2               |
+------------------+
2 rows in set (0.00 sec)

root@testdb> select count(*) from testdb.t1;
+----------+
| count(*) |
+----------+
|        5 |
+----------+
1 row in set (0.00 sec)

root@testdb> select count(*) from testdb.t2;
+----------+
| count(*) |
+----------+
|        6 |
+----------+
1 row in set (0.00 sec)
```

在同一台MySQL Server上创建导入目标库和对应用户：
```sql
create database testdb23;
create user testdb23 identified with mysql_native_password by 'XXXXXX';

grant select,insert,update,delete,create,drop,index,alter,execute,
create view,show view,create routine,alter routine,references on testdb23.* to testdb23 with grant option;
grant all on testdb23.* to testdb23 with grant option;
```

尝试使用dump-schemas导出源库所有表和视图：
```bash
mkdir /mydata/backup/testdmp

mysqlsh root:@127.0.0.1 -- util dump-schemas testdb --outputUrl=/mydata/backup/testdmp --threads=4 
```

确定要导入的库开启了**LOCAL_INFILE**参数：
```sql
SQL> set global local_infile=ON;
```

恢复到指定的数据库：
```bash
[mysql@dbhost ~]$ mysqlsh root:@127.0.0.1 -- util load-dump /mydata/backup/testdmp --threads=16 --schema=testdb23
WARNING: Using a password on the command line interface can be insecure.
Loading DDL and Data from '/mydata/backup/testdmp' using 16 threads.
Opening dump...
ERROR: The dump was not created by the util.dumpTables() function, the 'schema' option cannot be used.
ERROR: Invalid option: schema.
```
可以看到，dump-schemas导出的备份在导入时不支持schema参数指定导入的目标库。


下面使用dump-tables导出源库所有表和视图。

使用`dump-tables DBNAME [] --all`导出DBNAME库中的所有表和视图。
```bash
[mysql@dbhost ~]$ mysqlsh root:@127.0.0.1 -- util dump-tables testdb [] --all --outputUrl=/mydata/backup/testdmp --threads=4
WARNING: Using a password on the command line interface can be insecure.
Acquiring global read lock
Global read lock acquired
Initializing - done 
2 tables and 0 views will be dumped.
Gathering information - done 
All transactions have been started
Locking instance for backup
Global read lock has been released
Writing global DDL files
Running data dump using 4 threads.
NOTE: Progress information uses estimated values and may not be accurate.
Writing schema metadata - done       
Writing DDL - done       
Writing table metadata - done       
Starting data dump
110% (11 rows / ~10 rows), 0.00 rows/s, 0.00 B/s uncompressed, 0.00 B/s compressed
Dump duration: 00:00:00s                                                          
Total duration: 00:00:00s                                                         
Schemas dumped: 1                                                                 
Tables dumped: 2                                                                  
Uncompressed data size: 209 bytes                                                 
Compressed data size: 212 bytes                                                   
Compression ratio: 1.0                                                            
Rows written: 11                                                                  
Bytes written: 212 bytes                                                          
Average uncompressed throughput: 209.00 B/s                                       
Average compressed throughput: 212.00 B/s    
```

将导出的备份恢复到同一个MySQL Server中不同的数据库：
```bash
[mysql@dbhost ~]$ mysqlsh root:@127.0.0.1 -- util load-dump /mydata/backup/testdmp --threads=4 --schema=testdb23
WARNING: Using a password on the command line interface can be insecure.
Loading DDL and Data from '/mydata/backup/testdmp' using 4 threads.
Opening dump...
Target is MySQL 8.0.30. Dump was produced from MySQL 8.0.30
Scanning metadata - done       
Checking for pre-existing objects...
Executing common preamble SQL
Executing DDL - done       
Executing view DDL - done       
Starting data load
Executing common postamble SQL                       
100% (209 bytes / 209 bytes), 0.00 B/s, 2 / 2 tables done
Recreating indexes - done 
2 chunks (11 rows, 209 bytes) for 2 tables in 1 schemas were loaded in 0 sec (avg throughput 209.00 B/s)
0 warnings were reported during the load. 
```

检查导入的数据：
```sql
root@testdb> use testdb23;
Database changed

root@testdb23> show tables;
+--------------------+
| Tables_in_testdb23 |
+--------------------+
| t1                 |
| t2                 |
+--------------------+
2 rows in set (0.01 sec)

root@testdb23> select * from t1;
+------------------------+-------+
| title                  | price |
+------------------------+-------+
| Death Stranding        |   198 |
| Elden Ring             |   298 |
| Black Souls III        |   193 |
| Divinity: Origin Sin 2 |    53 |
| Titanfall 2            |    24 |
+------------------------+-------+
5 rows in set (0.00 sec)

root@testdb23> select * from t2;
+-----------------+-------+
| title           | price |
+-----------------+-------+
| Death Stranding |   198 |
| Elden Ring      |   298 |
| Dark Souls III  |   193 |
| Cuphead         |    38 |
| The Witcher 3   |    58 |
| Limbo           |    11 |
+-----------------+-------+
6 rows in set (0.00 sec)
```

# 导入时可能的报错

导入备份时可能收到以下报错：
```bash
ERROR: Error executing DDL script for view `testdb23`.`v_inf_top_operater`: 
MySQL Error 1146 (42S02): Table 'testdb23.t_ods_top_operater' doesn't exist: ...
Executing view DDL - done       
ERROR: Table 'testdb23.t_ods_top_operater' doesn't exist
```
原因是创建视图v_inf_top_operater时，发现其定义中引用了不存在的表（t_ods_top_operater），数据导入中断。

解决办法：通过`excludeTables`关键字排除报错的视图或表，然后继续导入。
```bash
mysqlsh root:@127.0.0.1 -- util load-dump /mydata/backup/testdmp --threads=4 --schema=testdb23 \
--excludeTables=testdb.v_inf_top_operater,testdb.v_xx_xxx,...
```
默认会从中断时的进度继续导入，无需清理数据重新开始。也可以手动清理数据后使用`--resetProgress`参数重置导入进度。


如果有太多太多视图导入报错，可以排除掉所有视图，只导入表。
```sql
--梳理源库中所有的视图名称
select table_schema,table_name from information_schema.views where table_schema='testdb';   --> 只包含视图

select table_schema,table_name from information_schema.tables where table_schema='testdb';  --> 包含表和视图

--拼接数据库名和视图名称
select group_concat(table_name) from information_schema.views where table_schema='testdb'\G

select group_concat(concat_ws('.',table_schema,table_name) separator ',') 
from information_schema.views where table_schema='testdb'\G
```

最后建议：数据量不大的话，时效要求不高的话，推荐优先使用mysqldump，只需替换SQL文件中的数据库名即可。


**References**
【1】https://gottdeskrieges.blog.csdn.net/article/details/130033301
【2】https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-load-dump.html#mysql-shell-utilities-load-dump-opt-control





