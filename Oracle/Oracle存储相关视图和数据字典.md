---
tags: [oracle]
title: Oracle存储相关视图和数据字典
created: '2022-12-19T01:21:21.649Z'
modified: '2022-12-19T12:51:15.566Z'
---

Oracle存储相关视图和数据字典

# v$asm_disk
描述了ASM磁盘的相关信息。

- `name`：ASM磁盘名称。
- `group_number`：磁盘所在磁盘组编号。
- `mount_status`：磁盘的挂载状态。可能的值包括MISSING、CLOSED、OPENED、CACHED、IGNORED或CLOSING。
- `mode_status`：可以接受I/O请求类型的状态。可能的值包括ONLINE、OFFLINE或SYNCING。
- `state`：相对于磁盘组的状态。可能的值包括UNKNOWN、NORMAL、ADDING、DROPPING、HUNG或FORCING。
- `failgroup`：磁盘所在的故障组名称。
- `free_mb`：磁盘中未被使用的容量大小，单位为MB。
- `total_mb`：磁盘的总容量大小，单位为MB。

# v$asm_diskgroup
描述了ASM磁盘组的相关信息。

- `name`：ASM磁盘组名称。
- `group_number`：磁盘组编号。
- `state`：磁盘组相对于实例的状态。可能的值包括BROKEN、CONNECTED、DISMOUNTED、MOUNTED、QUIESCING、RESTRICTED或UNKNOWN。
- `type`：磁盘组的冗余类型。可能的值包括EXTEND、EXTERN、FLEX、HIGH或NORMAL。
- `free_mb`：磁盘组中未使用的空间大小，单位为MB。
- `total_mb`：磁盘组的总容量大小，单位为MB。

# dba_data_files
描述了Oracle数据文件的相关信息，可以用来统计表空间的大小。常见的列属性有：

- `file_name`：数据文件的名称。
- `tablespace_name`：数据文件所属的表空间的名称。
- `bytes`：数据文件的大小，以bytes表示。
- `blocks`：数据文件的大小，以Oracle数据块表示。
- `status`：文件状态，其值为**AVAILABLE**或**INVALID**。INVALID表示文件未被使用。
- `online_status`：其值可以为SYSOFF、SYSTEM、OFFLINE、ONLINE或RECOVER。
- `autoextensible`：是否可以自动扩展。


# dba_free_space
描述了数据库表空间中可用的区（extent）的信息，可以用来统计表空间的剩余容量。常见的列属性有：

- `file_id`：包含区的数据文件的绝对文件编号。
- `tablespace_name`：区所属的表空间的名称。
- `bytes`：区的大小，以bytes表示。
- `blocks`：区的大小，以Oracle数据块表示。

# dba_temp_files
描述了数据库中所有临时文件的信息，可以用来统计临时表空间的大小。常见的列属性有：

- `file_name`：临时文件的名称。
- `tablespace_name`：临时文件所属的表空间名称。
- `bytes`：临时文件的大小，以bytes表示。
- `blocks`：临时文件的大小，以Oracle数据块表示。
- `status`：文件状态，其值为**OFFLINE**、**ONLINE**或**UNKNOWN**。
- `autoextensible`：是否可以自动扩展。

# dba_temp_free_space
描述了表空间层面的临时空间使用情况。

- `tablespace_name`：表空间名称。
- `free_space`：剩余可用临时空间大小，以bytes表示。包含已经分配且可用的空间、以及尚未被分配的空间。

# dba_tablespace_usage_metrics
描述了所有类型表空间的使用情况。

- `tablespace_name`：表空间名称。
- `tablespace_size`：表空间大小，以Oracle数据块表示。如果表空间中包含可以自动增长（autoextend）的数据文件，该列的值表示的是表空间所在底层存储的剩余可用空间；如果表空间的数据文件**仅**包含不能自动增长的数据文件，该列的值表示的是已经分配的表空间大小。
- `used_space`：已使用的空间大小，以Oracle数据块表示。对于undo表空间而言，该列的值同时包含了已过期和未过期的undo段所消耗的空间。
- `used_percent`：已使用空间占表空间大小的百分比。

借助`select value from v$parameter where name='db_block_size'`可以将数据块表示的表空间大小转化为bytes。

# dba_undo_extents
描述了undo表空间中的段（segment）所包含的区（extents）的信息。

- `tablespace_name`：区所属的段所在undo表空间的名字。
- `bytes`：区的大小，以bytes表示。
- `status`：区的undo状态，其值可以为**ACTIVE**、**EXPIRED**或**UNEXPIRED**。

# dba_segments
描述了数据库中所有segment被分配的存储信息。

- `segment_name`：段的名称。
- `segment_type`：段的类型，可以是TYPE2 UNDO、TABLE、INDEX、ROLLBACK等类型。
- `tablespace_name`：段所在的表空间的名称。
- `bytes`：段的大小，以bytes表示。

# v$rollname
描述了数据库打开时所有在线回滚段的名字和编号信息。

- `name`：回滚段的名称。
- `usn`：undo回滚段编号。


**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_DATA_FILES.html#GUID-0FA17297-73ED-4B5D-B511-103993C003D3
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-ASM_DISK.html#GUID-8E2E5721-6D4E-48C2-8DF3-A0EEBD439606


