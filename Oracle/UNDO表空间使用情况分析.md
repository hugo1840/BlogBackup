---
tags: [oracle]
title: UNDO表空间使用情况分析
created: '2023-06-12T11:54:13.211Z'
modified: '2023-06-12T12:10:51.543Z'
---

UNDO表空间使用情况分析

# 用量分析
检查表空间用量： 
```sql
select a.tablespace_name tbsname, 
round((max - (a.alloc - nvl (b.free, 0)))/1024/1024/1024, 2) free_gb, 
round ((a.alloc - nvl(b.free,0)) / decode (max,0,1,max) * 100) percnt_used, 
round(max/1024/1024/1024,2) max_gb 
from (
  select t.tablespace_name, sum(bytes) alloc, 
  sum( 
    case 
      when autoextensible = 'YES' then maxbytes 
      when autoextensible = 'NO' then bytes 
    end) max 
  from dba_data_files f join dba_tablespaces t 
  on (f.tablespace_name = t.tablespace_name and t.contents in ('PERMANENT','UNDO')) 
  group by t.tablespace_name) a, 
  (
  select ts.name tablespace_name, sum (fs.blocks) * ts.blocksize free 
  from DBA_LMT_FREE_SPACE fs, sys.ts$ ts 
  where ts.ts# = fs.tablespace_id group by ts.name, ts.blocksize) b 
where a.tablespace_name = b.tablespace_name(+);
```

# 扩容UNDO
检查UNDO表空间管理模式：
```sql
show parameter undo
```
如果`undo_management`值为**AUTO**或者**NULL**，表示启用了自动undo管理；`undo_retention`表示undo数据保留的最短时间（秒）。

查看UNDO表空间文件以及是否自动扩展：
```sql
set lines 200 
col file_name for a80 
select file_name,tablespace_name,
bytes/1024/1024 total_mb,maxbytes/1024/1024 max_mb,autoextensible 
from dba_data_files where tablespace_name like 'UNDOTBS%';
```

扩容undo表空间：
```sql
alter tablespace undotbs1 add datafile;
```

# 追踪用户
统计当前正在使用回滚段的用户：
```sql
set lines 200 
col username for a20 
col program for a20 
col machine for a30 
select s.username,s.sql_id,s.sql_hash_value, 
substr(s.program,1,25) program,s.machine, 
u.name as rollback_segment_name, 
r.rssize/1024/1024 as rollback_segment_mb, r.status 
from v$transaction t, v$rollstat r, v$rollname u, v$session s 
where s.taddr=t.addr and t.xidusn=r.usn and r.usn=u.usn 
order by s.username,r.rssize desc;
```

查看占用回滚段最多的SQL文本： 
```sql
select sql_fulltext from v$sqlarea where sql_id='xxxxxx';
```



