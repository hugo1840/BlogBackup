---
tags: [oracle]
title: 【日积跬步】Oracle查看与修改最大连接数
created: '2023-01-18T12:55:16.133Z'
modified: '2023-01-18T13:36:23.160Z'
---

【日积跬步】Oracle查看与修改最大连接数

查看最大进程数、连接数：
```sql
SQL> show parameter processes
SQL> show parameter sessions
SQL> select name,value from v$parameter where name in ('processes','sessions');
```

查看当前进程数、连接数：
```sql
SQL> select count(*) from v$process;
SQL> select count(*) from v$session;
SQL> select inst_id,count(*) from gv$session group by inst_id;   -- for RAC
```
其中，`v$session`记录的主要是客户端连接，`v$process`记录的则是Oracle服务进程信息。

查看当前并发连接数：
```sql
SQL> select count(*) from v$session where status='ACTIVE';
SQL> select inst_id,count(*) from gv$session where status='ACTIVE' group by inst_id;
```

统计不同用户的连接数：
```sql
SQL> select username,count(username) from v$session where username is not null
group by username;
```

查看起库以来的最大进程数、最大连接数：
```sql
SQL> select resource_name,max_utilization,limit_value 
from v$resource_limit where resource_name in ('processes','sessions');
```

修改最大进程数、连接数：
```sql
SQL> alter system set processes=1500 scope=spfile;
SQL> shutdown immediate;
SQL> startup;
```
这里只用修改processes和sessions其中的一个参数就行。




