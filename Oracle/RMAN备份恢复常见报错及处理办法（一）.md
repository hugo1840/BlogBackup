---
tags: [oracle]
title: RMAN备份恢复常见报错及处理办法（一）
created: '2023-03-15T14:22:10.782Z'
modified: '2023-03-16T10:38:35.635Z'
---

RMAN备份恢复常见报错及处理办法（一）

# ORA-19809和ORA-19804
不完全恢复后，打开数据库时时收到下面的报错：
```sql
alter database open resetlogs;
*
ERROR at line 1:
ORA-19809: limit exceeded for recovery files
ORA-19804: cannot reclaim 4294967296 bytes disk space from 53687091200 limit
```

解决办法：调大恢复区的大小。
```sql
SQL> archive log list;
SQL> show parameter db_recover
SQL> alter system set db_recovery_file_dest_size=300G scope=both;
```

# ORA-00059
RMAN中挂载数据库时收到下面的报错信息：
```bash
alter database mount;
*
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of alter db command at 03/10/2023 16:32:03
ORA-00059: maximum number of DB_FILES exceeded
```

解决办法：调大`db_files`参数，并重新挂载。
```sql
SQL> show parameter db_files
SQL> alter system set db_files=2048 scope=spfile;

System altered.

SQL> shutdown immediate;
```

# RMAN-06820和ORA-17629
使用RMAN备份数据库时，收到下面的报错：
```bash
RMAN-06820: WARNING: failed to archive current log at primary database
ORACLE error from target database: 
ORA-17629: Cannot connect to the remote database server
ORA-17627: ORA-00942: table or view does not exist
```

解决办法：使用`rman target sys/XXXX`方式登录，而不是`rman target /`。

# RMAN-11003和ORA-01143
恢复数据库时收到报错RMAN-11003，并且尝试关闭归档模式时会报错ORA-01143和ORA-01110。
```sql
RMAN-03002: failure of recover command at 03/11/2023 08:53:37
RMAN-11003: failure during parse/execution of SQL statement: alter database recover logfile '/oradata/arch/log_<db_name>_1_264356_989478510.arc'
ORA-10877: error signaled in parallel recovery slave

SQL> alter database noarchivelog
*
ERROR at line 1:
ORA-01143: cannot disable media recovery - file 317 needs media recovery
ORA-01111: name for data file 317 is unknown - rename to correct file
ORA-01110: data file 317: '/oracle/app/product/11204/dbs/UNNAMED00317'
```

解决办法：备份对应的数据文件（这里是datafile 317），然后重新进行恢复。
```bash
RMAN> restore datafile 317;

RMAN-03002: failure of restore command at 03/11/2023 10:18:30
RMAN-06085: must use SET NEWNAME command to restore datafile /oracle/app/product/11204/dbs/UNNAMED00317

RMAN> run {
       set newname for datafile 317 to new;        
       restore datafile 317;                        
       switch datafile 317; 
      }
  
RMAN> recover database;
```
如果还是不行，可以尝试去重新备份数据库文件进行恢复。


# ORA-01547、ORA-01194和RMAN-06025
恢复数据库时，Restore完成后，开始Recover时收到下面的报错信息：
```bash
Oracle Error: 
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/oradata/<db_unique_name>/datafile/o1_mf_system_xxxxx_.dbf'

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 03/16/2023 13:49:02
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 20620 and starting SCN of 959994799549 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 20619 and starting SCN of 959994645797 found to restore
```

解决办法：去备库备份缺少的归档日志，从SCN最小的时间点开始备份，然后继续恢复。
```sql
--备份
backup as compressed backupset archivelog from scn 959994645797 format '/oradata/backup/<db_unique_name>/%d_%T_arc_%s.bkp';

--恢复
catalog start with '/oradata/backup/<db_unique_name>/' noprompt;
recover database;
```

# ORA-01547、ORA-01194和RMAN-06102
恢复数据库时，Restore完成后，开始Recover时收到下面的报错信息：
```bash
Oracle Error: 
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/oradata/<db_unique_name>/datafile/o1_mf_system_xxxxx_.dbf'

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 03/16/2023 14:17:55
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06102: no channel to restore a backup or copy of archived log for thread 1 with sequence 20612 and starting SCN of 959865083425
RMAN-06102: no channel to restore a backup or copy of archived log for thread 1 with sequence 20610 and starting SCN of 959672485437
```

解决办法：RMAN-06102一般是由于备库缺少部分归档日志（可能已被清理），需要去别的地方备份缺少的归档日志，或者重新备份数据库（如果数据量整体不大），然后重新恢复数据库。




