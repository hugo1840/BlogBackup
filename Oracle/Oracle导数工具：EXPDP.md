---
tags: [oracle]
title: Oracle导数工具：EXPDP
created: '2023-01-15T07:59:56.810Z'
modified: '2023-03-27T13:13:26.938Z'
---

Oracle导数工具：EXPDP

Datapump是Oracle提供的导数工具，包含两个命令行工具（`expdp`和`impdp`）和一个PL/SQL包（`DBMS_DATAPUMP`）。

# Datapump导数权限
普通用户如果要导入导出全库数据（或者其他用户的数据），当前用户必须具有`datapump_imp_full_database`和`datapump_exp_full_database`权限，否则就只能导入导出自己的数据。

```sql
SQL> grant datapump_exp_full_database to miguel;
Grant succeeded.

SQL> grant datapump_imp_full_database to miguel;
Grant succeeded.

SQL> select grantee,granted_role from dba_role_privs where grantee='MIGUEL';

GRANTEE      GRANTED_ROLE
------------ ------------------------------
MIGUEL       DATAPUMP_EXP_FULL_DATABASE
MIGUEL       RESOURCE
MIGUEL       CONNECT
MIGUEL       DATAPUMP_IMP_FULL_DATABASE

SQL> select grantee,granted_role from dba_role_privs where grantee='pablo';

GRANTEE      GRANTED_ROLE
------------ ------------------------------
pablo        CONNECT
pablo        RESOURCE
```

# Directory创建及授权
创建Directory对象来存储导出的数据（必须有`CREATE ANY DIRECTORY`权限）。
```sql
SQL> create directory dumpdir1 as '/oradata/dumpdir';
Directory created.

SQL> select owner,directory_name,directory_path from dba_directories where directory_name='DUMPDIR1';

OWNER
--------------------------------------------------------------------------------
DIRECTORY_NAME
--------------------------------------------------------------------------------
DIRECTORY_PATH
--------------------------------------------------------------------------------
SYS
DUMPDIR1
/oradata/dumpdir
```

将Directory的读写权限授予要导数的用户：
```sql
SQL> grant read,write on directory dumpdir1 to MIGUEL;
Grant succeeded.

SQL> grant read,write on directory dumpdir1 to "pablo";
Grant succeeded.
```


# 使用expdp导出数据
EXPDP命令常用的参数有：

- `directory`：存储导出数据的Directory名称；
- `dumpfile`：导出的dump文件命名；
- `filesize`：导出的每个dump文件的最大容量，默认值为0，表示最大限制为16TB；
- `logfile`：导出任务的日志名称，默认为`export.log`；
- `exclude`：用于过滤无需导出的对象，多个对象用逗号分隔；
- `parallel`：导出任务的并行度，默认为1；
- `compression`：是否对导出数据进行压缩，默认为`METADATA_ONLY`。可选值还有`ALL`、`DATA_ONLY`、`NONE`；
- `full`：是否进行全库导出，默认值为**NO**；
- `schemas`：要导出的SCHEMA集合，多个schema用逗号分隔。默认为当前用户schema；
- `tables`：要导出的表集合，多个表用逗号分隔。

不指定`full`、`schemas`和`tables`时，默认导出当前用户Schema。

## 管理用户导出数据
管理用户导出全库数据：
```bash
expdp \'/ as sysdba\' directory=dumpdir1 \
dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp \
logfile=dump_${ORACLE_SID}_`date +%F`.log \
exclude=STATISTICS parallel=2 compression=all full=yes
```

管理用户导出指定Schema的数据：
```bash
expdp \'/ as sysdba\' directory=dumpdir1 \
dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp \
logfile=dump_${ORACLE_SID}_`date +%F`.log \
exclude=STATISTICS,db_link parallel=2 compression=all schemas=MIGUEL
```

## 普通用户导出数据
普通用户导出全库数据（需要有`datapump_exp_full_database`权限）：
```bash
expdp miguel/Xqc\$689 directory=dumpdir1 \
dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp \
logfile=dump_${ORACLE_SID}_`date +%F`.log \
exclude=STATISTICS parallel=2 compression=all full=yes
```

普通用户导出自己的数据（注意对密码中的字符`$`转义）：
```bash
[oracle@oracledb dumpdir]$ expdp miguel/Xqc\$689 directory=dumpdir1 \
> dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp \
> logfile=dump_${ORACLE_SID}_`date +%F`.log \
> exclude=STATISTICS parallel=2 compression=all

Export: Release 19.0.0.0.0 - Production on Sun Jan 15 19:26:43 2023
Version 19.3.0.0.0

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
ORA-31626: job does not exist
ORA-31633: unable to create master table "MIGUEL.SYS_EXPORT_SCHEMA_05"
ORA-06512: at "SYS.DBMS_SYS_ERROR", line 95
ORA-06512: at "SYS.KUPV$FT", line 1163
ORA-01950: no privileges on tablespace 'OMF_TBS1'
ORA-06512: at "SYS.KUPV$FT", line 1056
ORA-06512: at "SYS.KUPV$FT", line 1044
```

如果出现上述报错，需要调整用户在自己默认表空间中的Quota:
```sql
SQL> select default_tablespace from dba_users where username='MIGUEL';
DEFAULT_TABLESPACE
------------------------------
OMF_TBS1

SQL> alter user miguel quota unlimited on omf_tbs1;
User altered.
```
重新执行上面的expdp语句即可。

如果用户名是小写，在终端执行expdp命令时需要采用`'\"<username>\"/<password>'`转义：
```bash
expdp '\"pablo\"/Milf377' directory=dumpdir1 \
dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp \
logfile=dump_${ORACLE_SID}_`date +%F`.log \
exclude=STATISTICS parallel=2 compression=all
```



**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/12.2/sutil/oracle-data-pump-export-utility.html#GUID-5F7380CE-A619-4042-8D13-1F7DDE429991
【2】https://docs.oracle.com/en/database/oracle/oracle-database/12.2/sutil/oracle-data-pump-overview.html#GUID-EEB32B50-8A00-40B0-8787-CC2C8BA05DC5
【3】https://stackoverflow.com/questions/21671008/ora-01950-no-privileges-on-tablespace-users



