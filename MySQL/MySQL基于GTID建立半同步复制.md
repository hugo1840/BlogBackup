
@[TOC](MySQL基于GTID建立半同步复制)

# 基于GTID创建半同步的官方流程

## Step 1. 同步主备库
如果是新上线的服务器，直接跳到Step 3。如果主备库已经在使用非GTID的方式进行binlog同步了，需要将所有数据库设置为只读（没有提交的事务会回滚，会影响业务），等待备库赶上主库。

```sql
set @@GLOBAL.read_only=ON;
```

未开启GTID模式前生成的binlog，在开启GTID模式后的半同步架构中无法使用。如果需要将binlog用于除半同步复制之外的其他用途，比如备份恢复，那么必须等到没有GTID的binlog都过期。理想情况下，要等到服务器上所有的binlog都被purge掉，且已有的备份都过期。


## Step 2. 停止数据库
停止主备库的数据库服务。
```sql
mysqladmin -uroot -p shutdown 
```

## Step 3. 以GTID模式启动主备库
配置以下系统变量来启用GTID模式：
```
gtid-mode=ON
enforce-gtid-consistency=ON
```

在启动备库时，指定参数`--skip-slave-start`。如果不想备库记录binlog，可以开启 `--skip-log-bin`和`--log-slave-updates=OFF`参数。

## Step 4. 配置备库利用GTID实现binlog自动定位
在主库上创建半同步用户：
```sql
create user 'repl'@'%' identified by 'Repl&p@$$';
grant replication slave on *.* to 'repl'@'%';
```

在每个备库上执行：
```sql
CHANGE MASTER TO
MASTER_HOST = '192.168.136.128',   -- 主库IP
MASTER_PORT = 3306,
MASTER_USER = 'repl',              -- 半同步用户，需要提前创建
MASTER_PASSWORD = 'Repl&p@$$',
MASTER_AUTO_POSITION = 1;
```
可以发现，这里没有使用`MASTER_LOG_FILE`和`MASTER_LOG_POS`来手动定位binlog。

## Step 5. 备份
开启GTID之前的备份都将无法使用，因此最好对数据库做一次备份。

## Step 6. 启动备库复制线程并关闭只读模式
启动备库复制线程。
```sql
start slave;
set @@GLOBAL.read_only = OFF;
```

# 实验：基于两套单机mysql创建半同步
现有一套已投入使用的单机mysql-node1。假设我们新搭建了另一套单机mysql-node2，并在这两台单机之间配置半同步（以mysql-node1为主库）。两套MySQL的版本都是5.7。

查看GTID模式是否已经开启：
```sql
mysql> show variables like 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | OFF   |
+---------------+-------+
1 row in set (0.00 sec)

mysql> show variables like 'enforce_gtid%';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| enforce_gtid_consistency | OFF   |
+--------------------------+-------+
1 row in set (0.00 sec)
```

两套MySQL都未开启GTID模式。

将两套库都设置为只读：
```sql
mysql> set @@GLOBAL.read_only=ON;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'read_only';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| read_only     | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

停止目标主备库数据库服务：
```bash
[root@mysql-node1 ~]# mysqladmin -uroot -p shutdown
Enter password:
[root@mysql-node1 ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: inactive (dead) since Sat 2022-08-27 10:01:06 EDT; 8s ago
```

修改目标主备库的配置文件`my.cnf`，在`[mysqld]`的部分添加以下两行：
```
gtid-mode=ON
enforce-gtid-consistency=ON
```

启动目标主库：
```bash
systemctl start mysqld
```

启动目标备库。这里需要修改mysql守护进程文件，在启动命令中加上`--skip-slave-start`参数，使得从库启动时不要自动带起半同步复制线程。
```bash
[root@mysql-node2 ~]# grep mysqld /usr/lib/systemd/system/mysqld.service
Documentation=man:mysqld(8)
# Disable service start and stop timeout logic of systemd for mysqld service.
ExecStart=/mysql/mysql-5.7/bin/mysqld --defaults-file=/etc/my.cnf --skip-slave-start --daemonize
[root@mysql-node2 ~]# 
[root@mysql-node2 ~]# systemctl daemon-reload
[root@mysql-node2 ~]# systemctl restart mysqld
```

查看GTID模式是否已经开启：
```sql
mysql> show variables like '%gtid%';
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.00 sec)
```

在目标主库上创建半同步用户：

```sql
mysql> create user 'repuser'@'%' identified by 'Repl&p@33';
Query OK, 0 rows affected (0.01 sec)

mysql> grant replication slave on *.* to 'repuser'@'%';
Query OK, 0 rows affected (0.00 sec)
```

确认目标备库复制线程未启动：
```sql
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000010
         Position: 154
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

mysql> show slave status\G
Empty set (0.00 sec)
```

在目标备库上配置同步信息：
```sql
mysql> change master to
    -> MASTER_HOST = '192.168.124.142',   --主库IP
    -> MASTER_PORT = 3306,
    -> MASTER_USER = 'repuser',           --主库上的半同步用户
    -> MASTER_PASSWORD = 'Repl&p@33',
    -> MASTER_AUTO_POSITION = 1;
Query OK, 0 rows affected, 2 warnings (0.01 sec)
```

开启GTID之前的备份都将无法使用，因此最好对主库做一次全库备份。

启动备库半同步复制线程：
```sql
mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.124.142
                  Master_User: repuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000009
          Read_Master_Log_Pos: 601
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 814
        Relay_Master_Log_File: mysql-bin.000009
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 601
              Relay_Log_Space: 1015
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 142
                  Master_UUID: 3566d4fe-208e-11ed-87b0-000c29fc42a9
             Master_Info_File: /mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-2
            Executed_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-2
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

```

查看主库状态：
```sql
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000009
         Position: 601
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-2
1 row in set (0.00 sec)
```

# 一个陷阱：从库SQL线程故障
在主库插入新数据，测试是否可以成功复制：
```sql
mysql> use apptest;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from app01;
+----+---------------+---------+
| ID | name          | country |
+----+---------------+---------+
|  1 | Stephen Curry | USA     |
+----+---------------+---------+
1 row in set (0.00 sec)

mysql> insert into app01 (ID,name,country) values
    -> (2,'Yao Ming','China'),
    -> (3,'Cristiano Ronaldo','Portuguese');
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> select * from app01;
+----+-------------------+------------+
| ID | name              | country    |
+----+-------------------+------------+
|  1 | Stephen Curry     | USA        |
|  2 | Yao Ming          | China      |
|  3 | Cristiano Ronaldo | Portuguese |
+----+-------------------+------------+
3 rows in set (0.00 sec)
```

检查备库是否同步过来数据：
```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
```
发现从库没有同步过来apptest库下app01表中的数据。

检查从库半同步线程：
```sql
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.124.142
                  Master_User: repuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000009
          Read_Master_Log_Pos: 918
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 814
        Relay_Master_Log_File: mysql-bin.000009
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 1146
                   Last_Error: Error executing row event: 'Table 'apptest.app01' doesn't exist'
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 601
              Relay_Log_Space: 1332
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 1146
               Last_SQL_Error: Error executing row event: 'Table 'apptest.app01' doesn't exist'
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 142
                  Master_UUID: 3566d4fe-208e-11ed-87b0-000c29fc42a9
             Master_Info_File: /mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp: 220827 10:56:19
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-3
            Executed_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-2
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

发现从库报错`Last_Error: Error executing row event: 'Table 'apptest.app01' doesn't exist'`，且SQL线程已经停止。比较`Retrieved_Gtid_Set`和`Executed_Gtid_Set`可以得知，出现问题的SQL对应的GTID为`3566d4fe-208e-11ed-87b0-000c29fc42a9:3`。

在主库上查看导致从库SQL线程错误的binlog文件中的事件：
```sql
mysql> show binlog events in 'mysql-bin.000009';
+------------------+-----+----------------+-----------+-------------+------------------------------------------------------------------------------------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                                             |
+------------------+-----+----------------+-----------+-------------+------------------------------------------------------------------------------------------------------------------+
| mysql-bin.000009 |   4 | Format_desc    |       142 |         123 | Server ver: 5.7.39-log, Binlog ver: 4                                                                            |
| mysql-bin.000009 | 123 | Previous_gtids |       142 |         154 |                                                                                                                  |
| mysql-bin.000009 | 154 | Gtid           |       142 |         219 | SET @@SESSION.GTID_NEXT= '3566d4fe-208e-11ed-87b0-000c29fc42a9:1'                                                |
| mysql-bin.000009 | 219 | Query          |       142 |         402 | CREATE USER 'repuser'@'%' IDENTIFIED WITH 'mysql_native_password' AS '*4CB48C8B00F8BC991FEF5E333E866EBBB20089C4' |
| mysql-bin.000009 | 402 | Gtid           |       142 |         467 | SET @@SESSION.GTID_NEXT= '3566d4fe-208e-11ed-87b0-000c29fc42a9:2'                                                |
| mysql-bin.000009 | 467 | Query          |       142 |         601 | GRANT REPLICATION SLAVE ON *.* TO 'repuser'@'%'                                                                  |
| mysql-bin.000009 | 601 | Gtid           |       142 |         666 | SET @@SESSION.GTID_NEXT= '3566d4fe-208e-11ed-87b0-000c29fc42a9:3'                                                |
| mysql-bin.000009 | 666 | Query          |       142 |         741 | BEGIN                                                                                                            |
| mysql-bin.000009 | 741 | Table_map      |       142 |         798 | table_id: 109 (apptest.app01)                                                                                    |
| mysql-bin.000009 | 798 | Write_rows     |       142 |         887 | table_id: 109 flags: STMT_END_F                                                                                  |
| mysql-bin.000009 | 887 | Xid            |       142 |         918 | COMMIT /* xid=37 */                                                                                              |
| mysql-bin.000009 | 918 | Gtid           |       142 |         983 | SET @@SESSION.GTID_NEXT= '3566d4fe-208e-11ed-87b0-000c29fc42a9:4'                                                |
| mysql-bin.000009 | 983 | Query          |       142 |        1086 | create database appgame                                                                                          |
+------------------+-----+----------------+-----------+-------------+------------------------------------------------------------------------------------------------------------------+
13 rows in set (0.00 sec)
```

可以发现GTID`3566d4fe-208e-11ed-87b0-000c29fc42a9:3`对应的SQL正是前面的对`apptest.app01`的INSERT语句。

这是由于apptest库是在创建半同步前于主库上创建的，当时生成的binlog是没有开启GTID模式的，因此也不会被同步到从库。而在创建半同步复制后，最新的INSERT语句会被同步到从库回放，但是却找不到对应的apptest库。

因此，MySQL开启GTID模式创建半同步前，**一定要保证主从库的数据是一致的**，要么是两套全新的空库，要么是传统复制模式下从库已经追平主库的数据。

# 一个技巧：修复从库SQL线程
在从库上跳过执行出错的事务：
```sql
mysql> stop slave;
Query OK, 0 rows affected (0.00 sec)

mysql> set global sql_slave_skip_counter=1;
ERROR 1858 (HY000): sql_slave_skip_counter can not be set when the server is running with @@GLOBAL.GTID_MODE = ON. Instead, for each transaction that you want to skip, generate an empty transaction with the same GTID as the transaction
mysql>
mysql> set gtid_next='3566d4fe-208e-11ed-87b0-000c29fc42a9:3';
Query OK, 0 rows affected (0.00 sec)

mysql> begin;commit;
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

mysql> set gtid_next='AUTOMATIC';
Query OK, 0 rows affected (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

由于GTID模式下不支持使用`sql_slave_skip_counter`参数来跳过事务，解决办法是关闭slave线程后，**在从库上提交一个与导致问题的SQL相同GTID的空事务**，然后再启动slave线程。

查看从库半同步状态是否恢复正常：
```sql
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.124.142
                  Master_User: repuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000009
          Read_Master_Log_Pos: 1086
               Relay_Log_File: relay-log.000003
                Relay_Log_Pos: 454
        Relay_Master_Log_File: mysql-bin.000009
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1086
              Relay_Log_Space: 1800
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 142
                  Master_UUID: 3566d4fe-208e-11ed-87b0-000c29fc42a9
             Master_Info_File: /mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-4
            Executed_Gtid_Set: 3566d4fe-208e-11ed-87b0-000c29fc42a9:1-4
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

**References**
【1】https://dev.mysql.com/doc/refman/5.7/en/replication-gtids-howto.html
【2】https://dev.mysql.com/doc/refman/5.7/en/replication-options-gtids.html
【3】https://dev.mysql.com/doc/refman/5.7/en/replication-options-replica.html
【4】https://dev.mysql.com/doc/refman/5.7/en/replication-howto-repuser.html



