---
tags: [oracle]
title: Oracle统计信息问题排查常用SQL
created: '2023-09-27T11:01:34.283Z'
modified: '2023-09-27T11:36:41.190Z'
---

Oracle统计信息问题排查常用SQL

# 对表的基本情况分析
是否为临时表：
```sql
select owner,table_name,temporary from dba_tables where table_name='xxx';
```

是否为分区表：
```sql
select owner,table_name,partitioned from dba_tables where table_name='xxx';
```

检查表的大小：
```sql
select sum(bytes)/1024/1024/1024 as size_gb from dba_segments where segment_name='xxx';
```

查看特定用户下统计信息被锁定的表：
```sql
select owner,table_name,partition_name,stattype_locked from dba_tab_statistics 
where stattype_locked is not null and owner='xxx';
```

查看特定用户下统计信息已经过期的表：
```sql
select owner,table_name,partition_name from dba_tab_statistics 
where stattype_locked is null and stale_stats='YES' and owner='xxx';

select table_name,partition_name from user_tab_statistics 
where stattype_locked is null and STALE_STATS='YES';
```
其中，`stattype_locked`表示锁定的统计信息类型（data/cache/all），`stale_stats`表示统计信息是否过期。


# 统计信息收集作业分析

查看统计信息自动收集作业是否开启：
```sql
select client_name,status from dba_autotask_client where client_name like '%stats%';

select client_name,status from dba_autotask_client where client_name='auto optimizer stats collection';
```

查看历史统计信息自动收集记录：
```sql
select client_name,window_name,window_start_time,window_duration,window_end_time
from dba_autotask_client_history 
where client_name='auto optimizer stats collection'
order by window_start_time;
```

查看统计信息收集的具体操作：
```sql
select owner,program_name,program_type,program_action,enabled
from dba_scheduler_programs where program_name='GATHER_STATS_PROG';
```

查看统计信息自动收集作业窗口：
```sql
select window_name,repeat_interval,duration from dba_scheduler_windows where enabled='TRUE';
```

查看用户创建的统计信息收集定时作业：
```sql
select owner,job_name,enabled,state,program_name,job_action,schedule_name,
last_start_date,last_run_duration,next_run_date,run_count
from dba_scheduler_jobs where job_name='xxx';
```

# 最近一次的统计信息收集

查看指定表最近一次统计信息收集的时间和记录的行数：
```sql
--非分区表
select table_name,last_analyzed,num_rows 
from dba_tables where table_name='xxx';

--分区表
select table_owner,table_name,partition_name,last_analyzed,num_rows 
from dba_tab_partitions where table_name='xxx' order by partition_name;
```

查看指定表从最近一次统计信息收集以来的数据变化量：
```sql
select * from dba_tab_modifications where table_name='xxx';

select table_name,partition_name,inserts,updates,deletes,truncated,drop_segments
from dba_tab_modifications where table_name='xxx' 
order by partition_name;
```
其中，`INSERTS/UPDATES/DELETES`分别表示**从上一次收集表统计信息以来**插入/更新/删除的次数，`TRUNCATED`表示是否被TRUNCATE过，`DROP_SEGMENTS`表示被DROP过的分区和子分区的段数。

通过计算表被插入、更新和删除的总行数与`num_rows`的比值是否超过10%，可以大致估算是否会触发统计信息自动收集。


# 修改触发统计信息收集的阈值

查看自动统计信息收集触发的阈值（默认为10%）：
```sql
--全局参数值
SELECT dbms_stats.get_prefs(pname => 'STALE_PERCENT') FROM dual;

--指定用户表
SELECT dbms_stats.get_prefs(pname => 'STALE_PERCENT',ownname => 'xxx',tabname => 'xxx') FROM dual;
```
其中`STALE_PERCENT`是指DML操作导致表的行记录被修改/增删的比例。

修改自动统计信息收集触发的阈值为5%：
```sql
--修改全局级别的参数值
EXEC dbms_stats.set_global_prefs(pname => 'STALE_PERCENT',pvalue => 5);
--set_global_prefs对所有表生效，对新建的表也生效

--修改全库级别的参数值
EXEC dbms_stats.set_database_prefs(pname => 'STALE_PERCENT',pvalue => 5);  
--set_database_prefs默认不影响Oracle内置表，对新建的表不生效

--修改指定用户表的参数值
EXEC dbms_stats.set_table_prefs(ownname => 'xxx',tabname => 'xxx',pname => 'STALE_PERCENT',pvalue => 5);
```


**References**
[1] https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_STATS.html#GUID-2C00FE80-1553-404C-85B6-220895561FE8
[2] https://www.modb.pro/db/543228






