---
tags: [oracle]
title: Oracle清理归档&创建索引
created: '2023-02-07T15:10:31.697Z'
modified: '2023-02-07T16:19:22.320Z'
---

Oracle清理归档&创建索引

# 手动清理归档
归档日志一般会定期删除，也可以使用RMAN手动删除。但一定不能从文件系统层面直接删除。

```bash
# 校验归档日志
RMAN> crosscheck archivelog all;                        

using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=411 device type=DISK
validation succeeded for archived log
archived log file name=/oradata/arch/arch_1_3_1125605354.log RECID=1 STAMP=1128209085
Crosschecked 1 objects

# 删除7天以前的归档日志
RMAN> delete archivelog all completed before 'sysdate-7';

released channel: ORA_DISK_1
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=411 device type=DISK

# 删除所有归档日志
RMAN> delete archivelog all;

released channel: ORA_DISK_1
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=411 device type=DISK
List of Archived Log Copies for database with db_unique_name BANGKOK
=====================================================================

Key     Thrd Seq     S Low Time 
------- ---- ------- - ---------
1       1    3       A 15-JAN-23
        Name: /oradata/arch/arch_1_3_1125605354.log


Do you really want to delete the above objects (enter YES or NO)? YES
deleted archived log
archived log file name=/oradata/arch/arch_1_3_1125605354.log RECID=1 STAMP=1128209085
Deleted 1 objects

RMAN> crosscheck archivelog all;

released channel: ORA_DISK_1
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=411 device type=DISK
specification does not match any archived log in the repository
```

# 创建和删除索引
用户在被授予**CREATE TABLE**权限后，就可以在自己创建的表上创建和删除索引了。

通过`all_indexes`、`dba_indexes`和`user_indexes`表可以查询索引的相关信息。
```sql
SQL> create index idx_cc on test_table(c1,c2) online;

Index created.

SQL> set lines 200
SQL> col index_name for a30
SQL> col table_owner for a20
SQL> col table_name for a30
SQL> select index_name, index_type, table_owner, table_name
from user_indexes where table_name='TEST_TABLE';

INDEX_NAME                     INDEX_TYPE                  TABLE_OWNER          TABLE_NAME
------------------------------ --------------------------- -------------------- ------------------------------
SYS_C007195                    NORMAL                      MIGUEL               TEST_TABLE
IDX_CC                         NORMAL                      MIGUEL               TEST_TABLE
```

通过`all_ind_columns`、`dba_ind_columns`和`user_ind_columns`表可以查询索引中包含的列。
```sql
SQL> col column_name for a30
SQL> select index_name, table_name, column_name 
from user_ind_columns 
where table_name='TEST_TABLE' and index_name='IDX_CC';

INDEX_NAME                     TABLE_NAME                     COLUMN_NAME
------------------------------ ------------------------------ ------------------------------
IDX_CC                         TEST_TABLE                     C1
IDX_CC                         TEST_TABLE                     C2
```

使索引对优化器不可见：
```sql
SQL> alter index idx_cc invisible;
```

删除索引：
```sql
SQL> drop index idx_cc;

Index dropped.

SQL> select index_name, table_name, column_name 
from user_ind_columns where table_name='TEST_TABLE';

INDEX_NAME                     TABLE_NAME                     COLUMN_NAME
------------------------------ ------------------------------ ------------------------------
SYS_C007195                    TEST_TABLE                     ID
```



