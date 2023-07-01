---
tags: [mysql]
title: MySQL 8性能监控常用SQL汇总
created: '2023-07-01T05:35:47.886Z'
modified: '2023-07-01T11:31:30.176Z'
---

MySQL 8性能监控常用SQL汇总

>:dolphin:数据库版本：MySQL 8.0.30

本文中的系统表和视图，如未说明，默认在**performance_schema**数据库中。

# 监控SQL语句执行性能

主要用到以下三张表：
- events_statements_current
- events_statements_history
- events_statements_history_long

统计等待时间最长的SQL：
```sql
select t1.thread_id,user,event_name,
sys.format_time(timer_wait) wait_time,
sys.format_time(lock_time) lock_time,
sql_text,current_schema schema_name,message_text,
rows_affected,rows_sent,rows_examined
from performance_schema.events_statements_history t1
join performance_schema.threads t2 
on t1.thread_id = t2.thread_id
join performance_schema.processlist t3 
on t2.processlist_id = t3.id
where current_schema != 'performance_schema'
order by timer_wait desc limit 10\G
```

统计没有使用索引的SQL：
```sql
select t1.thread_id, user, substr(sql_text,1,50) as sql_text,
rows_sent, rows_examined, created_tmp_tables, 
no_index_used, no_good_index_used
from performance_schema.events_statements_history t1
join performance_schema.threads t2 
on t1.thread_id = t2.thread_id
join performance_schema.processlist t3 
on t2.processlist_id = t3.id
where no_index_used=1 or no_good_index_used=1\G
```

统计SQL执行过程中耗时长的阶段：
```sql
select t1.event_name,sql_text,
format_pico_time(t1.timer_wait) wait_time
from performance_schema.events_stages_history_long t1
join performance_schema.events_statements_history_long t2
on (t1.nesting_event_id = t2.event_id)
where t1.timer_wait > 1*100000000000\G
```

需要提前开启`events_stages_xxx`性能事件收集：
```sql
SQL> call sys.ps_setup_enable_consumer('events_stages');
```

# 监控锁
主要用到以下四张视图：
- data_locks
- data_lock_waits
- metadata_locks
- table_handles

## 数据锁（行锁&表锁）
查询当前存在的数据锁：
```sql
SQL> select * from performance_schema.data_locks\G
```
其中**LOCK_TYPE=TABLE**为表锁，**LOCK_TYPE=RECORD**为行锁。LOCK_MODE包含字母X为独占锁。

查询数据锁阻塞的线程信息：
```sql
SQL> select * from performance_schema.data_lock_waits\G

*************************** 1. row ***************************
                          ENGINE: INNODB
       REQUESTING_ENGINE_LOCK_ID: 140642486936360:30:4:6:140642395157712
REQUESTING_ENGINE_TRANSACTION_ID: 25680230
            REQUESTING_THREAD_ID: 411
             REQUESTING_EVENT_ID: 856
REQUESTING_OBJECT_INSTANCE_BEGIN: 140642395157712
         BLOCKING_ENGINE_LOCK_ID: 140642486937168:30:4:6:140642395163040
  BLOCKING_ENGINE_TRANSACTION_ID: 25680229
              BLOCKING_THREAD_ID: 414
               BLOCKING_EVENT_ID: 454
  BLOCKING_OBJECT_INSTANCE_BEGIN: 140642395163040
1 row in set (0.00 sec)
```
其中BLOCKING_THREAD_ID为阻塞源线程，REQUESTING_THREAD_ID为被阻塞的线程。

查询`sys.innodb_lock_waits`表可以看到更详细的信息，包括等待时间、被锁住的数据库对象名称、被阻塞的SQL语句、阻塞源SQL语句及其trx_id和pid、杀会话语句。
```sql
SQL> select * from sys.innodb_lock_waits\G
*************************** 1. row ***************************
                wait_started: 2023-06-27 16:29:46
                    wait_age: 00:00:06
               wait_age_secs: 6
                locked_table: `testdb`.`t1`
         locked_table_schema: testdb
           locked_table_name: t1
      locked_table_partition: NULL
   locked_table_subpartition: NULL
                locked_index: PRIMARY
                 locked_type: RECORD
              waiting_trx_id: 25680230
         waiting_trx_started: 2023-06-27 16:26:39
             waiting_trx_age: 00:03:13
     waiting_trx_rows_locked: 2
   waiting_trx_rows_modified: 0
                 waiting_pid: 375
               waiting_query: update t1 set price=193 where title='Black Souls III'
             waiting_lock_id: 140642486936360:30:4:6:140642395157712
           waiting_lock_mode: X,REC_NOT_GAP
             blocking_trx_id: 25680229
                blocking_pid: 378
              blocking_query: NULL
            blocking_lock_id: 140642486937168:30:4:6:140642395163040
          blocking_lock_mode: X,REC_NOT_GAP
        blocking_trx_started: 2023-06-27 16:27:50
            blocking_trx_age: 00:02:02
    blocking_trx_rows_locked: 1
  blocking_trx_rows_modified: 1
     sql_kill_blocking_query: KILL QUERY 378
sql_kill_blocking_connection: KILL 378
1 row in set (0.01 sec)
```

## 元数据锁
下面是一个DDL语句（例如truncate语句）导致锁表的例子。
```sql
SQL> select object_schema,object_name,lock_type,
lock_status,owner_thread_id
from performance_schema.metadata_locks
where object_name='t1';

+---------------+-------------+-------------+-------------+-----------------+
| object_schema | object_name | lock_type   | lock_status | owner_thread_id |
+---------------+-------------+-------------+-------------+-----------------+
| testdb        | t1          | SHARED_READ | GRANTED     |             367 |
| testdb        | t1          | EXCLUSIVE   | PENDING     |             411 |
+---------------+-------------+-------------+-------------+-----------------+
2 rows in set (0.01 sec)

SQL> select * from sys.schema_table_lock_waits\G

*************************** 1. row ***************************
               object_schema: testdb
                 object_name: t1
           waiting_thread_id: 411
                 waiting_pid: 375
             waiting_account: appuser@127.0.0.1
           waiting_lock_type: EXCLUSIVE
       waiting_lock_duration: TRANSACTION
               waiting_query: truncate table testdb.t1
          waiting_query_secs: 18
 waiting_query_rows_affected: 0
 waiting_query_rows_examined: 0
          blocking_thread_id: 367
                blocking_pid: 331
            blocking_account: appuser@127.0.0.1
          blocking_lock_type: SHARED_READ
      blocking_lock_duration: TRANSACTION
     sql_kill_blocking_query: KILL QUERY 331
sql_kill_blocking_connection: KILL 331
1 row in set (0.00 sec)

SQL> select t1.thread_id,t1.processlist_id,t2.user,t2.command,t2.info 
from performance_schema.threads t1 
join performance_schema.processlist t2 
on t1.processlist_id=t2.id;

+-----------+----------------+--------+---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| thread_id | processlist_id | user   | command | info                                                                                                                                                                   |
+-----------+----------------+--------+---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|       515 |            479 | appuser | Query   | select t1.thread_id,t1.processlist_id,t2.user,t2.command,t2.info from performance_schema.threads t1  join performance_schema.processlist t2 on t1.processlist_id=t2.id |
|       518 |            482 | dbops  | Sleep   | NULL                                                                                                                                                                   |
|       367 |            331 | appuser | Sleep   | NULL                                                                                                                                                                   |
|       411 |            375 | appuser | Query   | truncate table testdb.t1                                                                                                                                               |
+-----------+----------------+--------+---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
4 rows in set (0.00 sec)
```

查看阻塞源的SQL文本：
```sql
SQL> select event_id,current_schema,sql_text  
from performance_schema.events_statements_history a 
where thread_id=367  
and nesting_event_type='transaction' 
and nesting_event_id in ( select event_id 
from performance_schema.events_transactions_current b where a.thread_id = b.thread_id);

+----------+----------------+------------------+
| event_id | current_schema | sql_text         |
+----------+----------------+------------------+
|    62213 | testdb         | select * from t1 |
+----------+----------------+------------------+
1 row in set (0.00 sec)
```

查看元数据锁的超时时间（秒），超过该设定时间等待元数据锁的事务会自动回滚。
```sql
SQL> show variables like 'lock_wait_timeout';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| lock_wait_timeout | 60    |
+-------------------+-------+
1 row in set (0.00 sec)
 ```

## table_handles
视图table_handles中记录了持有和请求表锁的信息：
```sql
SQL> lock table testdb.t1 read;
SQL> select object_type,object_schema,object_name,owner_thread_id 
from performance_schema.table_handles where owner_thread_id is not null;
+-------------+---------------+-------------+-----------------+
| object_type | object_schema | object_name | owner_thread_id |
+-------------+---------------+-------------+-----------------+
| TABLE       | testdb        | t1          |             367 |
+-------------+---------------+-------------+-----------------+
1 row in set (0.00 sec)

SQL> select object_schema,object_name,lock_type,
lock_status,owner_thread_id
from performance_schema.metadata_locks
where object_name='t1';

+---------------+-------------+------------------+-------------+-----------------+
| object_schema | object_name | lock_type        | lock_status | owner_thread_id |
+---------------+-------------+------------------+-------------+-----------------+
| testdb        | t1          | SHARED_READ_ONLY | GRANTED     |             367 |
+---------------+-------------+------------------+-------------+-----------------+
1 row in set (0.00 sec)

SQL> unlock tables;
```

# 查询当前等待事件
主要用到以下三张表：
- events_waits_current
- events_waits_history
- events_waits_history_long

需要更新`performance_schema.setup_instruments`来启用wait性能事件收集。

统计当前等待事件中等待时间最长的事件：
```sql
select thread_id,event_name,
format_pico_time(timer_wait) wait_time,operation
from performance_schema.events_waits_current
where event_name != 'idle'
and event_name != 'wait/synch/cond/mysqlx/scheduler_dynamic_worker_pending'
order by timer_wait desc limit 10;

+-----------+----------------------------------------+-----------+-----------+
| thread_id | event_name                             | wait_time | operation |
+-----------+----------------------------------------+-----------+-----------+
|        16 | wait/io/file/innodb/innodb_log_file    | 1.28 ms   | sync      |
|        10 | wait/io/file/innodb/innodb_data_file   | 837.44 us | sync      |
|         9 | wait/io/file/innodb/innodb_data_file   | 605.82 us | sync      |
|        11 | wait/io/file/innodb/innodb_data_file   | 438.21 us | sync      |
|        19 | wait/io/file/innodb/innodb_log_file    | 51.45 us  | close     |
|        18 | wait/io/file/innodb/innodb_log_file    | 32.01 us  | write     |
|        14 | wait/io/file/innodb/innodb_log_file    | 10.86 us  | close     |
|        13 | wait/io/file/innodb/innodb_data_file   | 9.94 us   | write     |
|        12 | wait/io/file/innodb/innodb_data_file   | 5.20 us   | sync      |
|         1 | wait/synch/mutex/sql/LOCK_thread_cache | 2.59 us   | lock      |
+-----------+----------------------------------------+-----------+-----------+
10 rows in set (0.00 sec)
```

# 查询错误语句
主要用到以下五张视图：
- events_errors_summary_by_account_by_error
- events_errors_summary_by_host_by_error
- events_errors_summary_by_thread_by_error
- events_errors_summary_by_user_by_error
- events_errors_summary_global_by_error

下面给出了一个示例：
```sql
SQL> select * from t3;
ERROR 1146 (42S02): Table 'testdb.t3' does not exist
 
SQL> select user,host,error_name,sql_state,last_seen
  from performance_schema.events_errors_summary_by_account_by_error
  where error_number=1146 order by last_seen desc limit 5;
+--------+-----------+------------------+-----------+---------------------+
| user   | host      | error_name       | sql_state | last_seen           |
+--------+-----------+------------------+-----------+---------------------+
| appuser | 127.0.0.1 | ER_NO_SUCH_TABLE | 42S02     | 2023-06-28 11:14:32 |
| NULL   | NULL      | ER_NO_SUCH_TABLE | 42S02     | NULL                |
| dbops  | 127.0.0.1 | ER_NO_SUCH_TABLE | 42S02     | NULL                |
+--------+-----------+------------------+-----------+---------------------+
3 rows in set (0.00 sec)

SQL> select thread_id,sql_text,message_text 
from performance_schema.events_statements_history where mysql_errno=1146\G
*************************** 1. row ***************************
   thread_id: 367
    sql_text: select * from t3
message_text: Table 'testdb.t3' doesn't exist
1 row in set (0.00 sec)
```

不知道错误号时，查找出错的语句：
```sql
SQL> select * from performance_schema.events_statements_history_long where errors>0;
```

按账号分组查询死锁信息：
```sql
SQL> select user,host,sql_state,last_seen 
from performance_schema.events_errors_summary_by_account_by_error
where error_name='er_lock_deadlock';
```

# 监控表I/O
对表的读写，可能是从缓存中，也可以是从磁盘中。
- table_io_waits_summary_by_table：按表进行分组记录IO信息；
- table_io_waits_summary_by_index_usage：按索引进行分组记录表IO信息。

查询一张表的IO：
```sql
SQL> select * from performance_schema.table_io_waits_summary_by_table 
where object_schema='testdb' and object_name='t1'\G
*************************** 1. row ***************************
     OBJECT_TYPE: TABLE
   OBJECT_SCHEMA: testdb
     OBJECT_NAME: t1
      COUNT_STAR: 189
  SUM_TIMER_WAIT: 59664112377100
  MIN_TIMER_WAIT: 1142812
  AVG_TIMER_WAIT: 315683134162
  MAX_TIMER_WAIT: 30030700745166
      COUNT_READ: 174
  SUM_TIMER_READ: 59662368871512
  MIN_TIMER_READ: 1142812
  AVG_TIMER_READ: 342887177336
  MAX_TIMER_READ: 30030700745166
     COUNT_WRITE: 15
 SUM_TIMER_WRITE: 1743505588
 MIN_TIMER_WRITE: 14609936
 AVG_TIMER_WRITE: 116233678
 MAX_TIMER_WRITE: 300117312
     COUNT_FETCH: 174
 SUM_TIMER_FETCH: 59662368871512
 MIN_TIMER_FETCH: 1142812
 AVG_TIMER_FETCH: 342887177336
 MAX_TIMER_FETCH: 30030700745166
    COUNT_INSERT: 7
SUM_TIMER_INSERT: 913110968
MIN_TIMER_INSERT: 61476932
AVG_TIMER_INSERT: 130444424
MAX_TIMER_INSERT: 300117312
    COUNT_UPDATE: 4
SUM_TIMER_UPDATE: 481373816
MIN_TIMER_UPDATE: 58841024
AVG_TIMER_UPDATE: 120343454
MAX_TIMER_UPDATE: 198425436
    COUNT_DELETE: 4
SUM_TIMER_DELETE: 349020804
MIN_TIMER_DELETE: 14609936
AVG_TIMER_DELETE: 87254992
MAX_TIMER_DELETE: 157647864
1 row in set (0.00 sec)
```

以索引为单位查询一张表的IO：
```sql
SQL> select concat(object_schema,'.',object_name) table_name, index_name,
count_read,count_write,count_fetch,count_update,count_insert,count_delete 
from performance_schema.table_io_waits_summary_by_index_usage 
where object_schema='testdb' and object_name='t1';

+------------+------------+------------+-------------+-------------+--------------+--------------+--------------+
| table_name | index_name | count_read | count_write | count_fetch | count_update | count_insert | count_delete |
+------------+------------+------------+-------------+-------------+--------------+--------------+--------------+
| testdb.t1  | PRIMARY    |          1 |           8 |           1 |            4 |            0 |            4 |
| testdb.t1  | NULL       |        173 |           7 |         173 |            0 |            7 |            0 |
+------------+------------+------------+-------------+-------------+--------------+--------------+--------------+
2 rows in set (0.00 sec)
```

# 监控文件I/O
主要用到以下三张视图：
- events_waits_summary_global_by_event_name：记录了按事件名汇总的事件等待信息。事件名以wait/io/file开头的对应了磁盘IO的等待信息，一共有51类。 
- file_summary_by_event_name：记录了51类文件IO的详细等待信息。
- file_summary_by_instance：按硬盘上的实际文件记录的IO信息。

访问磁盘的IO信息，不包括对缓存的访问。

统计系统中IO最繁忙的事件：
```sql
SQL> select * from performance_schema.events_waits_summary_global_by_event_name 
where event_name like 'wait/io/file/%' order by sum_timer_wait desc limit 1\G
*************************** 1. row ***************************
    EVENT_NAME: wait/io/file/innodb/innodb_log_file
    COUNT_STAR: 8989
SUM_TIMER_WAIT: 30552309782764
MIN_TIMER_WAIT: 0
AVG_TIMER_WAIT: 3398854976
MAX_TIMER_WAIT: 675329382794
1 row in set (0.00 sec)
```

查看对具体InnoDB日志文件IO的信息：
```sql
SQL> select file_name, count_star,sum_timer_wait 
from performance_schema.file_summary_by_instance
where event_name='wait/io/file/innodb/innodb_log_file';
```

统计IO最繁忙的10个表空间文件：
```sql
SQL> select file_name from performance_schema.file_summary_by_instance
where event_name='wait/io/file/innodb/innodb_data_file' 
order by sum_timer_wait desc limit 10;

+--------------------------------------+
| file_name                            |
+--------------------------------------+
| /mydata/3306/ibdata/ibdata1          |
| /mydata/3306/data/mysql.ibd          |
| /mydata/3306/ibdata/undotbs2.ibu     |
| /mydata/3306/ibdata/undo_001         |
| /mydata/3306/ibdata/undotbs1.ibu     |
| /mydata/3306/ibdata/undo_002         |
| /mydata/3306/ibdata/ibtmp1           |
| /mydata/3306/data/testdb/t1.ibd      |
| /mydata/3306/data/testdb/t2.ibd      |
| /mydata/3306/data/sys/sys_config.ibd |
+--------------------------------------+
10 rows in set (0.00 sec)
```

# 查询连接情况
按账号和IP查询当前并发连接数、总的连接数：
```sql
SQL> select * from performance_schema.accounts;
```

查询当前会话的连接属性：
```sql
SQL> select * from performance_schema.session_account_connect_attrs;

+----------------+-----------------+------------+------------------+
| PROCESSLIST_ID | ATTR_NAME       | ATTR_VALUE | ORDINAL_POSITION |
+----------------+-----------------+------------+------------------+
|            331 | _pid            | 100281     |                0 |
|            331 | _platform       | x86_64     |                1 |
|            331 | _os             | Linux      |                2 |
|            331 | _client_name    | libmysql   |                3 |
|            331 | os_user         | mysql      |                4 |
|            331 | _client_version | 8.0.30     |                5 |
|            331 | program_name    | mysql      |                6 |
+----------------+-----------------+------------+------------------+
7 rows in set (0.01 sec)
```

# 查询当前事务
查询事务信息主要用到以下8张表：
- events_transactions_current                          
- events_transactions_history                          
- events_transactions_history_long                     
- events_transactions_summary_by_thread_by_event_name  
- events_transactions_summary_by_account_by_event_name 
- events_transactions_summary_by_user_by_event_name    
- events_transactions_summary_by_host_by_event_name    
- events_transactions_summary_global_by_event_name

查询当前的活动事务：
```sql
SQL> select thread_id, event_name, state, access_mode, isolation_level
from performance_schema.events_transactions_current where state='active';
```

# 查询内存使用情况
主要用到以下五张表：
- memory_summary_global_by_event_name     
- memory_summary_by_account_by_event_name 
- memory_summary_by_host_by_event_name    
- memory_summary_by_thread_by_event_name  
- memory_summary_by_user_by_event_name




