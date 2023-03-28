---
tags: [oracle]
title: Oracle Datapump导出导入Schema示例
created: '2023-03-28T12:10:25.404Z'
modified: '2023-03-28T12:42:31.016Z'
---

Oracle Datapump导出导入Schema示例

**数据库版本：11.2**

# 源库：expdp导出用户数据

1. 创建数据泵：

```sql
SQL> select owner,directory_name,directory_path from dba_directories; 
SQL> create directory dumpdir as '/oradata/backup';
```

2. 导出用户数据：

```bash
expdp '/ as sysdba' directory=dumpdir 
dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp 
logfile=dump_${ORACLE_SID}_`date +%F`.log 
exclude=STATISTICS,db_link parallel=2 compression=all schemas=APP_USER
```
注意dump文件名称中要有`%U`，用于区分多个dump文件。

3. 传输dump文件到目标服务器。


# 目标库：impdp导入用户数据
## 创建数据泵

```sql
SQL> select owner,directory_name,directory_path from dba_directories; 
SQL> create directory dumpdir as '/oradata/backup';
```
确保要导入的dump文件位于数据泵对应的目录下。

## 清理用户数据

**DROP USER ... CASCADE**不会清理**PACKAGE**、**FUNCTION**、**PROCEDURE**、**VIEW**，需要提前删除。

生成删除用户对象的脚本： 
```sql
set lines 300 
set heading off 
spool /home/oracle/delUserObj.sql

select 
  'drop '||object_type||' '||owner||'.'||object_name||';' 
  from dba_objects 
  where object_type in ('PACKAGE','FUNCTION','PROCEDURE','VIEW') 
  and owner = upper('&&ownername') 
union all 
select 
  'exec dbms_scheduler.drop_job('''||owner||'.'||object_name||''',force=>true);' 
  from dba_objects 
  where object_type = 'JOB' and owner = upper('&&ownername') 
  order by 1;

spool off
```

执行上面的语句时，会提示为变量`ownername`输入一个值。检查生成的`delUserObj.sql`脚本，然后调用清理：

```sql
SQL> @/home/oracle/delUserObj
``` 

> 1. `&`用来创建一个临时变量，每次遇到这个临时变量时，都会提示你输入一个值； 
> 2. `&&`用来创建一个持久变量，当用`&&`引用这个变量时，只在第一次遇到时提示输入一个值。

删除用户、表和索引： 

```sql
SQL> select username, default_tablespace from dba_users where username='APP_USER'; 
SQL> drop user APP_USER cascade;
```
**DROP USER ... CASCADE**命令会删除用户，并清理用户表空间中的数据，但是会保留表空间本身。

## 扩容表空间 
确保表空间能够容纳导入的数据。

检查表空间大小： 
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
```

扩容表空间： 
```sql
alter tablespace APP_USER add datafile;
```

## 导入数据 
```bash
impdp '/ as sysdba' directory=dumpdir 
dumpfile=dumpfile_01.dmp,dumpfile_02.dmp 
parallel=2 schemas=APP_USER
```

其中，dumpfile参数不能为OS中的绝对路径，否则会报错： 
```bash
ORA-39001: invalid argument value 
ORA-39000: bad dump file specification 
ORA-39088: file name cannot contain a path specification
```

IMPDP导入DUMP文件的过程会自动创建用户，并关联表空间权限，无需手动提前创建APP_USER用户。 如果提前创建了用户，会提示以下信息，但不会影响数据导入。 
```bash
ORA-31684: Object type USER:"APP_USER" already exists
```

## 导入后检查 
检查用户数据量： 
```sql
select sum(bytes/1024/1024/1024) as size_gb,owner,tablespace_name 
from dba_segments group by owner,tablespace_name;
```

检查用户权限： 
```sql
select grantee,table_name,owner,privilege from dba_tab_privs where grantee='APP_USER'; 
select grantee,privilege from dba_sys_privs where grantee='APP_USER'; 
select * from dba_role_privs where grantee='APP_USER';
```
视图**dba_sys_privs**中的某些系统权限可能要重新授予。

检查导入的数据库对象：
```sql
--检查表是否导入 
select owner,table_name,tablespace_name from dba_tables where owner='APP_USER';

--检查视图是否导入
select owner,view_name from dba_views where owner='APP_USER';

--检查索引是否导入
select owner,index_name,table_name,table_owner from dba_indexes where table_owner='APP_USER';
```

检查是否存在无效对象：
```sql
select owner,object_name,object_type,created,status from dba_objects 
where owner='APP_USER' and status <> 'VALID';
```



**References** 
【1】https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/DROP-USER.html#GUID-F766E1A2-6686-4734-89BA-0C5B4120B90E 
【2】https://docs.oracle.com/en/database/oracle/oracle-database/12.2/sutil/datapump-import-utility.html


