---
tags: [oracle]
title: Oracle统计某个时段运行的SQL
created: '2023-06-12T12:11:54.412Z'
modified: '2023-06-12T12:39:20.764Z'
---

Oracle统计某个时段运行的SQL

# 统计当前正在运行的SQL 
```sql
set line 200 pages 1000 
col event for a30 
col program for a35 
col username for a10 
col exec_time for 9999999999 
col sql_id for a15 
col machine for a30 
col ssid for a13 
col state for a20 
col status for a10 
select inst_id ||':'|| sid ||','|| serial# ssid,
username,sql_id,event,
substr(program,1,25) program,machine,
state,last_call_et exec_time,status 
from gv$session 
where wait_class<>'Idle' and username is not null 
order by last_call_et;
```

# 统计指定时间段运行的SQL 
```sql
set lines 220 
col sample_time for a20 
col username for a10 
col ssid for a15 
col event for a20 
col sql_id for a15 
col program for a35 
col machine for a20 
select to_char(h.sample_time,'yyyymmdd hh24:mi:ss') sample_time,s.username,
h.instance_number ||':'|| h.session_id ||':'|| h.session_serial# as ssid, 
h.sql_id,h.sql_plan_hash_value as plan_hash, h.event,
h.session_state,h.blocking_session_status, 
substr(h.program,1,25) program,h.machine 
from dba_hist_active_sess_history h, v$session s 
where h.user_id=s.user# and s.username is not null 
and h.sample_time>=to_date('&begin_time','mmdd-hh24:mi:ss') 
and h.sample_time<to_date('&end_time','mmdd-hh24:mi:ss') 
order by h.sample_time,s.username;
```

