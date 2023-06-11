---
tags: [OCP]
title: RMAN备份恢复之常用命令
created: '2023-06-11T08:00:28.796Z'
modified: '2023-06-11T09:28:25.006Z'
---

RMAN备份恢复之常用命令

# show
```sql
--显示当前数据库的RMAN备份配置
show all;
```

# report
```sql
--列出当前数据中的数据文件和临时文件
report schema;

--列出需要备份的数据库文件（根据冗余度配置）
report need backup;
report need backup redundancy=2;

--列出由于无法恢复而需要备份的数据文件
report unrecoverable;

--列出多余的备份（根据冗余度配置）
report obsolete;
```

# restore 
用于还原数据库。

# recover
用于恢复数据库。

# backup
用于备份数据库。
```sql
--备份数据库
backup as compressed backupset database
skip offline skip readonly
include current controlfile plus archivelog 
format '/u01/app/oracle/backup/U%' tag='mario' delete input;

--增量备份
backup incremental level 0 database;
backup incremental level 0 database root;
backup incremental level 0 database pdb1;
backup incremental level 0 cumulative database root;
backup incremental level 1 pluggable database pdb1;

--备份其他数据库文件
backup spfile format '/u01/app/oracle/backup/U%';
backup current controlfile;
backup as copy current controlfile;
backup datafile 3 include current controlfile;
backup archivelog all;
backup archivelog sequence between 50 and 80;
backup tablespace pdb1:users,pdb2:users;
```

# list
```sql
--显示所有备份记录
list backup;
list backup summary;
list backup by file;

--显示数据库备份记录
list backup of database;
list backup of database completed after 'sysdate-2';
list backup of pluggable database pdb1,pdb2;

--显示指定数据文件的备份记录
list backup of datafile 1;

--显示归档日志的备份记录
list archivelog all;
list backup of archivelog from sequence 80;
list backup of archivelog from scn 6708889;
list backup of archivelog time between "to_date('22-JUL-2021 00:00:00','DD-MON-YYYY HH24:MI:SS')" and "to_date('27-JUL-2021 00:00:00','DD-MON-YYYY HH24:MI:SS')";

--显示基于image copy的备份记录
list copy;
list copy of tablespace 'SYSTEM';

--显示数据库化身（每次resetlogs后都会生成一个新的incarnation）
list incarnation;
```

# crosscheck
```sql
--校验备份集
crosscheck backup;

--校验镜像拷贝
crosscheck copy;

--校验其他备份文件
crosscheck backup of database;
crosscheck backup of spfile;
crosscheck backup of datafile 2;
crosscheck backup of archivelog sequence 66;

--校验所有归档日志
crosscheck archivelog all;
```

# delete
```sql
--删除已过期的备份集和镜像拷贝
delete expired copy;
delete expired backup;

--删除指定BP Key的备份集
delete backupset 20;

--删除指定的备份片
delete backuppiece '/u01/app/oracle/backup/0j146sfg_1_1';

--删除超过冗余度配置的备份集
delete obsolete;
delete obsolete redundancy 2;

--备份后删除旧的归档日志
backup database plus archivelog delete input;

--清理归档日志
delete archivelog all;
delete archivelog all completed before 'sysdate-7';
```


