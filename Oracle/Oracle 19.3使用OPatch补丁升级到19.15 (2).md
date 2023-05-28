---
tags: [OCP, oracle]
title: Oracle 19.3使用OPatch补丁升级到19.15
created: '2023-05-21T05:44:35.842Z'
modified: '2023-05-27T13:23:34.574Z'
---

Oracle 19.3使用OPatch补丁升级到19.15

>:shark:需要从官方MOS网站上下载：
>- 补丁包：`p33806152_190000_Linux-x86-64.zip`
>- Opatch工具：`p6880880_210000_Linux-x86-64.zip`


# 补丁须知
关于Patch 33806152的一些重要须知如下：
- 如果是DG架构，主库和备库都需要打补丁（`Doc 278641.1`）；
- 如果是RAC环境，可以进行滚动更新（没有downtime），具体参考`Doc 244241.1`；
- 如果不是RAC环境，打补丁前需要关闭所有活动实例和数据库监听；
- 多租户环境中，执行Datapatch之前需要打开所有的PDB。


# 准备工作
备份Opatch目录或`$ORACLE_HOME`目录：
```bash
# 备份原有OPatch目录
[oracle@localhost ~]$ mv $ORACLE_HOME/OPatch $ORACLE_HOME/OPatch.bak

# 磁盘空间足够的话，也可以直接备份$ORACLE_HOME目录
[oracle@oracle19c ~]$ cp -r $ORACLE_HOME/ /u01/app/oracle/product/19.0.0/dbhome_1.bak
```

解压补丁工具和补丁包：
```bash
# 解压最新的OPatch工具
[oracle@localhost ~]$ unzip -q p6880880_210000_Linux-x86-64.zip -d $ORACLE_HOME/

# 解压补丁包
[oracle@localhost ~]$ mkdir $ORACLE_HOME/patches
[oracle@localhost ~]$ unzip -q p33806152_190000_Linux-x86-64.zip -d $ORACLE_HOME/patches
[oracle@localhost ~]$ ls $ORACLE_HOME/patches
33806152  PatchSearch.xml
```

# 补丁冲突检查
进入补丁解压目录下，使用OPatch工具检查补丁冲突：
```bash
[oracle@localhost ~]$ cd $ORACLE_HOME/patches/33806152
[oracle@localhost 33806152]$ $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph ./

Oracle Interim Patch Installer version 12.2.0.1.30

PREREQ session

Oracle Home       : /u01/app/oracle/product/version/db_1
Central Inventory : /u01/app/oraInventory
   from           : /u01/app/oracle/product/version/db_1/oraInst.loc
OPatch version    : 12.2.0.1.30
OUI version       : 12.2.0.7.0
Log file location : /u01/app/oracle/product/version/db_1/cfgtoollogs/opatch/opatch2023-05-21_04-18-08AM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
```

# OPatch打补丁
关闭数据库：
```sql
sys@ORCLCDB> show pdbs;
sys@ORCLCDB> alter pluggable database all close immediate;
sys@ORCLCDB> shutdown immediate;
```

停止数据库监听：
```bash
[oracle@localhost 33806152]$ lsnrctl stop
[oracle@localhost 33806152]$ lsnrctl status
```

使用OPatch工具打补丁：
```bash
# 工作为目录为补丁解压缩目录
[oracle@localhost 33806152]$ $ORACLE_HOME/OPatch/opatch apply
```

其中有几个地方需要手动输入y确认：
```bash
Oracle Interim Patch Installer version 12.2.0.1.30

Oracle Home       : /u01/app/oracle/product/19.0.0/dbhome_1
Central Inventory : /u01/app/oraInventory
   from           : /u01/app/oracle/product/19.0.0/dbhome_1/oraInst.loc
OPatch version    : 12.2.0.1.30
OUI version       : 12.2.0.7.0
Log file location : /u01/app/oracle/product/19.0.0/dbhome_1/cfgtoollogs/opatch/opatch2023-05-27_10-38-50AM_1.log

Verifying environment and performing prerequisite checks...
OPatch continues with these patches:   33806152

Do you want to proceed? [y|n]
y
User Responded with: Y
All checks passed.

Please shutdown Oracle instances running out of this ORACLE_HOME on the local system.
(Oracle Home = '/u01/app/oracle/product/19.0.0/dbhome_1')

Is the local system ready for patching? [y|n]
y
User Responded with: Y
Backing up files...
Applying interim patch '33806152' to OH '/u01/app/oracle/product/19.0.0/dbhome_1'
ApplySession: Optional component(s) [ oracle.network.gsm, 19.0.0.0.0 ] , [ oracle.rdbms.ic, 19.0.0.0.0 ] , [ oracle.rdbms.tg4db2, 19.0.0.0.0 ] , [ oracle.tfa, 19.0.0.0.0 ] , [ oracle.network.cman, 19.0.0.0.0 ] , [ oracle.options.olap.api, 19.0.0.0.0 ] , [ oracle.ons.cclient, 19.0.0.0.0 ] , [ oracle.rdbms.tg4ifmx, 19.0.0.0.0 ] , [ oracle.rdbms.tg4sybs, 19.0.0.0.0 ] , [ oracle.options.olap, 19.0.0.0.0 ] , [ oracle.xdk.companion, 19.0.0.0.0 ] , [ oracle.net.cman, 19.0.0.0.0 ] , [ oracle.ons.eons.bwcompat, 19.0.0.0.0 ] , [ oracle.oid.client, 19.0.0.0.0 ] , [ oracle.rdbms.tg4tera, 19.0.0.0.0 ] , [ oracle.rdbms.tg4msql, 19.0.0.0.0 ] , [ oracle.jdk, 1.8.0.191.0 ]  not present in the Oracle Home or a higher version is found.

Patching component oracle.bali.ewt, 11.1.1.6.0...

Patching component oracle.help.ohj, 11.1.1.7.0...

...

Patching component oracle.precomp.lang, 19.0.0.0.0...

Patching component oracle.jdk, 1.8.0.201.0...
Patch 33806152 successfully applied.
Sub-set patch [29517242] has become inactive due to the application of a super-set patch [33806152].
Please refer to Doc ID 2161861.1 for any possible further required actions.
Log file location: /u01/app/oracle/product/19.0.0/dbhome_1/cfgtoollogs/opatch/opatch2023-05-27_10-38-50AM_1.log

OPatch succeeded.
```


# Post patch操作
上述打补丁的过程完成后，启动监听：
```bash
[oracle@localhost 33806152]$ lsnrctl start
[oracle@localhost 33806152]$ lsnrctl status
```

启动数据库，打开所有的PDB：
```sql
idle> startup;
idle> alter pluggable database all open;
idle> show pdbs;
```

运行datapatch来执行post patch步骤：
```bash
[oracle@localhost 33806152]$ $ORACLE_HOME/OPatch/datapatch -verbose
```

>:shark:Datapatch运行的时间可能会很长（几十分钟甚至几个小时），相关问题参见`Doc 1585822.1`。


# 打补丁后检查
```sql
--检查已安装的补丁信息
sys@MARIO> select patch_id,status,action_time from dba_registry_sqlpatch;
sys@MARIO> select patch_id,status,action_time from cdb_registry_sqlpatch;

--检查数据库当前的版本
sys@MARIO> select * from v$version;
```


