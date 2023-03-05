---
tags: [mysql]
title: MySQL高可用架构之InnoDB Cluster部署
created: '2023-03-04T01:05:50.266Z'
modified: '2023-03-05T06:55:51.880Z'
---

MySQL高可用架构之InnoDB Cluster部署

**环境准备**
| 主机名 | IP | OS版本 | MySQL版本 |
| :-: | :-: | :-: | :-: |
| mysql-ic-1 | 172.16.x.y1 | CentOS 7.7 | 8.0.32 |
| mysql-ic-2 | 172.16.x.y2 | CentOS 7.7 | 8.0.32 |
| mysql-ic-3 | 172.16.x.y3 | CentOS 7.7 | 8.0.32 |

# InnoDB Cluster适用场景
MySQL InnoDB Cluster是一套完整部署和管理MySQL的高可用性解决方案，整合了MySQL的多项技术，以弥补组复制无法提供具有自动化故障转移功能的中间件，无法自动配置等不足。InnoDB Cluster需要至少**3**台MySQL服务器实例组成，并且提供高可用性和扩展功能。 

:pizza:InnoDB Cluster包括如下组件:
- **MySQL Shell**：MySQL的高级客户端、管理工具和代码编辑器。
- MySQL Server和**Group Replication**：使一组MySQL实例能够提供高可用性。组复制通过分布式状态机实现数据同步，组中的所有服务器都可以参与数据更新，并自动解决数据冲突。
- **X DevAPI**和**AdminAPI**：X DevAPI应用编程接口是一个类和方法库，实现了MySQL的NoSQL接口。使用X DevAPI必须启用X插件。AdminAPI实现了管理InnoDB Cluster的接口。
- **MySQL Router**：一种轻量级的中间件，提供负载均衡功能，并可在应用程序和多台MySQL实例之间提供透明的连接路由。


:dolphin:搭建InnoDB Cluster需要满足的要求如下：
- InnoDB集群使用了Group Replication，因此必须满足使用组复制的要求。具体可以参见`https://dev.mysql.com/doc/refman/8.0/en/group-replication-requirements.html`。其中比较重要的几点有：
  - 必须开启二进制日志，并且日志格式为ROW，即`--log-bin`和`binglog_format=ROW`；
  - 必须开启副本更新日志，即`log_slave_updates=ON`；
  - 必须开启GTID，即`gtid_mode=ON`和`enforce_gtid_consistency=ON`。

- 存储引擎只能使用**InnoDB**。最好禁用其他存储引擎：
```bash
disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
```

- 任一实例上**不**能有入站复制通道（inbound replication channel）。
- `group_replication_tls_source`参数不能设置为`mysql_admin`。
- 必须启用Performance Schema。
- Shell环境中必须有**python**。
```bash
/usr/bin/env python
```

- 集群中的实例必须使用不同的`server_id`。
- 从8.0.23开始，集群中的实例要启用并行复制。需要配置以下系统变量：
```bash
binlog_transaction_dependency_tracking=WRITESET
slave_preserve_commit_order=ON
slave_parallel_type=LOGICAL_CLOCK
transaction_write_set_extraction=XXHASH64
```

- 事务隔离级别默认为可重复读。如果要使用多主集群，需要将`transaction_isolation`参数修改为提交读。
- 只支持一个参数文件（option file），**不**支持使用`--defaults-extra-file`参数选项。


:octopus:InnoDB Cluster的不足主要有：
- MySQL Server最好使用8.0.26及其以后的版本，否则可能与MySQL Shell出现兼容性问题。
- InnoDB Cluster不支持管理手动配置的异步复制通道。
- InnoDB Cluster是为局域网内部署而设计的。在广域网内部署的集群的写性能会受到明显的影响。
- 对AdminAPI操作，仅支持通过TCP/IP连接和经典MySQL协议连接到MySQL实例，不支持Unix socket、named pipes和X Protocol连接。
- 使用多主模式时，不支持在多个实例上同时对同一个数据库对象进行DDL和DML操作。


# 准备工作
需要先安装三台单机MySQL，具体教程参见[MySQL 8.0 Server单机安装教程](https://blog.csdn.net/Sebastien23/article/details/128890837)。

然后配置好三台服务器的`/etc/hosts`：
```bash
echo "172.16.x.y1   mysql-ic-1      mysql-ic-1" >> /etc/hosts
echo "172.16.x.y2   mysql-ic-2      mysql-ic-2" >> /etc/hosts
echo "172.16.x.y3   mysql-ic-3      mysql-ic-3" >> /etc/hosts
```

检查三台服务器MySQL的**server-id**和**server-uuid**，使其不同。如果相同，修改后重启MySQL服务。
```bash
grep 'server-id' /etc/my.cnf
grep 'server-uuid' /mysql/data/auto.cnf
```

创建`'root'@'%'`用户并授予管理员权限（如果没有）：
```sql
mysql> create user 'root'@'%' identified with mysql_native_password by 'XXXXX';

mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, SHUTDOWN, PROCESS, FILE, REFERENCES, INDEX, ALTER, SHOW DATABASES, SUPER, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER, CREATE TABLESPACE, CREATE ROLE, DROP ROLE ON *.* TO `root`@`%` WITH GRANT OPTION;

mysql> GRANT APPLICATION_PASSWORD_ADMIN,AUDIT_ABORT_EXEMPT,AUDIT_ADMIN,AUTHENTICATION_POLICY_ADMIN,BACKUP_ADMIN,BINLOG_ADMIN,BINLOG_ENCRYPTION_ADMIN,CLONE_ADMIN,CONNECTION_ADMIN,ENCRYPTION_KEY_ADMIN,FIREWALL_EXEMPT,FLUSH_OPTIMIZER_COSTS,FLUSH_STATUS,FLUSH_TABLES,FLUSH_USER_RESOURCES,GROUP_REPLICATION_ADMIN,GROUP_REPLICATION_STREAM,INNODB_REDO_LOG_ARCHIVE,INNODB_REDO_LOG_ENABLE,PASSWORDLESS_USER_ADMIN,PERSIST_RO_VARIABLES_ADMIN,REPLICATION_APPLIER,REPLICATION_SLAVE_ADMIN,RESOURCE_GROUP_ADMIN,RESOURCE_GROUP_USER,ROLE_ADMIN,SENSITIVE_VARIABLES_OBSERVER,SERVICE_CONNECTION_ADMIN,SESSION_VARIABLES_ADMIN,SET_USER_ID,SHOW_ROUTINE,SYSTEM_USER,SYSTEM_VARIABLES_ADMIN,TABLE_ENCRYPTION_ADMIN,XA_RECOVER_ADMIN ON *.* TO `root`@`%` WITH GRANT OPTION;

mysql> GRANT PROXY ON ``@`` TO `root`@`%` WITH GRANT OPTION;
```

# 安装MySQL Shell
官方推荐通过配置MySQL YUM源安装。这里我们直接通过下载解压压缩包来安装（两台服务器都安装）。
```bash
wget https://dev.mysql.com/get/Downloads/MySQL-Shell/mysql-shell-8.0.32-linux-glibc2.12-x86-64bit.tar.gz

tar xvf mysql-shell-8.0.32-linux-glibc2.12-x86-64bit.tar.gz -C /usr/local/ 
```

添加到环境变量：
```bash
ln -s /usr/local/mysql-shell-8.0.32-linux-glibc2.12-x86-64bit /usr/local/mysql-shell 

echo "export PATH=$PATH:/usr/local/mysql-shell/bin" >> /root/.bash_profile
source /root/.bash_profile
```

测试X Protocol连接：
```bash
mysqlsh
MySQL  JS > \connect mysqlx://root@localhost:33060
```

# 使用MySQL Shell搭建InnoDB Cluster
## 初始化第一个实例
检查实例配置是否满足要求：
```javascript
mysqlsh --mysqlx -hlocalhost -uroot -P33060
MySQL  localhost:33060+ ssl  JS > dba.checkInstanceConfiguration('mysql-ic-1:3306')
Please provide the password for 'root@mysql-ic-1:3306': **********
Save password for 'root@mysql-ic-1:3306'? [Y]es/[N]o/Ne[v]er (default No): y
Validating local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysql-ic-1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| Variable                               | Current Value | Required Value | Note                                             |
+----------------------------------------+---------------+----------------+--------------------------------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable                       |
| master_info_repository                 | FILE          | TABLE          | Update read-only variable and restart the server |
+----------------------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
NOTE: Please use the dba.configureInstance() command to repair these issues.

{
    "config_errors": [
        {
            "action": "server_update", 
            "current": "COMMIT_ORDER", 
            "option": "binlog_transaction_dependency_tracking", 
            "required": "WRITESET"
        }, 
        {
            "action": "server_update+restart", 
            "current": "FILE", 
            "option": "master_info_repository", 
            "required": "TABLE"
        }
    ], 
    "status": "error"
}
```

修复检测出来的配置问题：
```javascript
MySQL  localhost:33060+ ssl  JS > dba.configureInstance('mysql-ic-1:3306')
Configuring local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysql-ic-1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

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
Configuring instance...
The instance 'mysql-ic-1:3306' was configured to be used in an InnoDB cluster.
Restarting MySQL...
ERROR: Remote restart of MySQL server failed: MySQL Error 3707 (HY000): Restart server failed (mysqld is not managed by supervisor process).
Please restart MySQL manually (check https://dev.mysql.com/doc/refman/en/restart.html for more details).
Dba.configureInstance: Restart server failed (mysqld is not managed by supervisor process). (MYSQLSH 3707)
```

手动修改配置文件并重启数据库：
```bash
[root@mysql-ic-1 ~]# vim /etc/my.cnf
[root@mysql-ic-1 ~]# systemctl restart mysqld
[root@mysql-ic-1 ~]# 
[root@mysql-ic-1 ~]# grep 'master_info' /etc/my.cnf
master_info_repository = TABLE
```

重新初始化第一个实例：
```javascript
MySQL  localhost:33060+ ssl  JS > dba.checkInstanceConfiguration('mysql-ic-1:3306')
Validating local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysql-ic-1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...
Instance configuration is compatible with InnoDB cluster

The instance 'mysql-ic-1:3306' is valid to be used in an InnoDB cluster.

{
    "status": "ok"
}

MySQL  localhost:33060+ ssl  JS > dba.configureInstance('mysql-ic-1:3306')
Configuring local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysql-ic-1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

The instance 'mysql-ic-1:3306' is valid to be used in an InnoDB cluster.
The instance 'mysql-ic-1:3306' is already ready to be used in an InnoDB cluster.

Successfully enabled parallel appliers.
```

## 创建InnoDB Cluster
初始化完第一个实例后，就可以创建集群了。
```javascript
MySQL  localhost:33060+ ssl  JS > var mycluster = dba.createCluster('clusterdemo')
A new InnoDB Cluster will be created on instance 'mysql-ic-1:3306'.

Validating instance configuration at localhost:3306...

This instance reports its own address as mysql-ic-1:3306

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'mysql-ic-1:3306'. Use the localAddress option to override.

Creating InnoDB Cluster 'clusterdemo' on 'mysql-ic-1:3306'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.
```

## 添加副本实例
初始化第二个和第三个实例：
```javascript
MySQL  localhost:33060+ ssl  JS > dba.configureInstance('mysql-ic-2:3306')
Please provide the password for 'root@mysql-ic-2:3306': **********
Save password for 'root@mysql-ic-2:3306'? [Y]es/[N]o/Ne[v]er (default No): y
Configuring MySQL instance at mysql-ic-2:3306 for use in an InnoDB cluster...

This instance reports its own address as mysql-ic-2:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+----------------------------+
| Variable                               | Current Value | Required Value | Note                       |
+----------------------------------------+---------------+----------------+----------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable |
+----------------------------------------+---------------+----------------+----------------------------+

Do you want to perform the required configuration changes? [y/n]: y
Configuring instance...
The instance 'mysql-ic-2:3306' was configured to be used in an InnoDB cluster.
 MySQL  localhost:33060+ ssl  JS > 
 MySQL  localhost:33060+ ssl  JS > dba.configureInstance('mysql-ic-3:3306')
Please provide the password for 'root@mysql-ic-3:3306': **********
Save password for 'root@mysql-ic-3:3306'? [Y]es/[N]o/Ne[v]er (default No): y
Configuring MySQL instance at mysql-ic-3:3306 for use in an InnoDB cluster...

This instance reports its own address as mysql-ic-3:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

applierWorkerThreads will be set to the default value of 4.

NOTE: Some configuration options need to be fixed:
+----------------------------------------+---------------+----------------+----------------------------+
| Variable                               | Current Value | Required Value | Note                       |
+----------------------------------------+---------------+----------------+----------------------------+
| binlog_transaction_dependency_tracking | COMMIT_ORDER  | WRITESET       | Update the server variable |
+----------------------------------------+---------------+----------------+----------------------------+

Do you want to perform the required configuration changes? [y/n]: y
Configuring instance...
The instance 'mysql-ic-3:3306' was configured to be used in an InnoDB cluster.
```

添加副本实例到创建好的集群。如果提示副本实例的GTID与集群不一致，选择通过**Clone**方式覆盖副本实例上的数据即可。
```javascipt
MySQL  localhost:33060+ ssl  JS > var mycluster = dba.getCluster()

MySQL  localhost:33060+ ssl  JS > mycluster.addInstance('mysql-ic-2:3306')

WARNING: A GTID set check of the MySQL instance at 'mysql-ic-2:3306' determined that it contains transactions that do not originate from the cluster, which must be discarded before it can join the cluster.

mysql-ic-2:3306 has the following errant GTIDs that do not exist in the cluster:
79a3b9d1-bafc-11ed-be1d-00163e01c0d0:1-4

WARNING: Discarding these extra GTID events can either be done manually or by completely overwriting the state of mysql-ic-2:3306 with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

Having extra GTID events is not expected, and it is recommended to investigate this further and ensure that the data can be removed prior to choosing the clone recovery method.

Please select a recovery method [C]lone/[A]bort (default Abort): C
Validating instance configuration at mysql-ic-2:3306...

This instance reports its own address as mysql-ic-2:3306

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'mysql-ic-2:3306'. Use the localAddress option to override.

A new instance will be added to the InnoDB Cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Clone based state recovery is now in progress.

NOTE: A server restart is expected to happen as part of the clone process. If the
server does not support the RESTART command or does not come back after a
while, you may need to manually start it back.

* Waiting for clone to finish...
NOTE: mysql-ic-2:3306 is being cloned from mysql-ic-1:3306
** Stage DROP DATA: Completed
** Clone Transfer  
    FILE COPY  ############################################################  100%  Completed
    PAGE COPY  ############################################################  100%  Completed
    REDO COPY  ############################################################  100%  Completed

NOTE: mysql-ic-2:3306 is shutting down...

* Waiting for server restart... timeout 
WARNING: Clone process appears to have finished and tried to restart the MySQL server, but it has not yet started back up.

Please make sure the MySQL server at 'mysql-ic-2:3306' is restarted and call <Cluster>.rescan() to complete the process. To increase the timeout, change shell.options["dba.restartWaitTimeout"].
ERROR: MYSQLSH 51156: Timeout waiting for server to restart
Cluster.addInstance: Timeout waiting for server to restart (MYSQLSH 51156)
```

这里需要手动重启副本实例后，重新扫描副本实例将其加入集群：
```javascript
MySQL  localhost:33060+ ssl  JS > mycluster.rescan()
Rescanning the cluster...

Result of the rescanning operation for the 'clusterdemo' cluster:
{
    "name": "clusterdemo", 
    "newTopologyMode": null, 
    "newlyDiscoveredInstances": [
        {
            "host": "mysql-ic-2:3306", 
            "member_id": "79a3b9d1-bafc-11ed-be1d-00163e01c0d0", 
            "name": null, 
            "version": "8.0.32"
        }
    ], 
    "unavailableInstances": [], 
    "updatedInstances": []
}

A new instance 'mysql-ic-2:3306' was discovered in the cluster.
Would you like to add it to the cluster metadata? [Y/n]: y
Adding instance to the cluster metadata...
The instance 'mysql-ic-2:3306' was successfully added to the cluster metadata.

Fixing incorrect recovery account 'mysql_innodb_cluster_102' in instance 'mysql-ic-1:3306'
 
MySQL  localhost:33060+ ssl  JS > mycluster.status()
{
    "clusterName": "clusterdemo", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql-ic-1:3306", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures.", 
        "topology": {
            "mysql-ic-1:3306": {
                "address": "mysql-ic-1:3306", 
                "memberRole": "PRIMARY", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "replicationLag": "applier_queue_applied", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.32"
            }, 
            "mysql-ic-2:3306": {
                "address": "mysql-ic-2:3306", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLag": "applier_queue_applied", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.32"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql-ic-1:3306"
}
```

重复上面的流程，将第三个实例也添加到集群中后，检查集群状态：
```javascript
MySQL  localhost:33060+ ssl  JS > mycluster.status()
{
    "clusterName": "clusterdemo", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql-ic-1:3306", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "mysql-ic-1:3306": {
                "address": "mysql-ic-1:3306", 
                "memberRole": "PRIMARY", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "replicationLag": "applier_queue_applied", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.32"
            }, 
            "mysql-ic-2:3306": {
                "address": "mysql-ic-2:3306", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLag": "applier_queue_applied", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.32"
            }, 
            "mysql-ic-3:3306": {
                "address": "mysql-ic-3:3306", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLag": "applier_queue_applied", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.32"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql-ic-1:3306"
}
```
注意到集群状态已变为`"status": "OK"`和`"statusText": "Cluster is ONLINE and can tolerate up to ONE failure."`。


## 创建相关用户

1. **创建集群管理用户**

```javascript
MySQL  localhost:33060+ ssl  JS > mycluster.setupAdminAccount('icadmin')

Missing the password for new account icadmin@%. Please provide one.
Password for new account: *********
Confirm password: *********

Creating user icadmin@%.
Account icadmin@% was successfully created.
```

2. **创建MySQL Router用户**

```javascript
MySQL  localhost:33060+ ssl  JS > mycluster.setupRouterAccount('icrouter')

Missing the password for new account icrouter@%. Please provide one.
Password for new account: *********
Confirm password: *********

Creating user icrouter@%.
Account icrouter@% was successfully created.
```

# MySQL Router部署
## 安装MySQL Router
下载并解压MySQL Router安装包：
```bash
wget https://dev.mysql.com/get/Downloads/MySQL-Router/mysql-router-8.0.32-linux-glibc2.12-x86_64.tar.xz

tar xvf mysql-router-8.0.32-linux-glibc2.12-x86_64.tar.xz -C /usr/local/ 
```

添加到环境变量：
```bash
ln -s /usr/local/mysql-router-8.0.32-linux-glibc2.12-x86_64 /usr/local/mysql-router 

echo "export PATH=$PATH:/usr/local/mysql-router/bin" >> /root/.bash_profile
source /root/.bash_profile
```

## 引导MySQL Router
以MySQL用户来引导MySQL Router启动（不建议使用root）：
```bash
[root@mysql-ic-2 ~]# mysqlrouter --bootstrap root@localhost:3306 --directory=/home/mysql/myrouter --conf-use-sockets --account icrouter --user=mysql --force
Please enter MySQL password for root: 
# Bootstrapping MySQL Router instance at '/home/mysql/myrouter'...

Please enter MySQL password for icrouter: 
Fetching Cluster Members
trying to connect to mysql-server at mysql-ic-1:3306
- Creating account(s) (only those that are needed, if any)
- Verifying account (using it to run SQL queries that would be run by Router)
- Storing account in keyring
- Adjusting permissions of generated files
- Creating configuration /home/mysql/myrouter/mysqlrouter.conf

# MySQL Router configured for the InnoDB Cluster 'clusterdemo'

After this MySQL Router has been started with the generated configuration

    $ mysqlrouter -c /home/mysql/myrouter/mysqlrouter.conf

InnoDB Cluster 'clusterdemo' can be reached by connecting to:

## MySQL Classic protocol

- Read/Write Connections: localhost:6446, /home/mysql/myrouter/mysql.sock
- Read/Only Connections:  localhost:6447, /home/mysql/myrouter/mysqlro.sock

## MySQL X protocol

- Read/Write Connections: localhost:6448, /home/mysql/myrouter/mysqlx.sock
- Read/Only Connections:  localhost:6449, /home/mysql/myrouter/mysqlxro.sock
```

## 启动MySQL Router
使用生成的脚本启动MySQL Router：
```bash
sh /home/mysql/myrouter/start.sh 
```

测试MySQL Router连接：
```bash
# 经典MySQL协议连接
mysqlsh --mysql -hlocalhost -uroot -P6446

# X协议连接
mysqlsh --mysqlx -hlocalhost -uroot -P6448
```



