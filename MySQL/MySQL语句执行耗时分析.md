---
tags: [mysql]
title: MySQL语句执行耗时分析
created: '2023-04-13T11:37:02.080Z'
modified: '2023-04-16T12:16:25.157Z'
---

MySQL语句执行耗时分析

# MySQL Profile查看SQL执行各阶段耗时

```sql
--开启SQL Profiling
SQL> set profiling=1; 

--执行目标SQL
SQL> SELECT * FROM db.tabname;

--获取Query ID和SQL执行总时长（秒）
SQL> show profiles; 

--获取SQL执行各阶段时间和资源消耗 
SQL> show profile all for query 2; 
--获取SQL执行各阶段IO次数
SQL> show profile block for query 2; 
--获取SQL执行各阶段CPU耗时（秒）
SQL> show profile cpu for query 2; 
--获取SQL执行各阶段通信次数 
SQL> show profile ipc for query 2; 
--获取SQL执行各阶段swap交换次数
SQL> show profile swaps for query 2; 

--关闭SQL Profiling
SQL> set profiling=0;
```

>:shark:See more in https://dev.mysql.com/doc/refman/8.0/en/show-profile.html

# Performance Schema查看SQL执行各阶段耗时
MySQL Profile目前已被列为Deprecated，官方推荐使用Performance Schema替代。不过目前Performance Schema好像还不是很完善，只能查看SQL执行各阶段的耗时，而看不到CPU和IO等资源消耗（截止8.0.32）。

## 配置收集哪些用户的SQL执行信息 
查看搜集哪些用户的SQL执行历史信息： 
```sql
select * from performance_schema.setup_actors;
```

限制搜集SQL执行历史信息的用户为本地root用户连接（根据实际需求设置）： 
```sql
update performance_schema.setup_actors 
set enabled='NO', history='NO' 
where host='%' and user='%';

insert into performance_schema.setup_actors (host,user,role,enabled,history) 
values('localhost','root','%','YES','YES');

select * from performance_schema.setup_actors;
```

## 开启SQL执行信息收集的相关特性 
确保`setup_instruments`中的相关特性已开启： 
```sql
update performance_schema.setup_instruments 
set enabled='YES', TIMED='YES' 
where name like '%statement/%';

update performance_schema.setup_instruments 
set enabled='YES', TIMED='YES' 
where name like '%stage/%';
```

确保`setup_consumers`中的相关特性已开启： 
```sql
update performance_schema.setup_consumers 
set enabled='YES' where name like '%events_statements_%';

update performance_schema.setup_consumers 
set enabled='YES' where name like '%events_stages_%';
```

## 执行目标SQL 
```sql
SELECT * FROM employees.employees WHERE emp_no = 10001;
```

## 获取SQL执行的EVENT_ID 
从`events_statements_history_long`中获取执行SQL的EVENT_ID：
```sql
select event_id, truncate(timer_wait/1000000000000,6) as duration, sql_text 
from performance_schema.events_statements_history_long 
where sql_text like 'SELECT%';
```

## 获取SQL执行各阶段耗时
从`events_stages_history_long`中获取SQL执行各阶段的耗时：
```sql
--以nesting_event_id匹配上面得到的event_id
select event_name as stage, truncate(timer_wait/1000000000000,6) as duration 
from performance_schema.events_stages_history_long 
where nesting_event_id=299;
```

>:dolphin:See more in https://dev.mysql.com/doc/refman/8.0/en/performance-schema-query-profiling.html


