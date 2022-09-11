@[TOC](MySQL备份与恢复工具之MYSQLDUMP)

mysqldump是MySQL服务端自带的**逻辑备份**工具。

# :apple: 用法
```bash
#导出逻辑备份文件
mysqldump -uroot -p db-name > backup-file.sql

#导入mysqldump导出的dump文件
mysql -uroot -p db-name < /tmp/backup-file.sql
mysql -uroot -p -e "source /tmp/backup-file.sql" db-name
```

mysqldump命令有很多可选参数，其中比较重要的包括：

- `--all-databases`：导出所有数据库中所有表的数据。
- `--databases`：导出多个给定的数据库（不加这个参数就不会生成`CREATE DATABASE`建表语句）。
- `--triggers`：导出表中定义的的触发器。
- `--routines`：导出数据库中定义的存储过程和函数。
- `--events`：导出数据库中创建的事件（类似于Linux crontab）。
- `--skip-add-drop-table`：mysqldump默认会在`CREATE TABLE`语句前加上`DROP TABLE`语句，使用该参数则不会在恢复表数据前删除表。
- `--lock-all-tables`：一次性锁定所有数据库中的所有表，在整个导出dump期间会一直持有全局读锁（**Global Read Lock**）。使用该参数时会自动关闭`--single-transaction`和`--lock-tables`。
- `--lock-tables`：以数据库为单位，在导出数据时锁定单个库中的所有表（因此不能保证不同数据库之间的逻辑一致性）。对基于MyISAM的表采用Read Local锁，允许在导出数据的同时进行INSERT操作；对基于InnoDB的表，建议使用`--single-transaction`来替代以避免锁表。
- `--single-transaction`：会在备份开始前以RR隔离级别执行`START TRANSACTION`开启一个事务。备份的数据都来自于事务开始前获取的一致性视图，因此在备份过程中不会阻塞应用。
  * 备份时需要开启事务，因此仅支持InnoDB表的一致性备份；
  * 备份期间不会阻塞DML和DDL语句，但是为了保证dump文件有效，在备份期间其他会话不要执行`TRUNCATE/DROP/CREATE/ALTER/RENAME TBALE`语句；
  * 与`--lock-tables`参数互斥，因为后者会导致正在等待的事务隐式提交；
  * 不建议与`--set-gtid-purged`参数同时使用，因为可能会导致dump文件的不一致性；
  * 在备份大表时，建议与`--quick`参数同时使用。

- `--master-data`：用于从主库导出一个可以应用到从库的dump文件，其中会包含`CHANGE MASTER TO`语句。**默认值为1**。
  * 值等于1时，`CHANGE MASTER TO`语句会在导入从库时生效；
  * 值等于2时，`CHANGE MASTER TO`语句会被注释，因此不会生效。

使用`--master-data`参数时会自动关闭`--lock-tables`参数，并打开`--lock-all-tables`参数。为了避免在备份过程中一直加全局读锁，可以同时使用`--single-transaction`参数，此时只会**在备份开始前短时间持有一个全局读锁**。

- `--set-gtid-purged`：控制了是否往dump文件中写入GTID信息。默认值为**AUTO**。
  * 值为OFF时，不会在导出的dump文件中写入`SET @@SESSION.SQL_LOG_BIN=0`和`SET @@GLOBAL.gtid_purged`语句；
  * 值为ON时，会在导出的dump文件中写入`SET @@SESSION.SQL_LOG_BIN=0`和`SET @@GLOBAL.gtid_purged`语句。如果未开启GTID会报错；
  * 值为AUTO时，如果开启了GTID，就会自动在导出的dump文件中写入`SET @@SESSION.SQL_LOG_BIN=0`和`SET @@GLOBAL.gtid_purged`语句。


# :snake: 注意事项

使用mysqldump导出备份时，至少需要具备以下权限：

- 对被导出表的`SELECT`权限；
- 对被导出视图的`SHOW VIEW`权限；
- 对被导出触发器的`TRIGGER`权限；
- 如果没有使用`--single-transaction`参数，还需要有`LOCK TABLES`权限；
- 如果没有使用`--no-tablespaces`参数，还需要有`PROCESS`权限（MySQL 5.7.31版本）；

导入mysqldump备份文件时，必须有执行dump文件中所有SQL语句的权限，比如合适的`CREATE`权限来创建数据库对象、`ALTER`权限来执行备份文件中的`ALTER DATABASE`语句。

实际操作时，可以直接使用MySQL的root用户进行备份和恢复。

# :eagle: 示例

以不同方式导出备份文件：
```bash
[root@mysql-node1 tmp]# mysqldump -uroot -p --all-databases  > /tmp/bak-default.sql
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events.
[root@mysql-node1 tmp]#
[root@mysql-node1 tmp]# mysqldump -uroot -p --all-databases --master-data=1  > /tmp/bak-master.sql
[root@mysql-node1 tmp]#
[root@mysql-node1 tmp]# mysqldump -uroot -p --all-databases --master-data=1  --single-transaction > /tmp/bak-master-single.sql
[root@mysql-node1 tmp]#
[root@mysql-node1 tmp]# mysqldump -uroot -p --all-databases --master-data=1  --set-gtid-purged=OFF > /tmp/bak-master-gtidoff.sql
[root@mysql-node1 tmp]#
[root@mysql-node1 tmp]# mysqldump -uroot -p --all-databases --master-data=1  --single-transaction --set-gtid-purged=OFF > /tmp/bak-master-single-gtidoff.sql
```

比较有无`--master-data`和`--single-transaction`的情况：
```bash
[root@mysql-node1 tmp]# diff bak-default.sql bak-master.sql
20a21,26
> -- Position to start replication or point-in-time recovery from
> --
>
> CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000013', MASTER_LOG_POS=652;
>
> --
981c987
< -- Dump completed on 2022-09-11  6:22:14
---
> -- Dump completed on 2022-09-11  6:23:46
[root@mysql-node1 tmp]#
[root@mysql-node1 tmp]# diff bak-master.sql bak-master-single.sql
987c987
< -- Dump completed on 2022-09-11  6:23:46
---
> -- Dump completed on 2022-09-11  6:24:04
[root@mysql-node1 tmp]#
```
注意到，加上`--master-data=1`后，dump文件中会多出一行`CHANGE MASTER TO`语句，带有`MASTER_LOG_FILE`和`MASTER_LOG_POS`参数。显然该语句并不能用于GTID模式下的从库。


比较`--set-gtid-purged`参数是否开启：
```bash
[root@mysql-node1 tmp]# diff bak-master.sql bak-master-gtidoff.sql
17,18d16
< SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
< SET @@SESSION.SQL_LOG_BIN= 0;
971,976d968
<
< --
< -- GTID state at the end of the backup
< --
<
< SET @@GLOBAL.GTID_PURGED='3566d4fe-208e-11ed-87b0-000c29fc42a9:1-21';
987c979
< -- Dump completed on 2022-09-11  6:23:46
---
> -- Dump completed on 2022-09-11  6:25:08
[root@mysql-node1 tmp]#
```

注意到，加上`--set-gtid-purged=OFF`参数后，导出的dump文件中会缺少以下三行：
```sql
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;
SET @@GLOBAL.GTID_PURGED='3566d4fe-208e-11ed-87b0-000c29fc42a9:1-21';
```

也就是说，没有指定`--set-gtid-purged=OFF`时生成的dump文件，在被导入时不会记录binlog，而且导入该备份文件的数据库中的`Executed_Gtid_Set`也不会增加。因此，如果是在主库导入，这部分数据不会被同步到从库。

反之，如果指定了`--set-gtid-purged=OFF`，**dump文件在被导入时会记录binlog**，而且导入该备份文件的数据库中的`Executed_Gtid_Set`也会增加。因此，如果是在主库导入，这部分数据就会被同步到从库。

# :camera: GTID模式下备份的问题
如果mysqldump导出的备份文件中包含有MySQL系统表的数据，不建议在开启了GTID模式的数据库中导入该备份。因为mysqldump会对使用了**MyISAM**的系统表进行DML操作，而在开启了GTID模式的数据库中这种操作不可行。

```sql
mysql> select table_schema,table_name,engine from information_schema.tables where engine='MyISAM';
+--------------+------------------+--------+
| table_schema | table_name       | engine |
+--------------+------------------+--------+
| mysql        | columns_priv     | MyISAM |
| mysql        | db               | MyISAM |
| mysql        | event            | MyISAM |
| mysql        | func             | MyISAM |
| mysql        | ndb_binlog_index | MyISAM |
| mysql        | proc             | MyISAM |
| mysql        | procs_priv       | MyISAM |
| mysql        | proxies_priv     | MyISAM |
| mysql        | tables_priv      | MyISAM |
| mysql        | user             | MyISAM |
+--------------+------------------+--------+
10 rows in set (0.01 sec)
```


**References**
【1】https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html
【2】https://dev.mysql.com/doc/refman/5.7/en/using-mysqldump.html



