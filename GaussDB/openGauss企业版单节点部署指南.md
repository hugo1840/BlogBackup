---
tags: [gaussdb]
title: openGauss企业版单节点部署指南
created: '2023-06-16T10:42:23.555Z'
modified: '2023-06-16T11:26:18.149Z'
---

openGauss企业版单节点部署指南

>:car:部署环境：CentOS 7.9
>:train:服务器配置：16C32G
>:cake:安装包：`openGauss-5.0.0-CentOS-64bit-all.tar.gz`

# 操作系统配置
检查操作系统配置，并安装所需软件包：
```bash
# 检查是否已安装python3
python3 -V

# 安装相关软件包
yum install -y bzip2 libaio-devel flex bison ncurses-devel glibc-devel patch redhat-lsb-core readline-devel

# 检查SELINUX和防火墙是否已关闭
cat /etc/selinux/config | grep -v '^#' | grep SELINUX
systemctl status firewalld

# 修改为统一时区
date
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 检查网卡MTU
ifconfig | grep mtu
```

解压缩安装包到指定路径下：
```bash
# 创建安装包存放目录
mkdir -p /opt/software/openGauss
chmod 755 -R /opt/software

# 解压安装包
cd /opt/software/openGauss
tar -zxvf openGauss-5.0.0-CentOS-64bit-all.tar.gz
tar -zxvf openGauss-5.0.0-CentOS-64bit-om.tar.gz
```

# 单节点配置文件
拷贝XML部署配置文件：
```bash
[root@gsdb_node openGauss]# cp script/gspylib/etc/conf/cluster_config_template.xml cluster_config.xml
```

编辑XML部署配置文件，添加集群信息和节点信息：
```xml
[root@gsdb_node openGauss]# cat cluster_config.xml 
<?xml version="1.0" encoding="utf-8"?>
<ROOT>
  <CLUSTER>
    <PARAM name="clusterName" value="Gstest" />
    <PARAM name="nodeNames" value="gsdb_node"/>
    <PARAM name="gaussdbAppPath" value="/opt/openGauss/app" />
    <PARAM name="gaussdbLogPath" value="/opt/openGauss/log/omm" />
    <PARAM name="tmpMppdbPath" value="/opt/openGauss/tmp"/>
    <PARAM name="gaussdbToolPath" value="/opt/openGauss/tool" />
    <PARAM name="corePath" value="/opt/openGauss/corefile"/>
    <PARAM name="backIp1s" value="192.168.166.98"/>
  </CLUSTER>

  <DEVICELIST>
    <DEVICE sn="gsdb_node">
      <PARAM name="name" value="gsdb_node"/>
      <PARAM name="azName" value="AZ1"/>
      <PARAM name="azPriority" value="1"/>
      <PARAM name="backIp1" value="192.168.166.98"/>
      <PARAM name="sshIp1" value="192.168.166.98"/>
      <!-- dn -->
      <PARAM name="dataNum" value="1"/>
      <PARAM name="dataPortBase" value="15400"/>
      <PARAM name="dataNode1" value="/opt/openGauss/data/dn"/>
      <PARAM name="dataNodeXlogPath1" value="/opt/openGauss/log/gauss_xlog "/>
      <PARAM name="dataNode1_syncNum" value="0"/>
    </DEVICE>

  </DEVICELIST>
</ROOT>
```

# 安装前检查
在hosts文件中添加节点映射信息：
```bash
[root@gsdb_node openGauss]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

#Gauss OM IP Hosts Mapping
192.168.166.98    gsdb_node
#Gauss OM IP Hosts Mapping
```

创建数据库管理用户和数据目录：
```bash
groupadd pgauss
useradd gauss -g pgauss -d /home/gauss

mkdir /opt/openGauss/
chown -R gauss:pgauss /opt/openGauss/
chown -R gauss:pgauss /opt/software/openGauss
```

以root用户执行预安装操作：
```bash
[root@gsdb_node openGauss]# cd /opt/software/openGauss/script
[root@gsdb_node script]# ./gs_preinstall -U gauss -G pgauss -L -X /opt/software/openGauss/cluster_config.xml
Parsing the configuration file.
Successfully parsed the configuration file.
Installing the tools on the local node.
Successfully installed the tools on the local node.
Setting host ip env
Successfully set host ip env.
Preparing SSH service.
Successfully prepared SSH service.
Checking OS software.
Successfully check os software.
Checking OS version.
Successfully checked OS version.
Creating cluster's path.
Successfully created cluster's path.
Set and check OS parameter.
Setting OS parameters.
Successfully set OS parameters.
Warning: Installation environment contains some warning messages.
Please get more details by "/opt/software/openGauss/script/gs_checkos -i A -h gsdb_node --detail".
Set and check OS parameter completed.
Preparing CRON service.
Successfully prepared CRON service.
Setting user environmental variables.
Successfully set user environmental variables.
Setting the dynamic link library.
Successfully set the dynamic link library.
Setting Core file
Successfully set core path.
Setting pssh path
Successfully set pssh path.
Setting Cgroup.
Successfully set Cgroup.
Set ARM Optimization.
No need to set ARM Optimization.
Fixing server package owner.
Setting finish flag.
Successfully set finish flag.
Preinstallation succeeded.
```

检查warning项目：
```bash
[root@gsdb_node openGauss]# /opt/software/openGauss/script/gs_checkos -i A -h gsdb_node --detail
Checking items:
    A1. [ OS version status ]                                   : Normal     
        [gsdb_node]
        centos_7.9.2009_64bit

    A2. [ Kernel version status ]                               : Normal     
        The names about all kernel versions are same. The value is "3.10.0-1160.80.1.el7.x86_64".
    A3. [ Unicode status ]                                      : Normal     
        The values of all unicode are same. The value is "LANG=en_US.UTF-8".
    A4. [ Time zone status ]                                    : Normal     
        The informations about all timezones are same. The value is "+0800".
    A5. [ Swap memory status ]                                  : Normal     
        The value about swap memory is correct.            
    A6. [ System control parameters status ]                    : Warning    
        [gsdb_node]
        Warning reason: variable 'net.ipv4.tcp_retries1' RealValue '3' ExpectedValue '5'.
        Warning reason: variable 'net.ipv4.tcp_syn_retries' RealValue '6' ExpectedValue '5'.
        Check_SysCtl_Parameter warning.

    A7. [ File system configuration status ]                    : Warning    
        [gsdb_node]
        Warning reason: variable 'open files' RealValue '1024' ExpectedValue '1000000'
        Warning reason: variable 'max user processes' RealValue '127842' ExpectedValue 'unlimited'

    A8. [ Disk configuration status ]                           : Normal     
        The value about XFS mount parameters is correct.   
    A9. [ Pre-read block size status ]                          : Normal     
        The value about Logical block size is correct.     
    A11.[ Network card configuration status ]                   : Warning    
        [gsdb_node]
BondMode Null
        Warning reason: network 'eth0' 'mtu' RealValue '1500' ExpectedValue '8192'
        Warning reason: network 'eth0' 'tx' RealValue '512' ExpectValue '4096'.

    A12.[ Time consistency status ]                             : Normal     
        The ntpd service is started, local time is "2023-06-16 08:48:47".
    A13.[ Firewall service status ]                             : Normal     
        The firewall service is stopped.                   
    A14.[ THP service status ]                                  : Normal     
        The THP service is stopped.                        
Total numbers:13. Abnormal numbers:0. Warning numbers:3.
```

调整系统参数：
```bash
[root@gsdb_node openGauss]# cat /etc/sysctl.conf | grep 'net.ipv4.tcp_retries1'
net.ipv4.tcp_retries1 = 5
[root@gsdb_node openGauss]# cat /etc/sysctl.conf | grep 'net.ipv4.tcp_syn_retries'
net.ipv4.tcp_syn_retries = 5
[root@gsdb_node openGauss]# sysctl -p
```

修改网卡MTU配置：
```bash
[root@gsdb_node openGauss]# cat /etc/sysconfig/network-scripts/ifcfg-eth0 | grep -i mtu
MTU=8192
[root@gsdb_node openGauss]# systemctl restart network

[root@gsdb_node openGauss]# ifconfig eth0 | grep mtu
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 8192
```

资源限制要切换一次用户才会生效：
```bash
[root@gsdb_node ~]# cat /etc/security/limits.conf | grep -v '^#' | grep nofile 
root       soft    nofile  1000000
gauss       soft    nofile  1000000
root       hard    nofile  1000000
gauss       hard    nofile  1000000

[root@gsdb_node ~]# cat /etc/security/limits.conf | grep -v '^#' | grep nproc
root       soft    nproc  unlimited
gauss       soft    nproc  unlimited
root       hard    nproc  unlimited
gauss       hard    nproc  unlimited
```

关闭透明大页：
```bash
[root@gsdb_node ~]# cat /sys/kernel/mm/transparent_hugepage/enabled 
always madvise [never]

[root@gsdb_node ~]# cat /etc/default/grub | grep CMDLINE
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=VolGroup/lv_root rd.lvm.lv=VolGroup/lv_swap rhgb quiet transparent_hugepage=never"

[root@gsdb_node ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
```

# 执行openGauss安装
切换到gauss用户来执行数据库安装：
```bash
[root@gsdb_node ~]# su - gauss
[gauss@gsdb_node ~]$ cd /opt/software/openGauss/script
[gauss@gsdb_node script]$ gs_install -X /opt/software/openGauss/cluster_config.xml --gsinit-parameter="--locale=en_US.utf8"

Parsing the configuration file.
Check preinstall on every node.
Successfully checked preinstall on every node.
Creating the backup directory.
Successfully created the backup directory.
begin deploy..
Installing the cluster.
begin prepare Install Cluster..
Checking the installation environment on all nodes.
begin install Cluster..
Installing applications on all nodes.
Successfully installed APP.
begin init Instance..
encrypt cipher and rand files for database.
Please enter password for database:         #--输入数据库密码
Please repeat for database:
begin to create CA cert files
The sslcert will be generated in /opt/openGauss/app/share/sslcert/om
NO cm_server instance, no need to create CA for CM.
Non-dss_ssl_enable, no need to create CA for DSS
Cluster installation is completed.
Configuring.
Deleting instances from all nodes.
Successfully deleted instances from all nodes.
Checking node configuration on all nodes.
Initializing instances on all nodes.
Updating instance configuration on all nodes.
Check consistence of memCheck and coresCheck on database nodes.
Configuring pg_hba on all nodes.
Configuration is completed.
The cluster status is Normal.
Successfully started cluster.
Successfully installed application.
end deploy..
```

# 安装后验证
安装后验证：
```bash
[gauss@gsdb_node script]$ ps -ef | grep gaussdb
gauss     42377      1  5 10:25 ?        00:00:15 /opt/openGauss/app/bin/gaussdb -D /opt/openGauss/data/dn
gauss     43562  39608  0 10:29 pts/0    00:00:00 grep --color=auto gaussdb

[gauss@gsdb_node script]$ du -sh /opt/openGauss/
1.5G    /opt/openGauss/
```

检查集群状态：
```bash
[gauss@gsdb_node ~]$ gs_om -t status --detail
[   Cluster State   ]

cluster_state   : Normal
redistributing  : No
current_az      : AZ_ALL

[  Datanode State   ]

    node     node_ip         port      instance                       state
-------------------------------------------------------------------------------------------
1  gsdb_node 192.168.166.98    15400      6001 /opt/openGauss/data/dn   P Primary Normal
```

连接到postgres数据库：
```sql
[gauss@gsdb_node script]$ gsql -d postgres -p 15400
gsql ((openGauss 5.0.0 build a07d57c3) compiled at 2023-03-29 03:07:56 commit 0 last mr  )
Non-SSL connection (SSL connection is recommended when requiring high-security)
Type "help" for help.

openGauss=# help
You are using gsql, the command-line interface to gaussdb.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with gsql commands
       \g or terminate with semicolon to execute query
       \q to quit
openGauss=# \q
```

# 初始化数据库
连接到数据库，并执行建库和建表操作：
```sql
[gauss@gsdb_node script]$ gsql -d postgres -p 15400

openGauss=# CREATE DATABASE mydb WITH ENCODING 'GBK' template = template0;
ERROR:  encoding "GBK" does not match locale "en_US.utf8"
DETAIL:  The chosen LC_CTYPE setting requires encoding "UTF8".

openGauss=# CREATE DATABASE mydb WITH ENCODING 'UTF8' template = template0;
CREATE DATABASE

openGauss=# select datname from pg_database;
  datname  
-----------
 template1
 mydb
 template0
 postgres
(4 rows)

openGauss=# \l
                             List of databases
   Name    | Owner | Encoding |  Collate   |   Ctype    | Access privileges 
-----------+-------+----------+------------+------------+-------------------
 mydb      | gauss | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | gauss | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | gauss | UTF8     | en_US.utf8 | en_US.utf8 | =c/gauss         +
           |       |          |            |            | gauss=CTc/gauss
 template1 | gauss | UTF8     | en_US.utf8 | en_US.utf8 | =c/gauss         +
           |       |          |            |            | gauss=CTc/gauss
(4 rows)

openGauss=# \c mydb
Non-SSL connection (SSL connection is recommended when requiring high-security)
You are now connected to database "mydb" as user "gauss".
mydb=# CREATE TABLE customer_t1
(
    c_customer_sk             integer,
    c_customer_id             char(5),
    c_first_name              char(6),
    c_last_name               char(8),
    Amount                    integer
);
CREATE TABLE

mydb=# \dp
                              Access privileges
 Schema |    Name     | Type  | Access privileges | Column access privileges 
--------+-------------+-------+-------------------+--------------------------
 public | customer_t1 | table |                   | 
(1 row)
```

# 查看数据库用户
```sql
mydb-# \c postgres
Non-SSL connection (SSL connection is recommended when requiring high-security)
You are now connected to database "postgres" as user "gauss".

openGauss=# select username from pg_user;
ERROR:  column "username" does not exist
LINE 1: select username from pg_user;
               ^
CONTEXT:  referenced column: username

openGauss=# \d pg_user
                View "pg_catalog.pg_user"
      Column      |           Type           | Modifiers 
------------------+--------------------------+-----------
 usename          | name                     | 
 usesysid         | oid                      | 
 usecreatedb      | boolean                  | 
 usesuper         | boolean                  | 
 usecatupd        | boolean                  | 
 userepl          | boolean                  | 
 passwd           | text                     | 
 valbegin         | timestamp with time zone | 
 valuntil         | timestamp with time zone | 
 respool          | name                     | 
 parent           | oid                      | 
 spacelimit       | text                     | 
 useconfig        | text[]                   | 
 nodegroup        | name                     | 
 tempspacelimit   | text                     | 
 spillspacelimit  | text                     | 
 usemonitoradmin  | boolean                  | 
 useoperatoradmin | boolean                  | 
 usepolicyadmin   | boolean                  | 

openGauss=# select usename from pg_user;
 usename 
---------
 gauss
(1 row)
```

# openGauss启停
登录数据库主节点来执行集群的启停操作。

停止集群：
```bash
[gauss@gsdb_node openGauss]$ gs_om -t stop
Stopping cluster.
=========================================
Successfully stopped cluster.
=========================================
End stop cluster.

[gauss@gsdb_node openGauss]$ gs_om -t status --detail
[   Cluster State   ]

cluster_state   : Unavailable
redistributing  : No
current_az      : AZ_ALL

[  Datanode State   ]

    node     node_ip         port      instance                       state
-------------------------------------------------------------------------------------------
1  gsdb_node 192.168.166.98    15400      6001 /opt/openGauss/data/dn   P Primary Manually stopped
```

启动集群：
```bash
[gauss@gsdb_node openGauss]$ gs_om -t start
Starting cluster.
=========================================
[SUCCESS] gsdb_node
2023-06-16 13:52:13.385 648bf88d.1 [unknown] 139787170578688 [unknown] 0 dn_6001 01000  0 [BACKEND] WARNING:  could not create any HA TCP/IP sockets
2023-06-16 13:52:13.385 648bf88d.1 [unknown] 139787170578688 [unknown] 0 dn_6001 01000  0 [BACKEND] WARNING:  could not create any HA TCP/IP sockets
=========================================
Successfully started.

#检查集群状态
[gauss@gsdb_node openGauss]$ gs_om -t status --detail
[   Cluster State   ]

cluster_state   : Normal
redistributing  : No
current_az      : AZ_ALL

[  Datanode State   ]

    node     node_ip         port      instance                       state
-------------------------------------------------------------------------------------------
1  gsdb_node 192.168.166.98    15400      6001 /opt/openGauss/data/dn   P Primary Normal

#检查节点实例状态
[gauss@gsdb_node openGauss]$ gs_om -t status -h gsdb_node
-----------------------------------------------------------------------

cluster_state             : Normal
redistributing            : No

-----------------------------------------------------------------------

node                      : 1
node_name                 : gsdb_node
instance_id               : 6001
node_ip                   : 192.168.166.98
data_path                 : /opt/openGauss/data/dn
instance_port             : 15400
type                      : Datanode
instance_state            : Normal
az_name                   : AZ1
instance_role             : Normal

-----------------------------------------------------------------------
```


**References**
[1] https://docs.opengauss.org/en/docs/5.0.0/docs/GettingStarted/GettingStarted.html






