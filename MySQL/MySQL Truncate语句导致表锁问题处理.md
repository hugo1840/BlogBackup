---
tags: [mysql]
title: MySQL Truncate语句导致表锁问题处理
created: '2023-02-02T11:40:44.048Z'
modified: '2023-02-06T11:52:29.264Z'
---

MySQL Truncate语句导致表锁问题处理

>:cookie:数据库版本： MySQL 8.0.32


# 创建测试表
生成一个长度为10000的序列：
```bash
[root@mysqldb ~]# seq 1 10000 > /tmp/temp_seq
```

创建应用库和用户：
```sql
mysql> create user 'appuser'@'%' identified by 'XXXXX';
mysql> create database appdb;
mysql> grant all on appdb.* to 'appuser'@'%' with grant option;
```

使用appuser用户创建表：
```sql
mysql> use appdb;
mysql> create table temp_seq(id int, primary key(id));
mysql> load data infile '/tmp/temp_seq' replace into table temp_seq;
ERROR 1045 (28000): Access denied for user 'appuser'@'%' (using password: YES)
```

检查导数相关参数和权限：
```sql
--授予应用用户FILE权限
mysql> grant file on *.* to 'appuser'@'%';

mysql> show variables like 'secure_file_priv';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv | NULL  |
+------------------+-------+
1 row in set (0.00 sec)

mysql> show variables like 'local_infile';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| local_infile  | OFF   |
+---------------+-------+
1 row in set (0.00 sec)
```

配置secure_file_priv参数后，重启数据库：
```bash
[root@mysqldb ~]# systemctl restart mysqld
[root@mysqldb ~]# cat /etc/my.cnf | grep secure_file
secure_file_priv=''
```

导入数据并生成随机的测试表：
```sql
mysql> load data infile '/tmp/temp_seq' replace into table temp_seq;
Query OK, 10000 rows affected (0.04 sec)
Records: 10000  Deleted: 0  Skipped: 0  Warnings: 0

mysql> create table test_table(id int, c1 int, c2 varchar(100), primary key(id));

mysql> insert into test_table 
select id, round(rand()*10000), 
concat('SpongeBobSquarePants',id) from temp_seq;
Query OK, 10000 rows affected (0.09 sec)
Records: 10000  Duplicates: 0  Warnings: 0

mysql> select * from appdb.test_table limit 10;
+----+------+------------------------+
| id | c1   | c2                     |
+----+------+------------------------+
|  1 | 6412 | SpongeBobSquarePants1  |
|  2 | 3736 | SpongeBobSquarePants2  |
|  3 | 9445 | SpongeBobSquarePants3  |
|  4 | 6014 | SpongeBobSquarePants4  |
|  5 | 1736 | SpongeBobSquarePants5  |
|  6 |  639 | SpongeBobSquarePants6  |
|  7 | 7988 | SpongeBobSquarePants7  |
|  8 | 8021 | SpongeBobSquarePants8  |
|  9 | 6140 | SpongeBobSquarePants9  |
| 10 | 6636 | SpongeBobSquarePants10 |
+----+------+------------------------+
10 rows in set (0.01 sec)
```

# 问题场景模拟
## 模拟truncate导致的元数据锁
使用appuser用户打开Session 1：
```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from test_table where c1>365 and c1<500;
...
...
111 rows in set (0.00 sec)
```

使用appuser用户打开Session 2，对会话1查询的表执行truncate操作：
```sql
mysql> truncate test_table;
--被阻塞
```

## 表锁阻塞源排查
使用root用户打开Session 3来排查锁。

查看异常状态的连接进程信息：
```sql
mysql> select * from information_schema.processlist where state != '';
+----+-----------------+-----------+-------+---------+------+---------------------------------+----------------------------------------------------------------+
| ID | USER            | HOST      | DB    | COMMAND | TIME | STATE                           | INFO                                                           |
+----+-----------------+-----------+-------+---------+------+---------------------------------+----------------------------------------------------------------+
| 11 | appuser         | localhost | appdb | Query   |  130 | Waiting for table metadata lock | truncate test_table                                            |
|  5 | event_scheduler | localhost | NULL  | Daemon  | 1343 | Waiting on empty queue          | NULL                                                           |
| 13 | root            | localhost | NULL  | Query   |    0 | executing                       | select * from information_schema.processlist where state != ''; |
+----+-----------------+-----------+-------+---------+------+---------------------------------+----------------------------------------------------------------+
3 rows in set (0.00 sec)
```

查看有表锁的数据库对象和线程ID：
```sql
mysql> select * from performance_schema.metadata_locks;
+-------------------+--------------------+----------------+-----------------------+-----------------------+---------------------+---------------+-------------+-------------------+-----------------+----------------+
| OBJECT_TYPE       | OBJECT_SCHEMA      | OBJECT_NAME    | COLUMN_NAME           | OBJECT_INSTANCE_BEGIN | LOCK_TYPE           | LOCK_DURATION | LOCK_STATUS | SOURCE            | OWNER_THREAD_ID | OWNER_EVENT_ID |
+-------------------+--------------------+----------------+-----------------------+-----------------------+---------------------+---------------+-------------+-------------------+-----------------+----------------+
| TABLE             | appdb              | test_table     | NULL                  |       140688096288624 | SHARED_READ         | TRANSACTION   | GRANTED     | sql_parse.cc:6093 |              57 |             15 |
| SCHEMA            | appdb              | NULL           | NULL                  |       140688242394496 | INTENTION_EXCLUSIVE | EXPLICIT      | GRANTED     | dd_schema.cc:107  |              54 |             19 |
| GLOBAL            | NULL               | NULL           | NULL                  |       140688242865040 | INTENTION_EXCLUSIVE | STATEMENT     | GRANTED     | sql_base.cc:5474  |              54 |             19 |
| BACKUP LOCK       | NULL               | NULL           | NULL                  |       140688241613744 | INTENTION_EXCLUSIVE | TRANSACTION   | GRANTED     | sql_base.cc:5481  |              54 |             19 |
| SCHEMA            | appdb              | NULL           | NULL                  |       140688242241760 | INTENTION_EXCLUSIVE | TRANSACTION   | GRANTED     | sql_base.cc:5461  |              54 |             19 |
| TABLE             | appdb              | test_table     | NULL                  |       140688242692080 | EXCLUSIVE           | TRANSACTION   | PENDING     | sql_parse.cc:6093 |              58 |             19 |
| TABLE             | performance_schema | metadata_locks | NULL                  |       140688163388672 | SHARED_READ         | TRANSACTION   | GRANTED     | sql_parse.cc:6093 |              56 |              4 |
| SCHEMA            | performance_schema | NULL           | NULL                  |       140688163404432 | INTENTION_EXCLUSIVE | TRANSACTION   | GRANTED     | dd_schema.cc:107  |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | column_name           |       140688163387824 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | lock_duration         |       140688163400944 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | lock_status           |       140688163596864 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | lock_type             |       140688163400592 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | object_instance_begin |       140688163385264 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | object_name           |       140688163434592 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | object_schema         |       140688163435376 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | object_type           |       140688163436160 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | owner_event_id        |       140688163397616 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | owner_thread_id       |       140688163398400 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
| COLUMN STATISTICS | performance_schema | metadata_locks | source                |       140688163399184 | SHARED_READ         | STATEMENT     | GRANTED     | sql_base.cc:596   |              56 |              4 |
+-------------------+--------------------+----------------+-----------------------+-----------------------+---------------------+---------------+-------------+-------------------+-----------------+----------------+
19 rows in set (0.00 sec)
```

对`metadata_locks`做表连接，查看表锁相关的对象以及thread_id：
```sql
--pending表示发出了获取元数据锁的请求，但是还没有拿到锁（被阻塞的对象）
--granted表示发出了获取元数据锁的请求，并且已经拿到锁（阻塞源对象）
mysql> select a.object_schema, a.object_name, "Metadata Lock" as locked_type,
a.owner_thread_id as locked_thread_id, b.owner_thread_id blocking_thread_id
from performance_schema.metadata_locks a
join performance_schema.metadata_locks b
on a.object_schema = b.object_schema  
and a.object_name = b.object_name      
and a.lock_status = 'PENDING'          
and b.lock_status = 'GRANTED'         
and a.owner_thread_id <> b.owner_thread_id    
and a.lock_type = 'EXCLUSIVE';

+---------------+-------------+---------------+------------------+--------------------+
| object_schema | object_name | locked_type   | locked_thread_id | blocking_thread_id |
+---------------+-------------+---------------+------------------+--------------------+
| appdb         | test_table  | Metadata Lock |               58 |                 57 |
+---------------+-------------+---------------+------------------+--------------------+
1 row in set (0.00 sec)
```

查看被阻塞对象的进程信息：
```sql
mysql> select
processlist_id, processlist_time, processlist_info, processlist_state
from performance_schema.threads 
where thread_id = 58;

+----------------+------------------+---------------------+---------------------------------+
| processlist_id | processlist_time | processlist_info    | processlist_state               |
+----------------+------------------+---------------------+---------------------------------+
|             15 |             3572 | truncate test_table | Waiting for table metadata lock |
+----------------+------------------+---------------------+---------------------------------+
1 row in set (0.00 sec)
```

查看阻塞源的进程信息：
```sql
mysql> select
processlist_id, processlist_time, processlist_info, processlist_state
from performance_schema.threads 
where thread_id = 57;

+----------------+------------------+------------------+-------------------+
| processlist_id | processlist_time | processlist_info | processlist_state |
+----------------+------------------+------------------+-------------------+
|             14 |             3639 | NULL             | NULL              |
+----------------+------------------+------------------+-------------------+
1 row in set (0.00 sec)
```

查看阻塞源的SQL文本：
```sql
mysql> select event_id, event_name, sql_text
from performance_schema.events_statements_history
where thread_id = 57 order by event_id;

+----------+-----------------------------+--------------------------------------------------------------------------------------------------------+
| event_id | event_name                  | sql_text                                                                                               |
+----------+-----------------------------+--------------------------------------------------------------------------------------------------------+
|        6 | statement/sql/show_tables   | show tables                                                                                            |
|        8 | statement/com/Field List    | NULL                                                                                                   |
|        9 | statement/com/Field List    | NULL                                                                                                   |
|       10 | statement/sql/show_tables   | show tables                                                                                            |
|       12 | statement/sql/select        | select count(*) from test_table                                                                        |
|       14 | statement/sql/insert_select | insert into test_table select id, round(rand()*10000), concat('SpongeBobSquarePants',id) from temp_seq |
|       16 | statement/sql/select        | select count(*) from test_table                                                                        |
|       18 | statement/sql/select        | select * from test_table limit 20                                                                      |
|       20 | statement/sql/begin         | begin                                                                                                  |
|       22 | statement/sql/select        | select * from test_table where c1>365 and c1<500                                                       |
+----------+-----------------------------+--------------------------------------------------------------------------------------------------------+
10 rows in set (0.00 sec)
```
可以看到，最后两行开启了一个事务，执行了一个SELECT语句后一直未提交。

以上的过程可以浓缩成下面的语句：
```sql
--substring_index(string,sep,num)函数用于截取string中的字符串，以sep为分隔符，num表示位置（正整数为从左至右第num个，负整数为从右至左第num个）
--group_concat(column1 order by column2 separator ';')函数用于将同一个分组中的元素连接后合并输出，column1为要连接的列，按照column2排序，连接符为分号
--case when <condition_A> then <statement_B> else <statement_C> end 表示条件A成立时执行语句B，不成立时执行语句C

mysql> select locked_schema, locked_table, locked_type,
waiting_processlist_id, waiting_age, waiting_query, waiting_state,
blocking_processlist_id, blocking_age, sql_kill_blocking_connection,
substring_index(sql_text, "transaction_begin;", -1) as blocking_query
from (
    select b.owner_thread_id as granted_thread_id,
    a.object_schema as locked_schema,
    a.object_name as locked_table,
    "Metadata Lock" as locked_type,
    c.processlist_id as waiting_processlist_id,
    c.processlist_time as waiting_age,
    c.processlist_info as waiting_query,
    c.processlist_state as waiting_state,
    d.processlist_id as blocking_processlist_id,
    d.processlist_time as blocking_age,
    d.processlist_info as blocking_query,
    concat('KILL ', d.processlist_id) as sql_kill_blocking_connection
    from 
    performance_schema.metadata_locks a
    join 
    performance_schema.metadata_locks b
    on a.object_schema = b.object_schema   
    and a.object_name = b.object_name     
    and a.lock_status = 'PENDING'          
    and b.lock_status = 'GRANTED'          
    and a.owner_thread_id <> b.owner_thread_id    
    and a.lock_type = 'EXCLUSIVE'          
    join
    performance_schema.threads c 
    on a.owner_thread_id = c.thread_id     
    join
    performance_schema.threads d           
    on b.owner_thread_id = d.thread_id
) t1,
(
    select thread_id, 
    group_concat(case when event_name = 'statement/sql/begin' then "transaction_begin" 
    else sql_text end order by event_id separator ";") as sql_text
    from 
    performance_schema.events_statements_history
    group by thread_id
) t2
where t1.granted_thread_id = t2.thread_id\G

*************************** 1. row ***************************
               locked_schema: appdb
                locked_table: test_table
                 locked_type: Metadata Lock
      waiting_processlist_id: 15
                 waiting_age: 3755
               waiting_query: truncate test_table
               waiting_state: Waiting for table metadata lock
     blocking_processlist_id: 14
                blocking_age: 3768
sql_kill_blocking_connection: KILL 14
              blocking_query: select * from test_table where c1>365 and c1<500
1 row in set, 1 warning (0.00 sec)
```

## 杀会话释放元数据锁
杀死阻塞源的进程：
```sql
mysql> kill 14;

mysql> select * from performance_schema.metadata_locks;
+-------------+--------------------+----------------+-------------+-----------------------+-------------+---------------+-------------+-------------------+-----------------+----------------+
| OBJECT_TYPE | OBJECT_SCHEMA      | OBJECT_NAME    | COLUMN_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE   | LOCK_DURATION | LOCK_STATUS | SOURCE            | OWNER_THREAD_ID | OWNER_EVENT_ID |
+-------------+--------------------+----------------+-------------+-----------------------+-------------+---------------+-------------+-------------------+-----------------+----------------+
| TABLE       | performance_schema | metadata_locks | NULL        |       140687962198208 | SHARED_READ | TRANSACTION   | GRANTED     | sql_parse.cc:6093 |              59 |             48 |
+-------------+--------------------+----------------+-------------+-----------------------+-------------+---------------+-------------+-------------------+-----------------+----------------+
1 row in set (0.00 sec)

mysql> select * from information_schema.processlist where state != '';
+----+-----------------+-----------+------+---------+-------+------------------------+----------------------------------------------------------------+
| ID | USER            | HOST      | DB   | COMMAND | TIME  | STATE                  | INFO                                                           |
+----+-----------------+-----------+------+---------+-------+------------------------+----------------------------------------------------------------+
| 16 | root            | localhost | NULL | Query   |     0 | executing              | select * from information_schema.processlist where state != '' |
|  5 | event_scheduler | localhost | NULL | Daemon  | 16938 | Waiting on empty queue | NULL                                                           |
+----+-----------------+-----------+------+---------+-------+------------------------+----------------------------------------------------------------+
2 rows in set (0.00 sec)
```

可以看到Session 2 truncate语句随之执行完成。
```sql
mysql> truncate test_table;
Query OK, 0 rows affected (29 min 8.17 sec)
```

**References** 
[1] https://dev.mysql.com/doc/refman/8.0/en/performance-schema-metadata-locks-table.html 
[2] https://dev.mysql.com/doc/refman/8.0/en/performance-schema-threads-table.html 
[3] https://dev.mysql.com/doc/refman/8.0/en/performance-schema-events-statements-history-table.html 
[4] https://www.cnblogs.com/linus-tan/p/13214142.html 
[5] https://blog.csdn.net/Leon_Jinhai_Sun/article/details/125831693



