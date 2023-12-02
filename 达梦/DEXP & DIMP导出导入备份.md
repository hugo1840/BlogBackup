---
tags: [达梦]
title: DEXP & DIMP导出导入备份
created: '2023-12-02T03:20:46.037Z'
modified: '2023-12-02T03:22:18.764Z'
---

DEXP & DIMP导出导入备份

# 导出导入示例
将数据库的KNOWDB模式下下的所有数据库对象导出到/home/dmdba/路径下的dmp文件中：
```bash
cd /dameng/app/v8/bin/ 
./dexp USERID=SYSDBA/SYSDBA FILE=knowdb_20231122.dmp LOG=dexp.log \
DIRECTORY=/home/dmdba/ SCHEMAS=KNOWDB
```

将/home/dmdba/路径下的逻辑备份文件knowdb_20231123.dmp导入数据库的KNOWDB模式下：
```bash
cd /dameng/app/v8/bin/ 
./dimp USERID=SYSDBA/SYSDBA FILE=knowdb_20231123.dmp LOG=dimp.log \
DIRECTORY=/home/dmdba/ SCHEMAS=KNOWDB
```

# 可用参数一览
基本参数：
- USERID: 执行导入导出操作的用户连接信息；
- FILE：导入或导出的dump文件名；
- LOG：导入或导出的日志文件名；
- DIRECTORY：dump文件和日志文件存放的默认路径；
- PARALLEL：导出导入的并行度。默认为单线程，一般不应超过CPU核数；
- FILESIZE：单个导出文件大小的上限，例如100M[B]、1G[B]。

DUMP级别相关。四选一，默认导出级别为SCHEMAS。
- FULL：导出或导入整个数据库，默认N，可选Y；
- OWNER：用户清单，用于导出或导入一个或多个用户的数据库对象；
- SCHEMAS：模式清单，用于导出或导入一个或多个模式下的所有对象；
- TABLES：表清单，用于导出或导入一个或多个表OR表分区。

其他可选的导出或导入对象：
- CONSTRAINTS：导出约束，默认Y；
- GRANTS：导出权限信息，默认Y；
- INDEXES：导出索引，默认Y；
- TRIGGERS：导出触发器，默认Y；
- ROWS：导出数据行，默认Y。
- TABLESPACE：导出的对象定义是否包含表空间，默认N；

导出内容过滤：
- EXCLUDE：在导出时忽略指定的数据库对象。
  - 忽略表（TABLES导出级别不适用）：`EXCLUDE=TABLES:tablename1,tablename2`
  - 忽略模式：`ECLUDE=SCHEMAS:schemeaname1,schemaname2`

- INCLUDE：在导出时包含指定的数据库对象。
  示例：`INCLUDE=TABLES:tablename1,tablename2`
  
导出压缩备份：
- COMPRESS：是否压缩导出文件，默认N。
  
导入关系映射：
- REMAP_SCHEMA：将一个模式下的对象导入到另一个模式下，示例`SOURCE_SCHEMA:TARGET_SCHEMA`。
- REMAP_TABLE：将一张表的数据导入到另一张表，示例`SOURCE_SCHEMA.TABLENAME1:TARGET_SCHEMA.TABLENAME2`。
- REMAP_TABLESPACE：将源表空间映射到另一个表空间，示例`SOURCE_TABLESPACE:TARGET_TABLESPACE`。


