---
tags: [postgresql]
title: PostgreSQL逻辑备份与恢复
created: '2023-10-22T05:50:10.152Z'
modified: '2023-10-22T05:50:15.271Z'
---

PostgreSQL逻辑备份与恢复

PostgreSQL逻辑备份恢复的官方工具是pg_dump、pg_dumpall、pg_restore。
- **pg_dump**：支持对数据库或表进行备份，导出文件可以是SQL文件或归档文件（默认是纯SQL文本）。
- **pg_dumpall**：支持对整个数据库进行备份，只支持导出为SQL文件。
- **pg_restore**：用于将pg_dump导出的归档文件恢复到数据库。

# 逻辑备份：pg_dump
## 备份数据库

备份单个数据库：
```bash
#格式：pg_dump -h${DB_IP} -p${DB_PORT} -U ${BACKUP_USER} ${DBNAME} > dump.sql
pg_dump sekiro > sekiro_db.sql

#导出为可用于pg_restore恢复的dump文件（选项-Fc）
pg_dump -Fc sekiro > sekiro_db.dump
```

## 备份表

```bash
#备份指定表：-t
#格式：pg_dump -h${DB_IP} -p${DB_PORT} -U ${BACKUP_USER} -t ${SCHEMA.TABLENAME} ${DBNAME} > dump.sql

pg_dump -t public.map sekiro > sekiro_map.sql
pg_dump -t map sekiro > sekiro_map.sql          # public schema可省略

#备份指定表以外的表：-T
#格式：pg_dump -h${DB_IP} -p${DB_PORT} -U ${BACKUP_USER} -T ${SCHEMA.TABLENAME} ${DBNAME} > dump.sql

pg_dump -T map sekiro > sekiro_exclude_map.sql

#备份通配符匹配的表
pg_dump -t 'public.st*' sekiro > sekiro_table.sql    # 备份public schema中名称以st开头的表
pg_dump -T 'public.*day' sekiro > sekiro_table.sql   # 备份public schema中名称以day结尾的表以外的其他所有表
```

## 备份Schema

```bash
#备份指定Schema：-n，不能与-t结合使用
#格式：pg_dump -h${DB_IP} -p${DB_PORT} -U ${BACKUP_USER} -n ${SCHEMA} ${DBNAME} > dump.sql

pg_dump -n public sekiro > sekiro_public.sql

#备份指定Schema以外的Schema：-N
#格式：pg_dump -h${DB_IP} -p${DB_PORT} -U ${BACKUP_USER} -N ${SCHEMA} ${DBNAME} > dump.sql

pg_dump -N public sekiro > sekiro_exclude_public.sql
```

# 逻辑备份：pg_dumpall

pg_dumpall用于将一个数据库中的所有数据都备份到一个SQL文件中，还可以备份所有数据库的全局共用对象、数据库用户和角色权限。

```bash
#备份所有数据库
pg_dumpall > backup.sql

#备份全局角色和表空间
pg_dumpall --globals-only > globals.sql

#备份角色定义
pg_dumpall --roles-only > roles.sql
```

# 逻辑恢复：psql和pg_restore

PostgreSQL进行逻辑恢复可以用到psql和pg_restore工具。PSQL可以用来恢复pg_dump和pg_dumpall导出的SQL文本。PG_RESTORE可以用来恢复pg_dump导出的dump文件。

## 恢复数据库

**示例：PSQL恢复数据库**
```bash
psql -f backup.sql
psql < backup.sql
```

**示例：PG_RESTORE恢复数据库**
```bash
#备份数据库
pg_dump -Fc sekiro > sekiro_db.dump

#删除数据库
postgres=# drop database sekiro;
DROP DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(4 rows)


#恢复数据库（需要先创建库）
psql -c "create database sekiro with owner=sekiro tablespace=sekiro"
pg_restore -d sekiro sekiro_db.dump

#检查恢复后的数据库
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 sekiro    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(5 rows)

postgres=# \c sekiro sekiro
You are now connected to database "sekiro" as user "sekiro".
sekiro=> select * from map;
 map_id |    map_name     
--------+-----------------
      1 | Senhen Temple
      2 | Fountain Palace
(2 rows)
```


## 恢复表
恢复表时，如果用于恢复的备份文件不是单表备份，可以通过选项`-t`指定要恢复的表，选项`-d`指定要恢复到的数据库。

下面两个示例分别展示了使用psql和pg_restore工具恢复表的方法。

**示例：PSQL恢复表**
```bash
#导出map表备份
pg_dump -t map sekiro > sekiro_map.sql 

#删除map表
postgres=# \c sekiro sekiro
You are now connected to database "sekiro" as user "sekiro".
sekiro=> \dt
        List of relations
 Schema | Name  | Type  | Owner  
--------+-------+-------+--------
 public | map   | table | sekiro
 public | staff | table | sekiro
(2 rows)

sekiro=> drop table map;
DROP TABLE
sekiro=> \dt
        List of relations
 Schema | Name  | Type  | Owner  
--------+-------+-------+--------
 public | staff | table | sekiro
(1 row)

#恢复表（-d指定数据库）
psql -d sekiro < sekiro_map.sql 

#检查恢复后的数据
postgres=# \c sekiro sekiro
You are now connected to database "sekiro" as user "sekiro".
sekiro=> \dt
        List of relations
 Schema | Name  | Type  | Owner  
--------+-------+-------+--------
 public | map   | table | sekiro
 public | staff | table | sekiro
(2 rows)

sekiro=> select * from map;
 map_id |    map_name     
--------+-----------------
      1 | Senhen Temple
      2 | Fountain Palace
(2 rows)
```

**示例：PG_RESTORE恢复表**
```bash
#导出map表备份
pg_dump -t map -Fc sekiro > sekiro_map.dump 

#删除map表
postgres=# \c sekiro sekiro
You are now connected to database "sekiro" as user "sekiro".

sekiro=> drop table map;
DROP TABLE
sekiro=> \dt
        List of relations
 Schema | Name  | Type  | Owner  
--------+-------+-------+--------
 public | staff | table | sekiro
(1 row)

#恢复表（-d指定数据库）
pg_restore -d sekiro sekiro_map.dump

#检查恢复后的数据
postgres=# \c sekiro sekiro
You are now connected to database "sekiro" as user "sekiro".
sekiro=> \dt
        List of relations
 Schema | Name  | Type  | Owner  
--------+-------+-------+--------
 public | map   | table | sekiro
 public | staff | table | sekiro
(2 rows)

sekiro=> select * from map;
 map_id |    map_name     
--------+-----------------
      1 | Senhen Temple
      2 | Fountain Palace
(2 rows)
```

## 恢复Schema

恢复Schema与恢复表类似。使用pg_restore恢复schema时，使用`-n`选项来恢复指定的Schema。


