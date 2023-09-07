---
tags: [oracle]
title: Oracle开启Supplemental Logging补充日志
created: '2023-09-07T09:16:41.091Z'
modified: '2023-09-07T09:33:31.563Z'
---

Oracle开启Supplemental Logging补充日志

Flink CDC应用需要开启数据库附加日志（Supplemental Logging）。CDC（Change Data Capture）即数据变更抓取，通过为源端数据源开启CDC，作业可实现数据源的实时数据同步。

# 开启数据库归档
开启数据库归档：
```sql
archive log list;
shutdown immediate;
startup mount;
alter database archivelog;
alter database open;
```

# 数据库级别配置
开启最小补充日志：
```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```
最小补充日志开启后可以使得logmnr工具支持链式行、簇表和索引组织表。

在数据库级别开启下列补充日志会导致REDO日志量明显增加。
```sql
--在数据库级别开启所有列的补充日志：
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;

--在数据库级别开启主键的补充日志：
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;

--在数据库级别开启唯一索引的补充日志：
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (UNIQUE) COLUMNS;

--在数据库级别开启外键的补充日志：
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (FOREIGN KEY) COLUMNS;
```

**注意事项：**
- 在数据库级别开启列的补充日志会**隐式**开启最小补充日志。
- 数据库打开时，开启数据库补充日志命令执行非常慢，强烈建议在**MOUNT**状态下执行完成后打开数据库。
- 在数据库打开时开启补充日志会导致游标缓存中的DML游标失效，短期内SQL**硬解析**显著增加，对数据库性能会有暂时性的影响。

关闭补充日志：
```sql
ALTER DATABASE DROP SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
ALTER DATABASE DROP SUPPLEMENTAL LOG DATA (UNIQUE) COLUMNS;
ALTER DATABASE DROP SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
ALTER DATABASE DROP SUPPLEMENTAL LOG DATA (FOREIGN KEY) COLUMNS;
ALTER DATABASE DROP SUPPLEMENTAL LOG DATA;
```

# 单表级别配置
开启表级补充日志前，先开启数据库级最小补充日志：
```sql
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
```

对单个表的所有列开启补充日志：
```sql
ALTER TABLE hr.employees ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

对单个表的主键开启补充日志：
```sql
ALTER TABLE hr.employees ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
```

对单个表的唯一索引开启补充日志：
```sql
ALTER TABLE hr.employees ADD SUPPLEMENTAL LOG DATA (UNIQUE) COLUMNS;
```

关闭补充日志：
```sql
ALTER TABLE hr.employees DROP SUPPLEMENTAL LOG DATA (UNIQUE) COLUMNS;
ALTER TABLE hr.employees DROP SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
ALTER TABLE hr.employees DROP SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
```

# 检查补充日志配置
检查数据库级别的补充日志配置：
```sql
SQL> select SUPPLEMENTAL_LOG_DATA_MIN,
SUPPLEMENTAL_LOG_DATA_PK,
SUPPLEMENTAL_LOG_DATA_UI,
SUPPLEMENTAL_LOG_DATA_FK,
SUPPLEMENTAL_LOG_DATA_ALL,
SUPPLEMENTAL_LOG_DATA_PL from v$database;  

SUPPLEME SUP SUP SUP SUP SUP
-------- --- --- --- --- ---
NO	 NO  NO  NO  NO  NO
```

其中：
- `SUPPLEMENTAL_LOG_DATA_MIN`：是否在数据库级别开启了最小补充日志；
- `SUPPLEMENTAL_LOG_DATA_ALL`：是否在数据库级别开启了所有列的补充日志；
- `SUPPLEMENTAL_LOG_DATA_PK`：是否在数据库级别开启了主键的补充日志；
- `SUPPLEMENTAL_LOG_DATA_UI`：是否在数据库级别开启了唯一索引的补充日志；
- `SUPPLEMENTAL_LOG_DATA_FK`：是否在数据库级别开启了外键的补充日志；
- `SUPPLEMENTAL_LOG_DATA_PL`：是否在数据库级别开启了Oracle包程序调用的补充日志。


查询日志组中某列是否开启了补充日志：
```sql
SQL> select * from ALL_LOG_GROUP_COLUMNS;

OWNER			       LOG_GROUP_NAME		      TABLE_NAME		     COLUMN_NAME	    POSITION LOGGIN
------------------------------ ------------------------------ ------------------------------ -------------------- ---------- ------
SYS			       SEQ$_LOG_GRP		      SEQ$			     OBJ#			   1 LOG
SYS			       ENC$_LOG_GRP		      ENC$			     OBJ#			   1 LOG
SYS			       ENC$_LOG_GRP		      ENC$			     OWNER#			   2 LOG
```

其中**LOGGING_PROPERTY**指定日志组中某列是否开启了补充日志：
- `LOG`：该列已开启补充日志。
- `NO LOG`：该列未开启补充日志。


查询日志组的补充日志类型：
```sql
SQL> select * from ALL_LOG_GROUPS;

OWNER			       LOG_GROUP_NAME		      TABLE_NAME		     LOG_GROUP_TYPE		  ALWAYS      GENERATED
------------------------------ ------------------------------ ------------------------------ ---------------------------- ----------- --------------
SYS			       SEQ$_LOG_GRP		      SEQ$			     USER LOG GROUP		  ALWAYS      USER NAME
SYS			       ENC$_LOG_GRP		      ENC$			     USER LOG GROUP		  ALWAYS      USER NAME
HR			       SYS_C0069023		      D_ROLE_INFO		     PRIMARY KEY LOGGING	  ALWAYS      GENERATED NAME
HR			       SYS_C0069022		      D_USER_INFO		     PRIMARY KEY LOGGING	  ALWAYS      GENERATED NAME
```

其中**LOG_GROUP_TYPE**表示日志组的补充日志类型：
- `ALL COLUMN LOGGING`
- `FOREIGN KEY LOGGING`
- `PRIMARY KEY LOGGING`
- `UNIQUE KEY LOGGING`
- `USER LOG GROUP`

其中**ALWAYS**表示日志组记录补充日志的条件：
- `ALWAYS`：表中任意一列被更新时都记录补充日志。
- `CONDITIONAL`：表中指定一列被更新时才记录补充日志。


**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-logminer-utility.html#GUID-48D9DB83-BBC0-45EE-A81E-7CD047C908C1
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-logminer-utility.html#GUID-E3E015C4-B0EB-4072-92A6-FD3079C68242
【3】https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DATABASE.html#GUID-C62A7B96-2DD4-4E70-A0D9-26EE4BFBE256

