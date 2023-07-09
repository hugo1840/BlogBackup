---
tags: [oracle]
title: ORA-00742 DG备库日志文件损坏导致MRP线程中断
created: '2023-07-06T12:45:49.692Z'
modified: '2023-07-09T02:16:00.357Z'
---

ORA-00742 DG备库日志文件损坏导致MRP线程中断


# 报错信息梳理
查看DG备库同步状态，发现归档日志应用已中断：
```bash
Database Error(s): ORA-16766: Redo Apply is stopped
Database Warning(s): ORA-16854: apply lag could not be determined
Database Status: ERROR
```

检查告警日志中相关的报错信息：
```bash
[oracle@orahost ~]$ tail -n 5000 $ORACLE_BASE/diag/rdbms/ORCLDB/ORCLDB/trace/alert_ORCLDB.log | grep -i -B3 -A5 error 

RFS[22]: Assigned to RFS process 275242 
RFS[22]: Opened log for thread 1 sequence 268509 dbid -581712216 branch 946124520 
Sat Jun 24 09:34:09 2023 MRP0: Background Media Recovery terminated with error 742 
Errors in file /oracle/app/diag/rdbms/ORCLDB/ORCLDB/trace/ORCLDB_pr00_6904.trc:
ORA-00742: Log read detects lost write in thread %d sequence %d block %d 
ORA-00312: online log 11 thread 1: '/oradata/fast_recovery_area/ORCLDB/onlinelog/o1_mf_11_l7vmy78h_.log' 
ORA-00312: online log 11 thread 1: '/oradata/ORCLDB/onlinelog/o1_mf_11_l7vmy5vx_.log' 
Managed Standby Recovery not using Real Time Apply 
Archived Log entry 763 added for thread 1 sequence 268509 rlc 946124520 ID 0xe50b0ccc dest 5:
...
```

检查上面提到的Trace文件：
```bash
[oracle@orahost ~]$ less /oracle/app/diag/rdbms/ORCLDB/ORCLDB/trace/ORCLDB_pr00_6904.trc

DDE: Problem Key 'ORA 312' was flood controlled (0x1) (no incident) 
ORA-00312: online log 11 thread 1: '/oradata/fast_recovery_area/ORCLDB/onlinelog/o1_mf_11_l7vmy78h_.log' 
ORA-00312: online log 11 thread 1: '/oradata/ORCLDB/onlinelog/o1_mf_11_l7vmy5vx_.log' 
* 2023-06-24 09:34:09.059 4343 krsh.c 
MRP0: Background Media Recovery terminated with error 742 
ORA-00742: Log read detects lost write in thread %d sequence %d block %d 
ORA-00312: online log 11 thread 1: '/oradata/fast_recovery_area/ORCLDB/onlinelog/o1_mf_11_l7vmy78h_.log' 
ORA-00312: online log 11 thread 1: '/oradata/ORCLDB/onlinelog/o1_mf_11_l7vmy5vx_.log' 
* 2023-06-24 09:34:09.068 4343 krsh.c 
Managed Standby Recovery not using Real Time Apply 
MRP: Prodding archiver at standby for thread 1 seq 268509 
----- Redo read statistics for thread 1 ----- 
Read rate (SYNC): 726114622Kb in 1008195.97s => 0.70 Mb/sec 
Total redo bytes: 726114622Kb 
...
```

查看错误解释： 
```bash
[oracle@orahost ~]$ oerr ora 312 
00312, 00000, "online log %s thread %s: '%s'" 
// *Cause: This message reports the filename for details of another message. 
// *Action: Other messages will accompany this message. See the 
// associated messages for the appropriate action to take. 

[oracle@orahost ~]$ oerr ora 742 
00742, 00000, "Log read detects lost write in thread %d sequence %d block %d" 
// *Cause: Either a write issued by Oracle was lost by the underlying 
// operating system or storage system or an Oracle internal error 
// occurred. 
// *Action: The trace file shows the lost write location. Dump the problematic 
// log file to see whether it is a real lost write. Contact Oracle 
// Support Services.
```

# 解决思路
>:book: **MOS文档**：ORA-00742 Log read detects lost write on standby (Doc ID 2762519.1)
> 
>:one: **现象：**
> MRP Getting Terminated On Standby with ORA-00742 
> Log read detects lost write:
> MRP0: Background Media Recovery terminated with error 742 
>ORA-00742: Log read detects lost write in thread 2 sequence xxxxxxx block xxxxx 
> ORA-00312: online log xxx thread 2: '+DISK_GROUP_NAME/standby.log'
>
>:two: **原因：** 
> Lost write or corruption in the redo.
>
>:three: **措施：** 
> Clear the standby redo logs and restart the MRP OR add new standby redo logs, then drop the old standby redo log group and restart the MRP.
>
> - To clear: 
> SQL> alter database clear logfile group nn;
>
> - To Add/Drop: 
> SQL> alter database add standby logfile thread 1 group nn '<path/filename>' size nnnnM; 
> SQL> alter database drop standby logfile group nn;


# 问题处理
检查日志组编号，并生成清理语句：
```sql
sys@ORCLDB> select group#,thread#,sequence#,status from v$standby_log;

GROUP#    THREAD#  SEQUENCE# STATUS
11        1           0 UNASSIGNED
12        1           269146 ACTIVE
13        1           0 UNASSIGNED
14        1           0 UNASSIGNED
15        1           0 UNASSIGNED
16        1           0 UNASSIGNED
17        1           0 UNASSIGNED
7 rows selected.

sys@ORCLDB> select 'alter database clear logfile group '||group#||';' from v$standby_log;

'ALTERDATABASECLEARLOGFILEGROUP'||GROUP#||';'

alter database clear logfile group 11; 
alter database clear logfile group 12; 
alter database clear logfile group 13; 
alter database clear logfile group 14; 
alter database clear logfile group 15; 
alter database clear logfile group 16; 
alter database clear logfile group 17;

7 rows selected.
```

执行上面的语句清理日志组：
```sql
alter database clear logfile group 11; 
alter database clear logfile group 12; 
alter database clear logfile group 13; 
alter database clear logfile group 14; 
alter database clear logfile group 15; 
alter database clear logfile group 16; 
alter database clear logfile group 17;
```

重启备库MRP进程： 
```sql
SQL> alter database recover managed standby database cancel; 
SQL> alter database recover managed standby database using current logfile disconnect from session;
```

过一两分钟后检查备库同步状态：
```sql
SQL> select process,status,thread#,sequence# from v$managed_standby where status<>'IDLE'; 
SQL> select name,value,unit,time_computed from v$dataguard_stats 
where name in ('transport lag','apply lag');
```

# 可能的额外操作
如果上面重启MRP进程后还是一直没有恢复正常同步，很可能是由于主库和其他备库的归档日志被清理了，导致获取不到部分归档日志，这种情况下还需要额外进行增量备份恢复。

检查问题备库的告警日志：
```bash
5 Thu Jul 06 18:05:16 2023 
FAL[client]: Failed to request gap sequence 
GAP - thread 1 sequence 210386-210485 DBID 4110260087 branch 946578423 
FAL[client]: All defined FAL servers have been attempted.
```
上面这种情况就是归档日志中有部分序列从其他所有库都获取不到了，需要做一次增量备份恢复。




