---
tags: [mysql]
title: MySQL Shell备份恢复可能会遇到的报错
created: '2023-04-11T12:27:34.145Z'
modified: '2023-04-11T12:59:59.831Z'
---

MySQL Shell备份恢复可能会遇到的报错

使用MySQL Shell进行备份和恢复的方法参见[MySQL Shell 8.0的Dump Utility备份与恢复](http://t.csdn.cn/csGCS)。

# MySQL Error 1226

MySQL Error 1226总是发生在备份/恢复开始的时候。

:spider:报错信息：
```bash
ERROR: [Worker009] Error opening connection to MySQL: 
MySQL Error 1226 (42000): User 'root' has exceeded the 'max_user_connections' resource (current value: 10)
```

:bird:报错原因：

使用MySQL Shell备份或恢复的并发线程数`threads`大小超过了当前备份用户的最大并发连接数`max_user_connections`。

:fish:解决办法：

减小备份的并发线程数，或者调大备份用户的最大并发连接数。

```sql
SQL> alter user root with max_user_connections 0;
-- 0表示不限制用户的并发连接数
```


# MySQL Error 1114

MySQL Error 1114通常发生在备份/恢复的过程中，会导致备份/恢复失败（部分成功）。

:spider:报错信息：

```bash
ERROR: [Worker004] db_name@table_name@84.tsv.zst:
MySQL Error 1114 (HY000): The table 'table_name' is full: LOAD DATA LOCAL INFILE ...
ERROR: Aborting load ...
```

:bird:报错原因：

一般是MySQL数据/备份目录满了，导致备份/恢复无法继续进行。

:fish:解决办法：

扩容MySQL数据/备份目录，或者清理目录后，继续进行备份/恢复。如果是非生产环境，可以清理binlog文件。

```sql
SQL> show binary logs;
SQL> purge binary logs to 'mysql-bin.001500';
SQL> purge binary logs before '2023-04-01 23:59:59';
```

扩容或数据清理完成后，再次通过MySQL Shell进行备份恢复，会从原来的进度继续进行。







