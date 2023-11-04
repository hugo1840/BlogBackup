---
tags: [达梦]
title: 达梦数据库表空间管理常用SQL
created: '2023-11-04T02:33:01.404Z'
modified: '2023-11-04T02:34:55.412Z'
---

达梦数据库表空间管理常用SQL

查看数据库状态：
```sql
select name,instance_name,status$,mode$ from v$instance;
--mode$显示Primary为主库

select name,status$,role$ from v$database;
--status$：1 启动，2 启动Redo完成，3 Mount，4 Open，5 挂起，6 关闭
```

# 表空间容量分析
查数据文件大小信息：
```sql
select client_path,total_size*32/1024 TOTAL_MB,
free_size*32/1024 FREE_MB,status$,max_size,auto_extend from v$datafile;

select client_path,total_size*32/1024/1024 TOTAL_GB,
free_size*32/1024/1024 FREE_GB,status$,
max_size/1024 as MAX_GB,auto_extend from v$datafile;
```

查表空间大小信息：
```sql
select id,name,(total_size*page)/1024/1024/1024 total_GB,
(used_size*page)/1024/1024/1024 used_GB,
((total_size-used_size)*page)/1024/1024/1024 free_GB, 
max_size from v$tablespace;

--> 真实的表空间使用率
select a.id,a.name,
round((a.total_size*page)/1024/1024/1024,2) total_GB,
round(b.max_mb/1024,2) max_GB,
round((a.used_size*page)/1024/1024/1024,2) used_GB,
round((b.max_mb - a.used_size*page/1024/1024)/1024,2) free_GB, 
round((a.used_size*page)/1024/1024/b.max_mb*100,2) || '%' used_pct
from v$tablespace a,
( select group_id, sum(max_size) max_mb from v$datafile group by group_id
) b where a.id=b.group_id;
```


# 表空间创建与扩容
创建用户表空间：
```sql
--> 需要手动指定数据文件名，建议统一命名格式为：USERNAME_TIMESTAMP.dbf
create tablespace TABLESPACE_NAME datafile 'DATAFILE_NAME_1.dbf' 
size 200 autoextend on maxsize 256*1024;
```

为指定用户扩容表空间，需要手动指定数据文件名。
```sql
alter tablespace TABLESPACE_NAME add datafile 'DATAFILE_NAME_2.dbf' 
size 200 autoextend on maxsize 256*1024;
```

修改数据文件大小：
```sql
alter tablespace "表空间名" resize datafile '数据文件名' to 1128;
--> 例如将数据文件DMHR.DBF大小指定为1128MB：alter tablespace "DMHR" resize datafile 'DMHR.DBF' to 1128;
```

>:sunflower: 更多内容参见：https://blog.csdn.net/Sebastien23/article/details/131146304





