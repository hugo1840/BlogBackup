---
tags: [oracle]
title: DG Broker配置、日志传输、备库管理与主备切换
created: '2023-02-27T13:51:08.081Z'
modified: '2023-02-28T14:36:38.407Z'
---

DG Broker配置、日志传输、备库管理与主备切换

# DG Broker的配置与启用
DG Broker即Data Guard Broker，是Oracle官方提供的一个用于维护DG的工具。我们可以通过**dgmgrl**这个命令行工具来使用DG Broker。

## 启动DG Broker
目前我们现有环境为使用RMAN Duplicate手工搭建的一套主备，现在需要在该环境中启用DG Broker。

```sql
[oracle@standbydb ~]$ dgmgrl /

Connected to "bangkokdg"
Connected as SYSDG.
DGMGRL> show database 'bangkokdg';
ORA-16525: The Oracle Data Guard broker is not yet available.

Configuration details cannot be determined by DGMGRL
```

通过修改`dg_broker_start`参数来启用DG Broker：
```sql
SQL> show parameter dg_broker

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
dg_broker_config_file1               string      /u01/app/oracle/product/19.0.0
                                                 /dbhome_1/dbs/dr1bangkok.dat
dg_broker_config_file2               string      /u01/app/oracle/product/19.0.0
                                                 /dbhome_1/dbs/dr2bangkok.dat
dg_broker_start                      boolean     FALSE

SQL> alter system set dg_broker_start=true;

System altered.
```

登录DGMGRL命令行工具：
```sql
[oracle@primarydb ~]$ dgmgrl /

Connected to "bangkok"
Connected as SYSDG.
DGMGRL> show database 'bangkokdg';
ORA-16532: Oracle Data Guard broker configuration does not exist

Configuration details cannot be determined by DGMGRL
```
这里提示需要先配置DG Broker。

## 配置DG Broker
在主库配置DG Broker：
```sql
DGMGRL> create configuration 'BANGKOK' as primary database is BANGKOK connect identifier is BANGKOK;
Configuration "BANGKOK" created with primary database "bangkok"

DGMGRL> add database BANGKOKDG as connect identifier is BANGKOKDG maintained as physical;
Error: ORA-16796: one or more properties could not be imported from the member

Failed.
```

上面报错的解决办法是同时在主备库开启DG Broker，并重新配置。我们首先停用DG Broker：
```sql
SQL> alter system set dg_broker_start=false scope=both;

System altered.

SQL> show parameter dg_broker

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
dg_broker_config_file1               string      /u01/app/oracle/product/19.0.0
                                                 /dbhome_1/dbs/dr1bangkok.dat
dg_broker_config_file2               string      /u01/app/oracle/product/19.0.0
                                                 /dbhome_1/dbs/dr2bangkok.dat
dg_broker_start                      boolean     FALSE
```

移除启用DG Broker时生成的上面两个配置文件：
```bash
[oracle@primarydb ~]$ cd /u01/app/oracle/product/19.0.0/dbhome_1/dbs/
[oracle@primarydb dbs]$ rm dr1bangkok.dat 
[oracle@primarydb dbs]$ rm dr2bangkok.dat 
```

然后同时在主备库启用DG Broker：
```sql
SQL> alter system set dg_broker_start=true scope=both;

System altered.

SQL> show parameter dg_broker

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
dg_broker_config_file1               string      /u01/app/oracle/product/19.0.0
                                                 /dbhome_1/dbs/dr1bangkok.dat
dg_broker_config_file2               string      /u01/app/oracle/product/19.0.0
                                                 /dbhome_1/dbs/dr2bangkok.dat
dg_broker_start                      boolean     TRUE
```

重新配置DG Broker：
```sql
DGMGRL> create configuration 'BANGKOK' as primary database is BANGKOK connect identifier is BANGKOK;
Configuration "BANGKOK" created with primary database "bangkok"

DGMGRL> add database BANGKOKDG as connect identifier is BANGKOKDG maintained as physical;
Error: ORA-16698: member has a LOG_ARCHIVE_DEST_n parameter with SERVICE attribute set

Failed.
```

这里主库配置成功了，但是在添加备库时有报错。这个报错是由于主备库之前已经配置了`log_archive_dest_n`参数。启用DG Broker时需要在主库和备库手动清除该参数。
```sql
SQL> alter system set log_archive_dest_2='' scope=both;

System altered.
```

重新在DG Broker配置中添加备库：
```sql
DGMGRL> add database BANGKOKDG as connect identifier is BANGKOKDG maintained as physical;
Database "bangkokdg" added
```

启用DG Broker配置：
```sql
DGMGRL> enable configuration;
Enabled.
DGMGRL> enable database bangkok;
Enabled.
DGMGRL> enable database bangkokdg;
Enabled.
```

查看当前DG Broker配置：
```sql
DGMGRL> show configuration;

Configuration - BANGKOK

  Protection Mode: MaxPerformance
  Members:
  bangkok   - Primary database
    bangkokdg - Physical standby database 
      Warning: ORA-16853: apply lag has exceeded specified threshold

Fast-Start Failover:  Disabled

Configuration Status:
WARNING   (status updated 47 seconds ago)
```

>:moon:**注**：使用DG Broker还需要配置本地监听器，我们在本文后面会提到。

## 使用DG Broker查看数据库信息
查看主备库状态：
```sql
DGMGRL> show database bangkok;

Database - bangkok

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    bangkok

Database Status:
SUCCESS

DGMGRL> show database bangkokdg;

Database - bangkokdg

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 1 second ago)
  Apply Lag:          0 seconds (computed 1 second ago)
  Average Apply Rate: 71.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    bangkokdg

Database Status:
SUCCESS
```

查看主备库的详细信息：
```sql
DGMGRL> show database verbose bangkok;

Database - bangkok

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    bangkok

  Properties:
    DGConnectIdentifier             = 'bangkok'
    ObserverConnectIdentifier       = ''
    FastStartFailoverTarget         = ''
    PreferredObserverHosts          = ''
    LogShipping                     = 'ON'
    RedoRoutes                      = ''
    LogXptMode                      = 'ASYNC'
    DelayMins                       = '0'
    Binding                         = 'optional'
    MaxFailure                      = '0'
    ReopenSecs                      = '300'
    NetTimeout                      = '30'
    RedoCompression                 = 'DISABLE'
    PreferredApplyInstance          = ''
    ApplyInstanceTimeout            = '0'
    ApplyLagThreshold               = '30'
    TransportLagThreshold           = '30'
    TransportDisconnectedThreshold  = '30'
    ApplyParallel                   = 'AUTO'
    ApplyInstances                  = '0'
    StandbyFileManagement           = ''
    ArchiveLagTarget                = '0'
    LogArchiveMaxProcesses          = '0'
    LogArchiveMinSucceedDest        = '0'
    DataGuardSyncLatency            = '0'
    LogArchiveTrace                 = '0'
    LogArchiveFormat                = ''
    DbFileNameConvert               = ''
    LogFileNameConvert              = ''
    ArchiveLocation                 = ''
    AlternateLocation               = ''
    StandbyArchiveLocation          = ''
    StandbyAlternateLocation        = ''
    InconsistentProperties          = '(monitor)'
    InconsistentLogXptProps         = '(monitor)'
    LogXptStatus                    = '(monitor)'
    SendQEntries                    = '(monitor)'
    RecvQEntries                    = '(monitor)'
    HostName                        = 'primarydb'
    StaticConnectIdentifier         = '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=primarydb)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=bangkok_DGMGRL)(INSTANCE_NAME=bangkok)(SERVER=DEDICATED)))'
    TopWaitEvents                   = '(monitor)'
    SidName                         = '(monitor)'

  Log file locations:
    Alert log               : /u01/app/oracle/diag/rdbms/bangkok/bangkok/trace/alert_bangkok.log
    Data Guard Broker log   : /u01/app/oracle/diag/rdbms/bangkok/bangkok/trace/drcbangkok.log

Database Status:
SUCCESS

DGMGRL> show database verbose bangkokdg;

Database - bangkokdg

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 0 seconds ago)
  Average Apply Rate: 12.00 KByte/s
  Active Apply Rate:  0 Byte/s
  Maximum Apply Rate: 0 Byte/s
  Real Time Query:    ON
  Instance(s):
    bangkokdg

  Properties:
    DGConnectIdentifier             = 'bangkokdg'
    ObserverConnectIdentifier       = ''
    FastStartFailoverTarget         = ''
    PreferredObserverHosts          = ''
    LogShipping                     = 'ON'
    RedoRoutes                      = ''
    LogXptMode                      = 'ASYNC'
    DelayMins                       = '0'
    Binding                         = 'optional'
    MaxFailure                      = '0'
    ReopenSecs                      = '300'
    NetTimeout                      = '30'
    RedoCompression                 = 'DISABLE'
    PreferredApplyInstance          = ''
    ApplyInstanceTimeout            = '0'
    ApplyLagThreshold               = '30'
    TransportLagThreshold           = '30'
    TransportDisconnectedThreshold  = '30'
    ApplyParallel                   = 'AUTO'
    ApplyInstances                  = '0'
    StandbyFileManagement           = ''
    ArchiveLagTarget                = '0'
    LogArchiveMaxProcesses          = '0'
    LogArchiveMinSucceedDest        = '0'
    DataGuardSyncLatency            = '0'
    LogArchiveTrace                 = '0'
    LogArchiveFormat                = ''
    DbFileNameConvert               = ''
    LogFileNameConvert              = ''
    ArchiveLocation                 = ''
    AlternateLocation               = ''
    StandbyArchiveLocation          = ''
    StandbyAlternateLocation        = ''
    InconsistentProperties          = '(monitor)'
    InconsistentLogXptProps         = '(monitor)'
    LogXptStatus                    = '(monitor)'
    SendQEntries                    = '(monitor)'
    RecvQEntries                    = '(monitor)'
    HostName                        = 'standbydb'
    StaticConnectIdentifier         = '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=standbydb)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=bangkokdg_DGMGRL)(INSTANCE_NAME=bangkokdg)(SERVER=DEDICATED)))'
    TopWaitEvents                   = '(monitor)'
    SidName                         = '(monitor)'

  Log file locations:
    Alert log               : /u01/app/oracle/diag/rdbms/bangkokdg/bangkokdg/trace/alert_bangkokdg.log
    Data Guard Broker log   : /u01/app/oracle/diag/rdbms/bangkokdg/bangkokdg/trace/drcbangkokdg.log

Database Status:
SUCCESS
```

# 使用DG Broker管理日志传输
## 关闭到所有备库的redo传输
将主库的状态设置为**TRANSPORT-OFF**来停止对所有备库的日志传输：
```sql
DGMGRL> edit database 'BANGKOK' set state='TRANSPORT-OFF';
Succeeded.
DGMGRL> show database bangkok;

Database - bangkok

  Role:               PRIMARY
  Intended State:     TRANSPORT-OFF
  Instance(s):
    bangkok

Database Status:
SUCCESS
```

检查主库上负责日志传输的LNS进程：
```sql
SQL> select process,status,thread#,sequence# from v$managed_standby where process like 'LNS%';

PROCESS   STATUS          THREAD#  SEQUENCE#
--------- ------------ ---------- ----------
LNS       WRITING               1         20

SQL> alter system switch logfile;

System altered.

SQL> select process,status,thread#,sequence# from v$managed_standby where process like 'LNS%';

no rows selected
```

查看主库上的`log_archive_dest_state_n`参数：
```sql
SQL> show parameter log_archive_dest_state_2

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_dest_state_2             string      RESET
```

## 恢复到所有备库的redo传输
将主库的状态设置为**TRANSPORT-ON**来启用对所有备库的日志传输：
```sql
DGMGRL> edit database 'BANGKOK' set state='TRANSPORT-ON';
Succeeded.
DGMGRL> show database bangkok;

Database - bangkok

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    bangkok

Database Status:
SUCCESS
```

再次查看主库上的LNS进程：
```sql
SQL> select process,status,thread#,sequence# from v$managed_standby where process like 'LNS%';

PROCESS   STATUS          THREAD#  SEQUENCE#
--------- ------------ ---------- ----------
LNS       WRITING               1         22
LNS       CONNECTED             0          0
```

再次查看主库上的`log_archive_dest_state_n`参数：
```sql
SQL> show parameter log_archive_dest_state_2

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_dest_state_2             string      ENABLE
```
其实，DG Broker控制redo传输是通过修改主库的`log_archive_dest_state_n`参数来实现的。

## 关闭到指定备库的redo传输
将备库的LogShipping属性设置为OFF来暂停对该备库的日志传输：
```sql
DGMGRL> show database bangkokdg logshipping;
  LogShipping = 'ON'

DGMGRL> edit database 'BANGKOKDG' set property LogShipping='OFF';
Property "logshipping" updated

DGMGRL> show database bangkokdg logshipping;
  LogShipping = 'OFF'
```

## 恢复到指定备库的redo传输
将备库的LogShipping属性设置为ON来启用对该备库的日志传输：
```sql
DGMGRL> edit database 'BANGKOKDG' set property LogShipping='ON';
Property "logshipping" updated

DGMGRL> show database bangkokdg logshipping;
  LogShipping = 'ON'
```

# 使用DG Broker管理备库
通过DG Broker可以方便地将备库在只读的物理备库模式和可读写的快照备库模式之间转换。在快照备库模式下对备库的写操作是暂时性的，当备库切回物理备库模式时，这些写操作会全部回滚。物理备库转换为快照备库的前提是备库开启了快速闪回区（Fast Recovery Area）。

## 转换备库为可读写快照模式
查看备库状态：
```sql
SQL> select db_unique_name,open_mode,database_role,switchover_status from v$database;

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    SWITCHOVER_STATUS
------------------------------ -------------------- ---------------- --------------------
bangkokdg                      READ ONLY WITH APPLY PHYSICAL STANDBY NOT ALLOWED
```

将物理备库转换为快照备库：
```sql
DGMGRL> CONVERT DATABASE 'BANGKOKDG' TO SNAPSHOT STANDBY;
Converting database "BANGKOKDG" to a Snapshot Standby database, please wait...
Database "BANGKOKDG" converted successfully
```

查看备库状态：
```sql
SQL> select db_unique_name,open_mode,database_role,switchover_status from v$database;

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    SWITCHOVER_STATUS
------------------------------ -------------------- ---------------- --------------------
bangkokdg                      READ WRITE           SNAPSHOT STANDBY NOT ALLOWED
```
发现备库变为可读写。

在快照备库上进行写操作：
```sql
SQL> create table elden_npc(npc_id number(6), name varchar2(30), career varchar2(20));

Table created.

SQL> insert into elden_npc values (1, 'Margit', 'Fallen Omen');

1 row created.

SQL> insert into elden_npc values (2, 'Radahn', 'Half God');

1 row created.

SQL> insert into elden_npc values (3, 'Horoah Loux', 'Warrior');

1 row created.

SQL> select * from elden_npc;

    NPC_ID NAME                           CAREER
---------- ------------------------------ --------------------
         1 Margit                         Fallen Omen
         2 Radahn                         Half God
         3 Horoah Loux                    Warrior
```

## 转换备库为只读物理备库
尝试切换快照备库为物理备库：
```sql
DGMGRL> show database bangkokdg;

Database - bangkokdg

  Role:               SNAPSHOT STANDBY
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          8 minutes 16 seconds (computed 0 seconds ago)
  Instance(s):
    bangkokdg

Database Status:
SUCCESS

DGMGRL> CONVERT DATABASE 'BANGKOKDG' TO PHYSICAL STANDBY;
Converting database "BANGKOKDG" to a Physical Standby database, please wait...
Operation requires shut down of instance "bangkokdg" on database "bangkokdg"
Shutting down instance "bangkokdg"...
Unable to connect to database using (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=standbydb)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=bangkokdg_DGMGRL)(INSTANCE_NAME=bangkokdg)(SERVER=DEDICATED)))
ORA-12514: TNS:listener does not currently know of service requested in connect descriptor

Failed.

Please complete the following steps and reissue the CONVERT command:
        shut down instance "bangkokdg" of database "bangkokdg"
        start up and mount instance "bangkokdg" of database "bangkokdg"
```

在主备库配置DGMGRL本地监听，并重启监听器。
```bash
# 主库
[oracle@primarydb admin]$ cat listener.ora 
SID_LIST_LISTENER=
     (SID_LIST =
  (SID_DESC =
      (GLOBAL_DBNAME = bangkok)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = bangkok)
  )
  (SID_DESC =
      (GLOBAL_DBNAME = bangkok_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = bangkok)
  )
     )

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

# 备库
[oracle@standbydb admin]$ cat listener.ora 
SID_LIST_LISTENER=
     (SID_LIST =
  (SID_DESC =
      (GLOBAL_DBNAME = bangkokdg)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = bangkokdg)
  )
  (SID_DESC =
      (GLOBAL_DBNAME = bangkokdg_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = bangkokdg)
  )
     )

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = standbydb)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
```

再次尝试切换快照备库为物理备库：
```sql
DGMGRL> CONVERT DATABASE 'BANGKOKDG' TO PHYSICAL STANDBY;
Converting database "BANGKOKDG" to a Physical Standby database, please wait...
Operation requires shut down of instance "bangkokdg" on database "bangkokdg"
Shutting down instance "bangkokdg"...
ORA-01017: invalid username/password; logon denied


Please complete the following steps and reissue the CONVERT command:
        shut down instance "bangkokdg" of database "bangkokdg"
        start up and mount instance "bangkokdg" of database "bangkokdg"
```

**解决办法一**：根据提示手动重启备库并挂载。
```sql
SQL> shutdown immediate;
Database closed.

Database dismounted.
ORACLE instance shut down.

SQL> startup mount;
ORACLE instance started.

SQL> select db_unique_name,open_mode,database_role,switchover_status from v$database;

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    SWITCHOVER_STATUS
------------------------------ -------------------- ---------------- --------------------
bangkokdg                      MOUNTED              SNAPSHOT STANDBY NOT ALLOWED
```

重新转换：
```sql
dgmgrl /
DGMGRL> CONVERT DATABASE 'BANGKOKDG' TO PHYSICAL STANDBY;
Converting database "BANGKOKDG" to a Physical Standby database, please wait...
Database "BANGKOKDG" converted successfully
```

**解决办法二**：使用TNS方式连接DG Broker，而不是`dgmgrl /`。
```sql
[oracle@primarydb admin]$ dgmgrl /

Connected to "bangkok"
Connected as SYSDG.

DGMGRL> connect sys/XXXXX@bangkok
Connected to "bangkok"
Connected as SYSDBA.

DGMGRL> CONVERT DATABASE 'BANGKOKDG' TO PHYSICAL STANDBY;

Converting database "BANGKOKDG" to a Physical Standby database, please wait...
Operation requires shut down of instance "bangkokdg" on database "bangkokdg"
Shutting down instance "bangkokdg"...
Connected to "bangkokdg"
Database closed.
Database dismounted.
ORACLE instance shut down.
Operation requires start up of instance "bangkokdg" on database "bangkokdg"
Starting instance "bangkokdg"...
Connected to an idle instance.
ORACLE instance started.
Connected to "bangkokdg"
Database mounted.
Connected to "bangkokdg"
Continuing to convert database "BANGKOKDG" ...
Database "BANGKOKDG" converted successfully
```

查询快照备库期间写入的数据：
```sql
SQL> select db_unique_name,open_mode,database_role,switchover_status from v$database;

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    SWITCHOVER_STATUS
------------------------------ -------------------- ---------------- --------------------
bangkokdg                      READ ONLY WITH APPLY PHYSICAL STANDBY NOT ALLOWED

SQL> select * from miguel.elden_npc;
select * from miguel.elden_npc
                     *
ERROR at line 1:
ORA-00942: table or view does not exist
```
可以看到，快照备库期间写入的数据已经回滚了。

# 使用DG Broker进行主备切换
这里我们同样使用TNS方式连接DG Broker，否则会收到下面的报错：
```sql
ORA-01017: invalid username/password; logon denied
```

检查是否可以进行切换：
```sql
DGMGRL> connect sys/XXXXX@bangkok
Connected to "bangkok"
Connected as SYSDBA.

DGMGRL> validate database bangkokdg;

  Database Role:     Physical standby database
  Primary Database:  bangkok

  Ready for Switchover:  Yes
  Ready for Failover:    Yes (Primary Running)

  Flashback Database Status:
    bangkok  :  Off
    bangkokdg:  Off

  Managed by Clusterware:
    bangkok  :  NO             
    bangkokdg:  NO             
    Validating static connect identifier for the primary database bangkok...
    The static connect identifier allows for a connection to database "bangkok".

  Current Log File Groups Configuration:
    Thread #  Online Redo Log Groups  Standby Redo Log Groups Status       
              (bangkok)               (bangkokdg)                          
    1         3                       2                       Insufficient SRLs

  Future Log File Groups Configuration:
    Thread #  Online Redo Log Groups  Standby Redo Log Groups Status       
              (bangkokdg)             (bangkok)                            
    1         3                       1                       Insufficient SRLs

DGMGRL> validate database bangkok;  

  Database Role:    Primary database

  Ready for Switchover:  Yes

  Flashback Database Status:
    bangkok:  Off

  Managed by Clusterware:
    bangkok:  NO             
    Validating static connect identifier for the primary database bangkok...
    The static connect identifier allows for a connection to database "bangkok".
```

发起DG主备切换：
```sql
DGMGRL> switchover to bangkokdg;
Performing switchover NOW, please wait...
Operation requires a connection to database "bangkokdg"
Connecting ...
Connected to "bangkokdg"
Connected as SYSDBA.
New primary database "bangkokdg" is opening...
Operation requires start up of instance "bangkok" on database "bangkok"
Starting instance "bangkok"...
Connected to an idle instance.
ORACLE instance started.
Connected to "bangkok"
Database mounted.
Database opened.
Connected to "bangkok"
Switchover succeeded, new primary is "bangkokdg"
```

检查切换后新的主备库状态：
```sql
DGMGRL> show database bangkok;

Database - bangkok

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 0 seconds ago)
  Average Apply Rate: 12.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    bangkok

Database Status:
SUCCESS

DGMGRL> show database bangkokdg;

Database - bangkokdg

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    bangkokdg

Database Status:
SUCCESS
```

**References**
[1] https://cloud.tencent.com/developer/article/1063089 
[2] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/dgbkr/managing-oracle-data-guard-broker-configurations.html#GUID-FCCDC53A-C6F4-4AFB-A220-9D6622474027 
[3] https://blog.csdn.net/tianlesoftware/article/details/6068993 
[4] https://www.ucloud.cn/yun/129826.html 
[5] https://docs.oracle.com/zh-cn/solutions/standby-database-in-cloud/verify-failover.html#GUID-E6026AB5-1A7A-49D7-A79A-894DF24BD892 
[6] https://blog.csdn.net/u010692693/article/details/75733282
[7] https://dbaplus.cn/blog-57-682-1.html


