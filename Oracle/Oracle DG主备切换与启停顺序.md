---
tags: [oracle]
title: Oracle DG主备切换与启停顺序
created: '2023-02-03T10:54:06.254Z'
modified: '2023-02-05T02:53:48.337Z'
---

Oracle DG主备切换与启停顺序

# DG主备切换
## 准备检查阶段
检查监听器：
```bash
lsnrctl status
```

检查数据库状态：
```sql
--检查数据库是否打开：主备库都要打开
SQL> select instance_name,status from gv$instance;

--检查主备DG参数
col value for a100
SQL> select name,value from v$parameter where name in ('fal_server','fal_client',
'standby_file_management','log_archive_dest_1','log_archive_dest_2','log_archive_dest_state_2',
'log_archive_config','db_file_name_convert','log_file_name_convert');

--检查主库（status正常为VALID）
col dest_name for a30
SQL> select dest_id,dest_name,status,recovery_mode from v$archive_dest_status 
where dest_name='LOG_ARCHIVE_DEST_2';

--检查主备库归档进程数（需要大于等于4）
show parameter log_archive_max_processes
```

检查Redo日志：
```sql
--检查备库redo文件是否创建（standby的redolog要比主库的多一组）
col member for a70
SQL> select * from v$logfile;

--检查备库redo log是否需要清理
SQL> select distinct a.group# from v$log a, v$logfile b
where a.group# = b.group# and a.status 
not in ('UNUSED','CLEARING','CLEARING_CURRENT');

--检查redolog
SQL> select group#,thread#,sequence#,members,archived,status,bytes/1024/1024 size_mb
from v$log order by 1;

--主库查询当前的redo sequence
SQL> select thread#,sequence# from v$thread;

-- 备库查询应用到的redo sequence，应该与上面主库查到的相差不大（只差1到2个）
SQL> select thread#, max(sequence#) from v$archived_log 
where applied = 'YES' and resetlogs_change# = (
    select resetlogs_change# from v$database_incarnation where status = 'CURRENT'
) group by thread#;
```

检查数据文件：
```sql
--确认主备库临时文件一致，且所有数据文件都在线
col filename for a50
SQL> select tmp.name filename, bytes/1024/1024 size_mb, ts.name tablespace_name 
from v$tempfile tmp, v$tablespace ts
where tmp.ts# = ts.ts#;

SQL> select name from v$datafile where status='OFFLINE';
```

检查应用连接：
```sql
--关闭应用后，确认主库当前连接会话
SQL> select username,sid,status,event,program,machine,sql_id 
from v$session where username != 'SYS';

SQL> select username,sid,status,event,program,machine,sql_id,logon_time 
from gv$session where username != 'SYS' order by logon_time desc;
```

检查DG同步状态：
```sql
--检查是否有GAP（正常没有）
SQL> select thread#,low_sequence#,high_sequence# from v$archive_gap;

--检查备库当前应用进程是否有延时（正常为0）
set lines 200
col value for a30
SQL> select name,value,unit,time_computed from v$dataguard_stats 
where name in ('transport lag','apply lag');

--检查打开模式（主库read write、备库read only with apply）、日志模式（主备库都是archivelog）
--检查switchover_status（主库为to standby或session active，备库为not allowed）
set lines 220
col name for a25
col value for a120
col host_name for a15
col db_unique_name for a15
col switchover_status for a20

SQL> select a.inst_id,a.db_unique_name,a.database_role,
a.protection_level,a.protection_mode,a.open_mode,a.log_mode,a.switchover_status,
b.host_name,b.thread# 
from gv$database a left join gv$instance b 
on a.inst_id=b.inst_id order by a.inst_id; 

--> 主库为to standby表示可以直接切换
--> 主库为session active，但查询v$session都是系统会话，可以通过如下命令处理
SQL> alter database commit to switchover to physical standby with session shutdown;
```

## DG切换
```
primarydb_unique_name = bangkok
standbydb_unique_name = bangkokdg
```

### 新语法切换
检查主库是否具备切换条件：
```sql
alter database switchover to bangkokdg verify;
--不报错即可切换
```

在主库发起主备切换：
```sql
alter database switchover to bangkokdg;
```

打开新主库（在原备库执行）：
```sql
alter database open;
```

打开新备库（在原主库执行）：
```sql
startup mount;
alter database recover managed standby database disconnect from session;

--检查延迟是否为0
select name,value,unit,time_computed from v$dataguard_stats 
where name in ('transport lag','apply lag');

alter database recover managed standby database cancel;
alter database open;
alter database recover managed standby database disconnect from session;
```

最后检查DG状态和MRP进程：
```sql
select a.inst_id,a.db_unique_name,a.database_role,
a.protection_level,a.protection_mode,a.open_mode,a.log_mode,a.switchover_status,
b.host_name,b.thread# 
from gv$database a left join gv$instance b 
on a.inst_id=b.inst_id order by a.inst_id; 

select process,status,sequence#,thread# from v$managed_standby where process='MRP0';
```

### 旧语法切换
在原主库上执行：
```sql
select switchover_status from v$database;   --应该为TO STANDBY

--发起切换为备库
alter database commit to switchover to physical standby;  

--挂载
startup mount;
```

在原备库上执行：
```sql
select switchover_status from v$database;   --应该为TO PRIMARY

--发起切换为主库
alter database commit to switchover to primary with session shutdown;   

--打开新主库
alter database open;
```

打开新备库（在原主库上执行）：
```sql
alter database recover managed standby database disconnect from session;

--检查延迟是否为0
select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

alter database recover managed standby database cancel;
alter database open;
alter database recover managed standby database disconnect from session;
```

最后检查DG状态和MRP进程。

# DG主备启停顺序
以一主一备架构为例：启动DG时，先起备库，再起主库；停止DG时，先停主库，再停备库。

## 启动顺序
启动监听：
```bash
lsnrctl start   # 备库
lsnrctl start   # 主库
```

启动备库：
```sql
--挂载
startup nomount;
alter database mount standby database;

--打开库和MRP日志应用
alter database open;
alter database recover managed standby database using current logfile disconnect from session;
```

启动主库：
```sql
startup;
```

## 停库顺序
停主库：
```sql
shutdown immediate;
```

停备库：
```sql
alter database recover managed standby database cancel;
shutdown immediate;
```

停监听：
```bash
lsnrctl stop   # 主库
lsnrctl stop   # 备库
```


**References**
[1] https://blog.csdn.net/JiekeXu/article/details/120793233 
[2] https://www.modb.pro/db/500104
[3] http://t.csdn.cn/zkUhN



