---
tags: [oracle]
title: Oracle杀会话回滚时间长处理办法
created: '2023-10-22T05:54:48.046Z'
modified: '2023-10-22T05:58:41.430Z'
---

Oracle杀会话回滚时间长处理办法

获取被KILL会话的SID：
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
select inst_id||':'||sid||','||serial# ssid,username,sql_id,sql_exec_id,event,
substr(program,1,25) program,machine,state,last_call_et exec_time,status 
from gv$session 
where wait_class<>'Idle' and username is not null 
order by last_call_et;

...
1:633,52377   APPUSER   13bqm6vvsn5s6	 db file sequential read	 sqlplus@etlhost (TNS V1-V3)	etlhost	   WAITING	243509  KILLED
```
注意最后一列状态为**KILLED**。

获取被KILL会话对应的操作系统进程号PID：
```sql
select spid from v$process 
where addr in (select paddr 
from v$session 
where username='APPUSER' and sid=633);

SPID
------------------------
273322

--或者
select spid from v$process 
where addr in (select creator_addr 
from v$session 
where username='APPUSER' and status='KILLED');
```

或者：
```sql
SET LINESIZE 100
COLUMN spid FORMAT A10
COLUMN username FORMAT A10
COLUMN program FORMAT A45
 
SELECT s.inst_id,
       s.sid,
       s.serial#,
       p.spid,
       s.username,
       s.program
FROM   gv$session s
       JOIN gv$process p ON p.addr = s.paddr AND p.inst_id = s.inst_id
WHERE  s.type != 'BACKGROUND' 
and s.username='APPUSER';

   INST_ID	  SID	 SERIAL# SPID	    USERNAME   PROGRAM
---------- ---------- ---------- ---------- ---------- ---------------------------------------------
	 1	  633	   52377 273322     APPUSER    sqlplus@etlhost (TNS V1-V3)
	 1	  722	    1331 104848     APPUSER    sqlplus@etlhost (TNS V1-V3)
	 1	  777	   64051 104254     APPUSER    sqlplus@etlhost (TNS V1-V3)
	 1	  938	   51475 104793     APPUSER    sqlplus@etlhost (TNS V1-V3)
	 1	 1101	   55005 161435     APPUSER    plsqldev.exe
	 1	 1172	   21141 101804     APPUSER    sqlplus@etlhost (TNS V1-V3)
	 1	 2036	   42029 161411     APPUSER    plsqldev.exe

7 rows selected.
```

操作系统层面杀掉对应连接：
```bash
[oracle@dbhost ~]$ ps -ef | grep oracle | grep LOCAL | grep 273322
oracle   273322      1 28 Oct15 ?        19:25:50 oracleorcldb (LOCAL=NO)

[oracle@dbhost ~]$ kill -9 273322
```

**References**
[1] https://blog.51cto.com/u_13631369/6251425


