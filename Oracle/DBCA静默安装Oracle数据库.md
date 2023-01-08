---
tags: [oracle]
title: DBCA静默安装Oracle数据库
created: '2023-01-08T05:44:50.787Z'
modified: '2023-01-08T12:52:39.694Z'
---

DBCA静默安装Oracle数据库

本篇教程采用DBCA工具静默安装Oracle数据库。前置准备工作请参考[Oracle单机部署：安装前检查和配置](https://blog.csdn.net/Sebastien23/article/details/128242810)和[Oracle单机部署：数据库安装](https://blog.csdn.net/Sebastien23/article/details/128290845)两篇文章中的部分内容。

将软件安装包解压到`ORACLE_HOME`路径下：
```bash
[root@oracledb ~]# ls /u01/app
oracle  oraInventory
[oracle@oracledb ~]$ echo $ORACLE_HOME
/u01/app/oracle/product/19.0.0/dbhome_1
```

# DBCA静默安装Oracle软件
静默安装Oracle软件需要用到`$ORACLE_HOME/install/response/db_install.rsp`文件。

拷贝`db_install.rsp`文件到Oracle家目录下：
```bash
[oracle@oracledb ~]$ cp $ORACLE_HOME/install/response/db_install.rsp .
```

## 准备db_install.rsp文件
客户化`db_install.rsp`文件中的以下配置项：

1. **基本配置项**

```bash
oracle.install.option=INSTALL_DB_SWONLY   # 只安装软件，不创建数据库
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/u01/app/oraInventory
# 配置Oracle环境变量
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
# 安装企业版
oracle.install.db.InstallEdition=EE   
```

2. **配置用户属组**
确保操作系统中已存在下列用户组：
```bash
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=oper
oracle.install.db.OSBACKUPDBA_GROUP=backupdba
oracle.install.db.OSDGDBA_GROUP=dgdba
oracle.install.db.OSKMDBA_GROUP=kmdba
oracle.install.db.OSRACDBA_GROUP=racdba
```

3. **Root脚本执行配置**
```bash
oracle.install.db.rootconfig.executeRootScript=true
oracle.install.db.rootconfig.configMethod=ROOT
```
安装过程中会提示输入root用户密码以执行root脚本。

## 静默安装软件
```bash
[oracle@oracledb ~]$ $ORACLE_HOME/runInstaller -silent -responseFile /home/oracle/db_install.rsp
Launching Oracle Database Setup Wizard...

 Enter password for 'root' user:
[FATAL] [INS-13013] Target environment does not meet some mandatory requirements.
   CAUSE: Some of the mandatory prerequisites are not met. See logs for details. /tmp/InstallActions2023-01-08_03-22-01PM/installActions2023-01-08_03-22-01PM.log
   ACTION: Identify the list of failed prerequisite checks from the log: /tmp/InstallActions2023-01-08_03-22-01PM/installActions2023-01-08_03-22-01PM.log. Then either from the log file or from installation manual find the appropriate configuration to meet the prerequisites and fix it manually.
Moved the install session logs to:
 /u01/app/oraInventory/logs/InstallActions2023-01-08_03-22-01PM
```

检查安装日志：
```bash
[oracle@oracledb ~]$ cat /tmp/InstallActions2023-01-08_03-22-01PM/installActions2023-01-08_03-22-01PM.log | grep Error
INFO:  [Jan 8, 2023 3:22:28 PM] Error Message:PRVF-7532 : Package "gcc-c++" is missing on node "oracledb"
INFO:  [Jan 8, 2023 3:22:29 PM] ****************** CVU Error logs ******************
INFO:  [Jan 8, 2023 3:22:29 PM] ERROR: [Result.addErrorDescription:760]  PRVF-7566 : User "oracle" does not belong to group "oper" on node "oracledb"
INFO:  [Jan 8, 2023 3:22:29 PM] ERROR: [Result.addErrorDescription:771]  PRVF-7566 : User "oracle" does not belong to group "oper" on node "oracledb"
INFO:  [Jan 8, 2023 3:22:29 PM] ERROR: [Result.addErrorDescription:771]  PRVF-7566 : User "oracle" does not belong to group "oper" on node "oracledb"
...
```

安装`gcc-c++`软件包：
```bash
[root@oracledb ~]# yum install gcc-c++ -y
```

为oracle用户添加次要用户组oper：
```bash
[root@oracledb ~]# usermod -g oinstall -G dba,oper,asmdba,asmadmin,backupdba,dgdba,kmdba,racdba oracle
[root@oracledb ~]# id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(backupdba),54323(dgdba),54324(kmdba),54325(racdba),54326(dba),54327(asmdba),54329(oper),54330(asmadmin)
```

重新安装Oracle软件：
```bash
[oracle@oracledb ~]$ $ORACLE_HOME/runInstaller -silent -responseFile /home/oracle/db_install.rsp
Launching Oracle Database Setup Wizard...

[WARNING] [INS-32047] The location (/u01/app/oraInventory) specified for the central inventory is not empty.
   ACTION: It is recommended to provide an empty location for the inventory.

 Enter password for 'root' user:

The response file for this session can be found at:
 /u01/app/oracle/product/19.0.0/dbhome_1/install/response/db_2023-01-08_03-32-29PM.rsp

You can find the log of this install session at:
 /tmp/InstallActions2023-01-08_03-32-29PM/installActions2023-01-08_03-32-29PM.log
Successfully Setup Software with warning(s).
Moved the install session logs to:
 /u01/app/oraInventory/logs/InstallActions2023-01-08_03-32-29PM
```
安装完成，安装日志位于`/u01/app/oraInventory/logs/`路径下。


# DBCA静默安装数据库
## 准备工作
安装LVM卷管理工具包：
```bash
yum install -y lvm2
```

创建数据文件目录并挂载：
```bash
[root@oracledb ~]# pvcreate /dev/vdb
  Physical volume "/dev/vdb" successfully created.
[root@oracledb ~]# vgcreate oradata /dev/vdb
  Volume group "oradata" successfully created
[root@oracledb ~]# lvcreate -l 100%FREE -n lv_oradata oradata
  Logical volume "lv_oradata" created.

[root@oracledb ~]# mkdir /oradata
[root@oracledb ~]# mkfs.ext4 /dev/oradata/lv_oradata
[root@oracledb ~]# mount /dev/oradata/lv_oradata /oradata
[root@oracledb ~]# df -Th
```

创建归档、备份和恢复区目录：
```bash
[root@oracledb ~]# cd /oradata
[root@oracledb oracle]# mkdir arch     # 自定义归档路径
[root@oracledb oracle]# mkdir backup   # 自定义备份路径
[root@oracledb oracle]# mkdir fast_recovery_area   # 自定义快速恢复区路径

[root@oracledb oracle]# chown -R oracle:oinstall /oradata
```

静默安装数据库实例需要用到dbca Response文件和数据库模板文件。

## 准备dbca.rsp文件
DBCA Response文件在如下路径中：
```bash
[oracle@oracledb ~]$ ls $ORACLE_HOME/assistants/dbca/
dbca.rsp  doc  jlib  templates
```

`dbca.rsp`文件中可以指定要创建的数据库的相关配置参数，其中注释有`Mandatory: Yes`的为必填。这些配置参数也可以在`dbca -createDatabase`命令后直接指定。

单机部署时一般需要配置以下参数：
```bash
gdbName=orcl                      # 全局数据库名
sid=orcl                          # 数据库系统标识符
templateName=General_Purpose.dbc  # 数据库模板文件
sysPassword=pikachu               # SYS用户登录密码
systemPassword=pikachu            # SYSTEM用户登录密码
storageType=FS                    # 默认采用文件系统存储
characterSet=ZHS16GBK             # 采用中文字符集
# 此处可以覆盖模板文件中的同名参数，多个参数以逗号分
initParams=processes=1000,db_block_size=8192,db_recovery_file_dest=/oradata/fast_recovery_area                  
```

## 修改dbca模板文件
以下是Oracle软件自带的数据库XML模板文件：
```bash
[oracle@oracledb ~]$ ls $ORACLE_HOME/assistants/dbca/templates
Data_Warehouse.dbc  General_Purpose.dbc  New_Database.dbt  pdbseed.dfb  pdbseed.xml  Seed_Database.ctl  Seed_Database.dfb
```
其中，重点关注以下三类：
- `Data_Warehouse.dbc`：数仓模板，适用于OLAP业务；
- `General_Purpose.dbc`：通用模板，适用于OLTP业务；
- `New_Database.dbt`：定制类型，需要手动配置参数。

修改`New_Database.dbt`模板文件，将下面一行：
```xml
<initParam name="control_files" value="(&quot;{ORACLE_BASE}/oradata/{DB_UNIQUE_NAME}/control01.ctl&quot;, &quot;{ORACLE_BASE}/fast_recovery_area/{DB_UNIQUE_NAME}/control02.ctl&quot;)"/>
```
替换为
```xml
<initParam name="db_create_file_dest" value="/oradata"/>
```
以开启OMF数据库文件管理模式（Oracle Managed Files）。


## 静默安装数据库实例
方法一：通过`dbca.rsp`文件指定数据库初始化参数（使用的模板为`General_Purpose.dbc`）。
```bash
[oracle@oracledb ~]$ $ORACLE_HOME/bin/dbca -silent -createDatabase \
-responseFile /home/oracle/dbca.rsp -redoLogFileSize 2048 
...
Prepare for db operation
10% complete
Copying database files
40% complete
Creating and starting Oracle instance
42% complete
46% complete
50% complete
54% complete
60% complete
Completing Database Creation
66% complete
69% complete
70% complete
Executing Post Configuration Actions
100% complete
Database creation complete. For details check the logfiles at:
 /u01/app/oracle/cfgtoollogs/dbca/orcl.
Database Information:
Global Database Name:orcl
System Identifier(SID):orcl
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/orcl/orcl0.log" for further details.
```

检查数据库：
```sql
[oracle@oracledb ~]$ sqlplus / as sysdba

SQL> show parameter db_create_file_dest;   --未开启OMF

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_create_file_dest                  string

SQL> show parameter db_recovery

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      /oradata/fast_recovery_area
db_recovery_file_dest_size           big integer 24888M
```


方法二：通过命令行指定数据库初始化参数（使用的模板为`New_Database.dbt`）。
```bash
[oracle@oracledb ~]$ export ORACLE_SID=bangkok
[oracle@oracledb ~]$ $ORACLE_HOME/bin/dbca -silent -createDatabase \
-templateName /home/oracle/New_Database.dbt \
-gdbName bangkok -sid bangkok -sysPassword oraclesys -systemPassword oraclesys \
-characterSet ZHS16GBK -redoLogFileSize 2048 \
-initParams processes=1000,db_block_size=8192,db_recovery_file_dest=/oradata/fast_recovery_area
```

安装完成后检查：
```sql
[oracle@oracledb ~]$ sqlplus / as sysdba
Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> show parameter name

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cdb_cluster_name                     string
cell_offloadgroup_name               string
db_file_name_convert                 string
db_name                              string      bangkok
db_unique_name                       string      bangkok
global_names                         boolean     FALSE
instance_name                        string      bangkok
lock_name_space                      string
log_file_name_convert                string
pdb_file_name_convert                string
processor_group_name                 string

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
service_names                        string      bangkok
SQL>
SQL> show parameter db_create

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_create_file_dest                  string      /oradata     --已开启OMF
db_create_online_log_dest_1          string
db_create_online_log_dest_2          string
db_create_online_log_dest_3          string
db_create_online_log_dest_4          string
db_create_online_log_dest_5          string
SQL>
SQL> show parameter db_recovery

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      /oradata/fast_recovery_area
db_recovery_file_dest_size           big integer 24483M
```


# DBCA静默删除数据库
使用DBCA删除数据库时只需指定数据库名称或者对应的的`ORACLE_SID`，并且数据库必须处于**OPEN**状态。
```bash
[oracle@oracledb ~]$ dbca -silent -deleteDatabase -sourceDB orcl
Enter SYS user password:

[WARNING] [DBT-19202] The Database Configuration Assistant will delete the Oracle instances and datafiles for your database. All information in the database will be destroyed.
Prepare for db operation
32% complete
Connecting to database
35% complete
39% complete
42% complete
45% complete
48% complete
52% complete
65% complete
Updating network configuration files
68% complete
Deleting instance and datafiles
84% complete
100% complete
Database deletion completed.
Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/orcl/orcl1.log" for further details.
```



**References**
【1】https://blog.csdn.net/Sebastien23/article/details/128290845
【2】https://blog.csdn.net/zxs9999/article/details/122538068
【3】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/installing-and-configuring-oracle-database-using-response-files.html#GUID-D53355E9-E901-4224-9A2A-B882070EDDF7
【4】https://docs.oracle.com/en/database/oracle/oracle-database/21/multi/dbca-overview.html#GUID-A2129D0E-9976-4A66-B038-8A653C1D4420
【5】https://docs.oracle.com/en/database/oracle/oracle-database/21/multi/dbca-command.html#GUID-0A94814D-032B-4F6A-8B54-A35223A1E3EF
【6】https://www.cnblogs.com/wonchaofan/p/16747546.html
【7】https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/using-oracle-managed-files.html#GUID-4A3C4616-0D81-4BBA-8EAD-FCAA8AD5C15A
【8】https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DB_CREATE_FILE_DEST.html#GUID-8E6EE3CE-0F39-4A6C-BF3C-F7C07902678D



