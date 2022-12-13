---
tags: [oracle]
title: Oracle单机部署：数据库安装
created: '2022-12-08T06:20:40.202Z'
modified: '2022-12-13T04:51:11.045Z'
---

Oracle单机部署：数据库安装

:dolphin: 使用**oracle**用户来安装数据库。

# 安装前须知
## 数据库字符集
在创建数据库之后，更改字符集在时间和资源上的代价都是非常昂贵的。可能需要通过导出整个数据库并将其导入来转换所有字符数据。因此，在安装时选择合适的数据库字符集是很重要的。

从Oracle Database 12c Release 2(12.2)开始，默认的字符集是Unicode AL32UTF8。

## 自动内存管理
在使用Oracle Database Configuration Assistant（DBCA）创建数据库时，会启用自动内存管理。如果选择Advanced高级安装，则可以手动指定内存分配，也可以启用自动内存管理（automatic memory management）。

如果数据库实例的总物理内存大于4GB，则在安装和创建数据库时不能选择“Oracle自动内存管理”选项。相反，应该使用自动共享内存管理（automatic shared memory management）。自动共享内存管理自动将可用内存分配给各个组件，使系统最大限度地利用所有可用的SGA内存。

通过自动内存管理，Oracle数据库实例可以实现自动管理和调优内存。使用自动内存管理，我们可以设定一个目标内存，然后实例会自动在SGA和实例PGA之间分配内存。当内存需求发生变化时，实例将在SGA和实例PGA之间动态地重新分配内存。

我们可以在数据库安装期间或之后启用自动内存管理。在安装后启用自动内存管理需要关闭和重新启动数据库。

# 数据库安装配置
创建Oracle家目录，并修改文件权限：
```bash
[root@oraclehost ~]# mkdir /u01/app/oracle/product/19.0.0/dbhome_1
[root@oraclehost ~]# cp /install/LINUX.X64_193000_db_home.zip /u01/app/oracle/product/19.0.0/dbhome_1/
[root@oraclehost ~]# chown -R oracle:oinstall /u01/app/oracle/product/19.0.0/dbhome_1
[root@oraclehost ~]# chmod -R 775 /u01/app
```

解压数据库安装包：
```bash
[root@oraclehost ~]# su - oracle
[oracle@oraclehost ~]$ cd /u01/app/oracle/product/19.0.0/dbhome_1
[oracle@oraclehost dbhome_1]$ unzip -q LINUX.X64_193000_db_home.zip
[oracle@oraclehost dbhome_1]$ rm LINUX.X64_193000_db_home.zip
```

为oracle用户添加X Display权限：
```bash
[root@oraclehost ~]# cp /root/.Xauthority /home/oracle/
[root@oraclehost ~]# chown -R oracle:oinstall /home/oracle
[root@oraclehost ~]# echo $DISPLAY
localhost:10.0
[root@oraclehost ~]# su - oracle
[oracle@oraclehost ~]$ export DISPLAY=localhost:10.0
```

运行安装向导（oracle用户需要能够运行X Display）：
```bash
[oracle@oraclehost ~]$ cd /u01/app/oracle/product/19.0.0/dbhome_1
[oracle@oraclehost ~]$ ./runInstaller
```

# 图形化安装
1. 选择安装单实例（single instance database）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/49c9070cb3024b24a040eb075f55c4a2.png#pic_center)

2. 选择服务器级别（Server class）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/24e9f3d8cea64c95bdda3660d280f164.png#pic_center)

3. 选择企业版本。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4266163cbc1847868ff24176e48ae56d.png#pic_center)

4. 选择oracle用户的**Oracle Base**路径。

![在这里插入图片描述](https://img-blog.csdnimg.cn/427542d1b32b423b9219781a6319836e.png#pic_center)

5. 选择数据库安装类型为General Purpose。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b0fcb8b3ba5b4a7fb9dc9d7699248748.png#pic_center)

6. 定义数据库名称和SID。这里我们采用默认名称`orcl`。

![在这里插入图片描述](https://img-blog.csdnimg.cn/a094878918524526813b2e3b43a70ded.png#pic_center)

7. 自动内存管理配置。如果物理内存总量超过4GB，则不用勾选**Enable Automatic Memory Management**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/1b6210b723a6498a8843c473f5efa37c.png#pic_center)

8. 选择数据库字符集。这里我们采用默认的**Unicode AL32UTF8**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/564f9df7c07d441aa16cfaab709eadf1.png#pic_center)

9. 是否安装HR示例Schema。按需勾选。

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/b9b407d4864f40e690e2125c0b7af2aa.png#pic_center)

10. 选择存储。由于我们之前安装了GI并且创建了ASM磁盘组，这里可以选择**Oracle Automatic Storage Management**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d0178ffdd7a14aefbd9745d8a41b4ec9.png#pic_center)

11. Management Options：跳过。

![在这里插入图片描述](https://img-blog.csdnimg.cn/65ac0a61355745a89b189558e7a5ef13.png#pic_center)

12. 恢复区选项。由于我们之前安装了GI并且创建了ASM磁盘组，这里可以选择**Oracle Automatic Storage Management**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/785bfb1d5fdc4b44bd7089a1c196b389.png#pic_center)

13. 选择已经创建好的ASM磁盘组。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8cb58de1a3844d40bd4f230fbddb4432.png#pic_center)

14. 设置管理用户密码。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6ab37bc074d0432085e73dbe5bac6c78.png#pic_center)

15. 确认操作系统用户组信息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/39dd78a011c940bf9b45c379462ca35c.png#pic_center)

16. 数据库安装过程中，某些操作需要以root权限执行。输入root密码来授予oracle用户权限。

![在这里插入图片描述](https://img-blog.csdnimg.cn/dc17380a37ad4b35afebe930ad3ec3c6.png#pic_center)

17. 查看检查信息，如果有可以自动修复的问题，点击**Fix & Check Again**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6ef0476dbb3f406db0d65b4c579bf026.png#pic_center)

18.  核对信息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/eb339e6dc9ee4233a323d75f8d50f1db.png#pic_center)

19. 等待安装完成。

![在这里插入图片描述](https://img-blog.csdnimg.cn/6401b409eb864a46835ee6064d6140bf.png#pic_center)



# 安装后检查
为oracle用户配置环境变量：
```bash
[root@oraclehost ~]# su - oracle
[oracle@oraclehost ~]$ echo 'export ORACLE_SID=orcl' >> /home/oracle/.bash_profile
[oracle@oraclehost ~]$ echo 'export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1' >> /home/oracle/.bash_profile
[oracle@oraclehost ~]$ echo 'export PATH=$ORACLE_HOME/bin:$PATH' >> /home/oracle/.bash_profile
[oracle@oraclehost ~]$ . ./.bash_profile
[oracle@oraclehost ~]$ env | grep ORACLE
ORACLE_SID=orcl
ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
```

检查数据库能否登录：
```bash
[oracle@oraclehost ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Mon Dec 12 17:33:01 2022
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> set lines 200
SQL> col name format a20
SQL> col path format a20
SQL> select name,path,mount_status,group_number from v$asm_disk;

NAME                 PATH                 MOUNT_S GROUP_NUMBER
-------------------- -------------------- ------- ------------
DATA1                /dev/vdb             CACHED             1

SQL> select name,free_mb,total_mb from v$asm_diskgroup;

NAME                    FREE_MB   TOTAL_MB
-------------------- ---------- ----------
DATADG                    77172      81920
```

如果查询`v$asm_diskgroup`视图不显示磁盘组，但是执行`asmcmd lsdg`可以看到磁盘组已经正常挂载，可以检查`asm_diskstring`参数的配置。如果该参数为空，将其修改为ASM磁盘所在路径通配符即可。
```bash
[root@oraclehost ~]# su - grid
[grid@oraclehost ~]$ sqlplus / as sysasm

SQL> show parameter asm_diskstring

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
asm_diskstring                       string
SQL>
SQL> alter system set asm_diskstring='/dev/vd*';

System altered.

SQL> show parameter asm_diskstring

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
asm_diskstring                       string      /dev/vd*
SQL>
SQL> select name from v$asm_diskgroup;

NAME
--------------------
DATADG
```



**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/cwlin/oracle-grid-infrastructure-installation-checklist.html#GUID-71A93E07-7E50-449C-B425-02F04A2EA8E6
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/running-oracle-universal-installer-to-install-oracle-database.html#GUID-DD4800E9-C651-4B08-A6AC-E5ECCC6512B9
【3】https://www.modb.pro/db/495719


