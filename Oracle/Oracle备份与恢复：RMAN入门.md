@[TOC](Oracle备份与恢复：RMAN入门)

本文所讨论内容涉及的数据库版本为 Oracle 12.2。

# 备份、恢复与归档
**什么是备份**
Oracle数据库中的备份可以分为**物理备份**（Physical backup）与**逻辑备份**（Logical backup）。

物理备份是对用于存储和恢复数据库的物理文件的备份。这些物理文件包括**数据文件**、**控制文件**、以及**归档的 redo 日志文件**。物理备份又可以分为归档模式备份（又叫**热备份**或者**联机备份**）和非归档模式备份（又叫**冷备份**或者**脱机备份**）。归档模式备份是当数据库的模式设置成归档模式时对数据库进行的备份；非归档模式备份则是当数据库的模式设置成非归档模式时对数据库进行的备份。

逻辑备份主要是对逻辑数据的导入导出，比如**表**、**存储过程**。在 Oracle 10g 之前使用 IMP/EMP 的方式进行导入和导出操作，从 Oracle 10g 开始引入数据泵技术，使用 EXPDP/IMPDP 的方式对数据进行导入和导出。

任何一个可靠的备份和恢复策略都离不开物理备份。在许多情形下，逻辑备份可以作为物理备份的一个有用的补充手段，因为逻辑备份本身并不足以避免数据丢失。除非特别说明，在本文中“备份”一词专门指代“物理备份”。

**什么是恢复**
数据库恢复就是把从数据库中备份出来的数据重新还原到原来的数据库中。数据库恢复可以分为**完全恢复**和**不完全恢复**。完全恢复是指把数据库恢复到数据库**失败时**的数据库状态，不完全恢复则是指将数据库恢复到数据库**失败前某一时刻**的数据库状态。

数据库恢复也可以分为物理恢复和逻辑恢复。物理恢复就是把从数据库中备份出来的物理文件重新应用到原来的数据库中；逻辑恢复就是把从数据库中导出的数据再导入原来的数据库中。

**什么是归档**
数据归档（Data archival）可以看作是一种特殊的备份，但是并不用于常规的备份和恢复策略。数据归档中的备份会被归档到独立的存储介质上，以供长期保存。归档的数据**不会**用来进行灾难恢复。

# RMAN基础
恢复管理器（Recovery Manager, RMAN）是Oracle数据库提供的一种恢复和备份数据库的工具。RMAN 有以下四个主要特点：

- 可以备份数据库、表空间、数据文件、控制文件、以及日志文件；
- 压缩备份可以只备份发生变化的内容；
- 集成了第三方的磁带媒介软件；
- 可以在 Oracle 数据库的目录中存放备份信息。

## 基本概念
使用 RMAN 时常用的概念包括：

- **目标数据库**（target database）：目标数据库就是使用 RMAN 进行备份和还原的数据库。RMAN 使用 `TARGET` 关键字链接目标数据库。RMAN 在目标数据库的控制文件中维护对该数据库的备份和恢复操作的元数据。这些 RMAN 元数据被称为 RMAN repository。
- **RMAN 客户端**（RMAN client）：RMAN 客户端用于解释和执行 RMAN 命令，会在目标数据库的控制文件中记录自己的活动。一般不用单独安装。通常位于 `$ORACLE_HOME/bin` 目录下。
- **闪回区**（flash recovery area）：磁盘上的一个区域，用于存放与数据库备份和恢复相关的一些文件。使用闪回区能够方便用户备份和还原数据库。可以通过 `DB_RECOVERY_FILE_DEST` 和 `DB_RECOVERY_FILE_DEST_SIZE` 参数来设置闪回区的位置和大小。
- **介质管理器**（media manager）：RMAN 通过介质管理器把数据库备份到磁带中。介质管理器通常由第三方厂商提供。它将数据块中的数据流从 RMAN 通道进程传递到对应的磁带上。介质管理设备通常被称为 SBT (System Backup to Tape)。
- **恢复目录**（recovery catalog）： 恢复目录是一个独立的数据库，用于存放一个或多个目标数据库的备份信息。恢复目录中保存有 RMAN repository 元数据，可以简化目标数据库控制文件丢失后的恢复过程。


## 常用命令
**启动RMAN并连接到目标数据库**
在操作系统终端中执行 rman 命令启动恢复管理器。通过 RMAN 连接到目标数据库需要当前用户具有 **SYSDBA** 或者 **SYSBACKUP** 权限。使用 EXIT 命令退出 RMAN 客户端。

```bash
% rman
RMAN> CONNECT TARGET "sbu@prod AS SYSBACKUP"

target databse Password: password
connected to target database: PROD (DBID=39283562)

RMAN> EXIT
```

**查看RMAN默认配置**
RMAN 对每个目标数据库分别预配置备份还原环境，比如备份设备、通道（channels）、备份策略等。连接到目标数据库后，执行以下命令查看针对该数据库的当前默认配置：

```bash
RMAN> SHOW ALL;
```

# 数据库备份
使用 BACKUP 命令来创建备份。RMAN 默认在磁盘上创建备份。如果启用了闪回区并且在备份时未注明 FORMAT 参数，RMAN 就会在闪回区创建备份，并自动给备份创建唯一命名。

默认情况下，RMAN 会创建备份集（backup sets）而不是镜像拷贝（image copies）。一个备份集包含一个或多个备份片（backup pieces）物理文件，且只有 RMAN 可以访问。RMAN 可以将备份集写回磁盘或磁带。

如果备份时注明了 `BACKUP AS COPY`，RMAN 会将文件备份为镜像拷贝。镜像拷贝是对数据库文件的逐比特（bit-for-bit）拷贝。在数据库处于 OPEN 状态时可以使用 RMAN 来创建镜像拷贝。

查看数据库是否为归档模式的 SQL 语句为

```sql
SQL> archive log list;
```

## 归档模式下备份数据库
如果数据库运行在归档（**ARCHIVELOG**）模式下，就可以在数据库处于 **OPEN** 状态时进行备份。连接到目标数据库后，执行以下命令备份数据库和所有归档的 redo 日志文件：

```bash
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

如果备份包含了 checkpoint 之后的变化，就会被称为是一个**非一致性备份**（inconsistent backup）。

## 非归档模式下备份数据库
如果数据库运行在非归档（**NOARCHIVELOG**）模式下，那么只有**一致性备份**才是有效的备份。要创建一致性备份，数据库必须在**一致性关闭**（consistent shutdown）后挂载。如果在还原（**Restore**）备份后未进行数据库恢复（**Recovery**），就会丢失备份后的所有事务。可以利用一致性备份中的归档日志来恢复数据库，以减小数据损失。

连接到目标数据库后，一致性关闭数据库然后挂载数据库。

```bash
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP FORCE DBA;
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
```

接下来备份数据库。

```bash
RMAN> BACKUP DATABASE;  # 创建备份集
RMAN> BACKUP AS COPY DATABASE;  # 创建镜像拷贝（与上一步二选一）
```

最后打开数据库，恢复到正常运行状态。

```bash
RMAN> ALTER DATABASE OPEN;
```

## 增量备份
增量备份是在前一次增量备份的基础上对块（block）级别的变化进行备份。增量备份比全数据库备份更小、更快。使用增量备份进行恢复也比单独使用 redo 日志更快。当数据库处于非归档模式下时，不能进行增量备份，除非在一致性关闭数据库后挂载。

>block 是数据库中最小的存储和处理单位，大小在 2K 到 64K 之间。

采用增量备份策略时，需要先进行一次 **level 0** 增量备份，即对数据库中所有的 blocks 进行备份，然后才能在 level 0 备份的基础上进行 **level 1** 的增量备份。一个 level 1 的增量备份可以包含自最新的 level 0 备份以来的所有 blocks 的变化，此时该 level 1 增量备份也被称为**累计增量备份**（cumulative incremental backup）。一个 level 1 的增量备份如果只包含最近一次非 level 0 备份以来的 blocks 变化，则被称为**差异增量备份**（differential incremental backup）。差异增量备份一般是默认选择。

连接到目标数据库后，执行以下命令来进行增量备份：

```bash
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;  # 创建差异增量备份（与上一步二选一）
```

在还原数据库备份时，RMAN 首先会还原 level 0 级别的备份，然后按顺序自动应用 level 1 级别的增量备份和 redo 日志。


## 验证数据库文件和备份
RMAN 验证会对备份进行检查以确定该备份是否能够被还原。RMAN 验证也可以用来检查坏掉的 block 和丢失的文件。

连接到目标数据库后，执行 **VALIDATE** 命令来确认所有的数据库文件存在、保存在正确的位置、并且没有物理损坏。**CHECK LOGICAL** 命令用于检查逻辑区块的损坏情况。

```bash
# 验证所有数据库文件和归档的redo日志是否存在物理损坏和逻辑损坏
RMAN> BACKUP VALIDATE CHECK LOGICAL DATABASE ARCHIVELOG ALL;

# 检查数据文件4中第10到第13个blocks的物理损坏情况
RMAN> VALIDATE DATAFILE 4 BLOCK 10 TO 13;

# 检查第3个备份集的物理损坏情况
RMAN> VALIDATE BACKUPSET 3;
```

## RMAN脚本
创建一个如下的 RMAN 命令脚本

```bash
# my_command_file.txt
CONNECT TARGET /
BACKUP DATABASE PLUS ARCHIVELOG;
LIST BACKUP;
EXIT;
```
使用 `@` 命令执行脚本

```bash
RMAN> @/my_dir/my_command_file.txt
```

也可以在启动 RMAN 时输入命令

```bash
$ rman @/my_dir/my_command_file.txt
```


# RMAN报表
RMAN 可以利用 RMAN repository 中的信息来生成备份活动的报表。

## LIST命令
使用 **LIST BACKUP** 和 **LIST COPY** 命令可以查看 repository 中备份和数据文件拷贝的信息。

```bash
RMAN> LIST BACKUP OF DATABASE BY BACKUP;  # 显示备份信息，按备份集组织（默认）
RMAN> LIST BACKUP SUMMARY;  # 显示备份信息综述

# 显示repository中有信息但在备份路径中不存在的备份（可能在系统层面被删除）
RMAN> LIST EXPIRED COPY;  
# 显示可以被恢复和还原的备份信息
RMAN> LIST BACKUP RECOVERABLE;

RMAN> LIST COPY OF DATAFILE 1, 2;
RMAN> LIST BACKUP OF ARCHIVELOG FROM SEQUENCE 10;
RMAN> LIST BACKUPSET OF DATAFILE 1;
```

## REPORT命令
使用 **REPORT** 命令可以输出更为详细的报表分析。

```bash
RMAN> REPORT NEED BACKUP DATABASE;  # 显示在当前策略下需要备份的文件
RMAN> REPORT OBSOLETE;  # 显示在当前的备份策略下已经过时的备份
RMAN> REPORT SCHEMA;  # 列出数据库中的表空间和数据文件
RMAN> REPORT UNRECOVERABLE;  # 列出最近一次备份以来所有遭受过不可恢复操作的数据文件
```

# 管理RMAN备份
## 交叉验证
执行 **CROSSCHECK** 命令将 RMAN 备份和拷贝的逻辑记录与存储介质上的文件进行同步。如果备份位于磁盘上，该命令会判定文件的 header 是否有效；如果备份位于磁带上，RMAN 会向 repository 请求备份片的名称和位置。

在删除备份或拷贝前最好进行交叉验证。

```bash
RMAN> CROSSCHECK BACKUP;
RMAN> CROSSCHECK COPY;
```

## 删除过时的备份
**DELETE** 命令可以从磁盘和磁带中移除备份和拷贝，将文件在 repository 中的状态更新为 "DELETED"，同时从恢复目录移除相应记录（如果使用了 recovery catalog）。在交互模式下，如果没有注明 **NOPROMPT** 参数，DELETE 命令会列出文件，等用户确认后才会删除。执行以下命令可以删除已经过时的备份

```bash
RMAN> DELETE OBSOLETE;
```

# 失败诊断与修复
**数据恢复顾问**（Data recovery advisor, DRA）是Oracle数据库中用于诊断数据失败（data failure）和执行数据修复（repairs）的工具。

## 备份失败诊断
数据失败是 Health Monitor 监控到的持久化的数据损坏（data corruption），比如物理数据块损坏、逻辑数据块损害、数据文件丢失。一般使用失败优先级（**failure priority**）和失败状态（**failure status**）来描述数据失败。失败优先级分为 Critical、High 和 Low；失败状态分为 Open 和 Closed。

执行 **LIST FAILURE** 命令来查看已知的失败。**ADVISE FAILURE** 命令会分别给出手动和自动的修复选项。首先尝试手动修复，如果手动修复无法解决问题，再参考自动修复的建议。

```bash
RMAN> LIST FAILURE;

Databse Role: Primary
List of Database Failures
================================
Failure ID	Priority	Status	Time Detected	Summary
......

RMAN> ADVISE FAILURE;
......
Mandatory Manual Actions
================================
......
Optional Manual Actions
================================
......
Automated Repair Options
================================
......
```

## 备份失败修复
使用 **REPAIR FAILURE** 命令执行修复选项。如果未注明修复选项的数字，RMAN 会使用最近执行的 **ADVISE FAILURE** 命令输出的第一个修复选项。

```bash
RMAN> REPAIR FAILURE;
```

# 还原和恢复
使用 **RESTORE** 和 **RECOVER** 命令来还原和恢复数据库的物理文件。恢复由于介质失败导致文件损坏的数据库的前提是，已经有所需的备份。使用  **RESTORE ... PREVIEW** 命令来查看 RMAN 可以用来进行数据库恢复的备份，而不会进行恢复操作。

```bash
RMAN> REPORT SCHEMA;  # 查看表空间和数据文件
RMAN> RESTORE DATABASE PREVIEW SUMMARY;

Starting restore at 21-MAY-13
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=80 device type=DISK

List of Backups:
================
......

List of Archived Log Copies for database with db_unique_name PROD
=================================================================
......

Media recovery start SCN is 586534
......
Finished restore at 21-MAY-13
```

## 恢复数据库
恢复数据库的前提是对所有必需的文件进行了备份。所有的数据文件必须能够还原到原来的路径。如果原来的路径已经不可访问，则需要使用 **SET NEWNAME** 命令将数据文件还原到非默认位置。

>[About Restoring Data Files to a Nondefault Location](https://docs.oracle.com/en/database/oracle/oracle-database/12.2/bradv/rman-complete-database-recovery.html#GUID-A3DE37A6-3983-4834-858E-A2BCC3804681)

恢复整个数据库的步骤如下：

1. 执行 `RESTORE DATABASE PREVIEW SUMMARY;`，查看可以用于恢复的备份；
2. 执行 `STARTUP FORCE MOUNT;`，将数据库设置为已挂载状态；
3. 执行 `RESTORE DATABASE;`，还原数据库；
4. 执行 `RECOVER DATABASE;`，恢复数据库；
5. 执行 `ALTER DATABASE OPEN`，打开数据库。

## 恢复表空间
当数据库打开时，使用 **RSTORE TABLESPACE** 和 **RECOVER TABLESPACE** 命令来恢复单个表空间。这种情况下，必须先将需要恢复的表空间离线（offline），进行还原恢复后再将表空间上线（online）。

如果无法将数据文件还原到原来的位置，可以使用 **SET NEWNAME** 和 **RUN** 命令来设定新的文件名和还原位置。然后，使用 **SWITCH DATAFILE ALL** 命令（等价于 SQL语句：`ALTER DATABASE RENAME FILE`），来更新控制文件中该数据文件的名称和位置信息。

在数据库打开时恢复单个表空间的步骤如下：

1. 查看表空间和可恢复的备份；

```bash
RMAN> REPORT SCHEMA;  
RMAN> RESTORE DATABASE PREVIEW SUMMARY;
```

2. 将要恢复的表空间离线；

```bash
RMAN> ALTER TABLESPACE users OFFLINE;
```

3. 还原恢复表空间；

```bash
RUN
{
  SET NEWNAME FOR DATAFILE '/disk1/oradata/prod/users01.dbf'
     TO '/disk2/users01.dbf';
  RESTORE TABLESPACE users;
  SWITCH DATAFILE ALL;  # 将新的文件名称和路径更新到控制文件
  RECOVER TABLESPACE users;
}
```

4. 将表空间上线。

```bash
RMAN> ALTER TABLESPACE users ONLINE;
```

数据文件的还原恢复也可以使用 **RESTORE DATAFILE** 和 **RECOVER DATAFILE** 命令。

## 恢复数据块
RMAN 还可以恢复单个损坏的数据块（block）。当 RMAN 对某个文件或备份进行完整扫描时，会将其中损坏的 block 信息保存在 `V$DATABASE_BLOCK_CORRUPTION` 视图中。损坏信息通常会在告警日志（alert log）、跟踪文件（trace file）或者 SQL 查询结果中返回。

连接到目标数据库后，执行以下命令来恢复数据块：

```bash
RMAN> SELECT NAME, VALUE FROM V$DIAG_INFO;  # 获取损坏区块的编号
RMAN> RECOVER CORRUPTION LIST;  # 回复所有损坏的区块
# 也可以恢复单独的损坏区块（通过数据文件编号和区块编号指定）
RMAN> RECOVER DATAFILE 1 BLOCK 233, 235 DATAFILE 2 BLOCK 100 TO 200;
```


References:
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/bradv/getting-started-rman.html#GUID-871FF5B2-C82B-462E-8182-FA28CF7B3E3B
[2\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/rcmrf/preface.html#GUID-1D50CC05-FB9A-4D4F-A0AE-ECF63610828F
[3\] https://www.cnblogs.com/xjp3/p/11507155.html
[4\] https://blog.csdn.net/leshami/article/details/9020727
[5\] Oracle从入门到精通，秦靖、刘存勇等

