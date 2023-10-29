---
tags: [postgresql]
title: PG物理备份与恢复之pg_basebackup
created: '2023-10-29T02:03:37.691Z'
modified: '2023-10-29T02:07:36.521Z'
---

PG物理备份与恢复之pg_basebackup

# 开启WAL日志归档
通过数据库的全量备份和WAL日志，可以将数据库恢复到任意时间点。每个WAL日志文件大小通常是16MB。WAL日志文件个数增长到`wal_keep_segments + 1`个后，会覆盖已有的WAL日志文件。因此最好根据业务需求定时对WAL日志进行归档。

>:spider: 在PG 13版本中，wal_keep_segments被参数wal_keep_size取代。

创建归档目录：
```bash
mkdir /pg_arch
chown -R postgres:postgres /pg_arch
```

编辑配置文件`postgresql.conf`：
```
wal_level = replica
archive_mode = on
archive_command = 'DATE=`date +%Y%m%d`; DIR="/pg_arch/$DATE"; (test -d $DIR || mkdir -p $DIR) && cp %p $DIR/%f'
wal_keep_segments = 128
```
其中，`%p`表示包含完整路径的WAL日志文件名，`%f`表示不包含路径信息的WAL日志文件名。

重启数据库：
```bash
pg_ctl stop
pg_ctl start
```

手动触发日志切换：
```sql
postgres=# checkpoint;               --刷新内存脏页到磁盘
CHECKPOINT

postgres=# select pg_switch_wal();   --手动归档WAL日志
 pg_switch_wal 
---------------
 0/50003C0
(1 row)
```

验证归档：
```bash
[postgres@dbhost ~]$ ls -lh /pgdata/pg_wal/
total 32M
-rw------- 1 postgres postgres 16M Oct 26 11:17 000000010000000000000009
-rw------- 1 postgres postgres 16M Oct 26 11:17 00000001000000000000000A
drwx------ 2 postgres postgres 244 Oct 26 11:17 archive_status
 
[postgres@dbhost ~]$ ls -lh /pg_arch/20231026/
total 16M
-rw------- 1 postgres postgres 16M Oct 26 11:17 000000010000000000000009
```

# pg_basebackup备份工具
从PostgreSQL 9.1开始，支持使用pg_basebackup进行数据库物理备份（热备份）。pg_basebackup通过replication协议连接到数据库，因此在`pg_hba.conf`中要允许replication连接。

```bash
[postgres@dbhost ~]$ cat /pgdata/pg_hba.conf | grep replica | grep -v '^#'
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
host    replication     repl            0.0.0.0/0               md5
```

出于安全考虑，也可以创建单独的流复制用户配置到`pg_hba.conf`中，并禁用其他replication连接。
```sql
create role repl nosuperuser replication login connection limit 32 encrypted password 'Pwdabc123';
```

对本地数据库进行全库备份：
```bash
[postgres@dbhost ~]$ pg_basebackup -Urepl -Ft -Xf -z -Z5 -R -Pv -D /home/postgres/backup
39635/39635 kB (100%), 2/2 tablespaces

[postgres@dbhost ~]$ ls backup/
16386.tar.gz  base.tar.gz  pg_wal.tar.gz
```
其中，纯数字命名的备份文件是对`/pgdata/pg_tblspc`目录下表空间的备份。

pg_basebackup支持的常用参数选项如下：
- `-h host`：指定要连接的源数据库地址；
- `-p port`：指定要连接的源数据库端口；
- `-U username`：指定连接用户名；
- `-D path`：指定备份的存放路径。如果不存在，会自动创建；如果已存在且非空，则会执行失败。
- `-F p|t`：指定备份输出格式，默认为plain，t表示备份为tar包。
- `-X f|fetch|s|stream`：f或fetch表示把备份过程中产生的xlog文件也备份。需要设置`wal_keep_segments`参数以保证备份过程中WAL日志不会被覆写。s或stream表示备份开始后，启动另一个连接从源库接收WAL日志。需要设置`max_wal_senders`参数至少大于2。
- `-R`：生成用于恢复的recovery.conf文件。
- `-z`：与`-Ft`连用，表示对备份输出的tar包进行gzip压缩，得到格式为`.tar.gz`的备份文件。
- `-Z level`：指定gzip的压缩级别，取值范围为1~9，数字越大压缩率越高。
- `-l label`：为备份指定标签，便于识别和管理备份文件。
- `-P`：在备份过程中实时显示备份进度。
- `-v`：备份过程中显示详细信息。


# 全库恢复：recovery.conf
利用pg_basebackup备份恢复整个数据库。

检查表空间备份对应的恢复路径：
```sql
[postgres@dbhost ~]$ ll /pgdata/pg_tblspc/
total 0
lrwxrwxrwx 1 postgres postgres 14 Oct 26 13:08 16386 -> /pgdata/sekiro
```
可以得知`16386.tar.gz`是表空间sekiro的备份。

也可以查询pg_tablespace系统表获知：
```sql
postgres=# select oid,* from pg_tablespace;
  oid  |  spcname   | spcowner | spcacl | spcoptions 
-------+------------+----------+--------+------------
  1663 | pg_default |       10 |        | 
  1664 | pg_global  |       10 |        | 
 16386 | sekiro     |    16384 |        | 
(3 rows)
```

切换WAL日志：
```sql
postgres=# checkpoint;
postgres=# select pg_switch_wal();
```

删除数据：
```bash
#停库
pg_ctl stop

#移除数据目录下所有数据库文件
cd /pgdata && rm -rf *

#解压基础备份到数据目录
tar -xvf /home/postgres/backup/base.tar.gz -C /pgdata

#解压表空间备份到对应的数据库表空间路径（如果有）
tar -xvf /home/postgres/backup/16386.tar.gz -C /pgdata/sekiro

#解压日志备份到WAL日志路径（如果有）
tar -xvf /home/postgres/backup/pg_wal.tar.gz -C /pgdata/pg_wal
```

在`$PGDATA`路径下创建用于恢复的配置文件**recovery.conf**：
```bash
cp /usr/local/pgsql/share/recovery.conf.sample /pgdata/recovery.conf
chmod 600 recovery.conf
```

在**recovery.conf**中添加恢复命令，拷贝归档日志到数据库原始路径下；
```
echo "restore_command='cp /pg_arch/20231026/%f %p'" > /pgdata/recovery.conf
echo "recovery_target_timeline = 'latest'" >> /pgdata/recovery.conf
```

启动数据库，自动读取**recovery.conf**进入恢复恢复：
```bash
[postgres@dbhost ~]$ pg_ctl start -D /pgdata
waiting for server to start....2023-10-26 14:08:30.862 CST [10503] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2023-10-26 14:08:30.862 CST [10503] LOG:  listening on IPv6 address "::", port 5432
2023-10-26 14:08:30.864 CST [10503] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2023-10-26 14:08:30.939 CST [10503] LOG:  redirecting log output to logging collector process
2023-10-26 14:08:30.939 CST [10503] HINT:  Future log output will appear in directory "log".
 done
server started
```
恢复完成后，recovery.conf会被重命名为**recovery.done**。

检查告警日志中的恢复信息：
```bash
[postgres@dbhost ~]$ tail -f /pgdata/log/server_Thu.log 
cp: cannot stat ‘/pg_arch/20231026/000000020000000000000010’: No such file or directory
2023-10-26 14:08:31.609 CST [10505] LOG:  redo done at 0/F0000D0
2023-10-26 14:08:31.621 CST [10505] LOG:  restored log file "00000002000000000000000F" from archive
cp: cannot stat ‘/pg_arch/20231026/00000003.history’: No such file or directory
2023-10-26 14:08:32.032 CST [10505] LOG:  selected new timeline ID: 3
2023-10-26 14:08:32.531 CST [10505] LOG:  archive recovery complete
2023-10-26 14:08:32.533 CST [10505] LOG:  restored log file "00000002.history" from archive
2023-10-26 14:08:32.638 CST [10503] LOG:  database system is ready to accept connections
```

**References**
【1】https://blog.csdn.net/jnrjian/article/details/129945206
【2】https://www.postgresql.org/docs/9.4/runtime-config-replication.html
【3】https://www.modb.pro/db/332203

