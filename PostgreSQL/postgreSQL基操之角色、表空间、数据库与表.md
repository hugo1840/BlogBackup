---
tags: [postgresql]
title: postgreSQL基操之角色、表空间、数据库与表
created: '2023-10-12T08:16:16.570Z'
modified: '2023-10-12T08:16:44.427Z'
---

postgreSQL基操之角色、表空间、数据库与表

# 角色创建与管理

PostgreSQL数据库里没有User的概念，只有Role的概念。有的Role可以用于登录数据库，这些Role与其他数据库中的用户等价。
```sql
--创建可以登录的角色
create role sekiro with login password 'shadowDie2';

--创建可以登录的角色并赋予创建数据库的权限
create role dba createdb login password 'shadowDie2';

--创建可以登录的角色并设定密码有效期
create role ishin with login password 'shadowDie2' valid until '2023-10-12';

--创建可以登录的角色并设定并发连接上限
create role genji with login password 'shadowDie2' connection limit 100;
```

使用角色登录数据库：
```bash
#psql -U 角色名称 -W 数据库名称
psql -U sekiro -W postgres
```

列出已有的角色：
```sql
postgres=# \du
                             List of roles
 Role name |                   Attributes                   | Member of 
-----------+------------------------------------------------+-----------
 dba       | Create DB                                      | {}
 genji     | 100 connections                                | {}
 ishin     | Password valid until 2023-10-12 00:00:00+08    | {}
 postgres  | Superuser, Create role, Create DB, Replication | {}
 sekiro    |                                                | {}

postgres=# select rolname,rolcreatedb,rolconnlimit,rolcanlogin from pg_roles;
 rolname  | rolcreatedb | rolconnlimit | rolcanlogin 
----------+-------------+--------------+-------------
 postgres | t           |           -1 | t
 sekiro   | f           |           -1 | t
 dba      | t           |           -1 | t
 ishin    | f           |           -1 | t
 genji    | f           |          100 | t
(5 rows)
```

移除角色：
```sql
postgres=# drop role genji;
DROP ROLE
```

# 表空间创建与管理
创建表空间必须是SUPERUSER角色。创建表空间并指定属主：
```sql
# 指定的location必须事先存在
postgres=# create tablespace sekiro owner sekiro location '/pgdata/sekiro';
CREATE TABLESPACE

postgres=# \db
          List of tablespaces
    Name    |  Owner   |    Location    
------------+----------+----------------
 pg_default | postgres | 
 pg_global  | postgres | 
 sekiro     | sekiro   | /pgdata/sekiro
(3 rows)
```

修改表空间：
```sql
--重命名表空间
ALTER TABLESPACE sekiro RENAME TO wolf;

--修改属主
ALTER TABLESPACE sekiro OWNER TO ishin;
```

移除表空间：
```sql
postgres=# drop tablespace sekiro;
DROP TABLESPACE
```

# 数据库创建与管理
创建数据库需要CREATEDB权限或者SUPERUSER角色。创建数据库并指定属主和表空间：
```sql
create database sekiro
with owner=sekiro tablespace=sekiro encoding='UTF8';
```

列出已有的数据库：
```sql
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 sekiro    | sekiro   | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

# 表创建与管理
登录数据库：
```
psql -U sekiro -W sekiro
```

创建表：
```sql
CREATE TABLE staff(
    staff_id SERIAL PRIMARY KEY,
    first_name VARCHAR(45) NOT NULL,
    last_name VARCHAR(45) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE
);
```

检查当前数据库中的表：
```sql
sekiro=> \dt
        List of relations
 Schema | Name  | Type  | Owner  
--------+-------+-------+--------
 public | staff | table | sekiro
(1 row)

sekiro=> insert into staff(staff_id,first_name,last_name,email) values (1,'Kuro','Satoshi','kuro.satoshi@sekiro.com');
INSERT 0 1

sekiro=> select * from staff;
 staff_id | first_name | last_name |          email          
----------+------------+-----------+-------------------------
        1 | Kuro       | Satoshi   | kuro.satoshi@sekiro.com
(1 row)

sekiro=> \dt+
                     List of relations
 Schema | Name  | Type  | Owner  |    Size    | Description 
--------+-------+-------+--------+------------+-------------
 public | staff | table | sekiro | 8192 bytes | 
(1 row)
```

将表的查询权限授予其他用户：
```bash
[postgres@dbhost pgdata]$ psql -U ishin -W sekiro
Password for user ishin: 
psql (9.2.4)
Type "help" for help.

sekiro=> select * from staff;
ERROR:  permission denied for relation staff

sekiro=> \q

[postgres@dbhost pgdata]$ psql -U sekiro -W sekiro
Password for user sekiro: 
psql (9.2.4)
Type "help" for help.

sekiro=> grant select on staff to ishin;
GRANT
```

**References **
【1】https://www.postgresqltutorial.com/postgresql-administration/postgresql-schema/



