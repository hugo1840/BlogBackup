---
tags: [oracle]
title: Oracle授予普通用户杀会话权限
created: '2023-06-14T14:07:59.241Z'
modified: '2023-06-14T14:24:21.027Z'
---

Oracle授予普通用户杀会话权限

可以通过创建存储过程来授予普通用户杀自己会话的权限。

# 创建存储过程
```sql
create or replace procedure sys.kill_app_session( 
  appuser in varchar2, appsid in number, appserial in number) 
as 
  v_sid number; 
  v_serial number; 
  v_count number; 
  v_username varchar2(40); 
  v_sql_kill_session varchar2(2000); 
begin 
  select count(*) into v_count from v$session 
  where username = upper(appuser) and sid = appsid and serial# = appserial; 
  select username into v_username from v$session 
  where audsid = SYS_CONTEXT('USERENV', 'SESSIONID'); 
  if v_count = 1 
  then 
    if upper(appuser) = upper(v_username) 
    then 
      v_sid := appsid; 
      v_serial := appserial; 
      v_sql_kill_session := 'alter system kill session ' || '''' || v_sid || ',' || v_serial || ''''; 
      dbms_output.put_line(chr(10)); 
      dbms_output.put_line('###############################################'); 
      dbms_output.put_line('Executing ...' || v_sql_kill_session); 
      execute immediate v_sql_kill_session; 
      dbms_output.put_line('-----------------Finished----------------------'); 
      dbms_output.put_line('###############################################'); 
    else 
      dbms_output.put_line(chr(10)); 
      dbms_output.put_line('###############################################'); 
      dbms_output.put_line('Error : NOT allowed to kill session owned by OTHER user !'); 
      dbms_output.put_line('###############################################'); 
    end if; 
  else 
    dbms_output.put_line(chr(10)); 
    dbms_output.put_line('###############################################'); 
    dbms_output.put_line('Error : The given session does not exist !'); 
    dbms_output.put_line('###############################################'); 
  end if; 
end; 
/
```

# 授予相关权限
将存储过程的执行权限授予普通用户：
```sql
create public synonym kill_app_session for sys.kill_app_session; 
grant execute on kill_app_session to xxx;
```

还需要授予以下查询权限： 
```sql
grant select on gv_$session to xxx; 
grant select on v_$session to xxx; 
grant select on gv_$sql to xxx; 
grant select on v_$sql to xxx; 
grant select on gv_$process to xxx; 
grant select on v_$process to xxx; 
grant select on gv_$parameter to xxx; 
grant select on v_$parameter to xxx; 
grant select on gv_$sqlarea to xxx; 
grant select on v_$sqlarea to xxx; 
grant select on gv_$locked_object to xxx; 
grant select on v_$locked_object to xxx;
```

# 用户杀会话
普通用户查询正在运行的SQL，获取到目标SQL的`sid`和`serial#`： 
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
select inst_id ||':'|| sid ||','|| serial# ssid,username,sql_id,event,
substr(program,1,25) program,machine,state,last_call_et exec_time,status 
from gv$session where username is not null order by last_call_et;
```

普通用户杀会话： 
```sql
exec kill_app_session('<用户名>',<sid>,<serial#>);
```



