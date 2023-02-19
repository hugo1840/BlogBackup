---
tags: [oracle]
title: 记一次TEMP表空间撑爆的SQL执行计划分析
created: '2023-02-14T15:15:10.024Z'
modified: '2023-02-19T01:45:13.697Z'
---

记一次TEMP表空间撑爆的SQL执行计划分析

# 定位问题时间点
检查alert日志，定位问题发生时间点（同时跟应用复核确认）：
```bash
cd $ORACLE_BASE/diag/rdbms/<DB_UNIQUE_NAME>/<INSTANCE_NAME>/trace

tail -n 50000 alert_<INSTANCE_NAME>.log | less
tail -n 50000 alert_<INSTANCE_NAME>.log > /tmp/alert_01.log

# 如果alert日志太大，可以把问题时间点附近的日志单独摘出来
cat /tmp/alert_01.log | grep -n 'unable to extend temp'
sed -n '6780,8320p' /tmp/alert_01.log > /tmp/alert_02.log
more /tmp/alert_02.log
```

# 检查表空间使用情况
检查问题时间点附近临时表空间的使用情况。

如果问题时间点在`min(sample_time)`之后，则可以查询`v$active_session_history`视图；否则，需要查询`dba_hist_active_sess_history`视图。
```sql
select min(sample_time) from v$active_session_history;
```

检查**当前**临时表空间的使用情况：
```sql
select tablespace_name,round(sum(bytes)/1024/1024/1024,2) current_gb,
round(sum(decode(maxbytes,0,bytes,maxbytes))/1024/1024/1024,2) max_gb 
from dba_temp_files
group by tablespace_name order by tablespace_name;

select tablespace_name,round(tablespace_size/1024/1024/1024,2) total_gb,
round(allocated_space/1024/1024/1024,2) allocated_gb,
round(free_space/1024/1024/1024,2) free_gb
from dba_temp_free_space order by tablespace_name;

select a.inst_id,
a.tablespace_name,
round(a.total_blocks*b.block_size/1024/1024/1024,2) total_gb,
round(a.used_blocks*b.block_size/1024/1024/1024,2) active_gb,
round(a.free_blocks*b.block_size/1024/1024/1024,2) free_gb,
round(a.max_blocks*b.block_size/1024/1024/1024,2) max_gb,
round(a.max_used_blocks*b.block_size/1024/1024/1024,2) max_used_gb,
round(a.max_sort_blocks*b.block_size/1024/1024/1024,2) max_sort_gb
from gv$sort_segment a,dba_tablespaces b
where a.tablespace_name=b.tablespace_name
order by 1,2;

select a.inst_id,b.sid,b.serial#,a.tablespace,a.contents,a.segtype,b.sql_id,
round(a.blocks*c.block_size/1024/1024/1024,2) temp_used_gb
from gv$tempseg_usage a,gv$session b,dba_tablespaces c
where a.inst_id=b.inst_id(+)
and a.session_addr=b.saddr(+)
and a.session_num=b.serial#(+)
and a.tablespace=c.tablespace_name
and round(a.blocks*c.block_size/1024/1024/1024,2)>0
order by temp_used_gb desc;
```

检查**问题时间点**临时表空间的大小：
```sql
--查看表空间编号
select ts#,name,bigfile from v$tablespace where name='TEMP';

--检查问题时间点临时表空间大小
select round(sum(bytes)/1024/1024/1024,2) size_gb 
from v$tempfile
where ts# = 3 and 
creation_time <= to_date('2023-02-07 03:30:00','yyyy-mm-dd hh24:mi:ss');
```

检查**问题时间点**临时表空间的使用情况：
```sql
define begin_time='0207-03:31:00';
define end_time='0207-03:32:00';

--查看指定时间点的temp表空间使用情况
select instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss') sample_time,
round(sum(TEMP_SPACE_ALLOCATED)/1024/1024/1024,2) temp_used_gb
from dba_hist_active_sess_history
where sample_time>=to_date('&begin_time','mmdd-hh24:mi:ss')
and sample_time<to_date('&end_time','mmdd-hh24:mi:ss')
group by instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss')
order by sample_time,instance_number,temp_used_gb desc; 

--查看指定时间点temp表空间使用量超过10GB的SQL
select instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss') sample_time,
sql_id,count(*),
round(sum(TEMP_SPACE_ALLOCATED)/1024/1024/1024,2) temp_used_gb
from dba_hist_active_sess_history
where sample_time>=to_date('&begin_time','mmdd-hh24:mi:ss')
and sample_time<to_date('&end_time','mmdd-hh24:mi:ss')
group by instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss'),sql_id
having round(sum(TEMP_SPACE_ALLOCATED)/1024/1024/1024,2)>10
order by sample_time,temp_used_gb desc; 

--查看指定时间点temp表空间相关的会话信息
set lines 200 
col sess for a20
col qc_sess for a15
col sql_plan_operation for a20
col sql_plan_options for a20
select to_char(sample_time,'yyyymmdd hh24:mi:ss') sample_time,
instance_number || '_' || session_id || '_' || session_serial# sess,
QC_INSTANCE_ID || '-' || QC_SESSION_ID || '-' || QC_SESSION_SERIAL# qc_sess,
sql_id,sql_plan_hash_value,sql_plan_line_id,sql_plan_operation,sql_plan_options,
round(TEMP_SPACE_ALLOCATED/1024/1024/1024,2) temp_used_gb
from dba_hist_active_sess_history
where sample_time>=to_date('&begin_time','mmdd-hh24:mi:ss')
and sample_time<to_date('&end_time','mmdd-hh24:mi:ss')
and round(TEMP_SPACE_ALLOCATED/1024/1024/1024,2)>0
order by sample_time,instance_number,temp_used_gb desc; 

SAMPLE_TIME      SESS               QC_SESS           SQL_ID         SQL_PLAN_HASH_VALUE SQL_PLAN_LINE_ID SQL_PLAN_OPERATION   SQL_PLAN_OPTIONS     TEMP_USED_GB
----------------- -------------------- --------------- ------------- ------------------- ---------------- -------------------- -------------------- ------------
...
...
```
重点关注问题时间段占用TEMP表空间较大的`SQL_ID`，将其定位为问题目标SQL。

# 分析目标SQL历史执行计划
查看从问题发生时间段以来目标SQL的执行信息，包括执行计划、执行次数、资源消耗、等待时长，等等。

```sql
set line 300 pages 5000 timing off feedback off long 100000000 longchunk 100000000
col i for 9
col timestamp for a17
col exec_date for a9
col parse_user for a20

define begin_time='020603';
define end_time='021303';
define sql_id='xxxxxxxx';

select 
    a.instance_number i,
    a.parsing_schema_name parse_user,
    to_char(b.begin_interval_time,'yyyymmdd') exec_date,
    a.plan_hash_value,
    sum(a.EXECUTIONS_DELTA) total_executions,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.fetches_delta),
    sum(a.fetches_delta)/sum(a.EXECUTIONS_DELTA))) fetches_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.rows_processed_delta),
    sum(a.rows_processed_delta)/sum(a.EXECUTIONS_DELTA))) rows_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.BUFFER_GETS_DELTA),
    sum(a.BUFFER_GETS_DELTA)/sum(a.EXECUTIONS_DELTA))) buffergets_avg,    
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.DISK_READS_DELTA),
    sum(a.DISK_READS_DELTA)/sum(a.EXECUTIONS_DELTA))) diskreads_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.ELAPSED_TIME_DELTA)/1000,
    (sum(a.ELAPSED_TIME_DELTA)/sum(a.EXECUTIONS_DELTA))/1000),2) elapsedms_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.CPU_TIME_DELTA)/1000,
    (sum(a.CPU_TIME_DELTA)/sum(a.EXECUTIONS_DELTA))/1000),2) cpums_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.IOWAIT_DELTA)/1000,
    (sum(a.IOWAIT_DELTA)/sum(a.EXECUTIONS_DELTA))/1000),2) ioms_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.CLWAIT_DELTA)/1000,
    (sum(a.CLWAIT_DELTA)/sum(a.EXECUTIONS_DELTA))/1000),2) clusterms_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.APWAIT_DELTA)/1000,
    (sum(a.APWAIT_DELTA)/sum(a.EXECUTIONS_DELTA))/1000),2) applicationms_avg,
    round(decode(sum(a.EXECUTIONS_DELTA),0,sum(a.CCWAIT_DELTA)/1000,
    (sum(a.CCWAIT_DELTA)/sum(a.EXECUTIONS_DELTA))/1000),2) concurrencyms_avg    
from dba_hist_sqlstat a,dba_hist_snapshot b
where a.SNAP_ID =b.SNAP_ID
and a.INSTANCE_NUMBER=b.INSTANCE_NUMBER
and b.begin_interval_time>=to_date('&begin_time','mmddhh24')
and b.end_interval_time<=to_date('&end_time','mmddhh24')+10/60/24
and a.sql_id='&sql_id' 
group by 
a.instance_number,a.parsing_schema_name,to_char(b.begin_interval_time,'yyyymmdd'),a.plan_hash_value
order by 2,1,3;


 I PARSE_USER        EXEC_DATE PLAN_HASH_VALUE TOTAL_EXECUTIONS FETCHES_AVG     ROWS_AVG BUFFERGETS_AVG DISKREADS_AVG ELAPSEDMS_AVG  CPUMS_AVG   IOMS_AVG CLUSTERMS_AVG APPLICATIONMS_AVG CONCURRENCYMS_AVG
-- -------------------- --------- --------------- ---------------- ----------- ---------- -------------- ------------- ------------- ---------- ---------- ------------- ----------------- -----------------
...
...
```
按时间顺序查看问题发生时间段以来目标SQL的执行计划的变化情况，以及各个执行计划对应的资源消耗和执行效率。

查看目标SQL的所有执行计划：
```sql
define sql_id='xxxxxxxx';
select * from table(dbms_xplan.display_awr('&sql_id'));
```

关注是否有因为使用临时表无法收集统计信息，而导致执行计划中对结果集**Rows**的评估与实际情况误差较大，最终选错执行计划的情况。
```sql
--查看是否为临时表
select owner,table_name,num_rows,blocks,avg_row_len,
to_char(last_analyzed,'yyyymmdd hh24:mi:ss') as analyzed,
partitioned,temporary,
round(num_rows*avg_row_len/1024/1024/1024/0.9,2) est_GB 
from dba_tables 
where owner='APPX' and table_name='TMP_HJKHG_T';

OWNER                   TABLE_NAME            NUM_ROWS     BLOCKS AVG_ROW_LEN ANALYZED      PAR T     EST_GB
------------------------------ ------------------------------ ---------- ---------- ----------- ----------------- --- - ----------
...
...
```

# 绑定执行计划
使用官方MOS文档提供的`coe_xfr_sql_profile.sql`脚本来绑定执行计划。

如果目标SQL在问题时间点以后自动调整为了更优的执行计划，可以将其绑定到目标SQL。或者我们也可以自己从历史执行计划中选择最优的。

```sql
@/home/oracle/coe_xfr_sql_profile.sql
--输入目标sql_id
--从给出的执行计划中选择plan_hash_value
--最后执行生成的执行计划绑定脚本
@/home/oracle/coe_xfr_sql_profile_<sql_id>_<plan_hash_value>.sql
```


