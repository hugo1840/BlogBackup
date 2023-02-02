---
tags: [oracle]
title: 手动搭建一套Oracle Dataguard
created: '2023-01-15T07:59:35.377Z'
modified: '2023-02-02T14:50:42.085Z'
---

手动搭建一套Oracle Dataguard

**数据库版本：Oracle 19c**

服务器配置如下：
|主机名 | 私网IP | 操作系统 | 性能 | 角色 |
| :--: | :--: | :--: | :--: | :--: | 
| primarydb | 172.16.171.96  | Centos 7.5 | 4C16G | 主库 |
| standbydb | 172.16.171.97 | Centos 7.5 | 4C16G | 备库 |

数据库文件管理模式为**OMF**（Oracle本地文件管理）。

# 前置工作
>1. 在primarydb服务器上安装好Oracle数据库，配置好Oracle环境变量，例如主库配置`ORACLE_SID=bangkok`。
>2. 在standby服务器上仅安装Oracle软件（不安装数据库实例），配置`ORACLE_SID=bangkokdg`。
>3. 配置好主备库服务器的`/etc/hosts`文件：
```bash
primarydb   primarydb   172.16.171.96
standbydb   standbydb   172.16.171.97
```

# 主库配置
## 开启归档模式
确认主库有没有开启归档模式。如果没有，按照以下步骤开启归档：
```sql
alter database force logging;
shutdown immediate;

startup mount;
alter database archivelog;
archive log list;

alter database open;
select name, log_mode, force_logging from v$database;
```

## 配置监听和服务名解析
配置`$ORACLE_HOME/network/admin/listener.ora`：
```
SID_LIST_LISTENER=
  (SID_LIST = 
    (SID_DESC =
      (GLOBAL_DBNAME = bangkok)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = bangkok) 
    )
  )

LISTENER = 
  (DESCRIPTION_LIST = 
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT =1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521)) 
    )
  )
```

重启监听使配置生效：
```bash
lsnrctl stop
lsnrctl start
lsnrctl status
```

配置`$ORACLE_HOME/network/admin/tnsnames.ora`：
```
BANGKOK = 
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkok)
    )
  )

BANGKOKDG = 
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = standbydb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkokdg)
      (UR = A)
    )
  )
```

验证服务名解析：
```bash
tnsping bangkok
tnsping bangkokdg
```

## 创建standby日志组
在主库上添加standby日志组，日志大小与online日志保持一致，数量要比online日志多一组。
```sql
set lines 200
col member for a80
--查看日志文件
select * from v$logfile;
--查看日志组数量及大小
select thread#, group#, bytes/1024/1024 size_mb from v$log;

THREAD#   GROUP#    SIZE_MB
-------   ------    -------
     1        1        2048
     1        2        2048
     1        3        2048
```

根据上面SQL的结果可知当前实例有3个日志组，所以至少需要创建4个standby日志组。
```sql
alter database add standby logfile group 11 size 2048M;
alter database add standby logfile group 21 size 2048M;
alter database add standby logfile group 31 size 2048M;
alter database add standby logfile group 41 size 2048M;
```

再次检查日志文件和standby日志：
```sql
select * from v$logfile;
select thread#, group#, sequence#, archived, status from v$standby_log;
--archived列的值应为YES，status列的值为UNASSIGNED
```

## 配置DG参数
在主库上配置Dataguard相关参数：
```sql
--配置DG主备库
alter system set log_archive_config='DG_CONFIG=(bangkok,bangkokdg)' scope=both;

--配置本地归档路径
alter system set log_archive_dest_1='LOCATION=/oradata/arch VALID_FOR=(ALL_LOGFILES,ALL_ROLES) 
DB_UNIQUE_NAME=bangkok' scope=both;

--配置备库归档
alter system set log_archive_dest_2='SERVICE=bangkokdg LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,
PRIMARY_ROLE) DB_UNIQUE_NAME=bangkokdg' scope=both;

alter system set log_archive_dest_state_1=ENABLED scope=both;
alter system set log_archive_dest_state_2=ENABLED scope=both;

alter system set FAL_SERVER=bangkokdg scope=both;
alter system set FAL_CLIENT=bangkok scope=both;

alter system set standby_file_management=auto;

--配置主备库数据文件名称转换关系
alter system set db_file_name_convert='/oradata/BANGKOKDG/datafile', '/oradata/BANGKOK/datafile' scope=spfile;

--配置主备库日志文件名称转换关系
alter system set log_file_name_convert='/oradata/BANGKOKDG/onlinelog', '/oradata/BANGKOK/onlinelog', 
'/oradata/fats_recovery_area/BANGKOKDG/onlinelog', '/oradata/fats_recovery_area/BANGKOK/onlinelog' scope=spfile;
```

生成参数文件：
```sql
create pfile from spfile;
```

将参数文件和密码文件拷贝到备库：
```bash
scp $ORACLE_HOME/dbs/initbangkok.ora oracle@172.16.171.97:$ORACLE_HOME/dbs/
scp $ORACLE_HOME/dbs/orapwbangkok oracle@172.16.171.97:$ORACLE_HOME/dbs/
```

# 备库配置
## 创建数据库目录
```bash
mkdir -p /oradata/BANGKOKDG/controlfile
mkdir -p /oradata/BANGKOKDG/datafile
mkdir -p /oradata/BANGKOKDG/onlinelog
```

## 配置监听和服务名解析
配置`$ORACLE_HOME/network/admin/listener.ora`：
```
SID_LIST_LISTENER=
  (SID_LIST = 
    (SID_DESC =
      (GLOBAL_DBNAME = bangkokdg)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (SID_NAME = bangkok) 
    )
  )

LISTENER = 
  (DESCRIPTION_LIST = 
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = standbydb)(PORT =1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521)) 
    )
  )
```

配置`$ORACLE_HOME/network/admin/tnsnames.ora`：
```
BANGKOK = 
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkok)
    )
  )

BANGKOKDG = 
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = standbydb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkokdg)
      (UR = A)
    )
  )
```

## 修改参数文件
修改从主库拷贝过来的参数文件`$ORACLE_HOME/dbs/initbangkok.ora`。主要是对调主备库名位置。下面是有改动的部分。
```
*.audit_file_dest='/u01/app/oracle/admin/bangkokdg/adump'

*.control_files='/oradata/BANGKOKDG/controlfile/o1_mf_kvodmbdo_.ctl','/oradata/fast_recovery_area/BANGKOKDG/controlfile/o1_mf_kvodmbfp_.ctl'

*.db_file_name_convert='/oradata/BANGKOK/datafile','/oradata/BANGKOKDG/datafile'

*.db_name='bangkok'

*.db_recovery_file_dest='/oradata/fast_recovery_area'

*.db_unique_name='bangkokdg'

*.fal_client='BANGKOKDG'
*.fal_server='BANGKOK'

*.log_archive_config='DG_CONFIG=(bangkokdg,bangkok)'
*.log_archive_dest_1='LOCATION=/oradata/archVALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=bangkokdg'
*.log_archive_dest_2='SERVICE=bangkok LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=bangkok'
*.log_archive_dest_state_1='ENABLE'
*.log_archive_dest_state_2='ENABLE'

*.log_file_name_convert='/oradata/BANGKOK/onlinelog','/oradata/BANGKOKDG/onlinelog', '/oradata/fast_recovery_area/BANGKOK/onlinelog', '/oradata/fast_recovery_area/BANGKOKDG/onlinelog'
```

重命名参数文件：
```bash
cd $ORACLE_HOME/dbs/
mv initbangkok.ora initbangkokdg.ora
```

## 使用参数文件启动数据库为NOMOUNT
利用上面修改好的参数文件，启动备库到NOMOUNT状态：
```sql
create spfile from pfile='/u01/app/oracle/product/19.0.0/dbhome_1/dbs/initbangkokdg.ora';
startup nomount;

ORA-09925: Unable to create audit trail file
Linux-x86_64 Error: No such file or directory
```

手动创建adump目录：
```bash
mkdir /u01/app/oracle/admin/bangkokdg/adump
```

启动备库到NOMOUNT状态：
```sql
startup nomount
--出现Oracle instance Started即可
```

启动监听：
```bash
lsnrctl start
lsnrctl status
```

验证服务名解析：
```bash
tnsping bangkok
tnsping bangkokdg
```

## 使用RMAN duplicate主库到备库
检查数据库名称：
```sql
show parameter name
```

重命名从主库拷贝过来的密码文件：
```bash
cd $ORACLE_HOME/dbs/
mv orapwbangkok orapwbangkokdg
```

连接RMAN并duplicate主库到备库:
```bash
rman target sys/syspassword@bangkok auxiliary sys/syspassword@bangkokdg

RMAN> run {
  allocate channel cl1 type disk;
  allocate channel cl2 type disk;
  allocate auxiliary channel c1 type disk;
  allocate auxiliary channel c2 type disk;
  duplicate target database for standby from active database nofilenamecheck;
  release channel c1;
  release channel c2;
  release channel cl1;
  release channel cl2;
}
```

复制完成后检查备库状态：
```sql
archive log list;
--归档模式已打开

select database_role, protection_mode, protection_level, open_mode from v$database;
--数据库角色应为PHYSICAL STANDBY，打开模式为MOUNTED
```

## 开启日志应用进程
打开备库：
```sql
alter database open;
```

开启日志应用进程：
```sql
alter database recover managed standby database using current logfile disconnect from session;
```

# 检查主备状态
查看备库日志应用情况：
```sql
select name, sequence#, thread#, applied from v$archived_log;
select thread#, max(sequence#) from v$archived_log where applied='YES' order by thread#;
```

查看归档错误：
```sql
select dest_id, error from v$archived_dest where error is not null;
```

查看归档有无GAP：
```sql
select * from v$archive_gap;
```

查看备库日志状态：
```sql
select group#, thread#, sequence#, archived, status from v$standby_log;
```

查看备库状态信息：
```sql
select message from v$dataguard_status;
```

查看主备库的DG配置参数：
```sql
set lines 220
col name for a25
col value for a120

select name,value from v$parameter where name in ('fal_server','log_archive_dest_1',
'log_archive_dest_2','log_archive_dest_state_2',
'log_archive_dest_3','log_archive_dest_state_3',
'log_archive_config','db_file_name_convert','log_file_name_convert');
```

查看主备库的切换状态：
```sql
set lines 220
col host_name for a15
col db_unique_name for a15
col switchover_status for a20

select a.inst_id, a.db_unique_name, 
a.database_role, a.protection_level, a.protection_mode, a.open_mode, a.switchover_status,
b.host_name, b.thread# 
from gv$database a 
left join gv$instance b 
on a.inst_id=b.inst_id 
order by a.inst_id;
```

查看备库日志应用进程：
```sql
select process,status,thread#,sequence# from v$managed_standby where process like 'MRP%';
```


**REFERENCES**
 [1] https://blog.csdn.net/techsupporter/article/details/56831289 
 [2] https://www.cnblogs.com/Bccd/p/6362786.html 
 [3] https://www.modb.pro/db/491783 
 [4] https://www.modb.pro/db/58180 
 [5] https://www.shuzhiduo.com/A/gAJGrKL1zZ/



