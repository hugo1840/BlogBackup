---
tags: [mysql]
title: MySQL运维之系统库sys
created: '2023-07-03T13:29:52.159Z'
modified: '2023-07-05T14:19:36.705Z'
---

MySQL运维之系统库sys

>:dolphin:数据库版本：MySQL 8.0.30

查看sys库和MySQL版本：
```sql
SQL> select * from sys.version;
+-------------+---------------+
| sys_version | mysql_version |
+-------------+---------------+
| 2.1.2       | 8.0.30        |
+-------------+---------------+
1 row in set (0.00 sec)
```

查看sys库中的对象统计：
```sql
SQL> select * from sys.schema_object_overview where db='sys';
+------+---------------+-------+
| db   | object_type   | count |
+------+---------------+-------+
| sys  | BASE TABLE    |     1 |
| sys  | FUNCTION      |    22 |
| sys  | INDEX (BTREE) |     1 |
| sys  | PROCEDURE     |    26 |
| sys  | TRIGGER       |     2 |
| sys  | VIEW          |   100 |
+------+---------------+-------+
6 rows in set (0.00 sec)
```


# 存储过程
sys库中比较有用的存储过程如下：
- `execute_prepared_stmt()`：执行给定的SQL字符串。
```sql
SQL> call sys.execute_prepared_stmt('select count(*) from testdb.t2');
```

- `diagnostics()`：生成一个关于当前MySQL实例整体性能的诊断报告，并输出到工作目录下的指定文件。
```sql
SQL> tee diagnostics.log;
SQL> call sys.diagnostics(null,null,'current');
SQL> notee;
```

- `ps_trace_statement_digest()`：用于跟踪收集SQL执行过程中的性能诊断信息。
- `statement_performance_analyzer()`：用于生成当前MySQL实例中正在运行的SQL性能报告。
- `ps_trace_thread()`：用于跟踪线程执行的SQL语句，并输出性能报告。


# 函数
sys库中包含如下常用的格式转换函数：
- `format_bytes()`：把字节数转化为易于阅读的格式；
- `format_time()`：把皮秒转化为易于阅读的时间格式；
- `format_pico_time()`：把皮秒转化为易于阅读的时间格式。

```sql
root@(none)> select sys.format_bytes(20000), sys.format_time(999999999), format_pico_time(999999999);
+-------------------------+----------------------------+-----------------------------+
| sys.format_bytes(20000) | sys.format_time(999999999) | format_pico_time(999999999) |
+-------------------------+----------------------------+-----------------------------+
| 19.53 KiB               | 1000 us                    | 1000.00 us                  |
+-------------------------+----------------------------+-----------------------------+
```

sys库中还包含如下与performance_schema性能诊断信息收集相关的函数：
- `ps_is_account_enabled()`：对某个账户是否启用性能事件收集；
- `ps_is_consumer_enabled()`：对某个消费者是否启用性能事件收集；
- `ps_is_instrument_default_enabled()`：是否默认启用某个性能事件收集；
- `ps_is_instrument_default_timed()`：是否默认启用某个性能事件收集计时；
- `ps_is_thread_instrumented()`：对某个连接是否启用性能事件收集；
- `ps_thread_account()`：查询给定线程ID对应的用户名和主机名；
- `ps_thread_stack()`：查询给定线程ID对应的SQL语句、事件和阶段信息；
- `ps_thread_trx_info()`：查询给定线程ID对应的事务和SQL语句信息。


查看MySQL版本号也可以用到的几个函数：
```sql
root@(none)> select sys.version_major(),sys.version_minor(),sys.version_patch();
+---------------------+---------------------+---------------------+
| sys.version_major() | sys.version_minor() | sys.version_patch() |
+---------------------+---------------------+---------------------+
|                   8 |                   0 |                  30 |
+---------------------+---------------------+---------------------+
```


# 视图
## 主机汇总
按用户或主机分组的汇总视图：
```sql
root@(none)> select * from sys.host_summary;
+-----------+------------+-------------------+-----------------------+-------------+----------+-----------------+---------------------+-------------------+--------------+----------------+------------------------+
| host      | statements | statement_latency | statement_avg_latency | table_scans | file_ios | file_io_latency | current_connections | total_connections | unique_users | current_memory | total_memory_allocated |
+-----------+------------+-------------------+-----------------------+-------------+----------+-----------------+---------------------+-------------------+--------------+----------------+------------------------+
| 127.0.0.1 |     151884 | 16.53 min         | 6.53 ms               |       69939 |     1533 | 436.76 ms       |                   3 |              1190 |            3 | 8.30 MiB       | 45.67 GiB              |
+-----------+------------+-------------------+-----------------------+-------------+----------+-----------------+---------------------+-------------------+--------------+----------------+------------------------+
1 row in set (0.01 sec)

root@(none)> select * from sys.host_summary_by_statement_latency;
+------------+-------+---------------+-------------+--------------+-------------+-----------+---------------+---------------+------------+
| host       | total | total_latency | max_latency | lock_latency | cpu_latency | rows_sent | rows_examined | rows_affected | full_scans |
+------------+-------+---------------+-------------+--------------+-------------+-----------+---------------+---------------+------------+
| 127.0.0.1  | 50637 | 5.51 min      | 1.00 min    | 59.76 s      |   0 ps      |     46515 |       2746855 |            29 |      23318 |
| background |     0 |   0 ps        |   0 ps      |   0 ps       |   0 ps      |         0 |             0 |             0 |          0 |
+------------+-------+---------------+-------------+--------------+-------------+-----------+---------------+---------------+------------+
2 rows in set (0.00 sec)
```

## 查询事务执行进度
主要用到表`sys.processlist`和`sys.session`。

查看当前连接的会话（不包括自己）：
```sql
root@(none)> select * from sys.session where conn_id != connection_id()\G
*************************** 1. row ***************************
                thd_id: 1235
               conn_id: 1195
                  user: tktest@127.0.0.1
                    db: testdb
               command: Sleep
                 state: NULL
                  time: 2003
     current_statement: NULL
      execution_engine: PRIMARY
     statement_latency: NULL
              progress: NULL
          lock_latency:   0 ps
           cpu_latency:   0 ps
         rows_examined: 0
             rows_sent: 0
         rows_affected: 0
            tmp_tables: 0
       tmp_disk_tables: 0
             full_scan: NO
        last_statement: NULL
last_statement_latency: 54.00 us
        current_memory: 24.69 KiB
             last_wait: idle
     last_wait_latency: Still Waiting
                source: init_net_server_extension.cc:67
           trx_latency: 239.64 us
             trx_state: COMMITTED
        trx_autocommit: YES
                   pid: 101090
          program_name: mysql


root@(none)> select * from sys.processlist where conn_id != connection_id()\G
*************************** 1. row ***************************
                thd_id: 41
               conn_id: 6
                  user: sql/compress_gtid_table
                    db: NULL
               command: Daemon
                 state: Suspending
                  time: 693749
     current_statement: NULL
      execution_engine: PRIMARY
     statement_latency: NULL
              progress: NULL
          lock_latency: NULL
           cpu_latency: NULL
         rows_examined: NULL
             rows_sent: NULL
         rows_affected: NULL
            tmp_tables: NULL
       tmp_disk_tables: NULL
             full_scan: NO
        last_statement: NULL
last_statement_latency: NULL
        current_memory: 15.64 KiB
             last_wait: wait/synch/mutex/sql/LOCK_compress_gtid_table
     last_wait_latency: 224.88 ns
                source: rpl_gtid_persist.cc:762
           trx_latency: 1.45 ms
             trx_state: COMMITTED
        trx_autocommit: YES
                   pid: NULL
          program_name: NULL
*************************** 2. row ***************************
                thd_id: 1235
               conn_id: 1195
                  user: tktest@127.0.0.1
                    db: testdb
               command: Sleep
                 state: NULL
                  time: 53
     current_statement: NULL
      execution_engine: PRIMARY
     statement_latency: NULL
              progress: NULL
          lock_latency: 4.00 us
           cpu_latency:   0 ps
         rows_examined: 5
             rows_sent: 5
         rows_affected: 0
            tmp_tables: 0
       tmp_disk_tables: 0
             full_scan: YES
        last_statement: select * from t1
last_statement_latency: 300.03 us
        current_memory: 37.33 KiB
             last_wait: idle
     last_wait_latency: Still Waiting
                source: init_net_server_extension.cc:67
           trx_latency: 108.91 us
             trx_state: COMMITTED
        trx_autocommit: YES
                   pid: 101090
          program_name: mysql
```

## 查询IO情况
查询IO繁忙的线程及SQL语句：
```sql
root@(none)> select * from sys.io_by_thread_by_latency;
+-------------------------------+--------+---------------+-------------+-------------+-------------+-----------+----------------+
| user                          | total  | total_latency | min_latency | avg_latency | max_latency | thread_id | processlist_id |
+-------------------------------+--------+---------------+-------------+-------------+-------------+-----------+----------------+
| log_files_governor_thread     |   8029 | 28.45 s       | 42.94 us    | 3.54 ms     | 675.33 ms   |        19 |           NULL |
| buf_dump_thread               |  18933 | 17.23 s       | 4.23 us     | 909.95 us   | 132.94 ms   |        34 |           NULL |
| main                          | 109912 | 6.32 s        | 204.82 ns   | 299.94 us   | 281.42 ms   |         1 |           NULL |
| page_flush_coordinator_thread |   1248 | 901.69 ms     | 4.40 us     | 1.15 ms     | 98.91 ms    |        13 |           NULL |
| log_flusher_thread            |    285 | 330.90 ms     | 2.12 us     | 1.16 ms     | 8.25 ms     |        16 |           NULL |
| io_write_thread               |    101 | 153.16 ms     | 581.02 ns   | 1.52 ms     | 67.70 ms    |        10 |           NULL |
| log_checkpointer_thread       |    395 | 108.97 ms     | 1.81 us     | 275.88 us   | 3.18 ms     |        14 |           NULL |
| io_write_thread               |    104 | 66.30 ms      | 770.79 ns   | 637.54 us   | 21.02 ms    |        11 |           NULL |
| io_write_thread               |     77 | 49.82 ms      | 639.54 ns   | 646.99 us   | 20.88 ms    |         9 |           NULL |
| io_write_thread               |     86 | 31.42 ms      | 887.00 ns   | 365.36 us   | 2.02 ms     |        12 |           NULL |
| log_writer_thread             |    362 | 10.90 ms      | 902.04 ns   | 30.11 us    | 1.95 ms     |        18 |           NULL |
| tktest@127.0.0.1              |      2 | 1.04 ms       | 26.52 us    | 519.35 us   | 1.01 ms     |      1235 |           1195 |
| clone_gtid_thread             |      1 | 800.72 us     | 800.72 us   | 800.72 us   | 800.72 us   |        35 |           NULL |
| srv_worker_thread             |      2 | 96.20 us      | 27.59 us    | 48.10 us    | 68.61 us    |        38 |           NULL |
+-------------------------------+--------+---------------+-------------+-------------+-------------+-----------+----------------+
14 rows in set (0.01 sec)
```

统计最繁忙的十个文件的运行情况：
```sql
root@(none)> select * from sys.io_global_by_file_by_latency limit 10;
+------------------------------------------------------+-------+---------------+------------+--------------+-------------+---------------+------------+--------------+
| file                                                 | total | total_latency | count_read | read_latency | count_write | write_latency | count_misc | misc_latency |
+------------------------------------------------------+-------+---------------+------------+--------------+-------------+---------------+------------+--------------+
| @@innodb_data_home_dir/ibdata1                       | 18629 | 17.11 s       |      18522 | 17.01 s      |          52 | 3.94 ms       |         55 | 99.25 ms     |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1339_tmp |   259 | 1.08 s        |          0 |   0 ps       |         256 | 411.60 ms     |          3 | 670.93 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1340_tmp |   259 | 1.06 s        |          0 |   0 ps       |         256 | 382.20 ms     |          3 | 675.47 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1338_tmp |   259 | 1.04 s        |          0 |   0 ps       |         256 | 415.21 ms     |          3 | 627.93 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1327_tmp |   259 | 951.34 ms     |          0 |   0 ps       |         256 | 429.78 ms     |          3 | 521.55 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1331_tmp |   259 | 934.53 ms     |          0 |   0 ps       |         256 | 404.07 ms     |          3 | 530.46 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1316_tmp |   259 | 932.31 ms     |          0 |   0 ps       |         256 | 389.00 ms     |          3 | 543.31 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1328_tmp |   259 | 923.43 ms     |          0 |   0 ps       |         256 | 387.15 ms     |          3 | 536.28 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1320_tmp |   259 | 923.02 ms     |          0 |   0 ps       |         256 | 387.74 ms     |          3 | 535.29 ms    |
| @@innodb_data_home_dir/#innodb_redo/#ib_redo1341_tmp |   259 | 920.65 ms     |          0 |   0 ps       |         256 | 369.94 ms     |          3 | 550.71 ms    |
+------------------------------------------------------+-------+---------------+------------+--------------+-------------+---------------+------------+--------------+
10 rows in set (0.01 sec)
```

## 统计SQL执行情况
统计全表扫描的SQL语句：
```sql
root@(none)> select * from sys.statements_with_full_table_scans where db<>'sys' limit 1\G
*************************** 1. row ***************************
                   query: SELECT * FROM `performance_schema` . `data_lock_waits`
                      db: performance_schema
              exec_count: 3
           total_latency: 909.82 us
     no_index_used_count: 3
no_good_index_used_count: 0
       no_index_used_pct: 100
               rows_sent: 1
           rows_examined: 1
           rows_sent_avg: 0
       rows_examined_avg: 0
              first_seen: 2023-06-27 16:16:15.235407
               last_seen: 2023-06-27 16:29:49.385441
                  digest: 3861b6e2a1a19662528c9baf6d8e4f2c792ef0d78f888e2e6baa0f3fc964d94e
```

找出全表扫描的表：
```sql
root@(none)> select * from sys.schema_tables_with_full_table_scans\G
```

统计做了排序的SQL语句：
```sql
root@(none)> select * from sys.statements_with_sorting where db<>'sys'\G
```

统计使用了临时表的SQL语句：
```sql
root@(none)> select * from sys.statements_with_temp_tables where db<>'sys'\G
```

## 查看内存分配
查看MySQL分配的总内存：
```sql
root@(none)> select * from sys.memory_global_total;
+-----------------+
| total_allocated |
+-----------------+
| 18.74 GiB       |
+-----------------+
1 row in set (0.01 sec)
```

按线程查询分配的内存：
```sql
root@(none)> select * from sys.memory_by_thread_by_current_bytes limit 1\G
*************************** 1. row ***************************
         thread_id: 1236
              user: root@127.0.0.1
current_count_used: 1436
 current_allocated: 5.07 MiB
 current_avg_alloc: 3.61 KiB
 current_max_alloc: 4.00 MiB
   total_allocated: 1.98 GiB
1 row in set (0.03 sec)
```

按主机查询分配的内存：
```sql
root@(none)> select * from sys.memory_by_host_by_current_bytes limit 1\G
*************************** 1. row ***************************
              host: 127.0.0.1
current_count_used: 1458
 current_allocated: 3.25 MiB
 current_avg_alloc: 2.28 KiB
 current_max_alloc: 2.00 MiB
   total_allocated: 17.66 GiB
1 row in set (0.00 sec)
```




