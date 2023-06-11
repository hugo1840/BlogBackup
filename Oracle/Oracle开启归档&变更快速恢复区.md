---
tags: [oracle]
title: Oracle开启归档&变更快速恢复区
created: '2023-01-08T03:13:36.644Z'
modified: '2023-01-15T03:54:30.776Z'
---

Oracle开启归档&变更快速恢复区


# 开启归档模式
查看是否开启归档模式：
```sql
SQL> archive log list;
Database log mode              No Archive Mode   --未开启归档
Automatic archival             Disabled          --自动归档已禁用
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     5
Current log sequence           7
```

关闭数据库：
```sql
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
```

挂载数据库：
```sql
SQL> startup mount;
ORACLE instance started.

Total System Global Area 4915722768 bytes
Fixed Size                  9144848 bytes
Variable Size             922746880 bytes
Database Buffers         3976200192 bytes
Redo Buffers                7630848 bytes
Database mounted.
```

开启归档模式：
```sql
SQL> alter database archivelog;
Database altered.

SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     5
Next log sequence to archive   7
Current log sequence           7
```

# 修改归档路径

修改归档路径：
```sql
SQL> alter system set log_archive_dest_1='location=/oradata/arch';
System altered.
```

修改归档日志命名格式：
```sql
SQL> alter system set log_archive_format="arch_%t_%s_%r.log";
alter system set log_archive_format="arch_%t_%s_%r.log"
                 *
ERROR at line 1:
ORA-02095: specified initialization parameter cannot be modified

SQL> alter system set log_archive_format="arch_%t_%s_%r.log" scope=spfile;
System altered.
```

检查上面两个修改是否生效：
```sql
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /oradata/arch
Oldest online log sequence     1
Next log sequence to archive   3
Current log sequence           3

SQL> show parameter log_archive_format

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_format                   string      %t_%s_%r.dbf
```

打开数据库：
```sql
SQL> alter database open;
Database altered.
```

重启数据库使`log_archive_format`参数修改生效：
```sql
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup;
ORACLE instance started.

Total System Global Area 4915722040 bytes
Fixed Size                  8906552 bytes
Variable Size             889192448 bytes
Database Buffers         4009754624 bytes
Redo Buffers                7868416 bytes
Database mounted.
Database opened.
SQL> show parameter log_archive_format

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_format                   string      arch_%t_%s_%r.log
```


# 修改快速恢复区
查看快速恢复区（Fast Recovery Area）：
```sql
-- 查看快速恢复区位置和大小
SQL> show parameter db_recovery_file_dest;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      +DATADG
db_recovery_file_dest_size           big integer 12732M

-- 查看快速恢复区使用情况
SQL> select * from v$flash_recovery_area_usage;
SQL> select * from v$recovery_file_dest;
```

修改快速恢复区大小和位置（确保目标路径已创建并赋予oracle用户rwx权限）：
```sql
SQL> alter system set db_recovery_file_dest_size=20G;
SQL> alter system set db_recovery_file_dest='/u01/app/flash_recovery_area';

System altered.

--开启闪回区（需要开启归档模式）
SQL> alter database flashback on;

Database altered.

SQL> show parameter db_recovery_file_dest;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      /u01/app/flash_recovery_area
db_recovery_file_dest_size           big integer 20G
```
快速恢复区需要设置得足够大，能够放得下控制文件、数据文件的副本、以及联机重做日志和闪回日志，并且最好与Oracle数据文件目录位于不同的磁盘位置。


**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/configuring-rman-client-basic.html#GUID-F8BFD7F3-345D-4A9E-9596-86AEAB29507D
【2】https://www.cnblogs.com/liang-ning/articles/15883826.html
【3】https://blog.51cto.com/chenzm0592/5284522
【4】https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/configuring-rman-client-basic.html#GUID-84988C3B-8F97-460A-8882-C26979F53836
