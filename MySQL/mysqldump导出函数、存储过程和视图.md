---
tags: [mysql]
title: mysqldump导出函数、存储过程和视图
created: '2024-01-01T12:43:23.524Z'
modified: '2024-01-01T12:50:53.835Z'
---

mysqldump导出函数、存储过程和视图

# 导出函数和存储过程

查看函数和存储过程：
```sql
select routine_schema,routine_name,routine_type 
from information_schema.routines 
where routine_schema='DBNAME' and 
routine_type in ('FUNCTION','PROCEDURE');
```

mysqldump导出函数和存储过程（不导出表数据）：
```bash
mysqldump -h127.0.0.1 -P3306 -uroot --set-gtid-purged=OFF \
--single-transaction --skip-opt \
--routines --no-create-db --no-create-info --no-data \
-B DBNAME > dump_routines_DBNAME_`date +%F`.sql
```
或者
```bash
mysqldump -h127.0.0.1 -P3306 -uroot --set-gtid-purged=OFF \
--single-transaction --skip-opt \
-R -ndt -B DBNAME > dump_routines_DBNAME_`date +%F`.sql
```

生成删除函数和存储过程的语句：
```sql
select 'DROP ' || concat_ws(' ',routine_type,concat_ws('.',routine_schema,routine_name)) || ';'  as SQLTEXT 
from information_schema.routines
where routine_schema='DBNAME' and 
routine_type in ('FUNCTION','PROCEDURE');
```

修改并确认dump文件后，导入存储过程和函数：
```bash
mysql -h127.0.0.1 -P3306 --uroot -e "use DBNAME; source /backup/dump_routines_DBNAME_xxx.sql;"
```


# 导出视图定义

检查数据库下面所有视图：
```sql
select table_schema,table_name from information_schema.views 
where table_schema='DBNAME';
```

获取并拼接所有视图名称：
```sql
select group_concat(concat_ws('.',table_schema,table_name) separator ',')
from information_schema.views where table_schema='DBNAME'\G
```

mysqldump单独导出视图/表定义：
```bash
mysqldump -h127.0.0.1 -P3306 -uroot --set-gtid-purged=OFF \
--single-transaction --skip-opt \
--no-create-db --no-data \
DBNAME VIEW1 VIEW2 VIEW3 ... > dump_views_DBNAME_`date +%F`.sql
```

生产删除视图的语句：
```sql
select 'DROP VIEW ' || concat_ws('.',table_schema,table_name) || ';' as SQLTEXT
from information_schema.views where table_schema='DBNAME'; 
```

修改并确认dump文件后导入视图：
```bash
mysql -h127.0.0.1 -P3306 --uroot -e "use DBNAME; source /backup/dump_views_DBNAME_xxx.sql;"
```

>:snake:**注**：MySQL Shell导入视图定义（尤其是嵌套视图）会出现权限问题，官方不推荐。




