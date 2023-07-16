---
tags: [oracle]
title: 行锁导致的SQL阻塞问题分析处理
created: '2023-07-16T02:35:50.091Z'
modified: '2023-07-16T06:05:01.349Z'
---

行锁导致的SQL阻塞问题分析处理

# 行锁分析处理流程
查看表上是否有锁：
```sql
select inst_id,object_id,session_id sid,oracle_username, 
decode(locked_mode,0,'None',2,'Row Share Lock',3,'Row Exclusive Table Lock', 
4,'Share Table Lock',5,'Share Row Exclusive Table Lock',6,'Exclusive Table Lock','NULL') 
from gv$locked_object where object_id = ( select 
object_id from dba_objects where object_type = 'TABLE' 
and object_name = '&obj_name' and owner = '&owner');
```

找到对应`inst_id`和`sid`的会话SQL：
```sql
select inst_id||':'||sid||','||serial# ssid,username,sql_id,event, 
substr(program,1,25) program,machine,state,last_call_et exec_time,status 
from gv$session where inst_id='&inst_id' and sid='&session_id';
```

杀掉造成阻塞的对应语句：
```sql
alter system kill session '&sid,&serial';
```

# 锁与SQL阻塞分析脚本
执行`check_blocking_stats.sql`脚本：
```sql
alter session set nls_date_format='yyyymmdd hh24:mi:ss';
select sysdate from dual;
set lines 200 pages 1000
col blocking_instance format 99
col inst_id format 99
col sid format 99999999
col blocking_session format 999999
col event for a40
col file_name format a40
col sql_text format a60
col PROGRAM format a18
col module format a18
col machine format a18
COLUMN owner FORMAT A20
COLUMN username FORMAT A20
COLUMN object_owner FORMAT A20
COLUMN object_name FORMAT A30
COLUMN locked_mode FORMAT A15

prompt "blocking sessions"
select blocking_instance, blocking_session, count(*)
  from gv$session
 where blocking_session is not null
 group by blocking_instance, blocking_session
 order by 1, 3;

prompt
prompt
prompt "blocking detail"
select inst_id,
       sid,
       serial#,
       nvl(sql_id,prev_sql_id) sql_id,
       prev_sql_id,
       event,
       state,
       seconds_in_wait,
       p1,
       p2,
       blocking_instance,
       blocking_session
  from gv$session
 where blocking_session is not null
 order by blocking_instance,blocking_session,inst_id,seconds_in_wait ;

prompt "Holder and Waiter:"
select decode(request,0,'***holder:','     waiter:') holder,
sid,id1,id2,lmode,request,type,ctime,block
from v$lock
where (id1,id2,type) in
(select id1,id2,type from v$lock where request>0) 
order by id1,request;

prompt
prompt
prompt "Event enq: TX - row lock contention waiting object"

select se.inst_id,
       se.sid,
       se.serial#,
       obj.owner,
       obj.object_name,
       obj.object_type,
       se.ROW_WAIT_BLOCK#
  from gv$session se, 
       dba_objects obj 
where se.event='enq: TX - row lock contention'
  and se.row_wait_obj# = obj.object_id; 
 
prompt 
prompt "Lock info"
select INST_ID,SID,TYPE,ID1,ID2,LMODE,REQUEST,CTIME,BLOCK 
from GV$lock where block=1 or request<>0 ORDER BY 1,2; 
 
prompt
prompt
prompt "Locked objects:"
prompt "Locked objects:"
SELECT b.session_id AS sid,
       NVL(b.oracle_username, '(oracle)') AS username,
       a.owner AS object_owner,
       a.object_name,
       Decode(b.locked_mode, 0, 'None',
                             1, 'Null (NULL)',
                             2, 'Row-S (SS)',
                             3, 'Row-X (SX)',
                             4, 'Share (S)',
                             5, 'S/Row-X (SSX)',
                             6, 'Exclusive (X)',
                             b.locked_mode) locked_mode,
       b.os_user_name
FROM   dba_objects a,
       v$locked_object b
WHERE  a.object_id = b.object_id
ORDER BY 1, 2, 3, 4;

prompt
prompt
prompt "blocking sql text"
select se.inst_id,
       se.sid,
       se.sql_id,
       sq.sql_text
from
   gv$session se,v$sql sq
where se.sql_id =  sq.sql_id
  and (inst_id,sid) in
 (select inst_id,
        sid
   from
      (select distinct
             blocking_instance as inst_id,
             blocking_session  as sid
        from gv$session 
      where blocking_session is not null )
 );

prompt
prompt
prompt "blocked sql text"
select se.inst_id,
       se.sid,
       nvl(se.sql_id,se.prev_sql_id) sql_id,
       sq.sql_text
  from gv$session se,v$sql sq
 where blocking_session is not null
   and nvl(se.sql_id,prev_sql_id) = sq.sql_id;
   
prompt 
prompt blocking session info
select se.inst_id,
       se.sid,
       nvl(se.sql_id,se.prev_sql_id) sql_id,
       logon_time,
       program,
       username,
       machine,
       module,
       last_call_et
from
   gv$session se
where (inst_id,sid) in
 (select inst_id,
        sid
   from
      (select distinct
             blocking_instance as inst_id,
             blocking_session  as sid
        from gv$session 
      where blocking_session is not null )
 );
 
 prompt
 prompt
 prompt blocking session consume rollback segments
 select se.inst_id,
       se.sid,
       se.serial#,
       nvl(se.sql_id,se.prev_sql_id) sql_id,
       tr.used_ublk,
       tr.used_urec
from
   gv$session se,gv$transaction tr
where (se.inst_id,se.sid) in
 (select inst_id,
        sid
   from
      (select distinct
             blocking_instance as inst_id,
             blocking_session  as sid
        from gv$session 
      where blocking_session is not null )
 )
 and se.taddr = tr.addr;
```


