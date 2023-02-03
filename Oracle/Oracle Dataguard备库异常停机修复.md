---
tags: [oracle]
title: Oracle Dataguard备库异常停机修复
created: '2023-02-02T11:39:48.016Z'
modified: '2023-02-03T11:57:26.894Z'
---

Oracle Dataguard备库异常停机修复

由于异常关机，导致DG备库在启动时遇到以下报错：
```sql
SQL> startup nomount;
SQL> alter database mount standby database;
SQL> alter database open;  
alter database open
*
ERROR at line 1:
ORA-10458: standby database requires recovery
ORA-01196: file 1 is inconsistent due to a failed media recovery session
ORA-01110: data file 1:
'/oradata/BANGKOKDG/datafile/o1_mf_system_kxd67djb_.dbf'
```

下面介绍利用RMAN备份来对备库进行修复的方法。

# 检查备库SCN
```sql
SQL> select to_char(current_scn) from v$database;

TO_CHAR(CURRENT_SCN)
----------------------------------------
1464788
```

# 创建主库RMAN备份
在主库上基于SCN做一次RMAN备份：
```bash
RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    backup as compressed backupset incremental level 0 database format '/oradata/backup/%d_%T_dbf_%s.bkp';
    backup as compressed backupset archivelog from scn 1464788 format '/oradata/backup/%d_%T_arc_%s.bkp';
    }
```

在主库上创建备库控制文件：
```sql
alter database create standby controlfile as '/oradata/backup/bangkok_4standby_ctl_0201.bkp';
```

将备份文件拷贝到备库：
```bash
scp /oradata/backup/BANGKOK_20230201_arc_2* oracle@172.x.x.x:/oradata/backup/
scp /oradata/backup/BANGKOK_20230201_dbf_* oracle@172.x.x.x:/oradata/backup/
scp /oradata/backup/bangkok_4standby_ctl_0201.bkp oracle@172.x.x.x:/oradata/backup/
```

# 利用备份恢复备库
重启备库：
```sql
shutdown immediate;
startup nomount;
```

恢复备库控制文件，并挂载备库：
```bash
RMAN> restore controlfile from '/oradata/backup/bangkok_4standby_ctl_0201.bkp';
RMAN> sql 'alter database mount';
```

利用拷贝过来的备份恢复备库：
```bash
RMAN> catalog start with '/oradata/backup/' noprompt;
RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    set newname for database to new;        
    restore database;                        
    switch datafile all;                    
    switch tempfile all;
    recover database noredo;                        
    }
```
需要根据Oracle文件管理模式修改上面的语句（示例中为OMF模式）。

检查恢复后的数据文件与操作系统中是否一致：
```bash
RMAN> report schema;

RMAN-06139: warning: control file is not current for REPORT SCHEMA
Report of database schema for database with db_unique_name BANGKOKDG

List of Permanent Datafiles
===========================
File Size(MB) Tablespace           RB segs Datafile Name
---- -------- -------------------- ------- ------------------------
1    890      SYSTEM               ***     /oradata/BANGKOKDG/datafile/o1_mf_system_kxmjjs3g_.dbf
2    580      SYSAUX               ***     /oradata/BANGKOKDG/datafile/o1_mf_sysaux_kxmjjs4b_.dbf
3    1010     UNDOTBS1             ***     /oradata/BANGKOKDG/datafile/o1_mf_undotbs1_kxmjjs39_.dbf
4    5        USERS                ***     /oradata/BANGKOKDG/datafile/o1_mf_users_kxmjjs4q_.dbf
5    100      OMF_TBS1             ***     /oradata/BANGKOKDG/datafile/o1_mf_omf_tbs1_kxmjjs4b_.dbf
6    100      omf_tbs2             ***     /oradata/BANGKOKDG/datafile/o1_mf_omf_tbs2_kxmjjs4q_.dbf

List of Temporary Files
=======================
File Size(MB) Tablespace           Maxsize(MB) Tempfile Name
---- -------- -------------------- ----------- --------------------
1    20       TEMP                 32767       /oradata/BANGKOKDG/datafile/o1_mf_temp_%u_.tmp
```

备库开启日志应用进程：
```sql
SQL> alter database recover managed standby database using current logfile disconnect;
SQL> select process,status,sequence#,thread# from v$managed_standby where process='MRP0';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
MRP0      WAIT_FOR_GAP          9          1
```

检查备库延迟：
```sql
SQL> set lines 200
SQL> col value for a30
SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                             VALUE                          UNIT                           TIME_COMPUTED
-------------------------------- ------------------------------ ------------------------------ ------------------------------
transport lag                    +00 00:00:00                   day(2) to second(0) interval   02/01/2023 09:38:31
apply lag                        +00 00:00:00                   day(2) to second(0) interval   02/01/2023 09:38:31

SQL> select process,status,sequence#,thread# from v$managed_standby where process='MRP0';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
MRP0      APPLYING_LOG         10          1
```

等待`transport lag`和`apply lag`的VALUE值都变为0后，再打开备库：
```sql
SQL> alter database recover managed standby database cancel;

SQL> alter database open;

SQL> alter database recover managed standby database using current logfile disconnect from session;
```

# 检查DG同步状态
检查备库状态：
```sql
SQL> select database_role,protection_mode,protection_level,open_mode from v$database;

DATABASE_ROLE    PROTECTION_MODE      PROTECTION_LEVEL     OPEN_MODE
---------------- -------------------- -------------------- --------------------
PHYSICAL STANDBY MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE  READ ONLY WITH APPLY

SQL> select a.inst_id,a.db_unique_name,a.database_role,a.protection_level,a.protection_mode,a.open_mode,a.switchover_status,
b.host_name,b.thread# from gv$database a left join gv$instance b on a.inst_id=b.inst_id order by a.inst_id;   

   INST_ID DB_UNIQUE_NAME  DATABASE_ROLE    PROTECTION_LEVEL     PROTECTION_MODE      OPEN_MODE            SWITCHOVER_STATUS    HOST_NAME          THREAD#
---------- --------------- ---------------- -------------------- -------------------- -------------------- -------------------- --------------- ----------
         1 bangkokdg       PHYSICAL STANDBY MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE  READ ONLY WITH APPLY NOT ALLOWED          standbydb                1
--> NOT ALLOWED在备库表示未收到主库发来的切换请求
```

检查主库状态：
```sql
set lines 220
col host_name for a15
col db_unique_name for a15
col switchover_status for a20
SQL> select a.inst_id,a.db_unique_name,a.database_role,a.protection_level,a.protection_mode,a.open_mode,a.switchover_status,
b.host_name,b.thread# from gv$database a left join gv$instance b on a.inst_id=b.inst_id order by a.inst_id;    

   INST_ID DB_UNIQUE_NAME  DATABASE_ROLE    PROTECTION_LEVEL     PROTECTION_MODE      OPEN_MODE            SWITCHOVER_STATUS    HOST_NAME          THREAD#
---------- --------------- ---------------- -------------------- -------------------- -------------------- -------------------- --------------- ----------
         1 bangkok         PRIMARY          MAXIMUM PERFORMANCE  MAXIMUM PERFORMANCE  READ WRITE           TO STANDBY           primarydb                
--> TO STANDBY在主库表示可以切换为备库
```



