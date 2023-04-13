---
tags: [oracle]
title: IMPDP导入用户数据到不同的Schema
created: '2023-04-13T11:37:42.807Z'
modified: '2023-04-13T12:34:45.717Z'
---

IMPDP导入用户数据到不同的Schema

回顾一下使用EXPDP和IMPDP导入导出用户数据的方法。

检查数据泵路径：
```sql
SQL> select owner,directory_name,directory_path from dba_directories;
```

导出指定用户appuser1的数据：
```bash
expdp \'/ as sysdba\' directory=dumpdir \
dumpfile=dump_${ORACLE_SID}_`date +%F`_%U.dmp \
logfile=dump_${ORACLE_SID}_`date +%F`.log \
exclude=STATISTICS,db_link parallel=2 compression=all schemas=APPUSER1
```

# IMPDP导入数据到原有用户Schema
导入指定用户appuser1的数据：
```bash
impdp '/ as sysdba' directory=dumpdir \
dumpfile=dumpfile_01.dmp,dumpfile_02.dmp,dumpfile_03.dmp \ 
parallel=2 schemas=APPUSER1
```

# IMPDP导入数据到不同用户Schema
将导出的用户appuser1的数据，导入到用户appuser2的Schema中：
```bash
impdp '/ as sysdba' directory=dumpdir \
dumpfile=dumpfile_01.dmp,dumpfile_02.dmp,dumpfile_03.dmp \ 
parallel=2 remap_schema=APPUSER1:APPUSER2 \
remap_tablespace=APPTBS1:APPTBS2
```
其中，APPTBS1是用户appuser1的默认表空间，APPTBS2是用户appuser2的默认表空间。





