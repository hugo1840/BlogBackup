---
tags: [oracle]
title: 使用DG备份恢复测试库的流程以及可能出现的问题
created: '2023-02-14T15:20:52.728Z'
modified: '2023-02-25T14:27:38.248Z'
---

使用DG备份恢复测试库的流程以及可能出现的问题

**思路**：从DG备库备份全库和归档日志，从DG主库备份控制文件，传输到测试库后进行恢复。最好对导出的数据进行脱敏处理。

# 评估数据量和服务器存储空间
评估备库的数据量，并评估备库和测试库用于存放备份的存储空间是否足够：
```bash
du -sh /oradata/TRADE/datafile
df -Th
```

# 从DG备库备份全库和归档日志
查看备库当前SCN：
```sql
SQL> select to_char(timestamp_to_scn(sysdate-0.25/24), '9999999999999999') from dual;

TO_CHAR(TIMESTAMP_TO_SCN(SYSDATE-0
----------------------------------
     950760546489
```

在备库备份全库和归档日志：
```sql
RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    backup as compressed backupset incremental level 0 filesperset 8 database format '/oradata/backup/%d_%T_dbf_%s.bkp';
    backup as compressed backupset archivelog from scn 950760546489 format '/oradata/backup/%d_%T_arc_%s.bkp';
    }

RMAN> backup current controlfile format '/oradata/backup/%d_%T_ctl_%s.bkp';
```
控制文件应该从主库备份，不过这里我们先试试从备库备份控制文件看看效果。

将备份文件传输到测试库：
```bash
cat scpbackup_trade.sh
#!/bin/bash
scp /oradata/backup/TRADE_20230214_dbf_27530.bkp  oracle@{TESTDB_IP}:/oradata/backup/trade/
scp /oradata/backup/TRADE_20230214_dbf_27531.bkp  oracle@{TESTDB_IP}:/oradata/backup/trade/
scp /oradata/backup/TRADE_20230214_dbf_27532.bkp  oracle@{TESTDB_IP}:/oradata/backup/trade/
...

nohup ./scpbackup_trade.sh & 
```

# 清理测试库环境
停止测试库：
```sql
shutdown immediate;
```

清理测试环境的旧数据：
```bash
cd /oradata/TRADE
rm -rf controlfile/
rm -rf datafile/
rm -rf onlinelog/
mkdir controlfile
mkdir datafile
mkdir onlinelog

cd /oradata/fast_recovery_area/
rm -rf TRADE/
mkdir TRADE
```

# 测试库恢复备份（一）
在测试库上，利用传输过来的备份进行数据库恢复：
```sql
rman target /
--恢复控制文件
RMAN> startup nomount;
RMAN> restore controlfile from '/oradata/backup/trade/TRADE_20230214_ctl_7745.bkp';

RMAN> alter database mount;
RMAN> catalog start with '/oradata/backup/trade/@' noprompt;

--恢复数据库
RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    set newname for database to new;        
    restore database;                        
    switch datafile all;                    
    switch tempfile all;
    recover database;                        
    }
```

漫长的恢复过程后，收到以下的报错信息：
```bash
...    
starting media recovery

released channel: c1
released channel: c2
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 02/15/2023 08:18:11
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 56987 and starting SCN of 950850216284 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 56986 and starting SCN of 950848551602 found to restore
```

尝试打开数据库：
```sql
SQL> alter database disable block change tracking; 
SQL> alter database noarchivelog;
SQL> alter system set log_archive_config=NODG_CONFIG sid='*';    
SQL> alter database open resetlogs;

SQL>  alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01666: control file is for a standby database
```
会提示控制文件只能用于备库。

# 从DG主库备份控制文件
控制文件相对于数据和日志文件一般比较小，可以直接从主库备份。
```sql
rman target /
RMAN> backup current controlfile format '/oradata/backup/%d_%T_ctl_%s.bkp';
```

由于从最开始恢复已经过了不少时间，现在主库控制文件里的信息是最新的，所以之后在测试库恢复时，可能还需要重新备份最新的归档日志和数据文件。

# 测试库恢复备份（二）
使用从主库导出的控制文件，重新恢复测试库：
```sql
rman target /
RMAN> startup nomount;
RMAN> restore controlfile from '/oradata/backup/trade/TRADE_20230215_ctl_16997.bkp';

RMAN> alter database mount;
RMAN> catalog start with '/oradata/backup/trade/@' noprompt;

RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    set newname for database to new;        
    restore database;                        
    switch datafile all;                    
    switch tempfile all;
    recover database;                        
    }
```

恢复过程收到下面的报错信息：
```bash
...
starting media recovery

Oracle Error: 
ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/oradata/TRADE/datafile/o1_mf_system_kz0mt4jf_.dbf'

RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 02/16/2023 09:05:19
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 57070 and starting SCN of 951454639987 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 57069 and starting SCN of 951429447138 found to restore
...
...
RMAN-06025: no backup of archived log for thread 1 with sequence 56958 and starting SCN of 950813614718 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 56957 and starting SCN of 950809868702 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 56947 and starting SCN of 950655049884 found to restore
```

如果我们尝试打开数据库，会收到下面的报错：
```sql
idle> alter database disable block change tracking;
idle> alter database noarchivelog;
idle> alter system set log_archive_config=NODG_CONFIG sid='*'; 

idle> alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/oradata/TRADE/datafile/o1_mf_system_kz0mt4jf_.dbf'
```

这里我们需要根据RMAN恢复数据库的报错信息，去备库导出最新的归档日志。

# 从DG备库备份最新的归档日志
从测试库RMAN报错信息中选择缺少的归档日志中最小的SCN（上面是950655049884），在备库备份最新的归档日志，并传输到测试库。
```sql
rman target /
RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    backup as compressed backupset archivelog from scn 950655049884 format '/oradata/backup/%d_%T_arc_%s.bkp';
    }
```

# 测试库恢复备份（三）
在测试库直接回复最新的归档日志（无需重新恢复控制文件和数据文件）。
```sql
RMAN> catalog start with '/oradata/backup/trade/@' noprompt;

RMAN> recover database;
...
...
RMAN-08187: WARNING: media recovery until SCN 951502616026 complete
archived log file name=/oradata/arch/log_TRADE_1_57071_946129689.arc thread=1 sequence=57072
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 02/18/2023 19:40:05
ORA-00283: recovery session canceled due to errors
RMAN-11003: failure during parse/execution of SQL statement: alter database recover logfile '/oradata/arch/log_TRADE_1_57071_956129689.arc'
ORA-00283: recovery session canceled due to errors
ORA-01112: media recovery not started
RMAN-06054: media recovery requesting unknown archived log for thread 1 with sequence 57072 and starting SCN of 951502616026
```

这次RMAN报错信息里提示已经恢复到了`SCN 951502616026`的状态，也并未出现`ORA-01194`和`ORA-01110`之类数据文件不一致的错误。我们尝试打开数据库：
```sql
idle> alter database open resetlogs;

Database altered.
```
数据库打开成功。

# 需要单独备份数据文件的情况
如果备份期间，DG主库生成了新的数据文件，则还需要单独备份对应的数据文件。

查看下面的恢复过程：
```sql
rman target /
RMAN> startup nomount;
RMAN> restore controlfile from '/oradata/backup/asset/ASSET_20230218_ctl_67534.bkp';

RMAN> alter database mount;
RMAN> catalog start with '/oradata/backup/asset/@' noprompt;

RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    set newname for database to new;        
    restore database;                        
    switch datafile all;                    
    switch tempfile all;
    recover database;                        
    }

--报错
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of restore command at 02/18/2023 20:02:42
RMAN-06026: some targets not found - aborting restore
RMAN-06100: no channel to restore a backup or copy of datafile 313
```
这里恢复一开始就直接失败，提示缺少313号数据文件。

我们去DG备库上手动备份313号数据文件：
```sql
RMAN> run {
allocate channel c1 device type disk;
allocate channel c2 device type disk;
backup
    as compressed backupset
    datafile 313 format '/oradata/backup/%d_%T_dbf_%s.bkp';
}
```

传输到测试库后，重新恢复数据库：
```sql
RMAN> catalog start with '/oradata/backup/asset/@' noprompt;

RMAN> run {
    allocate channel c1 device type disk;
    allocate channel c2 device type disk;
    set newname for database to new;        
    restore database;                        
    switch datafile all;                    
    switch tempfile all;
    recover database;                        
    }
```
之后可能遇到的报错都可以参考前面的内容。


