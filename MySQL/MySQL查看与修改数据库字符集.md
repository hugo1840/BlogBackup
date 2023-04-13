---
tags: [mysql]
title: MySQL查看与修改数据库字符集
created: '2023-04-13T11:36:34.974Z'
modified: '2023-04-13T14:14:53.626Z'
---

MySQL查看与修改数据库字符集

# utf8 vs utf8mb3 vs utf8mb4
根据MySQL官方文档解释，目前MySQL中的utf8字符集，实际上是utf8mb3字符集，即用3个字节的Unicode编码；而utf8mb4才是真正意义上的4个字节的UTF8编码。

不过在较新的MySQL版本（8.0.32）中，已经只能查询到utf8mb3和utf8mb4两个UTF8编码，而看不到名为utf8的字符集。
```sql
SQL > show character set like 'utf8%';

+---------+---------------+--------------------+--------+
| Charset | Description   | Default collation  | Maxlen |
+---------+---------------+--------------------+--------+
| utf8mb3 | UTF-8 Unicode | utf8mb3_general_ci |      3 |
| utf8mb4 | UTF-8 Unicode | utf8mb4_0900_ai_ci |      4 |
+---------+---------------+--------------------+--------+
```

>:dolphin:https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb3.html


# 查看字符集和排序规则
查看数据库默认字符集和排序规则：
```sql
SQL > select * from information_schema.schemata 
where schema_name in ('app_game','app_work','appdb');

+--------------+-------------+----------------------------+------------------------+----------+--------------------+
| CATALOG_NAME | SCHEMA_NAME | DEFAULT_CHARACTER_SET_NAME | DEFAULT_COLLATION_NAME | SQL_PATH | DEFAULT_ENCRYPTION |
+--------------+-------------+----------------------------+------------------------+----------+--------------------+
| def          | appdb       | utf8mb3                    | utf8mb3_general_ci     | NULL     | NO                 |
| def          | app_game    | utf8mb3                    | utf8mb3_general_ci     | NULL     | NO                 |
| def          | app_work    | utf8mb3                    | utf8mb3_general_ci     | NULL     | NO                 |
+--------------+-------------+----------------------------+------------------------+----------+--------------------+
```

查看表的排序规则（通过排序规则可以看出字符集）：
```sql
 SQL > select table_schema,table_name,engine,table_collation from information_schema.tables 
 where table_schema in ('app_game','app_work','appdb');

+--------------+------------+--------+--------------------+
| TABLE_SCHEMA | TABLE_NAME | ENGINE | TABLE_COLLATION    |
+--------------+------------+--------+--------------------+
| app_game     | persons    | InnoDB | utf8mb4_0900_ai_ci |
| appdb        | temp_seq   | InnoDB | utf8mb3_general_ci |
| appdb        | test_table | InnoDB | utf8mb3_general_ci |
+--------------+------------+--------+--------------------+
```

# 默认字符集的继承关系
查看建库时指定的字符集：
```sql
SQL > show variables like 'character_set_server';
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| character_set_server | utf8mb3 |
+----------------------+---------+

SQL > show create database appdb;
+----------+------------------------------------------------------------------------------------------------------+
| Database | Create Database                                                                                      |
+----------+------------------------------------------------------------------------------------------------------+
| appdb    | CREATE DATABASE `appdb` /*!40100 DEFAULT CHARACTER SET utf8mb3 */ /*!80016 DEFAULT ENCRYPTION='N' */ |
+----------+------------------------------------------------------------------------------------------------------+

SQL > show create database app_game;
+----------+---------------------------------------------------------------------------------------------------------+
| Database | Create Database                                                                                         |
+----------+---------------------------------------------------------------------------------------------------------+
| app_game | CREATE DATABASE `app_game` /*!40100 DEFAULT CHARACTER SET utf8mb3 */ /*!80016 DEFAULT ENCRYPTION='N' */ |
+----------+---------------------------------------------------------------------------------------------------------+
```

:eagle:其中：
- 服务器级别的默认字符集为`utf8mb3`；
- 数据库`appdb`和`app_game`建库时都未指定字符集，因而沿用了服务器级别的字符集`utf8mb3`。


查看建表时指定的字符集和排序规则：
```sql
SQL > show create table app_game.persons;

+---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                                                               |
+---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| persons | CREATE TABLE `persons` (
  `id` int NOT NULL,
  `name` varchar(50) NOT NULL,
  `remark` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
+---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.0036 sec)

SQL > show create table appdb.temp_seq;

+----------+-------------------------------------------------------------------------------------------------------------+
| Table    | Create Table                                                                                                |
+----------+-------------------------------------------------------------------------------------------------------------+
| temp_seq | CREATE TABLE `temp_seq` (
  `id` int NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 |
+----------+-------------------------------------------------------------------------------------------------------------+
```

:eagle:其中:
- 数据库`appdb`和`app_game`的默认字符集都是`utf8mb3`；
- `app_game.persons`在建表时指定了字符集`utf8mb4`，覆盖了数据库级别的默认字符集`utf8mb3`；
- `appdb.temp_seq`在建表时未指定字符集（上面的`DEFAULT CHARSET=utf8mb3`是MySQL自动补充的），因而沿用了数据库级别的默认字符集`utf8mb3`。


# 修改字符集和排序规则
## 修改数据库的默认字符集
修改数据库的默认字符集和排序规则：
```sql
SQL > alter database appdb character set utf8mb4;
Query OK, 1 row affected (0.0038 sec)

SQL > show create database appdb;

+----------+---------------------------------------------------------------------------------------------------------------------------------+
| Database | Create Database                                                                                                                 |
+----------+---------------------------------------------------------------------------------------------------------------------------------+
| appdb    | CREATE DATABASE `appdb` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci */ /*!80016 DEFAULT ENCRYPTION='N' */ |
+----------+---------------------------------------------------------------------------------------------------------------------------------+
```

创建一张新表，不指定字符集：
```sql
SQL > create table appdb.animals (
                               -> `id` int not null,
                               -> `name` varchar(30) not null,
                               -> primary key(`id`)
                               -> ) engine=InnoDB;
                        
SQL > show create table appdb.animals;

+---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                         |
+---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| animals | CREATE TABLE `animals` (
  `id` int NOT NULL,
  `name` varchar(30) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
+---------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

查看appdb中所有表的字符集：
```sql
SQL > select table_schema,table_name,table_collation 
from information_schema.tables where table_schema='appdb';

+--------------+------------+--------------------+
| TABLE_SCHEMA | TABLE_NAME | TABLE_COLLATION    |
+--------------+------------+--------------------+
| appdb        | animals    | utf8mb4_0900_ai_ci |
| appdb        | temp_seq   | utf8mb3_general_ci |
| appdb        | test_table | utf8mb3_general_ci |
+--------------+------------+--------------------+
```

:eagle:其中:
- 在修改数据库字符集之前已有的表，其字符集和排序规则保持不变，即`utf8mb3`和`utf8mb3_general_ci`；
- 在修改数据库字符集之后创建的表，如果建表时未指定字符集，其字符集和排序规则为修改后的数据库字符集和排序规则，即`utf8mb4`和`utf8mb4_0900_ai_ci`。


## 修改表的字符集和排序规则
修改单张表的字符集：
```sql
SQL > alter table appdb.temp_seq character set utf8mb4;

Query OK, 0 rows affected (0.0073 sec)
Records: 0  Duplicates: 0  Warnings: 0

SQL > select table_schema,table_name,table_collation 
from information_schema.tables where table_schema='appdb';

+--------------+------------+--------------------+
| TABLE_SCHEMA | TABLE_NAME | TABLE_COLLATION    |
+--------------+------------+--------------------+
| appdb        | animals    | utf8mb4_0900_ai_ci |
| appdb        | temp_seq   | utf8mb4_0900_ai_ci |
| appdb        | test_table | utf8mb3_general_ci |
+--------------+------------+--------------------+
```

修改单张表的字符集和排序规则：
```sql
SQL > alter table appdb.temp_seq character set utf8mb4 collate utf8mb4_general_ci;

Query OK, 0 rows affected (0.0051 sec)
Records: 0  Duplicates: 0  Warnings: 0

SQL > select table_schema,table_name,table_collation 
from information_schema.tables where table_schema='appdb';

+--------------+------------+--------------------+
| TABLE_SCHEMA | TABLE_NAME | TABLE_COLLATION    |
+--------------+------------+--------------------+
| appdb        | animals    | utf8mb4_0900_ai_ci |
| appdb        | temp_seq   | utf8mb4_general_ci |
| appdb        | test_table | utf8mb3_general_ci |
+--------------+------------+--------------------+
```


**References**

[1] https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb3.html

[2] https://blog.csdn.net/Sebastien23/article/details/125124622




