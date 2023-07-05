---
tags: [mysql]
title: 通过mysqlbinlog恢复误删除数据
created: '2023-07-01T05:34:51.282Z'
modified: '2023-07-05T12:49:32.352Z'
---

通过mysqlbinlog恢复误删除数据

>:dolphin:数据库版本：MySQL 8.0.30

# 问题与解决思路
应用用户误操作使用DELETE语句删除了某张表中的几十万条数据，需要进行数据恢复。数据库备份场景为每天凌晨做一次全量备份。应用没有当天的应用库数据备份。

实际的恢复场景中，通常是搭一个新库，拿前一天的全备在新库做一个全量恢复，然后从原库拷贝当天的binlog文件进行增量恢复。等到新库恢复到数据删除时间点前时，对数据进行对比校验，再将被误删除的数据单独导出来，导入生产环境的原库即可。

生产环境不建议在原库直接进行恢复。因为同一个库中，同一个GTID号只能对应一个事务，因此直接执行下面的语句什么也不会发生：
```bash
mysqlbinlog --start-position=1927 --stop-position=2298 -d testdb /mysql/data/binlog/mysql-bin.000509 | mysql -uroot -h127.0.0.1
```
指定`skip-gtids=true`参数可以把binlog日志中已经执行过的语句当成一个新事务来执行才行，但是这样会把两个时间点中间的所有事务都重新执行一遍，可能会导致某些数据被重复插入，造成数据不一致。

## 场景一：直接使用binlog日志恢复
如果是在新库进行数据恢复，在全量恢复之后直接拷贝当天误删除动作时间点之前的binlog，重新执行即可。

1. **全量恢复**

根据需求，全量恢复可以使用xtrabackup、mysqlbackup、mysqldump或者MySQL Shell工具。
```bash
mysqlsh root:@127.0.0.1 -- util loadDump /mydata/backup/testdb/testdb_2023-06-19 --threads=8 --deferTableIndexes=all --analyzeTables=on 
```

>**XtraBackup**：https://blog.csdn.net/Sebastien23/article/details/126820058
>**MySQL Shell**：https://blog.csdn.net/Sebastien23/article/details/130033301


2. **增量恢复**

增量恢复要使用到mysqlbinlog日志解析工具。手动切换binlog日志文件，并检查最新的binlog文件名：
```sql
SQL> flush logs;
SQL> show master status\G
SQL> show binary logs;
```

通过show命令或者mysqlbinlog来定位问题发生时段的日志内容：
```sql
SQL> show binlog events in 'mysql-bin.000509';
SQL> show binlog events in 'mysql-bin.000509' from [pos];
```

按时间段或点位解析binlog，定位到日志中的DELETE语句：
```bash
mysqlbinlog --start-datetime="2023-04-19 14:11:44"  -vv --base64-output=decode-rows mysql-bin.000509 | more
mysqlbinlog --start-datetime="2023-04-19 14:11:44" --stop-position=489135201 -d testdb mysql-bin.000509 | more
```

重新执行当天删除时间点之前的所有事务。假设当天binlog中第一个事务的起始位置为489798090，删除动作前最后一个事务的结束位置为495135201：
```bash
mysqlbinlog --start-position=489798090 --stop-position=495135201 -d testdb binlog-del2insert.sql | mysql -uroot -h127.0.0.1
```

## 场景二：替换binlog中的删除事件

如果是直接在原库进行数据恢复（生产环境不建议直接操作），需要将binlog中的DELETE语句部分拷贝到单独的文件中，修改为INSERT语句后再重新插入数据即可：
```bash
mysql -uroot -p -D testdb < del2insert.sql
```

# 数据误删除恢复测试
## 模拟删除场景
创建测试表并插入数据：
```sql
root@(none)> use testdb;

root@testdb> create table t1 (
    title varchar(100) default null,
    price int not null) engine=innodb;

root@testdb> insert into t1 (title,price) values ('Death Stranding',198);
root@testdb> insert into t1 (title,price) values ('Elden Ring',298);
root@testdb> insert into t1 (title,price) values ('The Witcher 3',58);

root@testdb> select * from testdb.t1;
+-----------------+-------+
| title           | price |
+-----------------+-------+
| Death Stranding |   198 |
| Elden Ring      |   298 |
| The Witcher 3   |    58 |
+-----------------+-------+
3 rows in set (0.00 sec)

root@testdb> insert into t1 (title,price) values ('RDR2',88);

root@testdb> select * from testdb.t1;
+-----------------+-------+
| title           | price |
+-----------------+-------+
| Death Stranding |   198 |
| Elden Ring      |   298 |
| The Witcher 3   |    58 |
| RDR2            |    88 |
+-----------------+-------+
4 rows in set (0.00 sec)
```

模拟误删除场景：
```sql
root@testdb> delete from t1 where title='RDR2';
Query OK, 1 row affected (0.01 sec)

root@testdb> select * from testdb.t1;
+-----------------+-------+
| title           | price |
+-----------------+-------+
| Death Stranding |   198 |
| Elden Ring      |   298 |
| The Witcher 3   |    58 |
+-----------------+-------+
3 rows in set (0.00 sec)
```

## 定位删除时间点
检查最新的binlog中记录的事件：
```sql
--获取最新的binlog文件名
root@testdb> show binary logs;
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000509 |      2655 | No        |
+------------------+-----------+-----------+
1 row in set (0.00 sec)

--检查当天binlog文件中的事件
root@testdb> show binlog events in 'mysql-bin.000509';
root@testdb> show binlog events in 'mysql-bin.000509' from 760;
+------------------+------+-------------+-----------+-------------+--------------------------------------------------------------------------+
| Log_name         | Pos  | Event_type  | Server_id | End_log_pos | Info                                                                     |
+------------------+------+-------------+-----------+-------------+--------------------------------------------------------------------------+
| mysql-bin.000509 |  760 | Gtid        |       100 |         839 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722159' |
| mysql-bin.000509 |  839 | Query       |       100 |         916 | BEGIN                                                                    |
| mysql-bin.000509 |  916 | Rows_query  |       100 |         999 | # insert into t1 (title,price) values ('Death Stranding',198)            |
| mysql-bin.000509 |  999 | Table_map   |       100 |        1058 | table_id: 125 (testdb.t1)                                                |
| mysql-bin.000509 | 1058 | Write_rows  |       100 |        1123 | table_id: 125 flags: STMT_END_F                                          |
| mysql-bin.000509 | 1123 | Xid         |       100 |        1154 | COMMIT /* xid=196 */                                                     |
| mysql-bin.000509 | 1154 | Gtid        |       100 |        1233 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722160' |
| mysql-bin.000509 | 1233 | Query       |       100 |        1310 | BEGIN                                                                    |
| mysql-bin.000509 | 1310 | Rows_query  |       100 |        1388 | # insert into t1 (title,price) values ('Elden Ring',298)                 |
| mysql-bin.000509 | 1388 | Table_map   |       100 |        1447 | table_id: 125 (testdb.t1)                                                |
| mysql-bin.000509 | 1447 | Write_rows  |       100 |        1507 | table_id: 125 flags: STMT_END_F                                          |
| mysql-bin.000509 | 1507 | Xid         |       100 |        1538 | COMMIT /* xid=199 */                                                     |
| mysql-bin.000509 | 1538 | Gtid        |       100 |        1617 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722161' |
| mysql-bin.000509 | 1617 | Query       |       100 |        1694 | BEGIN                                                                    |
| mysql-bin.000509 | 1694 | Rows_query  |       100 |        1774 | # insert into t1 (title,price) values ('The Witcher 3',58)               |
| mysql-bin.000509 | 1774 | Table_map   |       100 |        1833 | table_id: 125 (testdb.t1)                                                |
| mysql-bin.000509 | 1833 | Write_rows  |       100 |        1896 | table_id: 125 flags: STMT_END_F                                          |
| mysql-bin.000509 | 1896 | Xid         |       100 |        1927 | COMMIT /* xid=201 */                                                     |
| mysql-bin.000509 | 1927 | Gtid        |       100 |        2006 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722162' |
| mysql-bin.000509 | 2006 | Query       |       100 |        2083 | BEGIN                                                                    |
| mysql-bin.000509 | 2083 | Rows_query  |       100 |        2154 | # insert into t1 (title,price) values ('RDR2',88)                        |
| mysql-bin.000509 | 2154 | Table_map   |       100 |        2213 | table_id: 125 (testdb.t1)                                                |
| mysql-bin.000509 | 2213 | Write_rows  |       100 |        2267 | table_id: 125 flags: STMT_END_F                                          |
| mysql-bin.000509 | 2267 | Xid         |       100 |        2298 | COMMIT /* xid=208 */                                                     |
| mysql-bin.000509 | 2298 | Gtid        |       100 |        2377 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722163' |
| mysql-bin.000509 | 2377 | Query       |       100 |        2454 | BEGIN                                                                    |
| mysql-bin.000509 | 2454 | Rows_query  |       100 |        2511 | # delete from t1 where title='RDR2'                                      |
| mysql-bin.000509 | 2511 | Table_map   |       100 |        2570 | table_id: 125 (testdb.t1)                                                |
| mysql-bin.000509 | 2570 | Delete_rows |       100 |        2624 | table_id: 125 flags: STMT_END_F                                          |
| mysql-bin.000509 | 2624 | Xid         |       100 |        2655 | COMMIT /* xid=211 */                                                     |
+------------------+------+-------------+-----------+-------------+--------------------------------------------------------------------------+
30 rows in set (0.00 sec)
```
其中，`Delete_rows`对应删除事件，事务开始的位置（**Pos**）为2377，结束的位置（**End_log_pos**）为2655。该事务对应GTID生成的SET语句的开始位置为**2298**。

手动触发日志切换，生成新的binlog：
```sql
root@(none)> show master status;
root@(none)> flush logs;
Query OK, 0 rows affected (0.01 sec)
```

同时使用mysqlbinlog工具解析binlog来分析定位删除事件，通过`-d`指定数据库：
```bash
#假设业务告知删除动作的时间点在10:35到10:40之间
[mysql@mysqldb ~]$ mysqlbinlog -vv --base64-output=decode-rows --start-datetime='2023-06-25 10:35:00' --stop-datetime='2023-06-25 10:40:00'  -d testdb /mysql/data/binlog/mysql-bin.000509 | grep -i delete
# delete from t1 where title='RDR2'
#230625 10:37:45 server id 100  end_log_pos 2624 CRC32 0x92b8cd10  Delete_rows: table id 125 flags: STMT_END_F
### DELETE FROM `testdb`.`t1`

[mysql@mysqldb ~]$ mysqlbinlog -vv --base64-output=decode-rows --start-datetime='2023-06-25 10:35:00' --stop-datetime='2023-06-25 10:40:00'  -d testdb /mysql/data/binlog/mysql-bin.000509 | less
```
这里我们获取到删除动作的时间为上午10点37分。

## 直接使用binlog日志恢复
假设我们已经拿前一天的备份在新库上做了全量恢复，就可以直接拿误删除数据当天的binlog来进行数据增量恢复。

前面我们已经确定了删除动作发生在10点37分，这里可以使用mysqlbinlog解析记录了删除事件的Binlog，来获取当天零点以及删除时间点对应的binlog日志位置（Pos和End_log_pos）：
```bash
[mysql@mysqldb ~]$ mysqlbinlog -vv --base64-output=decode-rows --start-datetime='2023-06-25 00:00:00' --stop-datetime='2023-06-25 10:40:00'  -d testdb /mysql/data/binlog/mysql-bin.000509 | less
```

通过解析binlog拿到日志位置信息后，与做完前一天全量恢复的新库中的事务信息进行对比（`show binlog events in 'xxx'`）。

假设最终我们确定了当天凌晨备份后第一个事务的起始位置为2006（`SET @@SESSION.GTID_NEXT`事件对应的**Pos**），误删除动作前的最后一个事务的结束位置为2298（`COMMIT`事件对应的**End_log_pos**）。

解析并执行对应区间的binlog日志来进行增量恢复：
```bash
mysqlbinlog --start-position=2006 --stop-position=2298 -d testdb /mysql/data/binlog/mysql-bin.000509 | mysql -uroot -h127.0.0.1
```

检查恢复后的内容：
```sql
root@(none)> select * from testdb.t1;
+-----------------+-------+
| title           | price |
+-----------------+-------+
| Death Stranding |   198 |
| Elden Ring      |   298 |
| The Witcher 3   |    58 |
| RDR2            |    88 |
+-----------------+-------+
4 rows in set (0.00 sec)

root@(none)> show binlog events in 'mysql-bin.000510';
```

## 替换binlog中的删除事件
由于事务与GTID的唯一对应性，如果想直接在原库进行数据恢复，需要找到删除动作对应的DELETE语句，并将其统一替换为INSERT语句来重新插入数据。该方法需要数据库开启了`binlog_rows_query_log_events`参数。

定位到DELETE语句的位置：
```sql
mysql> show binlog events in 'mysql-bin.000012' from 1833;
+------------------+------+-------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name         | Pos  | Event_type  | Server_id | End_log_pos | Info                                                               |
+------------------+------+-------------+-----------+-------------+--------------------------------------------------------------------+
| mysql-bin.000012 | 1833 | Gtid        |       100 |        1912 | SET @@SESSION.GTID_NEXT= '1be40f24-a524-11ed-8c6e-00163e01628b:37' |
| mysql-bin.000012 | 1912 | Query       |       100 |        1989 | BEGIN                                                              |
| mysql-bin.000012 | 1989 | Table_map   |       100 |        2045 | table_id: 90 (testdb.t1)                                           |
| mysql-bin.000012 | 2045 | Delete_rows |       100 |        2091 | table_id: 90 flags: STMT_END_F                                     |
| mysql-bin.000012 | 2091 | Xid         |       100 |        2122 | COMMIT /* xid=17 */                                                |
| mysql-bin.000012 | 2122 | Rotate      |       100 |        2169 | mysql-bin.000013;pos=4                                             |
+------------------+------+-------------+-----------+-------------+--------------------------------------------------------------------+
6 rows in set (0.01 sec)
```

截取对应时间段的binlog日志内容：
```bash
[root@mysqldb log]# mysqlbinlog -vv --base64-output=decode-rows --start-position=1833 -d testdb mysql-bin.000012
# The proper term is pseudo_replica_mode, but we use this compatibility alias
# to make the statement usable on server versions 8.0.24 and older.
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 197
#230701 15:49:03 server id 100  end_log_pos 126 CRC32 0x7afa177c        Start: binlog v 4, server v 8.0.32 created 230701 15:49:03 at startup
ROLLBACK/*!*/;
# at 1833
#230701 15:52:01 server id 100  end_log_pos 1912 CRC32 0x002cdfb4       GTID    last_committed=6        sequence_number=7       rbr_only=yes    original_committed_timestamp=1688197921717746     immediate_commit_timestamp=1688197921717746     transaction_length=289
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1688197921717746 (2023-07-01 15:52:01.717746 CST)
# immediate_commit_timestamp=1688197921717746 (2023-07-01 15:52:01.717746 CST)
/*!80001 SET @@session.original_commit_timestamp=1688197921717746*//*!*/;
/*!80014 SET @@session.original_server_version=80032*//*!*/;
/*!80014 SET @@session.immediate_server_version=80032*//*!*/;
SET @@SESSION.GTID_NEXT= '1be40f24-a524-11ed-8c6e-00163e01628b:37'/*!*/;
# at 1912
#230701 15:52:01 server id 100  end_log_pos 1989 CRC32 0xa67b0559       Query   thread_id=8     exec_time=0     error_code=0
SET TIMESTAMP=1688197921/*!*/;
SET @@session.pseudo_thread_id=8/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1073741824/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=255,@@session.collation_connection=255,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
BEGIN
/*!*/;
# at 1989
#230701 15:52:01 server id 100  end_log_pos 2045 CRC32 0x438a8227       Table_map: `testdb`.`t1` mapped to number 90
# has_generated_invisible_primary_key=0
# at 2045
#230701 15:52:01 server id 100  end_log_pos 2091 CRC32 0x8aabee8d       Delete_rows: table id 90 flags: STMT_END_F
### DELETE FROM `testdb`.`t1`
### WHERE
###   @1='RDR2' /* VARSTRING(300) meta=300 nullable=1 is_null=0 */
###   @2=88 /* INT meta=0 nullable=0 is_null=0 */
# at 2091
#230701 15:52:01 server id 100  end_log_pos 2122 CRC32 0x4b045b98       Xid = 17
COMMIT/*!*/;
# at 2122
#230701 15:53:17 server id 100  end_log_pos 2169 CRC32 0x39692bd0       Rotate to mysql-bin.000013  pos: 4
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;

# 保存到指定文件 
[root@mysqldb log]# mysqlbinlog -vv --base64-output=decode-rows --start-position=1833 -d testdb mysql-bin.000012 > /tmp/binlog_1833.log
```

获取到具体的DELETE语句：
```bash
[root@mysqldb tmp]# sed -n '/^###/p' binlog_1833.log > del2insert.sql
[root@mysqldb tmp]# sed -i 's/### //g' del2insert.sql 
[root@mysqldb tmp]# cat del2insert.sql 
DELETE FROM `testdb`.`t1`
WHERE
  @1='RDR2' /* VARSTRING(300) meta=300 nullable=1 is_null=0 */
  @2=88 /* INT meta=0 nullable=0 is_null=0 */
```

将DELETE语句替换为INSERT语句：
```bash
[root@mysqldb tmp]# sed -i 's/DELETE FROM/INSERT INTO/g' del2insert.sql
[root@mysqldb tmp]# sed -i 's/WHERE/SELECT/g' del2insert.sql
[root@mysqldb tmp]# cat del2insert.sql 
INSERT INTO `testdb`.`t1`
SELECT
  @1='RDR2' /* VARSTRING(300) meta=300 nullable=1 is_null=0 */
  @2=88 /* INT meta=0 nullable=0 is_null=0 */
```

生成最终可执行的SQL：
```bash
# 去掉行末的注释/*...*/
[root@mysqldb tmp]# sed -i 's# /.*#,#g' del2insert.sql
[root@mysqldb tmp]# cat del2insert.sql 
INSERT INTO `testdb`.`t1`
SELECT
  @1='RDR2',
  @2=88,

# 将每条INSERT语句的最后一个逗号替换为分号
[root@mysqldb tmp]# sed -ri 's#(@2=.*)(,)#\1;#g' del2insert.sql 
[root@mysqldb tmp]# cat del2insert.sql 
INSERT INTO `testdb`.`t1`
SELECT
  @1='RDR2',
  @2=88;

# 去掉@数字=  
[root@mysqldb tmp]# sed -ri 's#(@.*=)(.*)#\2#g' del2insert.sql 
[root@mysqldb tmp]# cat del2insert.sql 
INSERT INTO `testdb`.`t1`
SELECT
  'RDR2',
  88;

# 添加COMMIT语句
[root@mysqldb tmp]# sed -i '$a commit;' del2insert.sql 
[root@mysqldb tmp]# cat del2insert.sql 
INSERT INTO `testdb`.`t1`
SELECT
  'RDR2',
  88;
commit;
```

使用生成的INSERT脚本来恢复数据：
```bash
[root@mysqldb tmp]# mysql -uroot -p -D testdb < del2insert.sql
```

检查恢复后的数据：
```sql
mysql> select * from testdb.t1;
+-----------------+-------+
| title           | price |
+-----------------+-------+
| Death Stranding |   198 |
| Elden Ring      |   298 |
| The Witcher 3   |    58 |
| RDR2            |    88 |
+-----------------+-------+
4 rows in set (0.00 sec)

mysql> show binary logs;
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000011 |       197 | No        |
| mysql-bin.000012 |      2169 | No        |
| mysql-bin.000013 |       486 | No        |
+------------------+-----------+-----------+
3 rows in set (0.00 sec)

mysql> show binlog events in 'mysql-bin.000013';
+------------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                               |
+------------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
| mysql-bin.000013 |   4 | Format_desc    |       100 |         126 | Server ver: 8.0.32, Binlog ver: 4                                  |
| mysql-bin.000013 | 126 | Previous_gtids |       100 |         197 | 1be40f24-a524-11ed-8c6e-00163e01628b:1-37                          |
| mysql-bin.000013 | 197 | Gtid           |       100 |         276 | SET @@SESSION.GTID_NEXT= '1be40f24-a524-11ed-8c6e-00163e01628b:38' |
| mysql-bin.000013 | 276 | Query          |       100 |         353 | BEGIN                                                              |
| mysql-bin.000013 | 353 | Table_map      |       100 |         409 | table_id: 90 (testdb.t1)                                           |
| mysql-bin.000013 | 409 | Write_rows     |       100 |         455 | table_id: 90 flags: STMT_END_F                                     |
| mysql-bin.000013 | 455 | Xid            |       100 |         486 | COMMIT /* xid=57 */                                                |
+------------------+-----+----------------+-----------+-------------+--------------------------------------------------------------------+
7 rows in set (0.00 sec)
```





