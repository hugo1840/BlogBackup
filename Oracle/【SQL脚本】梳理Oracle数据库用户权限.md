---
tags: [oracle]
title: 【SQL脚本】梳理Oracle数据库用户权限
created: '2023-07-23T03:48:44.221Z'
modified: '2023-07-23T03:49:44.713Z'
---

【SQL脚本】梳理Oracle数据库用户权限

```sql
spool /home/oracle/user_privs_summary.log

--梳理系统权限
declare
  cursor cursor_all_sys_privs is 
    select grantee,privilege from dba_sys_privs where grantee in (select 
	  username from dba_users where account_status='OPEN' and username not in ('SYS','SYSTEM'))
	  order by grantee;
begin
  for item in cursor_all_sys_privs loop
    dbms_output.put_line(item.grantee || ' # ' || item.privilege);
  end loop;
end;
/


--梳理对象权限
declare
  cursor cursor_all_tab_privs is 
    select grantee,table_name,privilege from dba_tab_privs where grantee in (select 
	  username from dba_users where account_status='OPEN' and username not in ('SYS','SYSTEM'))
	  order by grantee;
begin
  for item in cursor_all_tab_privs loop
    dbms_output.put_line(item.grantee || ' # ' || item.table_name || ' # ' || item.privilege);
  end loop;
end;
/


--梳理角色权限
declare
  cursor cursor_all_role_privs is 
    select grantee,granted_role from dba_role_privs where grantee in (select 
	  username from dba_users where account_status='OPEN' and username not in ('SYS','SYSTEM'))
	  order by grantee;
begin
  for item in cursor_all_role_privs loop
    dbms_output.put_line(item.grantee || ' # ' || item.granted_role);
  end loop;
end;
/

spool off
```


