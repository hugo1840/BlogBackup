---
tags: [oracle]
title: DG故障切换及DG Broker失效配置清理
created: '2023-08-14T11:15:08.273Z'
modified: '2023-08-14T11:59:06.665Z'
---

DG故障切换及DG Broker失效配置清理

# DG主库故障强制切换
主库发生故障无法在短时间内恢复时，需要执行主备切换。此时由于DG Broker无法连接到主库，故不能通过Broker切换，只能手动在备库进行切主。
```sql
--断开备库MRP进程
alter database recover managed standby database cancel;

--手动切换备库为新的主库
alter database recover managed standby database finish force;
alter database commit to switchover to primary with session shutdown;

--重启备库使得切主生效
shutdown immediate;
startup;

--检查备库角色已转换为PRIMARY
select open_mode,database_role from v$database;
```

# DG Broker原有配置清理

故障切换后，需要移除旧的DG Broker配置。由于是故障切换，DG Broker配置信息没有更新，因此不能直接通过DGMGRL命令来移除旧的配置信息。
```bash
[oracle@primarydbhost ~]$ dgmgrl / "show configuration"; 

[oracle@primarydbhost ~]$ dgmgrl / "remove configuration"; 

Error: ORA-12545: Connect failed because target host or object does not exist 
Error: ORA-16625: cannot reach database "orcldb_1"

[oracle@primarydbhost ~]$ dgmgrl / "show configuration"; 

Configuration - dg_orcldb
Protection Mode: MaxPerformance 
Databases: 
  orcldb_1 - Primary database 
  orcldb_2 - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status: 
ORA-12545: Connect failed because target host or object does not exist 
ORA-16625: cannot reach database "orcldb_1" 
DGM-17017: unable to determine configuration status

[oracle@primarydbhost ~]$ dgmgrl / "remove database orcldb_1"; 
Connected. 

Primary database cannot be removed

[oracle@primarydbhost ~]$ dgmgrl / "disable configuration"; 
Connected. 

Error: ORA-12545: Connect failed because target host or object does not exist 
Error: ORA-16625: cannot reach database "orcldb_1"
```

可行的办法是关闭`dg_broker_start`参数，并清理相关配置文件，然后重新开启该参数即可。
```sql
sys@orcldb_2> show parameter dg_broker

NAME                    TYPE     VALUE
----------------------  -------  ---------------------------------------------
dg_broker_config_file1  string   /oracle/app/product/11204/dbs/dr1orcldb_2.dat 
dg_broker_config_file2  string   /oracle/app/product/11204/dbs/dr2orcldb_2.dat 
dg_broker_start         boolean  TRUE

orcldb_2> alter system set dg_broker_start=false scope=both;

orcldb_2> !rm /oracle/app/product/11204/dbs/dr1orcldb_2.dat 
orcldb_2> !rm /oracle/app/product/11204/dbs/dr2orcldb_2.dat

sys@orcldb_2> alter system set dg_broker_start=true scope=both;

System altered.
```

确认旧的DG Broker配置是否已删除：
```bash
[oracle@primarydbhost ~]$ dgmgrl / "show configuration" 
Connected. 

ORA-16532: Data Guard broker configuration does not exist

Configuration details cannot be determined by DGMGRL
```



