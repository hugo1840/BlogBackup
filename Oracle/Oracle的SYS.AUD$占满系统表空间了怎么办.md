---
tags: [oracle]
title: Oracle的SYS.AUD$占满系统表空间了怎么办
created: '2023-03-27T13:23:13.797Z'
modified: '2023-03-27T13:47:15.471Z'
---

Oracle的SYS.AUD$占满系统表空间了怎么办

# 问题分析
应该反馈无法连接数据库。查看告警日志：
```bash
[oracle]:/home/oracle> tail -f /oracle/app/oracle/diag/rdbms/<db_unique_name>/<instance_name>/trace/alert_<instance_name>.log 

Mon Mar 27 09:33:23 2023 
ORA-1653: unable to extend table SYS.AUD$ by 128 in tablespace SYSTEM 
ORA-1653: unable to extend table SYS.AUD$ by 8192 in tablespace SYSTEM 
ORA-1653: unable to extend table SYS.AUD$ by 128 in tablespace SYSTEM 
ORA-1653: unable to extend table SYS.AUD$ by 8192 in tablespace SYSTEM
...
```

查看报错解释：
```bash
[oracle]:/home/oracle> oerr ora 1653 
01653, 00000, "unable to extend table %s.%s by %s in tablespace %s" 
// *Cause: Failed to allocate an extent of the required number of blocks for 
// a table segment in the tablespace indicated. 
// *Action: Use ALTER TABLESPACE ADD DATAFILE statement to add one or more 
// files to the tablespace indicated.
```

即SYSTEM表空间被`SYS.AUD$`这个表占满了。

查看表空间使用率：
```sql
set linesize 200 
select total.tablespace_name, 
round(total.max_mb,2) max_mb, round(total.MB,2) total_mb, 
round(free.MB,2) free_mb, round(total.MB - free.MB,2) used_mb, 
round((1 - free.MB/total.MB)*100,2) as used_percent 
from 
  (select tablespace_name,sum(bytes)/1024/1024 MB 
  from dba_free_space group by tablespace_name) free, 
  (select tablespace_name,sum(bytes)/1024/1024 MB, sum(maxbytes)/1024/1024 max_mb
  from dba_data_files group by tablespace_name) total 
where free.tablespace_name = total.tablespace_name 
order by used_percent desc;

TABLESPACE_NAME MAX_MB    TOTAL_MB  FREE_MB USED_MB   USED_PERCENT

SYSTEM          32767.98  32730     58.69   32671.31  99.82 
USERS           425982.81 189438.94 1211.25 188227.69 99.36 
...
```

# 应急处理
为了尽快恢复业务，当然是马上扩容表空间了。
```sql
SQL> alter tablespace system add datafile;

Tablespace altered.
```

# 长远的解决方案
Oracle 11g中默认开启了审计功能，数据库标准审计信息写入`SYS.AUD$`表，细粒度审计信息写入`SYS.FGA_LOG$`表。从长远来看，为了避免审计表占满SYSTEM表空间导致数据库无法访问，测试环境可以直接关闭审计功能，生产环境则可以将审计表迁移到非系统表空间。 

## 测试环境：关闭审计功能 
```sql
--关闭审计
SQL> alter system set audit_trail=none scope=spfile; 

--重启数据库
SQL> shutdown immediate; 
SQL> startup; 

--清空审计表
SQL> truncate table SYS.AUD$;
```


## 生产环境：迁移表空间 
数据库版本：**Oracle 11.2**

```sql
--创建要迁移过去的表空间
SQL> create tablespace auditspace; 
SQL> alter tablespace auditspace add datafile;

--检查表大小，评估迁移时间 
SQL> select table_name, tablespace_name from dba_tables 
where table_name in ('AUD$', 'FGA_LOG$') order by table_name; 
SQL> select segment_name,bytes/1024/1024 size_mb from dba_segments 
where segment_name in ('AUD$', 'FGA_LOG$');

--迁移表空间 
BEGIN 
DBMS_AUDIT_MGMT.set_audit_trail_location( 
  audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD, 
  audit_trail_location_value => 'AUDITSPACE'); 
END; 
/

BEGIN 
DBMS_AUDIT_MGMT.set_audit_trail_location( 
  audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_FGA_STD, 
  audit_trail_location_value => 'AUDITSPACE'); 
END; 
/

--检查迁移结果
select table_name, tablespace_name from dba_tables 
where table_name in ('AUD$', 'FGA_LOG$') order by table_name;
```



**References**
【1】https://www.cnblogs.com/yaoyangding/p/12259576.html


