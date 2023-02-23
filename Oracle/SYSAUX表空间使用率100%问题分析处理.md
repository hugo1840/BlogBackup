---
tags: [oracle]
title: SYSAUX表空间使用率100%问题分析处理
created: '2023-02-23T08:56:06.900Z'
modified: '2023-02-23T13:01:15.365Z'
---

SYSAUX表空间使用率100%问题分析处理

# 问题背景
容量巡检时，发现某套库的SYSAUX表空间使用率接近100%，需要进行排查分析。

# 查看占用SYSAUX表空间的对象
查看SYSAUX表空间中占用空间最多的组件和对象：
```sql
SQL> set line 200
col OCCUPANT_NAME for a30
col OCCUPANT_DESC for a60
select OCCUPANT_NAME,OCCUPANT_DESC,SPACE_USAGE_KBYTES/1024 USAGE_MB
from V$SYSAUX_OCCUPANTS order by SPACE_USAGE_KBYTES desc;  

OCCUPANT_NAME               OCCUPANT_DESC                              USAGE_MB
------------------------------ ------------------------------------------------------------ ----------
SM/AWR                   Server Manageability - Automatic Workload Repository        27798.1875
SM/OPTSTAT               Server Manageability - Optimizer Statistics History          4128.625
SM/ADVISOR               Server Manageability - Advisor Framework               279.9375
XDB                   XDB                                  125.9375
EM                   Enterprise Manager Repository                    92.625
SDO                   Oracle Spatial                               66.4375
JOB_SCHEDULER               Unified Job Scheduler                           38.1875
ORDIM/ORDDATA               Oracle Multimedia ORDDATA Components                   13.5625
LOGMNR                   LogMiner                             13.375
SM/OTHER               Server Manageability - Other Components                 10.75
TEXT                   Oracle Text                             3.625
...
```

如果`OCCUPANT_NAME`列为`SM/AWR`（Server Manageability - Automatic Workload Repository），那么表示AWR信息占用过大；如果该列为`SM/OPTSTAT`（Server Manageability - Optimizer Statistics History），那么表示优化器统计信息占用过大。

上面是AWR信息过大，那么可以通过设置AWR的保留时间来减小AWR信息的存储空间，通过如下的SQL语句可以获取AWR的保留时间。
```sql
SQL> select * from dba_hist_wr_control;

      DBID SNAP_INTERVAL                                   RETENTION                                   TOPNSQL
---------- ------------------------------------------------------- ---------------------------------- ----------
 329070140 +00000 01:00:00.0                                   +00008 00:00:00.0                        DEFAULT
```
可以看到，AWR默认保留的时间为8天。

可是为什么还会占用这么多空间？AWR报告默认是采取**DELETE**的方式进行过期信息删除的，相比TRUNCATE而言，会产生大量的碎片（对于开启了自动扩展数据文件的表空间而言，碎片的现会象更加严重）。另一方面，ASH的信息有可能不受AWR快照保留策略的控制。

检查占用SYSAUX表空间前十的对象：
```sql
SQL> select * from 
  (select owner,segment_name,segment_type,sum(bytes)/1024/1024/1024 GB  
  from dba_segments where tablespace_name='SYSAUX'
  group by owner,segment_name,segment_type
  order by 4 desc )
  where rownum <10;   

OWNER                   SEGMENT_NAME                                     SEGMENT_TYPE            GB
------------------------------ --------------------------------------------------------------------------------- ------------------ ----------
SYS                   WRH$_ACTIVE_SESSION_HISTORY                             TABLE PARTITION    14.4580688
SYS                   WRH$_EVENT_HISTOGRAM_PK                                 INDEX PARTITION    1.57818604
SYS                   WRH$_ACTIVE_SESSION_HISTORY_PK                             INDEX PARTITION    1.56451416
SYS                   WRH$_EVENT_HISTOGRAM                                 TABLE PARTITION    1.53424072
SYS                   WRH$_LATCH                                     TABLE PARTITION     1.0010376
SYS                   I_WRI$_OPTSTAT_H_OBJ#_ICOL#_ST                             INDEX            .993164063
SYS                   WRH$_SYSSTAT_PK                                     INDEX PARTITION    .680725098
SYS                   WRH$_LATCH_MISSES_SUMMARY                             TABLE PARTITION    .664123535
SYS                   WRH$_SYSSTAT                                     TABLE PARTITION    .653381348

9 rows selected.
```
可以看到，占用SYSAUX空间最大的是表`WRH$_ACTIVE_SESSION_HISTORY`。

# 检查表`WRH$_ACTIVE_SESSION_HISTORY`
```sql
SQL> select min(snap_id),max(snap_id) from WRH$_ACTIVE_SESSION_HISTORY;

MIN(SNAP_ID) MAX(SNAP_ID)
------------ ------------
       1        32719

SQL> select sample_time from WRH$_ACTIVE_SESSION_HISTORY where snap_id=1;

SAMPLE_TIME
---------------------------------------------------------------------------
21-NOV-18 02.21.01.075 PM
21-NOV-18 02.22.41.135 PM
21-NOV-18 02.25.51.208 PM
21-NOV-18 02.30.21.300 PM
```

从上面的查询可得知，从SNAP_ID为1（对应采样时间为18年11月21日）的快照到目前为止的所有快照都还在数据库中保存着，使用`DBMS_WORKLOAD_REPOSITORY`包清理过期或者不需要的AWR数据，可以回收这部分空间，但是由于是delete操作，无法降低水位线，对于自动扩展的表空间，碎片化会更加严重。

检查`WRH$_ACTIVE_SESSION_HISTORY`的表分区：
```sql
SQL> select segment_name,partition_name,segment_type,bytes/1024/1024 MB 
from dba_segments where tablespace_name='SYSAUX' 
and segment_name='WRH$_ACTIVE_SESSION_HISTORY' order by 3;    

SEGMENT_NAME                                      PARTITION_NAME             SEGMENT_TYPE            MB
------------------------------------------------------------------------------- - -------------------- --------- 
WRH$_ACTIVE_SESSION_HISTORY                          WRH$_ACTIVE_329070140_0     TABLE PARTITION     14805
WRH$_ACTIVE_SESSION_HISTORY                          WRH$_ACTIVE_SES_MXDB_MXSN     TABLE PARTITION     .0625
```

# 稳妥地清理SYSAUX表空间
由于`DBMS_WORKLOAD_REPOSITORY`包清理AWR数据使用的是DELETE操作，会造成碎片，并且对于大表的清理速度非常慢，这里我们采用先备份后TRUNCATE的操作来清理表`WRH$_ACTIVE_SESSION_HISTORY`。

1. 检查要保留日期的SNAP_ID

```sql
SQL> select snap_id from
(select snap_id, sample_time from WRH$_ACTIVE_SESSION_HISTORY 
where sample_time > '15-Feb-23' and sample_time < '16-Feb-23'
order by 1)
where rownum<10;

   SNAP_ID
----------
     32716
     32716
     32716
     32716
     32716
     32716
     32716
     32716
     32716

9 rows selected.
```

2. 备份数据

```sql
SQL> create table WRH$_ACTIVE_SESSION_HISTORY_B as 
select * from WRH$_ACTIVE_SESSION_HISTORY where snap_id >= 32716;

SQL> select count(*) from WRH$_ACTIVE_SESSION_HISTORY_B;
```

3. truncate目标表

```sql
SQL> truncate table WRH$_ACTIVE_SESSION_HISTORY;
```

4. 恢复备份数据

```sql
SQL> insert into WRH$_ACTIVE_SESSION_HISTORY 
select * from WRH$_ACTIVE_SESSION_HISTORY_B;

SQL> select count(*) from WRH$_ACTIVE_SESSION_HISTORY;

SQL> drop table WRH$_ACTIVE_SESSION_HISTORY_B purge;
```

5. 检查表空间使用率

```sql
SQL> set linesize 200
select total.tablespace_name, round(total.max_mb,2) max_mb, round(total.MB,2) total_mb,
round(free.MB,2) free_mb, round(total.MB - free.MB,2) used_mb,
round((1 - free.MB/total.MB)*100,2) as used_percent 
from 
(select tablespace_name,sum(bytes)/1024/1024 MB
from dba_free_space group by tablespace_name) free,
(select tablespace_name,sum(bytes)/1024/1024 MB, sum(maxbytes)/1024/1024 max_mb
from dba_data_files group by tablespace_name) total
where free.tablespace_name = total.tablespace_name 
order by used_percent desc;

TABLESPACE_NAME            MAX_MB    TOTAL_MB    FREE_MB    USED_MB USED_PERCENT
------------------------------ ---------- ---------- ---------- ---------- ------------
USERS                425982.81  175038.94   18891.19  156147.75      89.21
SYSTEM                 32767.98      32730   11144.75   21585.25      65.95
UNDOTBS1             32767.98   32767.98   13798.63   18969.36      57.89
SYSAUX                 32767.98   32767.98   16466.88   16301.11      49.75
...
```

最后，可以重建目标表的分区索引：
```sql
SQL> select index_name from dba_indexes where table_name='WRH$_ACTIVE_SESSION_HISTORY';

INDEX_NAME
-----------
WRH$_ACTIVE_SESSION_HISTORY_PK

SQL> select segment_name,partition_name,segment_type,bytes/1024/1024 MB 
from dba_segments where tablespace_name='SYSAUX' and
segment_name='WRH$_ACTIVE_SESSION_HISTORY' order by 3;

SEGMENT_NAME   PARTITION_NAME   SEGMENT_TYPE   MB
------------   --------------   ------------   --------
WRH$_ACTIVE_SESSION_HISTORY  WRH$_ACTIVE_329070140_0   TABLE PARTITION   3 
WRH$_ACTIVE_SESSION_HISTORY  WRH$_ACTIVE_SES_MXDB_MXSN   TABLE PARTITION   .0625

--重建分区索引
SQL> alter index WRH$_ACTIVE_SESSION_HISTORY_PK rebuild partition WRH$_ACTIVE_329070140_0; 
SQL> alter index WRH$_ACTIVE_SESSION_HISTORY_PK rebuild partition WRH$_ACTIVE_SES_MXDB_MXSN;
```

**References**
[1] https://www.shuzhiduo.com/A/Vx5M3K0pzN/ 
[2] https://blog.csdn.net/renyanjie123/article/details/115723348 
[3] http://blog.itpub.net/26148431/viewspace-2135213/ 
[4] MOS Doc ID 387914.1


