---
tags: [oracle]
title: Oracle DG复制中断：RMAN-03015和RMAN-06094
created: '2023-08-20T13:34:07.761Z'
modified: '2023-08-21T12:25:01.227Z'
---

Oracle DG复制中断：RMAN-03015和RMAN-06094

>:star:**DB_UNIQUE_NAME**
> - 主库：ORCLDB_0 
> - 备库：ORCLDB_1

# 背景与错误信息
通过RMAN DUPLICATE搭建DG备库，一段时间后收到RMAN-03015和RMAN-06094:
```bash
rman target sys/oracle@${ORACLE_SID}_0 auxiliary sys/oracle@${ORACLE_SID}_1
RMAN> run {
allocate channel c1 device type disk;
allocate channel c2 device type disk;
allocate auxiliary channel aux1 device type disk;
allocate auxiliary channel aux2 device type disk;
duplicate target database for standby
    from active database
    dorecover
    nofilenamecheck;
}

...
RMAN-03002: failure of Duplicate Db command at 08/13/2023 06:22:28
RMAN-05501: aborting duplication of target database
RMAN-03015: error occurred in stored script Memory Script
RMAN-06094: datafile 499 must be restored
```

检查报错信息：
```bash
[oracle@dbhost ~]$ oerr rman 6094
6094, 1, "datafile %s must be restored"
// *Cause: A RECOVER command was issued, and the recovery catalog indicates
//         the specified datafile should be part of the recovery, but
//         this datafile is not listed in the control file, and cannot be
//         found on disk.
// *Action: Issue a RESTORE command for this datafile, using the same
//          UNTIL clause specified to the RECOVER command (if any), then
//          reissue the RECOVER.
```
初步判断复制中断原因是主库太大（数据量超过30TB），RMAN Duplicate同步时间太长（接近一周），中间生成了新的数据文件，并且新生成的数据文件没有记录路在控制文件中。

# 主库备份数据文件
查看主库新增数据文件：
```bash
RMAN> report schema;
...
499  65500    TS_BASE              ***     /oradata/ORCLDB_0/datafile/o1_mf_ts_base_lf6qq695_.dbf
500  65500    TS_BASE              ***     /oradata/ORCLDB_0/datafile/o1_mf_ts_base_lf6qq696_.dbf
```

在主库备份新增的数据文件：
```sql
RMAN> run {
allocate channel c1 device type disk;
allocate channel c2 device type disk;
backup
    as compressed backupset
    datafile 499,500 format '/oradata/backup/%d_%T_dbf_%s.bkp';
}

RMAN> backup current controlfile for standby format '/oradata/backup/%d_%T_ctl_%s.bkp';
```
**注意**：备份控制文件一定要加`for standby`，否则备份出来的是主库控制文件！！！

# 备库恢复数据文件
错误的恢复方法：
```bash
rman target sys/oracle@${ORACLE_SID}_0 auxiliary sys/oracle@${ORACLE_SID}_1

#--恢复数据文件
...
```
以辅助数据库方式连接备库，恢复操作会在主库进行！

正确的备库恢复如下。

检查备库状态：
```sql
sys@ORCLDB_1> show parameter pfile

NAME                     TYPE     VALUE
------------------------------------ ----------- ------------------------------
spfile                   string     /oracle/app/product/11204/dbs/spfileORCLDB.ora

sys@ORCLDB_1> select db_unique_name,open_mode,database_role,switchover_status from v$database;

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    SWITCHOVER_STATUS
------------------------------ -------------------- ---------------- --------------------
ORCLDB_1                       MOUNTED              PHYSICAL STANDBY RECOVERY NEEDED
```

启动备库到NOMOUNT状态，依次恢复最新的控制文件和数据文件：
```sql
SQL> shutdown immediate;

rman target /
RMAN> startup nomount;

--恢复控制文件
RMAN> restore controlfile from '/oradata/backup/ORCLDB_20230814_ctl_9998.bkp';

RMAN> alter database mount;
RMAN> catalog start with '/oradata/backup/' noprompt; 

RMAN> report schema;
...
498  0        UNDOTBS              ***     /oradata/ORCLDB_0/datafile/o1_mf_undotbs_lf2rnobr_.dbf
499  0        TS_BASE              ***     /oradata/ORCLDB_0/datafile/o1_mf_ts_base_lf6qq695_.dbf
500  0        TS_BASE              ***     /oradata/ORCLDB_0/datafile/o1_mf_ts_base_lf6qq696_.dbf
--> 这里注意到文件名还是主库的文件名，此处埋个伏笔

--恢复数据文件
run {
allocate channel c1 device type disk;
allocate channel c2 device type disk;
set newname for datafile 499 to new;
set newname for datafile 500 to new;
restore datafile 499,500;
}

--恢复数据库
run {
allocate channel c1 device type disk;
allocate channel c2 device type disk;
recover database; 
}

--暂时忽略以下报错信息
...
RMAN-03002: failure of recover command at 08/14/2023 10:21:19
RMAN-06094: datafile 1 must be restored
```

# DG Broker创建DG
通过DG Broker创建DG配置：
```bash
dgmgrl sys/oracle@${ORACLE_SID}_0 "create configuration dg_${ORACLE_SID} as primary database is ${ORACLE_SID}_0 connect identifier is ${ORACLE_SID}_0";
dgmgrl sys/oracle@${ORACLE_SID}_0 "add database ${ORACLE_SID}_1 as connect identifier is ${ORACLE_SID}_1";
dgmgrl sys/oracle@${ORACLE_SID}_0 "enable configuration";
dgmgrl sys/oracle@${ORACLE_SID}_0 "show configuration";
```

发现备库MRP进程中断，检查备库告警日志：
```bash
[oracle@dbhost ~]$ tail -f /oracle/app/diag/rdbms/ORCLDB_1/ORCLDB/trace/alert_ORCLDB.log
...
Errors in file /oracle/app/diag/rdbms/ORCLDB_1/ORCLDB/trace/ORCLDB_dbw0_269929.trc:
ORA-01157: cannot identify/lock data file 500 - see DBWR trace file
ORA-01110: data file 500: '/oradata/ORCLDB_0/datafile/o1_mf_ts_base_lf6qq696_.dbf'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
MRP0: Background Media Recovery terminated with error 1110
Errors in file /oracle/app/diag/rdbms/ORCLDB_1/ORCLDB/trace/ORCLDB_pr00_274426.trc:
ORA-01110: data file 1: '/oradata/ORCLDB_0/datafile/o1_mf_system_9av350t0_.dbf'
ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
ORA-01110: data file 1: '/oradata/ORCLDB_0/datafile/o1_mf_system_9av350t0_.dbf'

[oracle@dbhost ~]$ oerr ora 1110
01110, 00000, "data file %s: '%s'"
// *Cause:  Reporting file name for details of another error. The reported
//          name can be of the old file if a data file move operation is
//          in progress.
// *Action: See associated error message.
```
这里发现备库告警日志中显示的数据文件名对应的是是主库数据目录（ORCLDB_0）。

通过DG Broker移除备库：
```bash
dgmgrl sys/oracle@${ORACLE_SID}_0 "remove database ORCLDB_1";
```

# 更新备库数据文件名
更新备库控制文件中的数据文件名称：
```sql
--切换到备库数据目录
RMAN> catalog start with '/oradata/ORCLDB_1/datafile' noprompt; 
RMAN> switch database to copy;

...
datafile 499 switched to datafile copy "/oradata/ORCLDB_1/datafile/o1_mf_ts_base_lfm2lq6z_.dbf"
datafile 500 switched to datafile copy "/oradata/ORCLDB_1/datafile/o1_mf_ts_base_lfm2lq71_.dbf"

--检查数据文件名
RMAN> report schema;

--恢复数据库
RMAN> run {
allocate channel c1 device type disk;
allocate channel c2 device type disk;
recover database; 
}

...
unable to find archived log
archived log thread=1 sequence=2542
released channel: c1
released channel: c2
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 08/14/2023 13:33:53
RMAN-06054: media recovery requesting unknown archived log for thread 1 with sequence 2542 and starting SCN of 1017335704413
```

再次添加备库到DG：
```bash
dgmgrl sys/oracle@${ORACLE_SID}_0 "add database ${ORACLE_SID}_1 as connect identifier is ${ORACLE_SID}_1";
dgmgrl sys/oracle@${ORACLE_SID}_0 "enable configuration";
dgmgrl sys/oracle@${ORACLE_SID}_0 "show configuration";

[oracle@dbhost ~]$ dgmgrl sys/oracle@${ORACLE_SID}_0 "show database ORCLDB_1";
DGMGRL for Linux: Version 11.2.0.4.0 - 64bit Production

Database - ORCLDB_1

  Role:            PHYSICAL STANDBY
  Intended State:  APPLY-ON
  Transport Lag:   (unknown)
  Apply Lag:       1 day(s) 17 hours 17 minutes 9 seconds (computed 26 seconds ago)
  Apply Rate:      (unknown)
  Real Time Query: OFF
  Instance(s):
    ORCLDB

Database Status:
SUCCESS
```

# 解决归档日志不传输
上面DGMGRL命令查看到的备库的Apply Lag超过一天，而且Transport Lag未知。

检查备库发现归档日志不传输：
```sql
sys@ORCLDB_1> select process,status,thread#,sequence#,block# from v$managed_standby where status<>'IDLE';

PROCESS   STATUS      THREAD#  SEQUENCE#     BLOCK#
--------- ------------ ---------- ---------- ----------
ARCH      CONNECTED        0       0          0
ARCH      CONNECTED        0       0          0
ARCH      CONNECTED        0       0          0
ARCH      CONNECTED        0       0          0
MRP0      WAIT_FOR_GAP     1    2542          0

sys@ORCLDB_1> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                   VALUE                  UNIT                 TIME_COMPUTED
------------------------------ ------------------------------ ------------------------------ ------------------------------
transport lag                              day(2) to second(0) interval   08/14/2023 13:40:06
apply lag               +01 17:17:09       day(2) to second(0) interval   08/14/2023 13:40:06
```

检查备库告警日志：
```bash
[oracle@dbhost ~]$ tail -f /oracle/app/diag/rdbms/ORCLDB_1/ORCLDB/trace/alert_ORCLDB.log
...
Primary database is in MAXIMUM PERFORMANCE mode
RFS[8]: Assigned to RFS process 319930
RFS[8]: No standby redo logfiles available for thread 1 
Errors in file /oracle/app/diag/rdbms/ORCLDB_1/ORCLDB/trace/ORCLDB_rfs_319930.trc:

ORA-19815: WARNING: db_recovery_file_dest_size of 214748364800 bytes is 100.00% used, and has 0 remaining bytes available.
************************************************************************
You have following choices to free up space from recovery area:
1. Consider changing RMAN RETENTION POLICY. If you are using Data Guard,
   then consider changing RMAN ARCHIVELOG DELETION POLICY.
2. Back up files to tertiary device such as tape using RMAN
   BACKUP RECOVERY AREA command.
3. Add disk space and increase db_recovery_file_dest_size parameter to
   reflect the new space.
4. Delete unnecessary files using RMAN DELETE command. If an operating
   system command was used to delete files, then use RMAN CROSSCHECK and
   DELETE EXPIRED commands.
```

发现快速恢复区满了，原因是归档路径不知道为啥在快速回复区（200G肯定不够用啊）。
```sql
sys@ORCLDB_1> show parameter log_archive

NAME                     TYPE     VALUE
------------------------------------ ----------- ------------------------------
log_archive_config       string     dg_config=(ORCLDB_1,ORCLDB_0)
log_archive_dest         string
log_archive_dest_1       string     location=USE_DB_RECOVERY_FILE_DEST, valid_for=(ALL_LOGFILES,ALL_ROLES)
```

扩容快速回复区：
```sql
sys@ORCLDB_1> show parameter db_recovery_file_dest

NAME                              TYPE         VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest             string       /oradata/fast_recovery_area
db_recovery_file_dest_size        big integer  200G

sys@ORCLDB_1> alter system set db_recovery_file_dest_size=300G;

System altered.
```

检查并修改备库本地归档路径为不限制容量的目录：
```sql
sys@ORCLDB_1> alter system set log_archive_dest_1='LOCATION=/oradata/arch' scope=both;

System altered.
```

检查并修改DG Broker中的本地归档日志路径配置：
```bash
[oracle@dbhost ~]$ dgmgrl sys/oracle@${ORACLE_SID}_0 "show database ORCLDB_1";
DGMGRL for Linux: Version 11.2.0.4.0 - 64bit Production

Database - ORCLDB_1

  Role:            PHYSICAL STANDBY
  Intended State:  APPLY-ON
  Transport Lag:   (unknown)
  Apply Lag:       1 day(s) 17 hours 27 minutes 21 seconds (computed 165 seconds ago)
  Apply Rate:      (unknown)
  Real Time Query: OFF
  Instance(s):
    ORCLDB
      Warning: ORA-16714: the value of property StandbyArchiveLocation is inconsistent with the database setting
      Warning: ORA-16714: the value of property AlternateLocation is inconsistent with the database setting

[oracle@dbhost ~]$ dgmgrl sys/oracle@${ORACLE_SID}_0 "show database verbose ORCLDB_1";
...
StandbyArchiveLocation          = 'USE_DB_RECOVERY_FILE_DEST'
AlternateLocation               = ''

#修改DG Broker中记录的备库本地归档位置
[oracle@dbhost ~]$ dgmgrl sys/oracle@${ORACLE_SID}_0 "edit database ${ORACLE_SID}_1 set property StandbyArchiveLocation='/oradata/arch'";
```

重启备库：
```sql
sys@ORCLDB_1> shutdown abort;
ORACLE instance shut down.

sys@ORCLDB_1> startup mount;

ORACLE instance started.
Database mounted.

sys@ORCLDB_1> select db_unique_name,open_mode,database_role,switchover_status from v$database;
sys@ORCLDB_1> select process,status,thread#,sequence#,block# from v$managed_standby where process <> 'IDLE';

DB_UNIQUE_NAME         OPEN_MODE        DATABASE_ROLE     SWITCHOVER_STATUS
------------------------------ -------------------- ---------------- --------------------
ORCLDB_1               MOUNTED          PHYSICAL STANDBY  RECOVERY NEEDED

sys@ORCLDB_1> 
PROCESS   STATUS      THREAD#  SEQUENCE#     BLOCK#
--------- ------------ ---------- ---------- ----------
ARCH      CONNECTED        0       0          0
ARCH      CONNECTED        0       0          0
ARCH      CONNECTED        0       0          0
ARCH      CONNECTED        0       0          0
RFS       RECEIVING        1    4937      88065
RFS       RECEIVING        1    4935     374785
RFS       RECEIVING        1    4932     378881
RFS       IDLE             1    4945         31

8 rows selected.
```
归档日志恢复传输。

验证主备库传输状态：
```bash
[oracle@dbhost ~]$ dgmgrl sys/oracle@${ORACLE_SID}_1 "show database ORCLDB_0"
Connected.

Database - ORCLDB_0

  Role:            PRIMARY
  Intended State:  TRANSPORT-ON  #--> 开启对所有备库的日志传输
  Instance(s):
    ORCLDB

Database Status:
SUCCESS

[oracle@dbhost ~]$ dgmgrl sys/oracle@${ORACLE_SID}_1 "show database ORCLDB_1 logshipping"
Connected.

  LogShipping = 'ON'   #--> 开启接收归档日志
```





