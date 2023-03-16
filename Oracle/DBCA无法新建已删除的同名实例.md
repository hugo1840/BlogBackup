---
tags: [oracle]
title: DBCA无法新建已删除的同名实例
created: '2023-03-16T08:31:32.950Z'
modified: '2023-03-16T09:18:45.687Z'
---

DBCA无法新建已删除的同名实例

使用dbca删除数据库时，数据库必须处于打开状态。如果已经shutdown实例，并且从操作系统层面删除了包含控制文件的数据目录，就不能重新打开数据库来使用dbca删库。

此时只能手动删库，需要确保以下内容删除干净，否则可能无法使用dbca创建同名的实例。

1. 删除数据目录，包括对应实例的controlfile、datafile、onlinelog目录下的所有控制文件、数据文件和联机redo日志。

```bash
cd ${ORACLE_DATADIR}/<db_unique_name>
rm -rf controlfile
rm -rf onlinelog
rm -rf datafile
mkdir controlfile onlinelog datafile
```

2. 删除闪回区下的实例数据，包括对应实例的archivelog、controlfile、onlinelog目录下的所有文件。

```bash
cd ${ORACLE_DATADIR}/<fast_recovery_area>/<db_unique_name>
rm -rf controlfile
rm -rf onlinelog
rm -rf archivelog
mkdir controlfile onlinelog archivelog
```

3. 移除`/etc/oratab`文件中的实例名称记录。

```bash
grep <ORACLE_SID> /etc/oratab 
```

4. 移除数据库实例的trace和告警日志目录（可选）。

```bash
cd ${ORACLE_BASE}/diag/rdbms/
rm -rf <db_unique_name>/
```




