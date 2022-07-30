@[TOC](Oracle ADG状态查看与相关视图)

定义相关变量
```
PRIMARY_DB_IP1=
PRIMARY_DB_IP2=
PRIMARY_DB_UNIQ_NAME=
SECOND_DB_IP1=
SECOND_DB_IP2=
SECOND_DB_UNIQ_NAME=
THIRD_DB_IP1=
THIRD_DB_IP2=
THIRD_DB_UNIQ_NAME=
```

# ADG状态相关视图
## `v$parameter`
记录了当前会话中生效的初始化参数。
- `fal_server`：即Fetch Archive Log服务器，该参数仅用于使用物理备库的ADG。该参数的值一般为一个Oracle Net服务名，配置在物理备库上，用于备库接收的归档日志不连续的问题。
- `log_archive_dest_n`：归档日志的存放路径。必须指定`LOCATION`或者`SERVICE`参数。`LOCATION`为本地磁盘中归档日志路径，`SERVICE`为远程备库的Oracle Net服务名。默认使用`ARCn`进程来收集和传输归档日志，也可以显示指定使用`LGWR`进程。
- `log_archive_dest_state_n`：定义了归档日志存放路径的可用性。默认为`ENABLE`，表示可用。`DEFER`和`RESET`表示不可用，需要重新enable。`ALTERNATE`表示备用的归档路径。
- `log_archive_config`：主备库的`DB_UNIQUE_NAME`。
```bash
#主库
LOG_ARCHIVE_DEST_1='LOCATION=/arch1/NYC/ VALID_FOR=(ALL_LOGFILES,ALL_ROLES)'
LOG_ARCHIVE_DEST_STATE_1=ENABLE
LOG_ARCHIVE_DEST_2='SERVICE=SECOND_DB_SRV_NAME LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=SECOND_DB_UNIQ_NAME'
LOG_ARCHIVE_DEST_STATE_2=ENABLE
DG_CONFIG=(PRIMARY_DB_UNIQ_NAME, SECOND_DB_UNIQ_NAME)

#备库
LOG_ARCHIVE_DEST_1='LOCATION=/arch1/NYC/ VALID_FOR=(ALL_LOGFILES,ALL_ROLES)'
LOG_ARCHIVE_DEST_STATE_1=ENABLE
LOG_ARCHIVE_DEST_2='SERVICE=PRIMARY_DB_SRV_NAME LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=PRIMARY_DB_UNIQ_NAME'
LOG_ARCHIVE_DEST_STATE_2=ENABLE
DG_CONFIG=(PRIMARY_DB_UNIQ_NAME, SECOND_DB_UNIQ_NAME)
```

- `db_file_name_convert`：定义了主备库之间数据文件名称的转化关系。
- `log_file_name_convert`：定义了主备库之间日志文件名称的转化关系。
```bash
#主库
DB_FILE_NAME_CONVER='/oracle/oradata/SECOND_DB_UNIQ_NAME/DATAFILE/','/oracle/oradata/PRIMARY_DB_UNIQ_NAME/DATAFILE/','/oracle/oradata/SECOND_DB_UNIQ_NAME/TEMPFILE/','/oracle/oradata/PRIMARY_DB_UNIQ_NAME/TEMPFILE/'
LOG_FILE_NAME_CONVER='/oracle/oradata/SECOND_DB_UNIQ_NAME/LOGFILE/','/oracle/oradata/PRIMARY_DB_UNIQ_NAME/LOGFILE/'

#备库
DB_FILE_NAME_CONVER='/oracle/oradata/PRIMARY_DB_UNIQ_NAME/DATAFILE/','/oracle/oradata/SECOND_DB_UNIQ_NAME/DATAFILE/','/oracle/oradata/PRIMARY_DB_UNIQ_NAME/TEMPFILE/','/oracle/oradata/SECOND_DB_UNIQ_NAME/TEMPFILE/'
LOG_FILE_NAME_CONVER='/oracle/oradata/PRIMARY_DB_UNIQ_NAME/LOGFILE/','/oracle/oradata/SECOND_DB_UNIQ_NAME/LOGFILE/'
```

## `gv$database`
- `INST_ID`：RAC架构中两节点的实例ID；
- `DB_UNIQUE_NAME`：数据库唯一名称，用于区分ADG的主备库。
- `DATABASE_ROLE`：数据库的角色，可以是主库`PRIMARY`、物理备库`PHYSICAL STANDBY`、逻辑备库`LOGICAL STANDBY`、快照备库`SNAPSHOT STANDBY`。
- `protection_level`：数据库保护等级，可以是最大保护模式`MAXIMUM PROTECTION`、最大可用模式`MAXIMUM AVAILABILITY`、最大性能模式`MAXIMUM PERFORMANCE`等。
- `PROTECTION_MODE`：数据库保护模式，可以是最大保护模式`MAXIMUM PROTECTION`、最大可用模式`MAXIMUM AVAILABILITY`、最大性能模式`MAXIMUM PERFORMANCE`等。ADG中默认为最大性能模式。
- `OPEN_MODE`：数据打开模式，可以为`MOUNTED`、`READ WRITE`、`READ ONLY`、`READ ONLY WITH APPLY`。其中，`READ ONLY WITH APPLY`表示物理备库处于只读状态。
- `switchover_status`：ADG切换状态，可以为以下值：
  - `NOT ALLOWED`：在主库上，表示不存在有效且可用的备库；在备库上，表示未收到主库发来的切换请求（不影响切换）。
  - `SESSIONS ACTIVE`：数据库中存在活动的会话。
  - `SWITCHOVER PENDING`：在物理备库上，表示收到了主库发来的切换请求且正在处理。
  - `SWITCHOVER LATENT`：在物理备库上，表示收到了主库发来的切换请求且正在处理时，源主库又切回了Primary角色。
  - `TO PRIMARY`：表示数据库已经准备好切换为主库。
  - `TO STANDBY`：表示数据库已经准备好切换为物理备库或者逻辑备库。
  - `TO LOGICAL STANDBY`：数据库已经准备好切换为逻辑备库。
  - `RECOVERY NEEDED`：在物理备库上，表示在准备好切换为主库之前，还有redo日志未被应用。
  - `PREPARING SWITCHOVER`：在主库上，表示正在接收逻辑备库发来的数据字典，为切换为备库做准备；在逻辑备库上，表示数据字典已经发送给主库和其他备库。
  - `PREPARING DICTIONARY`：在逻辑备库上，表示正在发送数据字典给主库和其他备库，为切换为主库做准备。
  - `FAILED DESTINATION`：在主库上，表示至少有一个备库发生了错误。
  - `RESOLVABLE GAP`：在主库上，表示至少有一个备库接收到了不连续的日志，但是可以自动从主库或者其他备库获取归档日志来解决日志不连续的问题。
  - `UNRESOLVABLE GAP`：在主库上，表示至少有一个备库接收到了不连续的日志，并且无法自动从主库或者其他备库获取归档日志来解决日志不连续的问题。
  - `LOG SWITCH GAP`：在主库上，表示至少有一个备库由于最近的日志切换丢失了redo数据。


## `gv$instance`
- `INST_ID`：实例ID；
- `HOST_NAME`：主机名；
- `THREAD#`：当前实例打开的Redo线程数。

## `gv$archived_log`
- `thread#`：Redo线程数；
- `sequence#`：Redo日志序列号。

## `v$managed_standby`
- `PROCESS`：进程名，例如`RFS`、`MRP0`、`ARCH`、`LGWR`等。
- `STATUS`：进程状态，例如`MRP0`在备库上一般为应用日志`APPLYING_LOG`状态。
- `THREAD#`：归档redo日志的线程数。
- `SEQUENCE#`：归档redo日志的序列号。


# 查询ADG主备库状态
检查主库状态
```bash
ssh ${PRIMARY_DB_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${PRIMARY_DB_UNIQ_NAME}1
    sqlplus / as sysdba <<EFF
       set lines 200
       col name for a25
       col value for a120
       col host_name for a15
       col db_unique_name for a15
       col switchover_status for a20
       select name, value from v\\\$parameter where name in ('fal_server', 'log_archive_dest_1', 'log_archive_dest_2','log_archive_dest_state_2',
	   'log_archive_dest_3','log_archive_dest_state_3','log_archive_config','db_file_name_convert','log_file_name_convert');
       select a.INST_ID, a.DB_UNIQUE_NAME, a.DATABASE_ROLE, a.protection_level, a.PROTECTION_MODE, a.OPEN_MODE, a.switchover_status, b.HOST_NAME, b.THREAD# THREAD 
	   from gv\\\$database a left join gv\\\$instance b on a.inst_id = b.inst_id order by a.inst_id;
       select thread# THREAD, max(sequence#) MAX_SEQUENCE from gv\\\$archived_log where archived='YES' group by thread# order by thread#;
       select PROCESS, STATUS, THREAD# THREAD, SEQUENCE# SEQUENCE from v\\\$managed_standby where process like 'MRP%';
EFF
EOF
```

检查备库状态
```bash
ssh ${SECOND_DB_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${SECOND_DB_UNIQ_NAME}1
    sqlplus / as sysdba <<EFF
       set lines 200
       col name for a25
       col value for a120
       col host_name for a15
       col db_unique_name for a15
       col switchover_status for a20
       select name, value from v\\\$parameter where name in ('fal_server', 'log_archive_dest_1', 'log_archive_dest_2','log_archive_dest_state_2',
	   'log_archive_dest_3','log_archive_dest_state_3','log_archive_config','db_file_name_convert','log_file_name_convert');
       select a.INST_ID, a.DB_UNIQUE_NAME, a.DATABASE_ROLE, a.protection_level, a.PROTECTION_MODE, a.OPEN_MODE, a.switchover_status, b.HOST_NAME, b.THREAD# THREAD 
	   from gv\\\$database a left join gv\\\$instance b on a.inst_id = b.inst_id order by a.inst_id;
       select thread# THREAD, max(sequence#) MAX_SEQUENCE from v\\\$archived_log where applied in ('YES','IN-MEMORY') group by thread# order by thread#;
       select PROCESS, STATUS, THREAD# THREAD, SEQUENCE# SEQUENCE from v\\\$managed_standby where process like 'MRP%';
EFF
EOF
```

**参考资料**
【1】https://docs.oracle.com/cd/B19306_01/server.102/b14237/initparams067.htm#REFRN10056
【2】https://blog.csdn.net/Sebastien23/article/details/125821548?spm=1001.2014.3001.5501
【3】https://docs.oracle.com/cd/B19306_01/server.102/b14237/initparams100.htm#REFRN10086
【4】https://docs.oracle.com/en/database/oracle/oracle-database/12.2/refrn/LOG_ARCHIVE_DEST_STATE_n.html#GUID-983A9C52-3046-4286-AEA7-800741EE0561
【5】https://docs.oracle.com/cd/B19306_01/server.102/b14239/log_arch_dest_param.htm#CACHDECE
【6】https://docs.oracle.com/cd/B19306_01/server.102/b14237/initparams048.htm#REFRN10038
【7】https://my.oschina.net/wangshengzhuang/blog/785024
【8】https://docs.oracle.com/cd/E11882_01/server.112/e40402/dynviews_1097.htm#REFRN30047
【9】https://docs.oracle.com/cd/B19306_01/server.102/b14237/dynviews_1169.htm#REFRN30144




