---
tags: [mysql]
title: MySQL高可用架构之InnoDB ReplicaSet部署
created: '2023-03-04T01:04:57.793Z'
modified: '2023-03-04T07:26:23.798Z'
---

MySQL高可用架构之InnoDB ReplicaSet部署

**环境准备**
| 主机名 | IP | OS版本 | MySQL版本 |
| :-: | :-: | :-: | :-: |
| mysql-rs-1 | 172.16.171.70 | CentOS 7.7 | 8.0.32 |
| mysql-rs-2 | 172.16.171.71 | CentOS 7.7 | 8.0.32 |

# ReplicaSet适用场景
InnoDB ReplicaSet至少由两台MySQL服务器实例组成，提供MySQL主从复制功能。InnoDB ReplicaSet基于**异步**的主从复制实现，因此适用于用户对高可用性要求不高的环境。可以通过MySQL Shell快速搭建及管理主从复制，避免了搭建主从复制时大量的手动操作。

InnoDB ReplicaSet的不足之处有：
- 不支持自动故障转移。主库不可用时，需要通过AdminAPI**手动**发起故障转移。
- 发生计划外的服务不可用时，可能会丢失部分数据。由于主备之间是异步复制，主库发生故障时，未提交的事务会丢失。
- 发生计划外的服务不可用时，可能会产生数据不一致。例如由于网络原因导致主库连不上，将备库提升为主库后，可能会同时存在两个主库，即发生“**脑裂**”。
- 不支持多主模式，即同一时刻只有一个主库可写。
- 读扩展受限，不能像组复制那样对流量进行控制。
- 所有的备库都从同一个主库复制数据。在有大量的小更新时，可能会对主库造成影响。
- 仅支持MySQL **8.0**及其以后的版本。
- 仅支持基于**GTID**的日志复制。
- 仅支持基于**行**的日志复制（Row-Based Replication, RBR），不支持基于SQL语句的复制（Statement-Based Replication, SBR）。
- 不支持复制过滤。
- RS为一个主库加多个从库的架构。需要通过MySQL Router监视RS中的实例，因此从库的数量不能无限制增加。
- 必须通过**MySQL Shell**配置和管理，包括复制用户的创建。


# 准备工作
需要先安装两台单机MySQL，具体教程参见[MySQL 8.0 Server单机安装教程](https://blog.csdn.net/Sebastien23/article/details/128890837)。

然后配置好两台服务器的`/etc/hosts`：
```bash
echo "172.16.171.70   mysql-rs-1      mysql-rs-1" >> /etc/hosts
echo "172.16.171.71   mysql-rs-2      mysql-rs-2" >> /etc/hosts
```

最后，注意对比两个MySQL实例的`server-id`和`server-uuid`是否不同。

# 安装MySQL Shell
官方推荐通过配置MySQL YUM源安装。这里我们直接通过下载解压压缩包来安装（两台服务器都安装）。
```bash
[root@mysql-rs-1 ~]# wget https://dev.mysql.com/get/Downloads/MySQL-Shell/mysql-shell-8.0.32-linux-glibc2.12-x86-64bit.tar.gz

[root@mysql-rs-1 ~]# tar xvf mysql-shell-8.0.32-linux-glibc2.12-x86-64bit.tar.gz -C /usr/local/ 
```

添加到环境变量：
```bash
[root@mysql-rs-1 ~]# ln -s /usr/local/mysql-shell-8.0.32-linux-glibc2.12-x86-64bit /usr/local/mysql-shell 

[root@mysql-rs-1 ~]# echo "export PATH=$PATH:/usr/local/mysql-shell/bin" >> /root/.bash_profile
[root@mysql-rs-1 ~]# source /root/.bash_profile
```

测试MySQL Shell连接到数据库：
```bash
# 通过经典MySQL协议连接到数据库
[root@mysql-rs-1 ~]# mysqlsh --mysql -u root -h localhost -C
Please provide the password for 'root@localhost': **********
Save password for 'root@localhost'? [Y]es/[N]o/Ne[v]er (default No): y
MySQL Shell 8.0.3

Creating a Classic session to 'root@localhost?compression=REQUIRED'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 9
MySQL  localhost  JS > \exit
Bye!

# 通过URL连接到数据库
[root@mysql-rs-1 ~]# mysqlsh mysql://root@localhost:3306
Please provide the password for 'root@localhost:3306': **********
MySQL Shell 8.0.32

Creating a Classic session to 'root@localhost:3306'
MySQL Error 1045 (28000): Access denied for user 'root'@'127.0.0.1' (using password: YES)

# 通过X Protocol连接到数据库
[root@mysql-rs-1 ~]# mysqlsh --mysqlx -u root -h localhost -P 33060
Please provide the password for 'root@localhost:33060': **********
MySQL Shell 8.0.32

Creating an X protocol session to 'root@localhost:33060'
MySQL Error 1045: Access denied for user 'root'@'127.0.0.1' (using password: YES)
```

出现上面报错的原因是数据库里没有`'root'@'127.0.0.1'`这个用户。我们先创建对应用户：
```sql
mysql> select user,host from mysql.user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| appuser          | %         |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+
5 rows in set (0.00 sec)

mysql> show grants for 'root'@'localhost';

--创建用户并授予管理员权限
mysql> create user 'root'@'%' identified with mysql_native_password by 'XXXXX';

mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, SHUTDOWN, PROCESS, FILE, REFERENCES, INDEX, ALTER, SHOW DATABASES, SUPER, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER, CREATE TABLESPACE, CREATE ROLE, DROP ROLE ON *.* TO `root`@`%` WITH GRANT OPTION;

mysql> GRANT APPLICATION_PASSWORD_ADMIN,AUDIT_ABORT_EXEMPT,AUDIT_ADMIN,AUTHENTICATION_POLICY_ADMIN,BACKUP_ADMIN,BINLOG_ADMIN,BINLOG_ENCRYPTION_ADMIN,CLONE_ADMIN,CONNECTION_ADMIN,ENCRYPTION_KEY_ADMIN,FIREWALL_EXEMPT,FLUSH_OPTIMIZER_COSTS,FLUSH_STATUS,FLUSH_TABLES,FLUSH_USER_RESOURCES,GROUP_REPLICATION_ADMIN,GROUP_REPLICATION_STREAM,INNODB_REDO_LOG_ARCHIVE,INNODB_REDO_LOG_ENABLE,PASSWORDLESS_USER_ADMIN,PERSIST_RO_VARIABLES_ADMIN,REPLICATION_APPLIER,REPLICATION_SLAVE_ADMIN,RESOURCE_GROUP_ADMIN,RESOURCE_GROUP_USER,ROLE_ADMIN,SENSITIVE_VARIABLES_OBSERVER,SERVICE_CONNECTION_ADMIN,SESSION_VARIABLES_ADMIN,SET_USER_ID,SHOW_ROUTINE,SYSTEM_USER,SYSTEM_VARIABLES_ADMIN,TABLE_ENCRYPTION_ADMIN,XA_RECOVER_ADMIN ON *.* TO `root`@`%` WITH GRANT OPTION;

mysql> GRANT PROXY ON ``@`` TO `root`@`%` WITH GRANT OPTION;
```

重新测试连接：
```bash
[root@mysql-rs-1 ~]# mysqlsh mysql://root@localhost:3306
Please provide the password for 'root@localhost:3306': **********
Save password for 'root@localhost:3306'? [Y]es/[N]o/Ne[v]er (default No): y
MySQL Shell 8.0.32

Creating a Classic session to 'root@localhost:3306'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 13
MySQL  localhost:3306 ssl  JS > \exit
Bye!

[root@mysql-rs-1 ~]# mysqlsh --mysqlx -u root -h localhost -P 33060
Please provide the password for 'root@localhost:33060': **********
Save password for 'root@localhost:33060'? [Y]es/[N]o/Ne[v]er (default No): y
MySQL Shell 8.0.32

Creating an X protocol session to 'root@localhost:33060'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 14 (X protocol)
MySQL  localhost:33060+ ssl  JS > \exit
Bye!
```

也可以在启动MySQL Shell后通过connect连接：
```javascript
[root@mysql-rs-1 ~]# mysqlsh
MySQL Shell 8.0.32

MySQL  JS > \connect mysqlx://root@localhost:33060
Creating an X protocol session to 'root@localhost:33060'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 16 (X protocol)
```

查看当前会话连接的信息：
```javascript
MySQL  localhost:33060+ ssl  JS > session
<Session:root@localhost:33060>

MySQL  localhost:33060+ ssl  JS > shell.status()
MySQL Shell version 8.0.32

Connection Id:                16
Default schema:               
Current schema:               
Current user:                 root@127.0.0.1
SSL:                          Cipher in use: TLS_AES_256_GCM_SHA384 TLSv1.3
Using delimiter:              ;
Server version:               8.0.32 MySQL Community Server - GPL
Protocol version:             X protocol
Client library:               8.0.32
Connection:                   localhost via TCP/IP
TCP port:                     33060
Server characterset:          utf8mb3
Schema characterset:          utf8mb3
Client characterset:          utf8mb4
Conn. characterset:           utf8mb4
Result characterset:          utf8mb4
Compression:                  Enabled (DEFLATE_STREAM)
Uptime:                       40 min 35.0000 sec
```

# 使用MySQL Shell搭建ReplicaSet
## 初始化第一个实例
配置第一个实例，并创建ReplicaSet管理员用户`rsadmin`（创建时会提示设置密码）。
```javascript
MySQL  localhost:33060+ ssl  JS > dba.configureReplicaSetInstance('root@mysql-rs-1:3306', {clusterAdmin: "'rsadmin'@'%'"})
Please provide the password for 'root@mysql-rs-1:3306': **********
Save password for 'root@mysql-rs-1:3306'? [Y]es/[N]o/Ne[v]er (default No): y
Configuring local MySQL instance listening at port 3306 for use in an InnoDB ReplicaSet...

This instance reports its own address as mysql-rs-1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.
Password for new account: *********
Confirm password: *********

applierWorkerThreads will be set to the default value of 4.

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| master_info_repository                 | FILE          | TABLE          | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
Do you want to perform the required configuration changes? [y/n]: y
Do you want to restart the instance after configuring it? [y/n]: y
Cluster admin user 'rsadmin'@'mysql-rs-1%' created.
Configuring instance...
The instance 'mysql-rs-1:3306' was configured to be used in an InnoDB ReplicaSet.
Restarting MySQL...
ERROR: Remote restart of MySQL server failed: MySQL Error 3707 (HY000): Restart server failed (mysqld is not managed by supervisor process).
Please restart MySQL manually (check https://dev.mysql.com/doc/refman/en/restart.html for more details).
Dba.configureReplicaSetInstance: Restart server failed (mysqld is not managed by supervisor process). (MYSQLSH 3707)
```

修改`my.cnf`配置文件中的`master_info_repository`为TABLE后，手动重启MySQL服务。
```bash
[root@mysql-rs-1 ~]# grep 'master_info_repository' /etc/my.cnf
master_info_repository = FILE
[root@mysql-rs-1 ~]# vim /etc/my.cnf
[root@mysql-rs-1 ~]# systemctl restart mysqld
[root@mysql-rs-1 ~]# grep 'master_info_repository' /etc/my.cnf
master_info_repository = TABLE
```

再次检查配置第一个实例：
```javascript
MySQL  localhost:33060+ ssl  JS > dba.configureReplicaSetInstance('root@mysql-rs-1:3306', {clusterAdmin: "'rsadmin'@'%'"})

Configuring local MySQL instance listening at port 3306 for use in an InnoDB ReplicaSet...

This instance reports its own address as mysql-rs-1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.
User 'rsadmin'@'mysql-rs-1%' already exists and will not be created.

applierWorkerThreads will be set to the default value of 4.

The instance 'mysql-rs-1:3306' is valid to be used in an InnoDB ReplicaSet.
The instance 'mysql-rs-1:3306' is already ready to be used in an InnoDB ReplicaSet.

Successfully enabled parallel appliers.
```

## 创建ReplicaSet
初始化完第一个ReplicaSet实例以后就可以创建ReplicaSet了。第一个实例会被选定为主实例。
```javascript
MySQL  localhost:33060+ ssl  JS > var rs = dba.createReplicaSet("rstest")
A new replicaset with instance 'mysql-rs-1:3306' will be created.

* Checking MySQL instance at mysql-rs-1:3306

This instance reports its own address as mysql-rs-1:3306
mysql-rs-1:3306: Instance configuration is suitable.

* Updating metadata...

ReplicaSet object successfully created for mysql-rs-1:3306.
Use rs.addInstance() to add more asynchronously replicated instances to this replicaset and rs.status() to check its status.
```

检查创建好的ReplicaSet状态：
```javascript
MySQL  localhost:33060+ ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "rstest", 
        "primary": "mysql-rs-1:3306", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "mysql-rs-1:3306": {
                "address": "mysql-rs-1:3306", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}
```

## 添加副本实例
初始化第二个实例：
```javascript
MySQL  localhost:33060+ ssl  JS > dba.configureReplicaSetInstance('root@mysql-rs-2:3306', {clusterAdmin: "'rsadmin'@'%'"})
```

将第二个实例添加到创建好的ReplicaSet中：
```javascript
MySQL  localhost:33060+ ssl  JS > var rs = dba.getReplicaSet()
You are connected to a member of replicaset 'rstest'.

MySQL  localhost:33060+ ssl  JS > rs.addInstance('mysql-rs-2:3306')
Adding instance to the replicaset...

* Performing validation checks
ReplicaSet.addInstance: server_uuid in mysql-rs-2:3306 is expected to be unique, but mysql-rs-1:3306 already uses the same value (MYSQLSH 51150)
```

修改两个实例的`/etc/my.cnf`中的`server-id`使其不同，然后重启数据库：
```bash
[root@mysql-rs-1 ~]# grep server-id /etc/my.cnf
server-id = 100
[root@mysql-rs-1 ~]# vim /etc/my.cnf
[root@mysql-rs-1 ~]# grep server-id /etc/my.cnf
server-id = 10
[root@mysql-rs-1 ~]# systemctl restart mysqld
```

尝试重新添加第二个ReplicaSet实例，发现还是报上面的错误：
```javascript
MySQL  localhost:33060+ ssl  JS > rs.addInstance('mysql-rs-2:3306')
Adding instance to the replicaset...

* Performing validation checks
ReplicaSet.addInstance: server_uuid in mysql-rs-2:3306 is expected to be unique, but mysql-rs-1:3306 already uses the same value (MYSQLSH 51150)
```

注意到上面提到的是`server_uuid`，检查对比MySQL数据目录下面的`auto.cnf`文件：
```bash
[root@mysql-rs-1 ~]# cat /mysql/data/auto.cnf
[auto]
server-uuid=1be40f24-a524-11ed-8c6e-00163e01628b

[root@mysql-rs-2 ~]# cat /mysql/data/auto.cnf 
[auto]
server-uuid=1be40f24-a524-11ed-8c6e-00163e01628b
```
发现**server-uuid**相同，因为这两台MySQL服务器是由同一个镜像生成的。

修改第二个实例的`server-uuid`，并重启数据库：
```bash
# 生成一个新的uuid
mysql> select uuid();
+--------------------------------------+
| uuid()                               |
+--------------------------------------+
| 4d71e8aa-ba49-11ed-ab7f-00163e011355 |
+--------------------------------------+
1 row in set (0.00 sec)

mysql> exit
Bye
[root@mysql-rs-2 ~]# vim /mysql/data/auto.cnf 
[root@mysql-rs-2 ~]# cat /mysql/data/auto.cnf 
[auto]
server-uuid=4d71e8aa-ba49-11ed-ab7f-00163e011355
[root@mysql-rs-2 ~]# systemctl restart mysqld
```

重新添加第二个实例到ReplicaSet：
```javascript
MySQL  localhost:33060+ ssl  JS > var rs = dba.getReplicaSet()
You are connected to a member of replicaset 'rstest'.
MySQL  localhost:33060+ ssl  JS > rs.addInstance('mysql-rs-2:3306')
Adding instance to the replicaset...

* Performing validation checks

This instance reports its own address as mysql-rs-2:3306
mysql-rs-2:3306: Instance configuration is suitable.

* Checking async replication topology...

* Checking transaction state of the instance...
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'mysql-rs-2:3306' with a physical snapshot from an existing replicaset member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

WARNING: It should be safe to rely on replication to incrementally recover the state of the new instance if you are sure all updates ever executed in the replicaset were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the replicaset or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.

Incremental state recovery was selected because it seems to be safely usable.

* Updating topology
** Changing replication source of mysql-rs-2:3306 to mysql-rs-1:3306
** Waiting for new instance to synchronize with PRIMARY...
** Transactions replicated  ############################################################  100% 

The instance 'mysql-rs-2:3306' was added to the replicaset and is replicating from mysql-rs-1:3306.

* Waiting for instance 'mysql-rs-2:3306' to synchronize the Metadata updates with the PRIMARY...
** Transactions replicated  ############################################################  100% 
```

如果提示第二个实例`has not been pre-provisioned (GTID set is empty)`，需要进行恢复，选择默认的**Clone**方式来恢复从库实例即可（会覆盖从库上已有的数据）。

最后，检查ReplicaSet状态：
```javascipt
MySQL  localhost:33060+ ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "rstest", 
        "primary": "mysql-rs-1:3306", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "mysql-rs-1:3306": {
                "address": "mysql-rs-1:3306", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }, 
            "mysql-rs-2:3306": {
                "address": "mysql-rs-2:3306", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for source to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}
```

在从库上可以看到同步的状态：
```sql
mysql> show replica status\G
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: mysql-rs-1
                  Source_User: mysql_innodb_rs_11
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000011
          Read_Source_Log_Pos: 3052
               Relay_Log_File: relay-log.000006
                Relay_Log_Pos: 3268
        Relay_Source_Log_File: mysql-bin.000011
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
          ...
```
上面的`mysql_innodb_rs_11`用户是从库在加入ReplicaSet后自动创建的主从同步用户。


# ReplicaSet手动故障转移
## 计划内的故障转移
使用`ReplicaSet.setPrimaryInstance()`命令来将指定的副本切换为主实例：
```javascript
MySQL  localhost:3306 ssl  JS > rs.setPrimaryInstance('mysql-rs-2:3306');
mysql-rs-2:3306 will be promoted to PRIMARY of 'rstest'.
The current PRIMARY is mysql-rs-1:3306.

* Connecting to replicaset instances
** Connecting to mysql-rs-1:3306
** Connecting to mysql-rs-2:3306
** Connecting to mysql-rs-1:3306
** Connecting to mysql-rs-2:3306

* Performing validation checks
** Checking async replication topology...
** Checking transaction state of the instance...

* Synchronizing transaction backlog at mysql-rs-2:3306
** Transactions replicated  ############################################################  100% 

* Updating metadata

* Acquiring locks in replicaset instances
** Pre-synchronizing SECONDARIES
** Acquiring global lock at PRIMARY
** Acquiring global lock at SECONDARIES

* Updating replication topology
** Changing replication source of mysql-rs-1:3306 to mysql-rs-2:3306

mysql-rs-2:3306 was promoted to PRIMARY.
```

## 计划外的故障转移
当主库不可用且无法恢复时，使用`ReplicaSet.forcePrimaryInstance()`命令来发起强制切换：
```javascript
MySQL  localhost:3306 ssl  JS > rs.forcePrimaryInstance('mysql-rs-2:3306');
MySQL  localhost:3306 ssl  JS > rs.status()
```

# 部署MySQL Router
MySQL Router可以为ReplicaSet提供应用程序的透明路由和读写分离，建议与应用安装在同一台服务器上。

## 创建MySQL Router账户
为MySQL Router创建ReplicaSet账号：
```javascript
MySQL  localhost:3306 ssl  JS > rs.setupRouterAccount('rs_router1')

Missing the password for new account rs_router1@%. Please provide one.
Password for new account: *********
Confirm password: *********

Creating user rs_router1@%.
Account rs_router1@% was successfully created.
```

## 安装MySQL Router
下载并解压MySQL Router安装包：
```bash
[root@mysql-rs-1 ~]# wget https://dev.mysql.com/get/Downloads/MySQL-Router/mysql-router-8.0.32-linux-glibc2.12-x86_64.tar.xz

[root@mysql-rs-1 ~]# tar xvf mysql-router-8.0.32-linux-glibc2.12-x86_64.tar.xz -C /usr/local/ 
```

添加到环境变量：
```bash
[root@mysql-rs-1 ~]# ln -s /usr/local/mysql-router-8.0.32-linux-glibc2.12-x86_64 /usr/local/mysql-router 

[root@mysql-rs-1 ~]# echo "export PATH=$PATH:/usr/local/mysql-router/bin" >> /root/.bash_profile
[root@mysql-rs-1 ~]# source /root/.bash_profile
```

## 引导启动MySQL Router
以MySQL用户来引导MySQL Router启动（不建议使用root）：
```bash
[root@mysql-rs-1 ~]# mysqlrouter --bootstrap root@mysql-rs-1:3306 --directory=/tmp/myrouter --conf-use-sockets --account rs_router1 --user=mysql
Please enter MySQL password for root: 
# Bootstrapping MySQL Router instance at '/tmp/myrouter'...

Please enter MySQL password for rs_router1: 
Error: It appears that a router instance named '' has been previously configured in this host. If that instance no longer exists, use the --force option to overwrite it.
[root@mysql-rs-1 ~]# mysqlrouter --bootstrap root@mysql-rs-1:3306 --directory=/tmp/myrouter --conf-use-sockets --account rs_router1 --user=mysql --force
Please enter MySQL password for root: 
# Bootstrapping MySQL Router instance at '/tmp/myrouter'...

Please enter MySQL password for rs_router1: 
- Creating account(s) (only those that are needed, if any)
- Verifying account (using it to run SQL queries that would be run by Router)
- Storing account in keyring
- Adjusting permissions of generated files
- Creating configuration /tmp/myrouter/mysqlrouter.conf

# MySQL Router configured for the InnoDB ReplicaSet 'rstest'

After this MySQL Router has been started with the generated configuration

    $ mysqlrouter -c /tmp/myrouter/mysqlrouter.conf

InnoDB ReplicaSet 'rstest' can be reached by connecting to:

## MySQL Classic protocol

- Read/Write Connections: localhost:6446, /tmp/myrouter/mysql.sock
- Read/Only Connections:  localhost:6447, /tmp/myrouter/mysqlro.sock

## MySQL X protocol

- Read/Write Connections: localhost:6448, /tmp/myrouter/mysqlx.sock
- Read/Only Connections:  localhost:6449, /tmp/myrouter/mysqlxro.sock
```
可以看到，MySQL Router生成了四个连接路由，两个读写和两个只读。

检查MySQL Router自动生成的启动脚本和配置文件：
```bash
[root@mysql-rs-1 ~]# ls /tmp/myrouter
data  log  mysqlrouter.conf  mysqlrouter.key  run  start.sh  stop.sh

[root@mysql-rs-1 ~]# cat /tmp/myrouter/mysqlrouter.conf
# File automatically generated during MySQL Router bootstrap
...
[routing:bootstrap_rw]  ## 读写连接
bind_address=0.0.0.0
bind_port=6446
socket=/tmp/myrouter/mysql.sock
destinations=metadata-cache://rstest/?role=PRIMARY
routing_strategy=first-available
protocol=classic

[routing:bootstrap_ro]  ## 只读连接
bind_address=0.0.0.0
bind_port=6447
socket=/tmp/myrouter/mysqlro.sock
destinations=metadata-cache://rstest/?role=SECONDARY
routing_strategy=round-robin-with-fallback
protocol=classic

[routing:bootstrap_x_rw]   ## 读写连接
bind_address=0.0.0.0
bind_port=6448
socket=/tmp/myrouter/mysqlx.sock
destinations=metadata-cache://rstest/?role=PRIMARY
routing_strategy=first-available
protocol=x

[routing:bootstrap_x_ro]  ## 只读连接
bind_address=0.0.0.0
bind_port=6449
socket=/tmp/myrouter/mysqlxro.sock
destinations=metadata-cache://rstest/?role=SECONDARY
routing_strategy=round-robin-with-fallback
protocol=x
...
```


## 启停MySQL Router
使用生成的脚本启动MySQL Router：
```bash
[root@mysql-rs-1 ~]# sh /tmp/myrouter/start.sh 
[root@mysql-rs-1 ~]# PID 7441 written to '/tmp/myrouter/mysqlrouter.pid'
stopping to log to the console. Continuing to log to filelog
```

测试MySQL Router的可读写连接：
```javascript
[root@mysql-rs-1 ~]# mysqlsh
MySQL Shell 8.0.32

MySQL  JS > \connect root@localhost:6446
Creating a session to 'root@localhost:6446'
Please provide the password for 'root@localhost:6446': **********
Your MySQL connection id is 431

MySQL  localhost:6446 ssl  JS > session
<ClassicSession:root@localhost:6446>

MySQL  localhost:6446 ssl  JS > \sql
Switching to SQL mode... Commands end with ;

MySQL  localhost:6446 ssl  SQL > select count(*) from mysql.user;
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.0009 sec)

MySQL  localhost:6446 ssl  SQL > create database testdb_01;
Query OK, 1 row affected (0.0019 sec)
MySQL  localhost:6446 ssl  SQL > \exit
Bye!
```

测试MySQL Router的只读连接：
```javascript
MySQL  JS > \connect root@localhost:6447
Creating a session to 'root@localhost:6447'
Please provide the password for 'root@localhost:6447': **********
Your MySQL connection id is 994

MySQL  localhost:6447 ssl  JS > \sql
Switching to SQL mode... Commands end with ;
Fetching global names for auto-completion... Press ^C to stop.
 
MySQL  localhost:6447 ssl  SQL > select count(*) from mysql.user;
+----------+
| count(*) |
+----------+
|       10 |
+----------+
1 row in set (0.0011 sec)

MySQL  localhost:6447 ssl  SQL > create database testdb_02;
ERROR: 1290 (HY000): The MySQL server is running with the --super-read-only option so it cannot execute this statement
MySQL  localhost:6447 ssl  SQL > \exit
Bye!
```

下面我们测试主从切换后MySQL Router能否正常工作。
```javascript
MySQL  JS > \connect root@localhost:6446
Your MySQL connection id is 1570

MySQL  localhost:6446 ssl  JS > var rs = dba.getReplicaSet()
You are connected to a member of replicaset 'rstest'.
MySQL  localhost:6446 ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "rstest", 
        "primary": "mysql-rs-1:3306", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "mysql-rs-1:3306": {
                "address": "mysql-rs-1:3306", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }, 
            "mysql-rs-2:3306": {
                "address": "mysql-rs-2:3306", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for source to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}

MySQL  localhost:6446 ssl  JS > rs.setPrimaryInstance('mysql-rs-2:3306')
...
mysql-rs-2:3306 was promoted to PRIMARY.

MySQL  localhost:6446 ssl  JS > rs.status()
ReplicaSet.status: mysql-rs-1:3306: Lost connection to MySQL server during query (MySQL Error 2013)
MySQL  localhost:6446 ssl  JS > \exit
Bye!
```

发生主从切换后，已有的连接会断开，需要重新建立连接。
```javascript
MySQL  JS > \connect root@localhost:6446
Your MySQL connection id is 1596

MySQL  localhost:6446 ssl  JS > var rs = dba.getReplicaSet()
You are connected to a member of replicaset 'rstest'.
MySQL  localhost:6446 ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "rstest", 
        "primary": "mysql-rs-2:3306", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "mysql-rs-1:3306": {
                "address": "mysql-rs-1:3306", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for source to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }, 
            "mysql-rs-2:3306": {
                "address": "mysql-rs-2:3306", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}

MySQL  localhost:6446 ssl  JS > \sql
MySQL  localhost:6446 ssl  SQL > show databases;
+-------------------------------+
| Database                      |
+-------------------------------+
| appdb                         |
| information_schema            |
| mysql                         |
| mysql_innodb_cluster_metadata |
| performance_schema            |
| sys                           |
| testdb_01                     |
+-------------------------------+
7 rows in set (0.0009 sec)
```
重新通过MySQL Router建立连接后，可以看到主实例已经更新为`mysql-rs-2`。

为了方便，可以将MySQL Router启动脚本`/tmp/myrouter/start.sh`添加到开机自启动。



