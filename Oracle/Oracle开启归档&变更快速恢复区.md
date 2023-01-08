---
tags: [oracle]
title: Oracle开启归档&变更快速恢复区
created: '2023-01-08T03:13:36.644Z'
modified: '2023-01-08T03:25:31.227Z'
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

打开数据库：
```sql
SQL> alter database open;
Database altered.
```

# 变更快速恢复区
修改快速恢复区（Fast Recovery Area）位置的前提条件是已开启归档模式。

查看快速恢复区位置：
```sql
SQL> show parameter db_recovery_file_dest;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      +DATADG
db_recovery_file_dest_size           big integer 12732M
```

修改快速恢复区位置（确保目标路径已创建并赋予oracle用户rwx权限）：
```sql
SQL> alter system set db_recovery_file_dest='/u01/app/flash_recovery_area';

System altered.

SQL> alter database flashback on;

Database altered.

SQL> show parameter db_recovery_file_dest;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      /u01/app/flash_recovery_area
db_recovery_file_dest_size           big integer 12732M
```


**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/configuring-rman-client-basic.html#GUID-F8BFD7F3-345D-4A9E-9596-86AEAB29507D
【2】https://www.cnblogs.com/liang-ning/articles/15883826.html
【3】https://blog.51cto.com/chenzm0592/5284522
【4】https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/configuring-rman-client-basic.html#GUID-84988C3B-8F97-460A-8882-C26979F53836
