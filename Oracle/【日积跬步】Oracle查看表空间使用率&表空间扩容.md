---
tags: [oracle]
title: 【日积跬步】Oracle查看表空间使用率&表空间扩容
created: '2023-01-18T12:55:42.413Z'
modified: '2023-01-18T14:16:26.342Z'
---

【日积跬步】Oracle查看表空间使用率&表空间扩容


查看所有表空间名称：
```sql
SQL> select tablespace_name from dba_tablespaces;
```

查看表空间使用率：
```sql
SQL> set linesize 200
SQL> select total.tablespace_name, round(total.size_mb,2) total_mb, 
round(total.size_mb - free.size_mb,2) used_mb,
round((1 - free.size_mb/total.size_mb)*100,2) || '%' as used_percent
from (
  select tablespace_name, sum(bytes)/1024/1024 size_mb from dba_free_space 
  group by tablespace_name
  ) free,
  (select tablespace_name, sum(bytes)/1024/1024 size_mb from dba_data_files
  group by tablespace_name
  ) total
where free.tablespace_name = total.tablespace_name
order by used_percent desc;
```

查看数据文件大小以及是否自动扩展：
```sql
SQL> select file_name, tablespace_name, bytes/1024/1024 total_mb,
maxbytes/1024/1024 max_mb, autoextensible
from dba_data_files where tablespace_name='XXX';
```

表空间扩容（以OMF为例）：
```sql
--OMF文件管理模式下，单个数据文件达到自动扩展的上限后，需要通过新增数据文件来对表空间扩容
SQL> alter tablespace <tablespace_name> add datafile;
```

查看临时表空间大小以及使用率：
```sql
SQL> select tablespace_name, tablespace_size/1024/1024 size_mb, 
free_space/1024/1024 free_mb, 
round((1 - nvl(free_space,0)/tablespace_size)*100,2) used_percent 
from dba_temp_free_space;
```

查看临时表空间文件以及是否自动扩展：
```sql
SQL> set linesize 200
SQL> col file_name format a60
SQL> select tablespace_name, file_name, bytes/1024/1024 size_mb,
maxbytes/1024/1024 max_mb, autoextensible from dba_temp_files;
```

临时表空间扩容（以OMF为例）：
```sql
SQL> alter tablespace temp add tempfile;
```



