---
tags: [oracle]
title: Oracle单机部署：GI安装
created: '2022-12-12T07:14:04.631Z'
modified: '2022-12-12T09:41:23.484Z'
---

Oracle单机部署：GI安装

:dolphin: 使用**grid**用户来安装GI。

# 存储配置
Oracle存储支持Oracle ASM、Oracle ACFS、本地文件系统、网络文件系统（NFS/NAS）、Oracle Memory Speed这五种方案。这里我们采用Oracle ASM。

使用ASM需要安装Oracle Grid Infrastructure（以下简称GI或Grid）。GI提供以下两种重要管理功能：
- Oracle自动存储管理（**ASM**）：Oracle数据库文件的卷管理器和文件系统，支持单实例和RAC架构。
- Oracle Restart功能：负责监控和（在故障时）重启数据库实例、数据库监听器、ASM实例，支持单实例数据库和ASM实例。


# ASM磁盘空间评估
为了评估ASM的存储需求，需要确定以下几个事项：

1. 确定是否将ASM磁盘组同时用于数据库文件和恢复文件，或者仅用于数据库文件。数据库文件包括数据文件、控制文件、redo日志文件、系统参数文件和密码文件。如果在安装数据库时没有启用恢复功能，则可以在安装完成后通过修改`DB_RECOVERY_FILE_DEST`参数来启用快速恢复区。

2. 选择ASM磁盘组的冗余级别（redundancy level）：

- **External redundancy**：Oracle ASM不管理磁盘组镜像，冗余由外部存储系统实现（比如RAID）。有效的磁盘空间等于所有磁盘空间之和。
- **Normal redundancy**：Oracle ASM为每份数据文件提供一份镜像数据，有效的磁盘空间等于所有磁盘空间之和的**一半**。每个磁盘组至少需要两块磁盘（或者两个故障组）。
- **High redundancy**：Oracle ASM为每份数据文件提供两份镜像数据，有效的磁盘空间等于所有磁盘空间之和的**1/3**。每个磁盘组至少需要三块磁盘（或者三个故障组）。
- **Flex redundancy**：弹性冗余，可以选择为每份数据文件提供零份、一份或者两份镜像数据。每个磁盘组至少需要三块磁盘（或者三个故障组）。
- **Extended redundancy**：与Flex redundancy类似，可用于Oracle扩展集群。

3. 评估数据文件和恢复文件所需的磁盘空间总量。下面给出了non-CDB数据库在不同冗余等级时的最小存储空间需求。

| 冗余等级 | 最小磁盘数量 | 数据文件所需最小空间 | 恢复文件所需最小空间 |  
| :-: | :-: | :-: | :-: | 
| External | 1 | 2.5GB | 7.5GB |  
| Normal | 2 | 5.2GB | 15.6GB |  
| High | 3 | 7.6GB | 22.8GB  |  

4. 可选项：确认ASM磁盘组中的故障组（failure groups）。

在Normal冗余级别下，如果一个磁盘组中的两个故障组都连接到了同一个HBA卡（Host bus adapter），那么一旦HBA卡故障，整个磁盘组就不可用了。为避免HBA单点故障，建议使用两个HBA卡，并在创建磁盘组时，为两个故障组分别指定连接到不同HBA卡的磁盘。

5. 如果没有合适的磁盘组，添加磁盘来创建新的磁盘组（disk group）。

添加到ASM磁盘组中的所有磁盘的容量和性能必须相同。这些**磁盘的属主必须是GI的安装用户**。不要将单个物理磁盘上的多个分区作为存储设备添加到ASM磁盘组中。不建议将逻辑卷作为存储设备添加到ASM磁盘组中。


# GI单机安装配置
创建Oracle Base和Oracle Inventory目录：
```bash
[root@oraclehost ~]# mkdir -p /u01/app/oracle
[root@oraclehost ~]# mkdir -p /u01/app/oraInventory
[root@oraclehost ~]# chown -R oracle:oinstall /u01/app/oracle
[root@oraclehost ~]# chown -R grid:oinstall /u01/app/oraInventory
[root@oraclehost ~]# chmod -R 775 /u01/app
```

创建Grid Home目录，将Grid安装包拷贝到Grid家目录并解压：
```bash
[root@oraclehost ~]# mkdir -p /u01/app/oracle/product/19.0.0/grid
[root@oraclehost ~]# cp /install/LINUX.X64_193000_grid_home.zip /u01/app/oracle/product/19.0.0/grid
[root@oraclehost ~]# chown -R grid:oinstall /u01/app/oracle/product/19.0.0/grid

[root@oraclehost ~]# su - grid
[grid@oraclehost ~]$ cd /u01/app/oracle/product/19.0.0/grid
[grid@oraclehost grid]$ unzip -q LINUX.X64_193000_grid_home.zip
[grid@oraclehost grid]$ rm LINUX.X64_193000_grid_home.zip
```

创建Grid Base目录，用于存储ASM和Clusterware相关的诊断文件和日志：
```bash
[root@oraclehost ~]# cd /u01/app/
[root@oraclehost app]# mkdir grid
[root@oraclehost app]# chown -R grid:oinstall grid/
```

以root身份设置`ORACLE_HOME`和`ORACLE_BASE`两个Shell变量：
```bash
[root@oraclehost ~]# set ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
[root@oraclehost ~]# set ORACLE_BASE=/tmp
[root@oraclehost ~]#
[root@oraclehost ~]# set | grep ORACLE
ORACLE_BASE=/tmp
ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
```

**注**：官方文档里是通过set命令设置`ORACLE_HOME`和`ORACLE_BASE`两个变量的。根据实际安装过程中的报错信息判断应该是使用export命令定义将其为环境变量。

以root身份配置`ORACLE_HOME`和`ORACLE_BASE`两个环境变量：
```bash
[root@oraclehost bin]# export ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
[root@oraclehost bin]# export ORACLE_BASE=/tmp
[root@oraclehost bin]#
[root@oraclehost bin]# env | grep ORACLE
ORACLE_BASE=/tmp
ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
```

通过ASMCMD的`afd_label`命令来标记要添加到ASM的磁盘：
```bash
[root@oraclehost bin]# cd /u01/app/oracle/product/19.0.0/grid/bin
[root@oraclehost bin]# ./asmcmd afd_label DATA1 /dev/vdb --init
```

检查标记结果：
```bash
[root@oraclehost bin]# ./asmcmd afd_lslbl /dev/vdb
--------------------------------------------------------------------------------
Label                     Duplicate  Path
================================================================================
DATA1                                 /dev/vdb
```

最后一定要删除`ORACLE_BASE`变量：
```bash
[root@oraclehost bin]# unset ORACLE_BASE
[root@oraclehost bin]#
[root@oraclehost bin]# env | grep ORACLE
ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
[root@oraclehost bin]#
[root@oraclehost bin]# set | grep ORACLE
ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
```

登录到grid用户，执行`gridSetup.sh`脚本来启动GI安装向导：
```bash
[root@oraclehost bin]# su - grid
[grid@oraclehost ~]$ cd /u01/app/oracle/product/19.0.0/grid
[grid@oraclehost grid]$ ./gridSetup.sh
ERROR: Unable to verify the graphical display setup. This application requires X display. Make sure that xdpyinfo exist under PATH variable.

No X11 DISPLAY variable was set, but this program performed an operation which requires it.
```

为grid用户配置X display权限：
```bash
[root@oraclehost ~]# cp /root/.Xauthority /home/grid/
[root@oraclehost ~]# chown -R grid:oinstall /home/grid
[root@oraclehost ~]# echo $DISPLAY
localhost:10.0
[root@oraclehost ~]# su - grid
[grid@oraclehost ~]$ export DISPLAY=localhost:10.0
[grid@oraclehost ~]$ xhost +
access control disabled, clients can connect from any host
```

重新运行GI安装向导来打开图形化安装界面：
```bash
[grid@oraclehost ~]$ cd /u01/app/oracle/product/19.0.0/grid
[grid@oraclehost grid]$ ./gridSetup.sh
```


# GI图形化安装流程
1. 选择单机安装（Standalone Server）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/92d5b2aec3da4dcaacb7aea257197326.png#pic_center)


2. 创建ASM磁盘组：磁盘组名称这里我们指定为`DATADG`；由于只有一块磁盘，冗余程度选择`External`。在**Change Discovery Path**里指定扫描的磁盘路径，勾选扫描出来的磁盘（我这里是`/dev/vdb`）。建议勾选最下面的**Configure Oracle ASM Filter Driver**，如果报错不支持，则取消勾选该项。

![在这里插入图片描述](https://img-blog.csdnimg.cn/56df0e79a3ab4adda0b5643085ac7686.png#pic_center)

3. 为ASM实例的管理用户和监控用户设置密码。

![在这里插入图片描述](https://img-blog.csdnimg.cn/fc1185988905459090ee1610cd974dca.png#pic_center)

4. 为ASM选择事先创建好的管理用户组（需要事先将这些用户组设置为Grid安装用户的次要用户组）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/25a69fbd63cb49f79f8359be27a7f74f.png#pic_center)

5. 设置GI安装的**Oracle Base目录**。Base目录用于存放ASM和Clusterware的诊断文件和管理日志，而GI安装的Oracle Home目录则用于存放GI软件。这里我们已经事先创建了Base目录和Home目录，并将权限赋予GI安装用户。

![在这里插入图片描述](https://img-blog.csdnimg.cn/05a1d66619d64a75a0cc96d7afe79ffa.png#pic_center)

6. 指定Inventory目录（已经事先创建）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/083c9890719f4bea9c30b7a8f5083fbc.png#pic_center)

7. 在GI安装过程中，有些命令需要以root权限执行。在这一步输入root密码或者赋予Grid安装用户sudo权限。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d129abbae08e423484a42e17e75e6f0e.png#pic_center)

8. 等待安装向导程序完成安装前的检查。

![在这里插入图片描述](https://img-blog.csdnimg.cn/f97aaa6fca9644679bc75dcfa37522d3.png#pic_center)

这里**Fixable**一列为Yes对应的是安装程序可以自动解决的问题，No对应的是需要手动解决的问题。

```bash
yum install -y gcc-c++.x86_64
yum install -y nfs-utils.x86_64
```
手动安装缺少的包后，点击**Fix & Check Again**自动解决其他问题。


9. 核对安装向导检查后给出的信息。确认无误后，点击**Install**开始GI安装。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b675725d48df4297a31d02340332b949.png#pic_center)

10. 处理安装报错：`[INS_20802] Oracle Net Configuration Assistant Failed`。这里检查对应日志没有发现有用信息，点击**Retry**重试后通过。

![在这里插入图片描述](https://img-blog.csdnimg.cn/885d1ecd28e049989a151ed94a33f707.png#pic_center)

11. 处理安装报错：`[INS_20802] Automatic Storage Management Configuration  Assistant Failed`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/934296d6beab4c8abeacc99e951d646a.png#pic_center)


这里检查对应安装日志后，发现如下错误信息：
```bash
[root@oraclehost app]# tail -n 15 /tmp/GridSetupActions2022-12-11_03-15-05PM/gridSetupActions2022-12-11_03-15-05PM.log
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line:
INFO:  [Dec 11, 2022 4:09:42 PM] [FATAL] [DBT-30002] Disk group DATADG creation failed.
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line: [FATAL] [DBT-30002] Disk group DATADG creation failed.
INFO:  [Dec 11, 2022 4:09:42 PM] ORA-15018: diskgroup cannot be created
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line: ORA-15018: diskgroup cannot be created
INFO:  [Dec 11, 2022 4:09:42 PM] ORA-15031: disk specification '/dev/vdb' matches no disks
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line: ORA-15031: disk specification '/dev/vdb' matches no disks
INFO:  [Dec 11, 2022 4:09:42 PM] ORA-15025: could not open disk "/dev/vdb"
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line: ORA-15025: could not open disk "/dev/vdb"
INFO:  [Dec 11, 2022 4:09:42 PM] ORA-27041: unable to open file
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line: ORA-27041: unable to open file
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line:
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line:
INFO:  [Dec 11, 2022 4:09:42 PM] Skipping line:
INFO:  [Dec 11, 2022 4:09:42 PM] Completed Plugin named: asmca
```

看上去是GI安装用户没有访问ASM磁盘的权限。将`/dev/vdb`的属主修改为GI安装用户：
```bash
[root@oraclehost app]# ll /dev/vdb
brw-rw---- 1 root disk 253, 16 Dec 11 14:16 /dev/vdb
[root@oraclehost app]#
[root@oraclehost app]# chown grid:oinstall /dev/vdb
[root@oraclehost app]# ll /dev/vdb
brw-rw---- 1 oracle oinstall 253, 16 Dec 11 14:16 /dev/vdb
```

12. 安装完成。

![在这里插入图片描述](https://img-blog.csdnimg.cn/fa84c21a4f21492089eef2781513d8f2.png#pic_center)



# 安装后测试
登录用户，定义环境变量：
```bash
[root@oraclehost ~]# su - grid
[grid@oraclehost ~]$ env | grep ORACLE
[grid@oraclehost ~]$ echo 'export ORACLE_SID=+ASM' >> .bash_profile
[grid@oraclehost ~]$ echo 'export ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid' >> .bash_profile
[grid@oraclehost ~]$ echo 'export PATH=$ORACLE_HOME/bin:$PATH' >> .bash_profile
[grid@oraclehost ~]$ . ./.bash_profile
[grid@oraclehost ~]$ 
[grid@oraclehost ~]$ env | grep ORACLE
ORACLE_SID=+ASM
ORACLE_HOME=/u01/app/oracle/product/19.0.0/grid
```

执行ASMCMD命令来列出磁盘组：
```bash
[grid@oraclehost ~]$ asmcmd lsdg
State    Type    Rebal  Sector  Logical_Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  EXTERN  N         512             512   4096  4194304     81920    81820                0           81820              0             N  DATADG/
```

如果是在阿里云服务器中安装的GI，利用自定义镜像新建实例后由于磁盘发生变化，lsdg命令的输出可能不会显示任何结果。我们可以检查磁盘的属主后重新挂载磁盘组：
```bash
# 修改磁盘属组
[root@oraclehost app]# chown grid:oinstall /dev/vdb

[root@oraclehost app]# su - grid
[grid@oraclehost ~]$ env | grep ORACLE

# 挂载磁盘组DATADG
[grid@oraclehost ~]$ asmcmd mount DATADG

# 检查磁盘组和ASM磁盘
[grid@oraclehost ~]$ asmcmd lsdg
[grid@oraclehost ~]$ asmcmd lsdsk
```



**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/disk_space_requirements_for_oracle_asm.html#GUID-DE737EEE-A352-4DC3-BB85-8CADE800E6C1
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/installing-oracle-grid-infrastructure-for-a-standalone-server-with-a-new-database-installation.html#GUID-0B1CEE8C-C893-46AA-8A6A-7B5FAAEC72B3
【3】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/disk_space_requirements_for_oracle_asm.html#GUID-DE737EEE-A352-4DC3-BB85-8CADE800E6C1
【4】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/installing-oracle-grid-infrastructure-for-a-standalone-server-with-a-new-database-installation.html#GUID-0B1CEE8C-C893-46AA-8A6A-7B5FAAEC72B3
【5】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/testing-the-oracle-automatic-storage-management-installation.html#GUID-125CBE93-C37C-46B6-A3AD-03D8AD210EFD
【6】https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/about-the-oracle-home-directory-for-oracle-grid-infrastrucrure.html#GUID-C07AF84D-F2E4-41F0-9205-64BAA61C0111
【7】https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/about-the-oracle-base-directory-for-the-grid-user.html#GUID-A36535B4-EB3C-44C9-A973-8BAFCAA2F2D6
【8】https://docs.oracle.com/en/database/oracle/oracle-database/19/ostmg/asmcmd-diskgroup-commands.html#GUID-7D948582-EA82-48D3-A797-392D3B82A0AD
【9】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/configuring-oracle-asm-disk-groups-manually-using-oracle-asmca.html#GUID-F1A4AB04-EB12-49F2-896D-7719F877DFDD


