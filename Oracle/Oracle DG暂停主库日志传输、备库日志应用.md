---
tags: [oracle]
title: Oracle DG暂停主库日志传输、备库日志应用
created: '2023-02-25T14:38:46.254Z'
modified: '2023-02-26T07:42:41.511Z'
---

Oracle DG暂停主库日志传输、备库日志应用

# 查看主备库DG相关进程
主备库日志同步为异步模式：
```sql
SQL> show parameter log_archive_dest_2

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_dest_2                   string      SERVICE=bangkokdg LGWR ASYNC V
                                                 ALID_FOR=(ONLINE_LOGFILES,PRIM
                                                 ARY_ROLE) DB_UNIQUE_NAME=bangk
                                                 okdg
```

查看主库上与DG相关的进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'LGWR';

no rows selected

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'LNS%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
LNS       WRITING              18          1

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'ARC%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
ARCH      CLOSING              17          1
ARCH      CONNECTED             0          0
ARCH      CLOSING              17          1
ARCH      CONNECTED             0          0

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'FAL%';

no rows selected
```

异步模式下，DG主库上的**LNS**进程负责把LGWR进程写好的联机重做日志传输到备库的**RFS**进程。

查看物理备库上与DG相关的进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'RFS%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
RFS       IDLE                  0          1
RFS       IDLE                  0          0
RFS       IDLE                 18          1

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'ARC%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
ARCH      CONNECTED             0          0
ARCH      CONNECTED             0          0
ARCH      CLOSING              17          1
ARCH      CONNECTED             0          0

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'MRP%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
MRP0      APPLYING_LOG         18          1
```

DG物理备库上，**RFS**进程负责接收主库传输过来的日志，并将其写入备库重做日志（Standby Redo Log）；**ARCH**进程负责对备库重做日志进行归档（Archived Redo Log）；**MRP**进程负责应用归档重做日志，将其恢复到备库。

# DG日志传输：`log_archive_dest_state_2`
## DG停止主库日志传输
查看主库日志传输状态：
```sql
SQL> show parameter log_archive_dest_state_2

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_dest_state_2             string      ENABLE
```

查看备库的延迟：
```sql
SQL> select * from v$archive_gap;

no rows selected

SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                             VALUE                UNIT                           TIME_COMPUTED
-------------------------------- -------------------- ------------------------------ ------------------------------
transport lag                    +00 00:00:00         day(2) to second(0) interval   02/26/2023 14:33:14
apply lag                        +00 00:00:00         day(2) to second(0) interval   02/26/2023 14:33:14
```
可以看到，transport lag和apply lag都为0。

在主库暂停对备库的日志传输：
```sql
SQL> alter system set log_archive_dest_state_2='defer';

System altered.

SQL> alter system switch logfile;

System altered.
```

查看主库上的LNS进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'LNS%';

no rows selected
```
可以看到，负责向备库传输日志的LNS进程没了。

查看备库上的RFS进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'RFS%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
RFS       IDLE                  0          0
```

主库不做任何更新和插入的情况下，查看备库延迟：
```sql
SQL> select * from v$archive_gap;

no rows selected

SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                             VALUE                UNIT                           TIME_COMPUTED
-------------------------------- -------------------- ------------------------------ ------------------------------
transport lag                    +00 00:00:00         day(2) to second(0) interval   02/26/2023 14:43:59
apply lag                        +00 00:00:00         day(2) to second(0) interval   02/26/2023 14:43:59
```
备库延迟信息没有变化。

在主库的应用库插入数据：
```sql
SQL> select * from miguel.rdr_npc;

 NPC_ID FIRST_NAME           LAST_NAME                   AGE MESSAGE
---------- -------------------- -------------------- ---------- ----------------------------------------
         1 Arthur               Morgan                       35 He's a good man.
         2 John                 Maston                       25 Revenge is a stupid game.
         3 Dutch                Vanderlin                    45 I have a god damn plan!
```

再次查看备库的延迟：
```sql
SQL> select * from v$archive_gap;

no rows selected

SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                                     VALUE                UNIT                           TIME_COMPUTED
---------------------------------------- -------------------- ------------------------------ ------------------------------
transport lag                            +00 00:00:00         day(2) to second(0) interval   02/26/2023 15:01:57
apply lag                                +00 00:00:00         day(2) to second(0) interval   02/26/2023 15:01:57

SQL> select * from miguel.rdr_npc;
select * from miguel.rdr_npc
                     *
ERROR at line 1:
ORA-00942: table or view does not exist
```
延迟依然没有变化。但是也查不到主库上新创建的表。

查看备库的MRP进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'MRP%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
MRP0      WAIT_FOR_LOG         19          1
```
显示在等待编号为19的日志序列（Log Sequence Number）。

查看主库日志归档情况：
```sql
SQL> select group#, thread#, sequence#, archived, status from v$log;

    GROUP#    THREAD#  SEQUENCE# ARC STATUS
---------- ---------- ---------- --- ----------------
         1          1         17 YES INACTIVE
         2          1         18 YES INACTIVE
         3          1         19 NO  CURRENT
```
显示编号为19的日志序列还未被归档。

## DG恢复主库日志传输
在主库重新恢复对物理备库的传输：
```sql
SQL> alter system set log_archive_dest_state_2='enable';

System altered.

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'LNS%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
LNS       CONNECTED             0          0
LNS       WRITING              19          1
```

查看备库上的进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'RFS%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
RFS       IDLE                  0          1
RFS       IDLE                  0          0
RFS       IDLE                 19          1

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'MRP%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
MRP0      APPLYING_LOG         19          1

SQL> select * from miguel.rdr_npc;

    NPC_ID FIRST_NAME           LAST_NAME                   AGE MESSAGE
---------- -------------------- -------------------- ---------- ----------------------------------------
         1 Arthur               Morgan                       35 He's a good man.
         2 John                 Maston                       25 Revenge is a stupid game.
         3 Dutch                Vanderlin                    45 I have a god damn plan!
```

# DG日志应用：MRP进程
## DG停止备库日志应用
在主库保持活跃的同时，停止备库的日志应用：
```sql
SQL> alter database recover managed standby database cancel;

Database altered.
```

查看备库上的DG进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'RFS%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
RFS       IDLE                  0          1
RFS       IDLE                  0          0
RFS       IDLE                 19          1

SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'MRP%';

no rows selected
```
可以看到，负责应用归档重做日志的MRP0进程没了。

查看备库延迟：
```sql
SQL> select * from v$archive_gap;

no rows selected

SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                                     VALUE                UNIT                           TIME_COMPUTED
---------------------------------------- -------------------- ------------------------------ ------------------------------
transport lag                            +00 00:00:00         day(2) to second(0) interval   02/26/2023 15:22:59
apply lag                                +00 00:02:40         day(2) to second(0) interval   02/26/2023 15:22:59

SQL> /

NAME                                     VALUE                UNIT                           TIME_COMPUTED
---------------------------------------- -------------------- ------------------------------ ------------------------------
transport lag                            +00 00:00:00         day(2) to second(0) interval   02/26/2023 15:23:18
apply lag                                +00 00:02:58         day(2) to second(0) interval   02/26/2023 15:23:18
```
可以看到，transport lag一直为0，但是apply lag随着时间一直在增长。


## DG恢复备库日志应用
恢复备库的日志应用：
```sql
SQL> alter database recover managed standby database using current logfile disconnect from session;

Database altered.
```

检查MRP进程：
```sql
SQL> select process,status,sequence#,thread# from v$managed_standby where process like 'MRP%';

PROCESS   STATUS        SEQUENCE#    THREAD#
--------- ------------ ---------- ----------
MRP0      APPLYING_LOG         19          1
```

检查备库延迟：
```sql
SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                                     VALUE                UNIT                           TIME_COMPUTED
---------------------------------------- -------------------- ------------------------------ ------------------------------
transport lag                            +00 00:00:00         day(2) to second(0) interval   02/26/2023 15:35:26
apply lag                                +00 00:00:00         day(2) to second(0) interval   02/26/2023 15:35:26
```
可以看到，apply lag已经降到0了。


**References**
[1] https://www.modb.pro/db/99314
[2] https://blog.csdn.net/Sebastien23/article/details/125821548
[3] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/refrn/LOG_ARCHIVE_DEST_STATE_n.html#GUID-983A9C52-3046-4286-AEA7-800741EE0561



