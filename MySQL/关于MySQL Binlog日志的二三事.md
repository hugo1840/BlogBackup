---
tags: [mysql]
title: 关于MySQL Binlog日志的二三事
created: '2023-07-01T05:35:23.431Z'
modified: '2023-07-01T05:58:25.763Z'
---

关于MySQL Binlog日志的二三事

>:dolphin:数据库版本：MySQL 8.0.30

# 两个重要参数

- `binlog_expire_logs_seconds`：表示binlog日志的失效时间，默认为`30*24*60*60`秒，即30天。用于取代MySQL 5.7中的`expire_logs_days`参数。

- `binlog_rows_query_log_events=ON`：表示在binlog中记录具体的SQL语句。

开启该参数后，使用`mysqlbinlog -vv`命令解析日志可以看到具体SQL文本（以`###`开头的注释行）。这些SQL文本是给人看的，恢复数据时mysql命令实际执行的是`BINLOG '...'/*!*/;`的部分。

```bash
[mysql@mysqldb binlog]$ mysqlbinlog  -vv  mysql-bin.000509 | tail -n 35
SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722163'/*!*/;
# at 2377
#230625 10:37:45 server id 100  end_log_pos 2454 CRC32 0x53b0c2ca  Query thread_id=12 exec_time=0 error_code=0
SET TIMESTAMP=1687660665/*!*/;
BEGIN
/*!*/;
# at 2454
#230625 10:37:45 server id 100  end_log_pos 2511 CRC32 0x3e222b5b  Rows_query
# delete from t1 where title='RDR2'
# at 2511
#230625 10:37:45 server id 100  end_log_pos 2570 CRC32 0xfb85bd70  Table_map: `testdb`.`t1` mapped to number 125
# at 2570
#230625 10:37:45 server id 100  end_log_pos 2624 CRC32 0x92b8cd10  Delete_rows: table id 125 flags: STMT_END_F

BINLOG '
eaiXZB1kAAAAOQAAAM8JAACAACFkZWxldGUgZnJvbSB0MSB3aGVyZSB0aXRsZT0nUkRSMidbKyI+
eaiXZBNkAAAAOwAAAAoKAAAAAH0AAAAAAAUABnRlc3RkYgACdDEAAwgPAwKQAQIBAYACA/z/AHC9
hfs=
eaiXZCBkAAAANgAAAEAKAAAAAH0AAAAAAAEAAgAD/wAEAAAAAAAAAAQAUkRSMlgAAAAQzbiS
'/*!*/;
### DELETE FROM `testdb`.`t1`
### WHERE
###   @1=4 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @2='RDR2' /* VARSTRING(400) meta=400 nullable=1 is_null=0 */
###   @3=88 /* INT meta=0 nullable=0 is_null=0 */
# at 2624
#230625 10:37:45 server id 100  end_log_pos 2655 CRC32 0xf09ed8f7  Xid = 211
COMMIT/*!*/;
# at 2655
```

# Binlog一些使用场景
Binlog是MySQL高可用架构中数据同步的基础。

可以使用binlog来恢复误删除的数据（具体做法参见另一篇博文）：
```bash
mysqlbinlog mysql-bin.000010 mysql-bin.000011 | mysql -uroot -p 
```

可以统计binlog中的大事务：
```bash
mysqlbinlog  mysql-bin.000509 | grep -A2 '^BEGIN' | egrep '^# at' | awk '{print $3}' | awk '{if(NR==1) {at=$1} else if(NR>1) {print ($1-at); at=$1} }'

# 等价于
mysqlbinlog  mysql-bin.000509 | grep -A2 '^BEGIN' | egrep '^# at' | awk '{print $3}' | awk 'NR==1 {at=$1} NR>1 {print ($1-at); at=$1}'
```

# Binlog日志记录格式
通过**SHOW BINLOG EVENTS**命令来查看binlog中记录的事件：
```sql
root@(none)> show binlog events in 'mysql-bin.000509';
+------------------+------+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos  | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                                                                                                  |
+------------------+------+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| mysql-bin.000509 |    4 | Format_desc    |       100 |         126 | Server ver: 8.0.30, Binlog ver: 4                                                                                                                                                                                                     |
| mysql-bin.000509 |  126 | Previous_gtids |       100 |         197 | 3e863e10-eede-11ed-a3fd-005056a3eca7:1-12722156                                                                                                                                                                                       |
| mysql-bin.000509 |  197 | Gtid           |       100 |         274 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722157'                                                                                                                                                              |
| mysql-bin.000509 |  274 | Query          |       100 |         388 | create database testdb /* xid=77 */                                                                                                                                                                                                   |
| mysql-bin.000509 |  388 | Gtid           |       100 |         467 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722158'                                                                                                                                                              |
| mysql-bin.000509 |  467 | Query          |       100 |         760 | use `testdb`; CREATE TABLE `t1` (
  `my_row_id` bigint unsigned NOT NULL AUTO_INCREMENT /*!80023 INVISIBLE */,
  `title` varchar(100) DEFAULT NULL,
  `price` int NOT NULL,
  PRIMARY KEY (`my_row_id`)
) ENGINE=InnoDB /* xid=193 */ |
| mysql-bin.000509 |  760 | Gtid           |       100 |         839 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722159'                                                                                                                                                              |
| mysql-bin.000509 |  839 | Query          |       100 |         916 | BEGIN                                                                                                                                                                                                                                 |
| mysql-bin.000509 |  916 | Rows_query     |       100 |         999 | # insert into t1 (title,price) values ('Death Stranding',198)                                                                                                                                                                         |
| mysql-bin.000509 |  999 | Table_map      |       100 |        1058 | table_id: 125 (testdb.t1)                                                                                                                                                                                                             |
| mysql-bin.000509 | 1058 | Write_rows     |       100 |        1123 | table_id: 125 flags: STMT_END_F                                                                                                                                                                                                       |
| mysql-bin.000509 | 1123 | Xid            |       100 |        1154 | COMMIT /* xid=196 */                                                                                                                                                                                                                  |
| mysql-bin.000509 | 1154 | Gtid           |       100 |        1233 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722160'                                                                                                                                                              |
| mysql-bin.000509 | 1233 | Query          |       100 |        1310 | BEGIN                                                                                                                                                                                                                                 |
| mysql-bin.000509 | 1310 | Rows_query     |       100 |        1388 | # insert into t1 (title,price) values ('Elden Ring',298)                                                                                                                                                                              |
| mysql-bin.000509 | 1388 | Table_map      |       100 |        1447 | table_id: 125 (testdb.t1)                                                                                                                                                                                                             |
| mysql-bin.000509 | 1447 | Write_rows     |       100 |        1507 | table_id: 125 flags: STMT_END_F                                                                                                                                                                                                       |
| mysql-bin.000509 | 1507 | Xid            |       100 |        1538 | COMMIT /* xid=199 */                                                                                                                                                                                                                  |
| mysql-bin.000509 | 1538 | Gtid           |       100 |        1617 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722161'                                                                                                                                                              |
| mysql-bin.000509 | 1617 | Query          |       100 |        1694 | BEGIN                                                                                                                                                                                                                                 |
| mysql-bin.000509 | 1694 | Rows_query     |       100 |        1774 | # insert into t1 (title,price) values ('The Witcher 3',58)                                                                                                                                                                            |
| mysql-bin.000509 | 1774 | Table_map      |       100 |        1833 | table_id: 125 (testdb.t1)                                                                                                                                                                                                             |
| mysql-bin.000509 | 1833 | Write_rows     |       100 |        1896 | table_id: 125 flags: STMT_END_F                                                                                                                                                                                                       |
| mysql-bin.000509 | 1896 | Xid            |       100 |        1927 | COMMIT /* xid=201 */                                                                                                                                                                                                                  |
| mysql-bin.000509 | 1927 | Gtid           |       100 |        2006 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722162'                                                                                                                                                              |
| mysql-bin.000509 | 2006 | Query          |       100 |        2083 | BEGIN                                                                                                                                                                                                                                 |
| mysql-bin.000509 | 2083 | Rows_query     |       100 |        2154 | # insert into t1 (title,price) values ('RDR2',88)                                                                                                                                                                                     |
| mysql-bin.000509 | 2154 | Table_map      |       100 |        2213 | table_id: 125 (testdb.t1)                                                                                                                                                                                                             |
| mysql-bin.000509 | 2213 | Write_rows     |       100 |        2267 | table_id: 125 flags: STMT_END_F                                                                                                                                                                                                       |
| mysql-bin.000509 | 2267 | Xid            |       100 |        2298 | COMMIT /* xid=208 */                                                                                                                                                                                                                  |
| mysql-bin.000509 | 2298 | Gtid           |       100 |        2377 | SET @@SESSION.GTID_NEXT= '3e863e10-eede-11ed-a3fd-005056a3eca7:12722163'                                                                                                                                                              |
| mysql-bin.000509 | 2377 | Query          |       100 |        2454 | BEGIN                                                                                                                                                                                                                                 |
| mysql-bin.000509 | 2454 | Rows_query     |       100 |        2511 | # delete from t1 where title='RDR2'                                                                                                                                                                                                   |
| mysql-bin.000509 | 2511 | Table_map      |       100 |        2570 | table_id: 125 (testdb.t1)                                                                                                                                                                                                             |
| mysql-bin.000509 | 2570 | Delete_rows    |       100 |        2624 | table_id: 125 flags: STMT_END_F                                                                                                                                                                                                       |
| mysql-bin.000509 | 2624 | Xid            |       100 |        2655 | COMMIT /* xid=211 */                                                                                                                                                                                                                  |
| mysql-bin.000509 | 2655 | Rotate         |       100 |        2702 | mysql-bin.000510;pos=4                                                                                                                                                                                                                |
+------------------+------+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
37 rows in set (0.00 sec)
```

MySQL Binlog中以Event为单位记录数据库中的变更信息。可以看到，主要有以下事件类型：
- `Format_desc`：binlog中的第一个事件，记录了MySQL版本和binlog版本。
- `Previous_gtids`：之前的binlog中已经执行过的GTID集合。
- `Gtid`：开启GTID模式后，每个事务开始的事件、以及对应事务的GTID。
- `Anonymous_Gtid`：未开启GTID模式时，每个事务开始的事件。
- `Query`：执行过的SQL语句，一般是事务开始的BEGIN语句。也可以是CREATE等其他语句。
- `Rows_query`：记录的实际SQL文本。只有`binlog_rows_query_log_events=TRUE`时才会记录。
- `Table_map`：记录接下来将要被修改的表信息。
- `Write_rows`：向表中插入一条记录。
- `Update_rows`：修改表中的一条记录。
- `Delete_rows`：从表中删除一条记录。
- `Xid`：事务提交时写入的事务ID。
- `Rotate`：binlog日志切换事件。记录了下一个binlog的文件名。




