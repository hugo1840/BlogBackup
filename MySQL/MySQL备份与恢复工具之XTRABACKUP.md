@[TOC](MySQL备份与恢复工具之XTRABACKUP)

Percona XtraBackup是一款开源的MySQL热备份工具。XtraBackup支持对基于InnoDB、XtraDB和MyISAM存储引擎的表的备份，且支持Oracle MySQL、Percona Server和MariaDB多个分支。目前官网提供2.4和8.0两个版本的下载，前者支持MySQL 5.1、5.5、5.6、5.7的备份，后者仅支持MySQL 8.0的备份。

# :coffee: 安装

> XtraBackup的下载地址为：https://www.percona.com/downloads/Percona-XtraBackup-2.4/Percona-XtraBackup-2.4.24/binary/redhat/7/x86_64/

**安装xtrabackup**

提前下好tar安装包，使用yum localinstall安装自动解决安装包依赖问题：
```bash
[root@mysql-node1 ~]# tar -xvf Percona-XtraBackup-2.4.24-rb4ee263-el7-x86_64-bundle.tar
[root@mysql-node1 ~]# yum localinstall -y percona-xtrabackup-24-2.4.24-1.el7.x86_64.rpm
```

如果使用rpm命令安装，需要手动安装依赖：
```bash
yum install -y perl-DBD-MySQL per-DBI perl-Time-HiRes libaio*
yum install -y perl-Digest-MD5 rsync
yum install -y libev
rpm -ivh percona-xtrabackup-24-2.4.24-1.el7.x86_64.rpm
```

**安装压缩备份工具qpress**
```bash
[root@mysql-node1 ~]# wget https://repo.percona.com/yum/release/7/RPMS/x86_64/qpress-11-1.el7.x86_64.rpm
[root@mysql-node1 ~]# yum localinstall qpress-11-1.el7.x86_64.rpm -y
```

# :feet: 原理
安装XtraBackup后会在`/usr/bin`下生成四个可执行文件：innobackupex、xtrabackup、xbcrypt、xbstream。其中，innobackupex用于备份非InnoDB引擎的表，同时还会调用xtrabackup备份InnoDB表。innobackupex还可以与MySQL Server进行交互，比如加读锁（FTWRL）、获取主从同步位点（show slave status）等。xbcrypt和xbstream分别用于备份恢复过程中的加解密和并发写支持。

> :eye: **注意**：XtraBackup 2.3版本用C语言对innobackupex进行了重写，所以2.3和2.4版本的innobackupex实际上是xtrabackup的一个软链接。

XtraBackup备份实现基于InnoDB存储引擎的故障恢复功能（crash-recovery）。也就是说，xtrabackup备份的数据最开始是不满足一致性的，但是通过模拟redo log故障恢复的过程最终能够获取到满足一致性的备份数据。

XtraBackup备份的流程大致如下：

1. innonackupex进程会fork一个xtrabackup子进程，用于备份InnoDB表。
2. xtrabackup进程会产生一个redo拷贝线程和一个ibd拷贝线程。Redo拷贝线程会记录备份开始时刻redo log中的日志序列号（LSN），并备份redo日志。ibd拷贝线程负责备份ibd数据文件。
3. InnoDB表备份完成后，ibd拷贝线程退出。
4. innobackupex进程执行`flush tables with read lock`加全局读锁。如果数据库是Percona Server，则会执行`lock tables for backup`加一个轻量的备份锁（Backup locks）。这里只会给MyISAM和其他非InnoDB表加备份锁，目的是为了避免在备份非InnoDB表的过程中阻塞修改InnoDB表的DML操作。
5. innobackupex进程开始备份非InnoDB表和数据库文件。备份的文件后缀包括`.frm`、`.MRG`、`.MYD`、`.MYI`、`.TRG`、`.TRN`、`.ARM`、`.ARZ`、`.CSM`、`.CSV`、`.par`、`.opt`。
6. 非InnoDB表备份完成后，innobackupex进程执行`lock binlog for backup`，以避免binlog一致性位点（`Exec_Master_Log_Pos`或者`Exec_Gtid_Set`）被修改。随后xtrabackup进程完成redo log备份，并通过执行`show master|slave status`取得binlog一致性位点。
7. innobackupex进程移除FTWRL全局读锁（或者备份锁）、binlog锁。
8. xtrabackup进程退出，备份结束。

在上面的备份流程中，XtraBackup是通过直接读取操作系统文件来进行备份的，只有在执行SQL命令时会和数据库有少量交互，基本对数据库运行无影响。**只有在备份非InnoDB表的时候会处于短时间的只读状态**（取决于MyISAM表的数据量，一般在几秒左右）。


# :deer: 用法

## 全量备份与还原

### STEP 1: BACKUP
将全量备份导出到/backup目录下，并将错误日志输出到backup.log中。

XtraBackup备份和还原时会从配置文件`/etc/my.cnf`中读取`datadir`、`innodb_data_home_dir`、`innodb_data_file_path`和`innodb_log_group_home_dir`变量。如果没有定义，则会报错Missing argument。

```bash
[root@mysql-node1 ~]# innobackupex --defaults-file=/etc/my.cnf --user=root --password=ABCcba123 \
/backup 2> /backup/backup.log
[root@mysql-node1 backup]# cat backup.log
xtrabackup: recognized server arguments: --datadir=/mysql/data --server-id=142 --log_bin=/mysql/log/mysql-bin --innodb_io_capacity=500 --innodb_flush_method=O_DIRECT --innodb_log_file_size=1G --innodb_file_per_table=1 --innodb_buffer_pool_size=3G --innodb_data_file_path=ibdata1:512M:autoextend --innodb_autoextend_increment=64 --innodb_checksum_algorithm=crc32 --innodb_read_io_threads=4 --innodb_log_buffer_size=8388608 --innodb_write_io_threads=4 --innodb_flush_log_at_trx_commit=1 --tmpdir=/mysql/tmp
220912 01:57:45 innobackupex: Missing argument
[root@mysql-node1 backup]#
```

如果没有定义这些变量，需要在备份命令中手动指定。其中，`innodb_data_home_dir`和`innodb_log_group_home_dir`如果没有在配置文件中定义，默认会与`datadir`相同。

```bash
[root@mysql-node1 backup]# innobackupex --defaults-file=/etc/my.cnf --user=root --password=ABCcba123 \
--innodb_data_home_dir=/mysql/data --innodb_log_group_home_dir=/mysql/data  /backup 2> /backup/backup.log
[root@mysql-node1 backup]#
[root@mysql-node1 backup]# cat backup.log
xtrabackup: recognized server arguments: --datadir=/mysql/data --server-id=142 --log_bin=/mysql/log/mysql-bin --innodb_io_capacity=500 --innodb_flush_method=O_DIRECT --innodb_log_file_size=1G --innodb_file_per_table=1 --innodb_buffer_pool_size=3G --innodb_data_file_path=ibdata1:512M:autoextend --innodb_autoextend_increment=64 --innodb_checksum_algorithm=crc32 --innodb_read_io_threads=4 --innodb_log_buffer_size=8388608 --innodb_write_io_threads=4 --innodb_flush_log_at_trx_commit=1 --tmpdir=/mysql/tmp --innodb_data_home_dir=/mysql/data --innodb_log_group_home_dir=/mysql/data
xtrabackup: recognized client arguments:
220912 02:09:28 innobackupex: Starting the backup operation

IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".

220912 02:09:28  version_check Connecting to MySQL server with DSN 'dbi:mysql:;mysql_read_default_group=xtrabackup' as 'root'  (using password: YES).
Failed to connect to MySQL server: DBI connect(';mysql_read_default_group=xtrabackup','root',...) failed: Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2) at - line 1314.
220912 02:09:28 Connecting to MySQL server host: localhost, user: root, password: set, port: not set, socket: not set
Failed to connect to MySQL server: Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2).
[root@mysql-node1 backup]#
```

如果`datadir`不是默认的`/var/lib/mysql`，则还需要指定`mysql.sock`的文件路径，否则会报错数据库连接失败。
```bash
[root@mysql-node1 backup]# innobackupex --defaults-file=/etc/my.cnf --user=root --password=ABCcba123 \
--innodb_data_home_dir=/mysql/data --innodb_log_group_home_dir=/mysql/data  \
--socket=/tmp/mysql.sock /backup 2> /backup/backup.log
[root@mysql-node1 backup]#
[root@mysql-node1 backup]# ls /backup
2022-09-12_02-17-32  backup.log
[root@mysql-node1 backup]# ls /backup/2022-09-12_02-17-32/
appgame        ib_buffer_pool  mysql               sys                     xtrabackup_checkpoints  xtrabackup_logfile
backup-my.cnf  ibdata1         performance_schema  xtrabackup_binlog_info  xtrabackup_info
[root@mysql-node1 backup]#
```

> :eye: **注意**：备份命令`innobackupex ... /backup`也可以替换为`xtrabackup ... --target-dir=/backup --backup`。


### STEP 2: PREPARE
假设我们不小心把应用库删了：
```sql
mysql> drop database appgame;
Query OK, 1 row affected (0.01 sec)
```

由于备份文件中的InnoDB数据在时间点上是不一致的，因此不能直接用于恢复数据。在利用备份文件进行恢复前，需要先进行Prepare。PREPARE会模仿InnoDB crash-recovery的过程，将备份数据还原到一个一致性位点。

```bash
[root@mysql-node1 backup]# xtrabackup --prepare --target-dir=/backup/2022-09-12_02-17-32/
...
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2806312
220912 03:26:36 completed OK!
```
出现以上信息则表示PREPARE完成。PREPARE过程中不能中断xtrabackup进程，否则有可能导致数据损坏，使得备份不可用。

### STEP 3: RESTORE
利用备份还原数据库之前，必须满足以下三个条件：

- 已经对备份数据进行PREPARE；
- MySQL Server服务已经停止（`systemctl stop mysqld`）；
- MySQL数据目录`datadir`必须是空目录（`rm -rf /mysql/data/*`）。

可以通过以下三种方法还原数据库：

- copy-back：保留原有的备份文件；
- move-back：将备份文件移动到MySQL数据目录下；
- rsync或cp：直接通过操作系统命令拷贝备份文件。例如：
`rsync -avrP /backup/2022-09-12_02-17-32/ /mysql/data/`

使用copy-back方法还原数据库：
```bash
[root@mysql-node1 backup]# xtrabackup --copy-back --target-dir=/backup/2022-09-12_02-17-32/
```

修改datadir权限并启动MySQL数据库服务：
```bash
[root@mysql-node1 backup]# chown -R mysql:mysql /mysql/data
[root@mysql-node1 backup]#
[root@mysql-node1 backup]# systemctl start mysqld
```

## 增量备份与还原
XtraBackup仅支持对InnoDB表进行增量备份，所以MyISAM表每次都是全量备份。

### STEP 1: BACKUP
在进行增量备份之前必须做一次全量备份，且全量备份的存放路径以`base`结尾：
```bash
[root@mysql-node1 backup]# mkdir base
[root@mysql-node1 backup]# xtrabackup --defaults-file=/etc/my.cnf --user=root --password=ABCcba123 \
--innodb_data_home_dir=/mysql/data --innodb_log_group_home_dir=/mysql/data  \
--socket=/tmp/mysql.sock --target-dir=/backup/base --backup 2> /backup/backup.log
```

创建一张新表，并插入数据：
```sql
mysql> create table `brands` (
    -> `id` int not null,
    -> `name` varchar(30) not null,
    -> `country` varchar(20),
    -> `celebrity` varchar(30),
    -> primary key pk_id(id)
    -> ) engine=InnoDB default charset=utf8mb4;
Query OK, 0 rows affected (0.01 sec)

...
mysql> select * from brands;
+----+--------------+---------+---------------+
| id | name         | country | celebrity     |
+----+--------------+---------+---------------+
|  1 | Nike         | America | Lebron James  |
|  2 | Under Armour | America | Stephen Curry |
|  3 | Anta         | China   | Klay Thompson |
|  4 | Adidas       | Germany | Derrick Rose  |
|  5 | Lining       | China   | Jimmy Butler  |
+----+--------------+---------+---------------+
5 rows in set (0.00 sec)
```

进行增量备份时，通过`--incremental-basedir`参数指定上一次备份的存放路径。

进行第一次增量备份，保存到`/backup/inc1`路径下：
```bash
[root@mysql-node1 backup]# xtrabackup --backup --user=root --password=ABCcba123 \
--innodb_data_home_dir=/mysql/data --innodb_log_group_home_dir=/mysql/data  \
--socket=/tmp/mysql.sock --target-dir=/backup/inc1 --incremental-basedir=/backup/base 2> /backup/backup.log
```

进行第二次增量备份，保存到`/backup/inc2`路径下：
```bash
[root@mysql-node1 backup]# xtrabackup --backup --user=root --password=ABCcba123 \
--innodb_data_home_dir=/mysql/data --innodb_log_group_home_dir=/mysql/data  \
--socket=/tmp/mysql.sock --target-dir=/backup/inc2 --incremental-basedir=/backup/inc1 2> /backup/backup.log
```

> :warning: **注意**：`--incremental-basedir`对应上一次备份的存放路径。

### STEP 2: PREPARE
在对全量备份做PREPARE时，未提交的事务会被rollback。考虑到这些事务可能会在增量备份时提交，因此对于会追加增量备份的每一次备份，在做PREPARE时必须跳过rollback的环节。这里可以通过在最后一次备份之前的所有备份PREPARE时加上`--apply-log-only`参数实现。

Prepare全量备份：
```bash
xtrabackup --prepare --apply-log-only --target-dir=/backup/base
```

Prepare第一次增量备份：
```bash
xtrabackup --prepare --apply-log-only --target-dir=/backup/base --incremental-dir=/backup/inc1
```

Prepare第二次增量备份：
```bash
xtrabackup --prepare --target-dir=/backup/base --incremental-dir=/backup/inc2
```

:warning: **注意**：
1. 按照备份顺序依次Prepare增量备份，并且`target-dir`一直都是全量备份路径，包括在最后还原数据库时copy-back的也是`/backup/base`。
2. 最后一次Prepare增量备份时，**不**要加`--apply-log-only`参数。


### STEP 3: RESTORE
还原备份前，一定要停止数据库服务，并清空MySQL数据目录。

使用copy-back方法还原数据库：
```bash
[root@mysql-node1 backup]# xtrabackup --copy-back --target-dir=/backup/base
```

修改datadir权限并启动MySQL数据库服务：
```bash
[root@mysql-node1 backup]# chown -R mysql:mysql /mysql/data
[root@mysql-node1 backup]#
[root@mysql-node1 backup]# systemctl start mysqld
```


**References**
【1】https://docs.percona.com/percona-xtrabackup/2.4/index.html
【2】https://docs.percona.com/percona-xtrabackup/2.4/how_xtrabackup_works.html
【3】http://mysql.taobao.org/monthly/2016/03/07/
【4】https://docs.percona.com/percona-xtrabackup/2.4/backup_scenarios/full_backup.html
【5】https://docs.percona.com/percona-xtrabackup/2.4/backup_scenarios/incremental_backup.html#

