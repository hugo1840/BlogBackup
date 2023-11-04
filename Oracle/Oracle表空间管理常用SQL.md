---
tags: [oracle]
title: Oracle表空间管理常用SQL
created: '2023-11-04T02:07:11.421Z'
modified: '2023-11-04T02:19:28.081Z'
---

Oracle表空间管理常用SQL

# 用户表空间
查看用户的默认表空间名称：
```sql
select username,default_tablespace from dba_users;
```

查看表空间使用率：
```sql
set linesize 200
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

set linesize 200
select total.tablespace_name, round(total.max_mb/1024,2) max_gb, round(total.MB/1024,2) total_gb,
round(free.MB/1024,2) free_gb, round((total.MB - free.MB)/1024,2) used_gb,
round((1 - free.MB/total.MB)*100,2) as used_percent 
from 
(select tablespace_name,sum(bytes)/1024/1024 MB
from dba_free_space group by tablespace_name) free,
(select tablespace_name,sum(bytes)/1024/1024 MB, sum(maxbytes)/1024/1024 max_mb
from dba_data_files group by tablespace_name) total
where free.tablespace_name = total.tablespace_name 
order by used_percent desc;


--> 删除数据后，数据文件不会缩小，推荐使用下面的语句
select a.tablespace_name tbsname,
    round((max - (a.alloc - nvl (b.free, 0)))/1024/1024/1024, 2) free_gb,
    round ((a.alloc - nvl (b.free, 0)) / decode (max, 0, 1, max) * 100) pct_used,
    round(max/1024/1024/1024,2) max_gb from (  
	    select t.tablespace_name,
            sum (bytes) alloc,
            sum (
                case
                    when autoextensible = 'YES' then maxbytes
                    when autoextensible = 'NO' then bytes
                    end) max
                        from dba_data_files f
                        join dba_tablespaces t
                        on (f.tablespace_name = t.tablespace_name
                            and t.contents in ('PERMANENT','UNDO'))
                        group by t.tablespace_name) a,
                        (select ts.name tablespace_name, sum (fs.blocks) * ts.blocksize free
                            from DBA_LMT_FREE_SPACE fs, sys.ts$ ts
                            where ts.ts# = fs.tablespace_id
                            group by ts.name, ts.blocksize) b
                        where a.tablespace_name = b.tablespace_name(+);
```
						
查看数据文件大小以及是否自动扩展：
```sql
set lines 200
col file_name for a80
select file_name,tablespace_name,bytes/1024/1024 total_mb,maxbytes/1024/1024 max_mb,autoextensible 
from dba_data_files where tablespace_name='APPDATA';
```


数据表空间扩容（OMF模式）：
```sql
alter tablespace <tablespaceName> add datafile;
```

# UNDO表空间
查看UNDO表空间文件以及是否自动扩展：
```sql
show parameter undo

set lines 200
col file_name for a80
select file_name,tablespace_name,bytes/1024/1024 total_mb,maxbytes/1024/1024 max_mb,autoextensible 
from dba_data_files where tablespace_name='UNDOTBS1';
```

如果**undo_management**值为AUTO或者null，表示启用了自动undo管理。**undo_retention**表示undo数据保留的最短时间（秒），如果undo空间不足，undo表空间会自动扩展（需要开启autoextend）。


# 临时表空间
查看临时表空间文件以及是否自动扩展：
```sql
set linesize 200
col file_name format a60
select tablespace_name,file_name,bytes/1024/1024 size_mb,
maxbytes/1024/1024 max_mb,autoextensible from dba_temp_files;
```

查看临时表空间大小以及使用率：
```sql
select a.tablespace_name,
  round((b.max_size - (b.alloc_size - nvl(a.free_space, 0)))/1024/1024/1024, 2) free_gb,
  round((b.alloc_size - nvl(a.free_space, 0)) / decode(max_size, 0, 1, max_size) * 100) pct_used,
  round(max_size/1024/1024/1024,2) max_gb
from (select tablespace_name,free_space from dba_temp_free_space) a,
(select tablespace_name,sum(bytes) alloc_size,sum(
        case
        when autoextensible = 'YES' then maxbytes
        when autoextensible = 'NO' then bytes
        end
) max_size from dba_temp_files group by tablespace_name) b
where a.tablespace_name = b.tablespace_name;
```

临时表空间扩容：
```sql
alter tablespace temp add tempfile;
```


# 表空间历史使用情况
检查问题时间点临时表空间使用情况：
```sql
define begin_time='0324-09:00:00';
define end_time='0324-10:00:00';

select instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss') sample_time,
round(sum(TEMP_SPACE_ALLOCATED)/1024/1024/1024,2) temp_used_gb
from dba_hist_active_sess_history
where sample_time>=to_date('&begin_time','mmdd-hh24:mi:ss')
and sample_time<to_date('&end_time','mmdd-hh24:mi:ss')
group by instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss')
order by sample_time,instance_number,temp_used_gb desc; 
```

查看指定时间点temp表空间使用量超过10GB的SQL：
```sql
select instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss') sample_time,sql_id,count(*),
round(sum(TEMP_SPACE_ALLOCATED)/1024/1024/1024,2) temp_used_gb
from dba_hist_active_sess_history
where sample_time>=to_date('&begin_time','mmdd-hh24:mi:ss')
and sample_time<to_date('&end_time','mmdd-hh24:mi:ss')
group by instance_number,to_char(sample_time,'yyyymmdd hh24:mi:ss'),sql_id
having round(sum(TEMP_SPACE_ALLOCATED)/1024/1024/1024,2)>10
order by sample_time,temp_used_gb desc; 
```


# 统计表的大小
查看表大小：
```sql
--非分区表
select owner, segment_name, sum(table_size) || 'g'
from (
      select owner, segment_name, round(bytes / 1024 / 1024 / 1024, 2) as table_size
      from dba_segments
      where segment_name = upper('TABLE_NAME')
        and owner = 'USERNAME')
group by owner, segment_name;

--分区表
set lines 200
col owner for a15
col partition_size_gb for a20
select owner,segment_name,partition_name,sum(size_gb) || 'G' as partition_size_gb
from 
(select owner,segment_name,partition_name,round(bytes/1024/1024/1024,2) as size_gb
from dba_segments where segment_name=upper('xxx') and owner='xxx')
group by owner,segment_name,partition_name
order by partition_name;
```




