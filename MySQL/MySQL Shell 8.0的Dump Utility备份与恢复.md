---
tags: [mysql]
title: MySQL Shell 8.0的Dump Utility备份与恢复
created: '2023-04-08T06:52:22.543Z'
modified: '2023-04-08T11:36:48.550Z'
---

MySQL Shell 8.0的Dump Utility备份与恢复

# mysqldump逻辑备份与恢复
MYSQLDUMP常用来做MySQL数据库逻辑备份与恢复。由于备份是以SQL语句的形式导出，在恢复时需要重放SQL语句，效率很低，因此一般在备份数据量较小时较为适用。

## mysqldump备份
MySQLDUMP备份数据库：
```bash
# 备份多个数据库
mysqldump -h${MYSQL_Server_IP} -P${MYSQL_Server_PORT} -u${BACKUP_User} \
--set-gtid-purged=OFF --single-transaction --skip-opt --add-drop-database \
-B 数据库名1 数据库名2 数据库名3 ... > /backup/dump_full_db_`date +%F`.sql

# 压缩备份 
gzip dump_full_db_xxxx.sql > dump_full_db_xxxx.sql.gz
```

也可以直接压缩备份： 
```bash
mysqldump -h${MYSQL_Server_IP} -P${MYSQL_Server_PORT} -u${BACKUP_User} \
--set-gtid-purged=OFF --single-transaction --skip-opt --add-drop-database \
-B 数据库名1 数据库名2 数据库名3 ... | gzip > /backup/dump_full_db_`date +%F`.sql.gz
```

MYSQLDUMP单独备份表（不加`-B`）：
```bash
mysqldump -h${MYSQL_Server_IP} -P${MYSQL_Server_PORT} -u${BACKUP_User} \
--set-gtid-purged=OFF --single-transaction --skip-opt --add-drop-database \
数据库名 表名1 表名2 表名3 ... > /backup/dump_dbname_`date +%F`.sql
```

## mysqldump恢复
恢复数据库（需要先清理目标库中的旧数据）：
```bash
# 解压
gunzip dump_full_db_xxxx.sql.gz

# 恢复数据库（方法一）
mysql -h${MYSQL_Server_IP} -P${MYSQL_Server_PORT} -u${BACKUP_User} \
-e "source dump_full_db_xxxx.sql" 数据库名

# 恢复数据库（方法二）
mysql -h${MYSQL_Server_IP} -P${MYSQL_Server_PORT} -u${BACKUP_User} 数据库名 < dump_full_db_xxxx.sql
```

恢复数据表：
```bash
mysql -h${MYSQL_Server_IP} -P${MYSQL_Server_PORT} -u${BACKUP_User} \
-e "use dbname; source /backup/dump_dbname_xxxx.sql"
```


# MySQL Shell 8.0的Dump & Load特性
MySQL Shell 8.0的Dump Utility特性支持实例、Schema、数据表三个级别的MySQL数据导出功能。Dump & Load特性自带兼容性检查、并行导入导出、以及备份文件压缩，而且效率比MYSQLDUMP更高。

主要的使用限制如下：
- 导入备份的目标库必须是MySQL 5.7或者更新的版本；
- MySQL Shell 8.0.27之前的版本无法导入由8.0.27及其之后的版本导出的备份；
- 数据库对象名称必须是latin1或者utf8字符集；
- 只能保证InnoDB表的数据一致性；
- 备份用户至少必须具有EVENT、RELOAD、SELECT、SHOW VIEW、TRIGGER权限；
- 表级别的备份不支持导出Routines。


## 备份实例：dump-instance 
MySQL Shell的实例备份特性可用于导出多个用户和Schema的数据。MySQL中的Schema可以近似理解为Database的概念。

检查要备份的用户Schema：
```sql
select group_concat(user) from mysql.user 
where user not like 'mysql%' and user not in ('root');
```

实例级别备份： 
```bash
mysqlsh mysql://root@localhost:3306 -- util dump-instance <导出备份的存放路径> \
--tzUtc=false --threads=4 \     
--excludeSchemas=mysql_innodb_cluster_metadata,mysql_schema \
--includeUsers=user_1,user_2,...
```
其中：
- `tzUtc=false`：保留源数据时间戳（不会因为导入目标库跨时区而变化）。
- `threads`：数据导出的并行度，默认为4，可按需调大。
- `excludeSchemas`：（仅备份实例可用参数）不用导出的Schema。注意：`information_schema`、`mysql`、`ndbinfo`、`performance_schema`、以及`sys`这几个数据库，在导出实例时默认不会被导出。
- `includeUsers`：（仅备份实例可用参数）要导出的用户清单。

示例：
```bash
[root@iZ0jl2qhfpcmxu641m5jntZ ~]# mysqlsh mysql://root@localhost:3306 -- util dump-instance /mysql/backup/dp_instance/ \
> --tzUtc=false --threads=4 \
> --excludeSchemas=mysql_innodb_cluster_metadata,mysql_schema \
> --includeUsers=appuser
Acquiring global read lock
Global read lock acquired
Initializing - done 
3 out of 7 schemas will be dumped and within them 3 tables, 0 views.
1 out of 6 users will be dumped.
Gathering information - done 
All transactions have been started
Locking instance for backup
Global read lock has been released
Writing global DDL files
Writing users DDL
Running data dump using 4 threads.
NOTE: Progress information uses estimated values and may not be accurate.
Writing schema metadata - done       
Writing DDL - done       
Writing table metadata - done       
Starting data dump
100% (20.01K rows / ~19.89K rows), 0.00 rows/s, 0.00 B/s uncompressed, 0.00 B/s compressed
Dump duration: 00:00:00s                                                                  
Total duration: 00:00:00s                                                                 
Schemas dumped: 3                                                                         
Tables dumped: 3                                                                          
Uncompressed data size: 395.80 KB                                                         
Compressed data size: 91.65 KB                                                            
Compression ratio: 4.3                                                                    
Rows written: 20006                                                                       
Bytes written: 91.65 KB                                                                   
Average uncompressed throughput: 395.80 KB/s                                              
Average compressed throughput: 91.65 KB/s                                                 
[root@iZ0jl2qhfpcmxu641m5jntZ ~]# ls /mysql/backup/dp_instance/ | wc -l
23
[root@iZ0jl2qhfpcmxu641m5jntZ ~]# ls /mysql/backup/dp_instance/ 
appdb.json                     appdb@temp_seq.json              appdb@test_table.json        app_game@persons@@0.tsv.zst.idx  app_work.json  @.post.sql
appdb.sql                      appdb@temp_seq.sql               appdb@test_table.sql         app_game@persons.json            app_work.sql   @.sql
appdb@temp_seq@@0.tsv.zst      appdb@test_table@@0.tsv.zst      app_game.json                app_game@persons.sql             @.done.json    @.users.sql
appdb@temp_seq@@0.tsv.zst.idx  appdb@test_table@@0.tsv.zst.idx  app_game@persons@@0.tsv.zst  app_game.sql                     @.json
```

然后可以对备份目录打包，传输到目标库进行恢复。

## 备份库：dump-schemas
MySQL Shell的Schema备份特性可用于导出指定的多个库的多张表数据。

检查要备份的用户Schema：
```sql
show databases;

select table_schema,table_name from information_schema.tables 
where table_schema='数据库名';
```

Schema级别备份： 
```bash
mysqlsh mysql://root@localhost:3306 -- util dump-schemas dbname1,dbname2,... \
--outputUrl=<导出备份的存放路径> --threads=4 \
--includeTables=dbname1.tabname_1,dbname2.tabname_2,...
```
可以使用`--includeTables`和`--excludeTables`参数来筛选要导出的表。

示例：
```bash
[root@iZ0jl2qhfpcmxu641m5jntZ backup]# mysqlsh mysql://root@localhost:3306 -- util dump-schemas appdb,app_game \
> --outputUrl=/mysql/backup/dp_schemas \
> --threads=4 --includeTables=appdb.test_table,app_game.persons
Acquiring global read lock
Global read lock acquired
Initializing - done 
2 schemas will be dumped and within them 2 out of 3 tables, 0 views.
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
101% (10.01K rows / ~9.89K rows), 0.00 rows/s, 0.00 B/s uncompressed, 0.00 B/s compressed
Dump duration: 00:00:00s                                                                 
Total duration: 00:00:00s                                                                
Schemas dumped: 2                                                                        
Tables dumped: 2                                                                         
Uncompressed data size: 346.90 KB                                                        
Compressed data size: 70.67 KB                                                           
Compression ratio: 4.9                                                                   
Rows written: 10006                                                                      
Bytes written: 70.67 KB                                                                  
Average uncompressed throughput: 346.90 KB/s                                             
Average compressed throughput: 70.67 KB/s                                                
[root@iZ0jl2qhfpcmxu641m5jntZ backup]# 
[root@iZ0jl2qhfpcmxu641m5jntZ backup]# ls dp_schemas/
appdb.json  appdb@test_table@@0.tsv.zst      appdb@test_table.json  app_game.json                app_game@persons@@0.tsv.zst.idx  app_game@persons.sql  @.done.json  @.post.sql
appdb.sql   appdb@test_table@@0.tsv.zst.idx  appdb@test_table.sql   app_game@persons@@0.tsv.zst  app_game@persons.json            app_game.sql          @.json       @.sql
```

## 备份表：dump-tables
MySQL Shell的表级别备份特性可用于导出指定的单个库的多张表数据。

检查要备份的表：
```sql
select table_schema,table_name from information_schema.tables 
where table_schema='数据库名';
```

表级别备份： 
```bash
mysqlsh mysql://root@localhost:3306 -- util dump-tables <数据库名> 
tabname1,tabname2,... \
--outputUrl=<导出备份的存放路径> --threads=4 
```

示例：
```bash
[root@iZ0jl2qhfpcmxu641m5jntZ backup]# mysqlsh mysql://root@localhost:3306 -- util dump-tables appdb temp_seq,test_table \
> --threads=4 --outputUrl=/mysql/backup/dp_tables
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
100% (20.00K rows / ~19.88K rows), 0.00 rows/s, 0.00 B/s uncompressed, 0.00 B/s compressed
Dump duration: 00:00:00s                                                                  
Total duration: 00:00:00s                                                                 
Schemas dumped: 1                                                                         
Tables dumped: 2                                                                          
Uncompressed data size: 395.61 KB                                                         
Compressed data size: 91.45 KB                                                            
Compression ratio: 4.3                                                                    
Rows written: 20000                                                                       
Bytes written: 91.45 KB                                                                   
Average uncompressed throughput: 395.61 KB/s                                              
Average compressed throughput: 91.45 KB/s                                                 
[root@iZ0jl2qhfpcmxu641m5jntZ backup]# ls dp_tables/
appdb.json  appdb@temp_seq@@0.tsv.zst      appdb@temp_seq.json  appdb@test_table@@0.tsv.zst      appdb@test_table.json  @.done.json  @.post.sql
appdb.sql   appdb@temp_seq@@0.tsv.zst.idx  appdb@temp_seq.sql   appdb@test_table@@0.tsv.zst.idx  appdb@test_table.sql   @.json       @.sql
```


## 恢复数据：load-dump
 MySQL Shell的loadDump特性用于导入使用Dump Utility导出的备份。

 其需要注意的主要有以下几点：
 - loadDump通过执行`LOAD DATA LOCAL INFILE`语句来导入数据，因此在备份导入期间目标库上的全局参数`local_infile`必须设置为**ON**。
 - 如果目标库上开启了`sql_require_primary_key`参数（默认为OFF），loadDump会检查导入表是否包含主键，如果没有主键就会报错。
 - loadDump不会主动在目标库应用从源库导出的GTID set。如果目标库要用于搭建副本从库，需要使用`updateGtidSet`参数手动导入GTID。

导入备份的语法如下：
```bash
mysqlsh mysql://root@localhost:3306 -- util load-dump <导入备份的存放路径> --threads=4
```

**示例**

事先清理旧数据：
```sql
drop database app_game;
drop database app_work;
drop database appdb;
```

检查LOCAL_INFILE参数：
```sql
SQL > show variables like 'local_infile';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| local_infile  | OFF   |
+---------------+-------+
1 row in set (0.0019 sec)

SQL > set global local_infile=ON;
```

导入备份：
```bash
[root@iZ0jl2qhfpcmxu641m5jntZ backup]# mysqlsh mysql://root@localhost:3306 -- util load-dump /mysql/backup/dp_instance
Loading DDL and Data from '/mysql/backup/dp_instance' using 4 threads.
Opening dump...
Target is MySQL 8.0.32. Dump was produced from MySQL 8.0.32
Scanning metadata - done       
Checking for pre-existing objects...
Executing common preamble SQL
Executing DDL - done       
Executing view DDL - done       
Starting data load
Executing common postamble SQL                       
100% (395.80 KB / 395.80 KB), 0.00 B/s, 3 / 3 tables done
Recreating indexes - done 
3 chunks (20.01K rows, 395.80 KB) for 3 tables in 3 schemas were loaded in 0 sec (avg throughput 395.80 KB/s)
0 warnings were reported during the load. 
```


**References**
[1] https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-dump-instance-schema.html
<br/>[2] https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-load-dump.html



