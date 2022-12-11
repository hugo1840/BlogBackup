---
tags: [oracle]
title: Oracle单机部署：安装前检查和配置
created: '2022-09-28T06:26:21.565Z'
modified: '2022-12-11T08:45:05.523Z'
---

Oracle单机部署：安装前检查和配置


**Oracle版本：19c**

到[Oracle官网](https://www.oracle.com/cn/database/technologies/oracle19c-linux-downloads.html)下载以下两个安装包：
- `LINUX.X64_193000_db_home.zip`
- `LINUX.X64_193000_grid_home.zip`


# 服务器检查和配置
## 启用X Windows和X11-Forwarding
我这里使用的是阿里云的远程云服务器，本地使用MobaXterm连接。

首先要在阿里云控制台的安全组中对`源:0.0.0.0/0`的授权。

远程服务器配置：
```bash
# 安装x11相关软件
yum install xorg-x11-xauth xorg-x11-fonts-* xorg-x11-font-utils xorg-x11-fonts-Type1 xclock -y

# 开启x11 forwarding
vim /etc/ssh/sshd_config
# 取消X11Forwarding yes该行的注释（如果是no修改为yes）

# 重启sshd服务
systemctl restart sshd.service
```

通过本地MobaXterm重新连接远程服务器（注意勾选X11-Forwarding）：
```bash
     │   • SSH compression : ✔                                            │
     │   • SSH-browser     : ✔                                            │
     │   • X11-forwarding  : ✔  (remote display is forwarded through SSH) │
     │   • DISPLAY         : ✔  (automatically set on remote server)      |
```
连接成功后，显示上面的信息，并且MobaXterm右上角X Server正常运行即可。

在云服务器端进行测试：
```bash
# 检查DISPLAY环境变量是否已定义
[root@oraclehost ~]# echo $DISPLAY
localhost:10.0

# 运行一个图形化程序
[root@oraclehost ~]# xclock
```
如果xclock命令执行成功，本地Windows主机上会出现一个时钟的图形化界面。

## 检查系统硬件和内存配置

- 检查runlevel：必须为**3或5**。
```bash
runlevel
```

- 检查物理内存总量：内存至少**16G**。
```bash
grep MemTotal /proc/meminfo
```

- 检查swap分区大小：物理内存在2G到16G之间时，Swap大小建议**等于物理内存**；物理内存大于16G时，Swap大小建议为**16G**。
```bash
grep SwapTotal /proc/meminfo
```

- 检查可用的内存和swap分区大小：
```bash
free -g
```

- 检查tmp目录大小：至少**1GB**剩余空间。
```bash
df -h /tmp
```

- 检查系统架构：必须为`x86_64`。
```bash
uname -m
```

- 检查共享内存挂载：挂载方式必须为`tmpfs`，有`rwx`权限，无`noexec`和`nosuid`的配置。
```bash
df -h /dev/shm
ll /dev/ | grep shm
```

**附**：创建Swap分区的方法如下：
```bash
# 从根分区中划分出16G用于创建swap分区
[root@oraclehost ~]# dd if=/dev/zero of=/swapfile1 bs=1024 count=15728640
15728640+0 records in
15728640+0 records out
16106127360 bytes (16 GB) copied, 109.583 s, 147 MB/s

# 创建swap交换文件
[root@oraclehost ~]# mkswap /swapfile1
Setting up swapspace version 1, size = 15728636 KiB
no label, UUID=5f78a45b-2ad6-4d9d-bd43-affb849b0e5d

# 激活swap交换文件
[root@oraclehost ~]# swapon /swapfile1
swapon: /swapfile1: insecure permissions 0644, 0600 suggested.

# 设置开机自动启用swap分区
[root@oraclehost ~]# echo '/swapfile1  swap swap  defaults  0 0' >> /etc/fstab
[root@oraclehost ~]# cp /etc/sysctl.conf /etc/sysctl.conf.bak20221208
[root@oraclehost ~]# sed -i 's/^vm.swappiness.*/vm.swappiness = 10/g' /etc/sysctl.conf

# 重启服务器后检查swap分区大小
[root@oraclehost ~]# reboot
[root@oraclehost ~]# free -g
              total        used        free      shared  buff/cache   available
Mem:             15           0          14           0           0          14
Swap:            14           0          14
```


# 操作系统配置
## 红帽7操作系统配置

- 检查已安装openSSH：
```bash
rpm -q openssh
```

- 检查操作系统版本：最好是红帽**7.5**之后的版本。
```bash
cat /etc/redhat-release 
```

- 检查Linux内核版本：不低于`3.10.0-862.11.6.el7`。
```bash
uname -sr
```

- 确保已安装以下软件包（`yum install -y`）：
```bash
bc
binutils
compat-libcap1
compat-libstdc++-33
elfutils-libelf
elfutils-libelf-devel
fontconfig-devel
glibc
glibc-devel
ksh
libaio
libaio-devel
libX11
libXau
libXi
libXtst
libXrender
libXrender-devel
libgcc
libstdc++
libstdc++-devel
libxcb
make
smartmontools
sysstat
```

- 安装JDK8：
```bash
yum install -y java-1.8.0-openjdk*
```

## tsc时钟源配置
将虚拟时钟源设置为**tsc**可以提高服务器性能：
```bash
# 检查可用的时钟源
[root@oraclehost ~]# cat /sys/devices/system/clocksource/clocksource0/available_clocksource
kvm-clock tsc acpi_pm

# 检查当前使用的时钟源
[root@oraclehost ~]# cat /sys/devices/system/clocksource/clocksource0/current_clocksource
kvm-clock

# 修改当前的时钟源为tsc
[root@oraclehost ~]# echo "tsc" > /sys/devices/system/clocksource/clocksource0/current_clocksource
```

将`clocksource=tsc`添加到`/etc/default/grub`中`GRUB_CMDLINE_LINUX`配置中，这样即使服务器重启后也会继续使用tsc时钟源。
```bash
[root@oraclehost ~]# cat /etc/default/grub | grep GRUB_CMDLINE_LINUX
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rhgb quiet net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs nvme_core.io_timeout=4294967295 nvme_core.admin_timeout=4294967295 clocksource=tsc"
```

## 关闭透明大页（THP）
透明大页（Transparent Huge Pages）会导致内存分配延迟，影响数据库性能，因此建议关闭。

检查透明大页是否开启：
```bash
[root@oraclehost ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
#[always]表示已启用透明大页
```

将`transparent_hugepage=never`添加到`/etc/default/grub`中`GRUB_CMDLINE_LINUX`配置中来关闭透明大页：
```bash
[root@oraclehost ~]# cat /etc/default/grub | grep GRUB_CMDLINE_LINUX
GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rhgb quiet net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs nvme_core.io_timeout=4294967295 nvme_core.admin_timeout=4294967295 clocksource=tsc transparent_hugepage=never"
```

执行下面的命令来重建`grub.cfg`文件：
```
[root@oraclehost ~]# grub2-mkconfig -o /boot/grub2/grub.cfg
```
最后**重启**服务器来永久关闭透明大页。


## 检查磁盘I/O调度器
查看Oracle数据盘使用的I/O调度器：
```bash
# 假设数据盘为vdb
[root@oraclehost ~]# cat /sys/block/vdb/queue/scheduler
[mq-deadline] kyber none
```
确保数据盘使用的I/O调度器类型为**Deadline**。

## 修改Linux内核参数
检查以下操作系统内核参数：
```bash
# 检查semmsl、semmns、semopm和semmni
sysctl -a | grep sem
# 检查shmall、shmmax和shmmni
sysctl -a | grep shm
# 检查file-max
sysctl -a | grep file-max

# 检查ip_local_port_range
sysctl -a | grep ip_local_port_range
# 检查rmem_default
sysctl -a | grep rmem_default
# 检查rmem_max
sysctl -a | grep rmem_max

# 检查wmem_default
sysctl -a | grep wmem_default
# 检查wmem_max
sysctl -a | grep wmem_max
# 检查aio-max-nr
sysctl -a | grep aio-max-nr
```

创建下面的配置文件，除**shmmax**和**shmall**根据服务器物理内存计算外，其余参数默认采用下面给出的值：
```bash
[root@oraclehost ~]# cat /etc/sysctl.d/97-oracle-database-sysctl.conf
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 4194304        # shmmax/2048
kernel.shmmax = 8589934592     # 物理内存的一半，单位bytes
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
```
服务器重启后上面文件中的内核参数会被持久化。

执行下面的命令来使内核参数修改立即生效：
```bash
sysctl --system
```

查看当前生效的所有内核参数：
```bash
sysctl -a
```


# Oracle用户和环境配置
## 创建部署用户组
如果是首次安装Oracle产品，需要创建oinstall用户组：
```bash
groupadd -g 54321 oinstall
```

创建OSOPER用户组oper，该组内用户具有**SYSOPER**权限：
```bash
groupadd oper
```

创建ASM管理用户组asmoper，组内用户具有ASM启动和停止的权限：
```bash
groupadd asmoper
```

## 创建管理用户组
创建数据库管理用户组dba，该用户组内的用户将拥有**SYSDBA**权限。
```bash
groupadd dba
```

创建ASM管理用户组asmdba和asmadmin，用于ASM管理。组内用户具有**SYSASM**权限。
```bash
groupadd asmdba
groupadd asmadmin
```

此外还需创建以下不同角色的管理用户组：
- OSBACKUPDBA用户组：用于数据库备份和恢复管理。
```bash
groupadd backupdba
```

- OSDGDBA用户组：用于Data Guard管理，组内用户具有SYSDG权限。
```bash
groupadd dgdba
```

- OSKMDBA用户组：用于加密密钥管理，组内用户具有SYSKM权限。
```bash
groupadd kmdba
```

- OSRACDBA用户组：用于RAC集群管理，组内用户具有SYSRAC权限。
```bash
groupadd racdba
```


## 创建部署用户
Oracle Grid安装需要用到grid用户，用于管理Clusterware和ASM。Oracle数据库安装则需要用到oracle用户。grid用户和oracle用户都应该属于oinstall用户组。

创建oracle用户，并指定uid和主要用户组：
```bash
useradd -u 54321 -g oinstall oracle
```

创建grid用户，并指定uid和主要用户组：
```bash
useradd -u 54331 -g oinstall grid
```

为grid和oracle用户添加次要用户组：
```bash
usermod -g oinstall -G dba,asmdba,asmadmin,backupdba,dgdba,kmdba,racdba oracle
usermod -g oinstall -G dba,asmdba,asmadmin,backupdba,dgdba,kmdba,racdba grid
```

查看用户组信息：
```bash
[root@oraclehost ~]# id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(backupdba),54323(dgdba),54324(kmdba),54325(racdba),54326(dba),54327(asmdba)
[root@oraclehost ~]# id grid
uid=54331(grid) gid=54321(oinstall) groups=54321(oinstall),54322(backupdba),54323(dgdba),54324(kmdba),54325(racdba),54326(dba),54327(asmdba)
[root@oraclehost ~]#
```

## umask与环境变量设置
需要将grid用户和oracle用户的默认文件权限掩码设umask置为`022`。

以grid用户为例：
```bash
su - grid
# 修改umask为022
echo 'umask 022' >> .bash_profile
. ./.bash_profile

# 查看umask文件权限掩码
umask
```

如果`.bash_profile`文件中定义了`ORACLE_SID`、`ORACLE_HOME`或者`ORACLE_BASE`这几个环境变量，需要将它们移除。

如果事先定义了`$ORACLE_HOME`、`$ORA_NLS10`、`$TNS_ADMIN`、`$ORA_CRS_HOME`等Oracle相关的环境变量，也需要将它们移除。
```bash
env | grep ORA
env | grep TNS
```

## 用户系统资源限制
对grid用户和oracle用户的系统资源限制如下：
|  | nofile | nproc | stack | memlock（仅oracle用户） |
| :--: | :--: | :--: | :--: | :--: |
| 软限制 | 至少1024 | 至少2047 | 至少10240KB | unlimited |
| 硬限制 | 至少65536 | 至少16384 | 至少10240KB，至多32768KB | unlimited |

检查grid用户和oracle用户的nofile、nproc和stack系统配额，以oracle用户为例：
```bash
su - oracle
# 检查nofile的软限制和硬限制
ulimit -Sn
ulimit -Hn
# 检查nproc的软限制和硬限制
ulimit -Su
ulimit -Hu
# 检查stack的软限制和硬限制
ulimit -Ss
ulimit -Hs
# 检查memlock的软限制和硬限制
ulimit -Sl
ulimit -Hl

# 列出当前用户的所有系统配额
ulimit -Sa
ulimit -Ha
```

使用root用户修改`/etc/security/limits.conf`来修改系统资源配额：
```bash
grid   soft   nofile   65535
grid   hard   nofile   65535

grid   soft   nproc    4096
grid   hard   nproc    62381

grid   soft   stack    10240
grid   hard   stack    32768

oracle   soft   nofile   65535
oracle   hard   nofile   65535

oracle   soft   nproc    4096
oracle   hard   nproc    62381

oracle   soft   stack    10240
oracle   hard   stack    32768

oracle   soft   memlock  unlimited
oracle   hard   memlock  unlimited
```
上述修改在切换到对应用户时生效。




**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/oracle-database-installation-checklist.html#GUID-E847221C-1406-4B6D-8666-479DB6BDB046
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/supported-red-hat-enterprise-linux-7-distributions-for-x86-64.html#GUID-2E11B561-6587-4789-A583-2E33D705E498
【3】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/server-configuration-checklist-for-oracle-database-installation.html#GUID-CD4657FB-2DDC-4B30-AAB4-2C927045A86D
【4】https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/changing-kernel-parameter-values.html#GUID-FB0CC366-61C9-4AA2-9BE7-233EB6810A31



