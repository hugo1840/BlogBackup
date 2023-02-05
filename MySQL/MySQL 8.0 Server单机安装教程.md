---
tags: [mysql]
title: MySQL 8.0 Server单机安装教程
created: '2023-02-05T04:18:08.711Z'
modified: '2023-02-05T07:24:02.942Z'
---

MySQL 8.0 Server单机安装教程

>:cookie:数据库版本：MySQL 8.0.32
>:coffee:操作系统版本：CentOS 7.7

# 准备工作
root用户执行：
```bash
# 卸载mariadb
rpm -qa | grep mariadb
yum remove mariadb-libs -y

# 关防火墙
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld

# 关SELinux
sed -i 's/enforcing/disabled/' /etc/sysconfig/selinux
setenforce 0

# 创建mysql用户
groupadd mysql
useradd -g mysql -s /bin/bash mysql
mkdir /mysql
chown -R mysql:mysql /mysql

# 创建LV并挂载mysql目录
lsblk
yum install -y lvm2
pvcreate /dev/vdb
vgcreate vg_mysql /dev/vdb
lvcreate -l 100%FREE -n lv_mysql vg_mysql
lvs

mkfs.xfs /dev/vg_mysql/lv_mysql
mount /dev/vg_mysql/lv_mysql /mysql
df -Th

cp mysql-8.0.32-el7-x86_64.tar.gz /mysql
chown mysql:mysql /mysql/mysql-8.0.32-el7-x86_64.tar.gz
```

配置环境变量：
```bash
echo "export PATH=/mysql/mysql-8.0/bin:$PATH" >> /etc/profile
source /etc/profile
```

安装libaio：
```bash
yum install -y libaio
```

# MySQL安装
```bash
su - mysql
cd /mysql
tar -zxvf mysql-8.0.32-el7-x86_64.tar.gz
mv mysql-8.0.32-el7-x86_64 mysql-8.0

mkdir data
mkdir log
mkdir tmp
```

## MySQL配置文件
创建MySQL配置文件`/etc/my.cnf`，编辑内容如下：
```
[mysqld]
gtid-mode = ON
enforce-gtid-consistency = ON
symbolic-links = 0
user = mysql
basedir = /mysql/mysql-8.0
datadir = /mysql/data
port = 3306
server-id = 100
core-file
log_bin = /mysql/log/mysql-bin
binlog_format = ROW
socket = /tmp/mysql.sock
log_output = FILE
character_set_server = utf8
slow_query_log_file = /mysql/log/slow.log
query_cache_type = 0
long_query_time = 3
max_connections = 1024
max_connect_errors = 1024
local_infile = 0
general_log = OFF
slow_query_log = ON
relay-log = /mysql/log/relay-log
expire_logs_days = 15
innodb_io_capacity = 500 
innodb_flush_method = O_DIRECT
innodb_file_format = Barracuda
innodb_file_format_max = Barracuda
innodb_log_file_size = 1G
innodb_file_per_table = ON
innodb_lock_wait_timeout = 5
innodb_buffer_pool_size = 5G
innodb_print_all_deadlocks = ON
#innodb_additional_mem_pool_size = 32M
innodb_data_file_path = ibdata1:512M:autoextend
innodb_autoextend_increment = 64
innodb_thread_concurrency = 0
innodb_old_blocks_time = 1000
innodb_buffer_pool_instances = 8
thread_cache_size = 200
innodb_lru_scan_depth = 512
innodb_flush_neighbors = 1
innodb_checksum_algorithm = crc32
table_definition_cache = 400
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
innodb_read_io_threads = 4
innodb_adaptive_flushing = ON
innodb_log_buffer_size = 8388608
innodb_purge_threads = 4
performance_schema = ON
innodb_write_io_threads = 4
skip-name-resolve = ON
skip_external_locking = ON
max_allowed_packet = 16M
table_open_cache = 400
innodb_flush_log_at_trx_commit = 1
log_bin_trust_function_creators = 1
sync_binlog = 1
slave_net_timeout = 30
relay_log_info_repository = TABLE # slave SQL thread crash safe
master_info_repository = FILE
relay_log_recovery = ON
lower_case_table_names = 1
sql_mode = NO_ENGINE_SUBSTITUTION
log_error = /mysql/log/mysqld_err.log
pid-file= mysqld.pid
tmpdir=/mysql/tmp

[mysql]
socket = /tmp/mysql.sock
```

修改配置文件权限：
```bash
chmod 0660 /etc/my.cnf
chown mysql:mysql /etc/my.cnf
```

## systemd服务文件
配置systemd服务文件`/usr/lib/systemd/system/mysqld.service`，内容如下：
```
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql

Type=forking

# Disable service start and stop timeout logic of systemd for mysqld service.
TimeoutSec=0

# Start main service 
ExecStart=/mysql/mysql-8.0/bin/mysqld --defaults-file=/etc/my.cnf --daemonize

# Use this to switch malloc implementation
EnvironmentFile=-/etc/sysconfig/mysql

# Sets open_files_limit
LimitNOFILE = 60000

#Restart=on-failure  #这里最好先注释掉这一行，否则初始化失败时会不停地自动重启服务
RestartPreventExitStatus=1

PrivateTmp=false
```

修改文件权限，并重载系统服务：
```bash
chmod 644 /usr/lib/systemd/system/mysqld.service

systemctl daemon-reload
```

## MySQL初始化
使用配置文件初始化MySQL：
```bash
su - mysql
/mysql/mysql-8.0/bin/mysqld --defaults-file=/etc/my.cnf --initialize
```

以root身份启动MySQL服务：
```bash
[root@mysqldb ~]# systemctl start mysqld
Job for mysqld.service failed because the control process exited with error code. See "systemctl status mysqld.service" and "journalctl -xe" for details.
```

检查MySQL错误日志：
```bash
...
2023-02-05T06:41:27.781751Z 0 [ERROR] [MY-013129] [Server] A message intended for a client cannot be sent there as no client-session is attached. Therefore, we're sending the information to the error-log instead: MY-001146 - Table 'mysql.component' doesn't exist
2023-02-05T06:41:27.781767Z 0 [Warning] [MY-013129] [Server] A message intended for a client cannot be sent there as no client-session is attached. Therefore, we're sending the information to the error-log instead: MY-003543 - The mysql.component table is missing or has an incorrect definition.
2023-02-05T06:41:27.781816Z 0 [ERROR] [MY-000067] [Server] unknown variable 'query_cache_type=0'.
2023-02-05T06:41:27.781859Z 0 [ERROR] [MY-010119] [Server] Aborting
```

## MySQL 8过期的配置参数
查阅MySQL 8.0官方手册的值，以下参数已被列为**Deprecated**（包括但不限于）：
- `expire_logs_days`
- `innodb_log_file_size`
- `relay_log_info_repository`
- `skip-slave-start`
- `slave_net_timeout`
- `sql_slave_skip_counter`
- `symbolic-links`

以下配置参数已被移除（包括但不限于）：
- `innodb_file_format`
- `innodb_file_format_max`
- `query_cache_type`
- `query_cache_size`


>:dolphin:更多被移除的参数参见：https://dev.mysql.com/doc/refman/8.0/en/added-deprecated-removed.html#optvars-deprecated

## 修改MySQL配置文件
移除`my.cnf`配置文件中的过期参数。修改后内容如下：
```
[mysqld]
gtid-mode = ON
enforce-gtid-consistency = ON
user = mysql
basedir = /mysql/mysql-8.0
datadir = /mysql/data
port = 3306
server-id = 100
core-file
log_bin = /mysql/log/mysql-bin
binlog_format = ROW
socket = /tmp/mysql.sock
log_output = FILE
character_set_server = utf8
slow_query_log_file = /mysql/log/slow.log
long_query_time = 3
max_connections = 1024
max_connect_errors = 1024
local_infile = 0
general_log = OFF
slow_query_log = ON
relay-log = /mysql/log/relay-log
innodb_io_capacity = 500
innodb_flush_method = O_DIRECT
innodb_file_per_table = ON
innodb_lock_wait_timeout = 5
innodb_buffer_pool_size = 5G
innodb_print_all_deadlocks = ON
innodb_data_file_path = ibdata1:512M:autoextend
innodb_autoextend_increment = 64
innodb_thread_concurrency = 0
innodb_old_blocks_time = 1000
innodb_buffer_pool_instances = 8
thread_cache_size = 200
innodb_lru_scan_depth = 512
innodb_flush_neighbors = 1
innodb_checksum_algorithm = crc32
table_definition_cache = 400
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup = ON
innodb_read_io_threads = 4
innodb_adaptive_flushing = ON
innodb_log_buffer_size = 8388608
innodb_purge_threads = 4
performance_schema = ON
innodb_write_io_threads = 4
skip-name-resolve = ON
skip_external_locking = ON
max_allowed_packet = 16M
table_open_cache = 400
innodb_flush_log_at_trx_commit = 1
log_bin_trust_function_creators = 1
sync_binlog = 1
master_info_repository = FILE
relay_log_recovery = ON
lower_case_table_names = 1
sql_mode = NO_ENGINE_SUBSTITUTION
log_error = /mysql/log/mysqld_err.log
pid-file= mysqld.pid
tmpdir=/mysql/tmp

[mysql]
socket = /tmp/mysql.sock
```

## 重新初始化MySQL
重新初始化MySQL：
```bash
# 清理MySQL目录
su - mysql
cd /mysql
rm -rf data
rm -rf log
mkdir data
mkdir log

# 重新初始化
/mysql/mysql-8.0/bin/mysqld --defaults-file=/etc/my.cnf --initialize
```

以root身份启动MySQL服务：
```bash
[root@mysqldb ~]# systemctl start mysqld
[root@mysqldb ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-02-05 15:10:32 CST; 9s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 12867 ExecStart=/mysql/mysql-8.0/bin/mysqld --defaults-file=/etc/my.cnf --daemonize (code=exited, status=0/SUCCESS)
 Main PID: 12869 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─12869 /mysql/mysql-8.0/bin/mysqld --defaults-file=/etc/my.cnf --daemonize

Feb 05 15:10:31 mysqldb systemd[1]: Starting MySQL Server...
Feb 05 15:10:32 mysqldb systemd[1]: Started MySQL Server.
```

获取临时密码：
```bash
[root@mysqldb ~]# cat /mysql/log/mysqld_err.log | grep  'temporary password'
2023-02-05T07:10:06.385517Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: ;,KzEaNZe5KZ
```

修改MySQL root用户密码：
```sql
SQL> alter user 'root'@'localhost' identified by 'XXXXX';
```

修改并重载MySQL服务：
```bash
vim /usr/lib/systemd/system/mysqld.service
#取消注释Restart=on-failure

systemctl daemon-reload
systemctl restart mysqld
#systemctl enable mysqld
```

登录数据库检查：
```sql
mysql> show variables like '%gtid%';
+----------------------------------+----------------------------------------+
| Variable_name                    | Value                                  |
+----------------------------------+----------------------------------------+
| binlog_gtid_simple_recovery      | ON                                     |
| enforce_gtid_consistency         | ON                                     |
| gtid_executed                    | 1be40f24-a524-11ed-8c6e-00163e01628b:1 |
| gtid_executed_compression_period | 0                                      |
| gtid_mode                        | ON                                     |
| gtid_next                        | AUTOMATIC                              |
| gtid_owned                       |                                        |
| gtid_purged                      |                                        |
| session_track_gtids              | OFF                                    |
+----------------------------------+----------------------------------------+
9 rows in set (0.01 sec)
```


**References**
[1] https://blog.csdn.net/Sebastien23/article/details/122517919
[2] https://dev.mysql.com/doc/refman/8.0/en/using-systemd.html
[3] https://dev.mysql.com/doc/refman/8.0/en/added-deprecated-removed.html



