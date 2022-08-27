
@[TOC](Linux批量新建分区)

# 手动创建和删除分区
现有一块新添加的磁盘sdc，手动创建分区并创建PV、VG的流程如下：
```bash
[root@mysql-node1 ~]# fdisk /dev/sdc
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0xafde06d0.

Command (m for help): n   #表示新建一个分区
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p):   #回车，默认建立一个主分区
Using default response p
Partition number (1-4, default 1):   #回车，默认分区编号
First sector (2048-20971519, default 2048):   #回车，默认起始位置
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): +5G   #根据需要选择大小，默认会使用剩余的全部磁盘空间来创建分区
Partition 1 of type Linux and of size 5 GiB is set

Command (m for help): w   #保存修改
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
[root@mysql-node1 ~]#
[root@mysql-node1 ~]# lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                 8:0    0   20G  0 disk
├─sda1              8:1    0    1G  0 part /boot
└─sda2              8:2    0   19G  0 part
  ├─centos-root   253:0    0   17G  0 lvm  /
  └─centos-swap   253:1    0    2G  0 lvm  [SWAP]
sdb                 8:16   0   20G  0 disk
└─rootvg-lv_mysql 253:2    0   20G  0 lvm  /mysql
sdc                 8:32   0   10G  0 disk
└─sdc1              8:33   0    5G  0 part
sr0                11:0    1  4.4G  0 rom
[root@mysql-node1 ~]#
[root@mysql-node1 ~]# pvcreate /dev/sdc1   #利用新建分区创建PV
  Physical volume "/dev/sdc1" successfully created.
[root@mysql-node1 ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda2  centos lvm2 a--  <19.00g    0
  /dev/sdb   rootvg lvm2 a--  <20.00g    0
  /dev/sdc1         lvm2 ---    5.00g 5.00g
[root@mysql-node1 ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   2   0 wz--n- <19.00g    0
  rootvg   1   1   0 wz--n- <20.00g    0
[root@mysql-node1 ~]# vgextend rootvg /dev/sdc1   #利用新建PV扩容VG
  Volume group "rootvg" successfully extended
[root@mysql-node1 ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   2   0 wz--n- <19.00g     0
  rootvg   2   1   0 wz--n-  24.99g <5.00g
```

手动移除VG、PV和分区的流程如下（移除VG和分区有风险，务必谨慎操作）：
```bash
[root@mysql-node1 ~]# vgreduce rootvg /dev/sdc1   #从VG中移除未使用的PV
  Removed "/dev/sdc1" from volume group "rootvg"
[root@mysql-node1 ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   2   0 wz--n- <19.00g    0
  rootvg   1   1   0 wz--n- <20.00g    0
[root@mysql-node1 ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda2  centos lvm2 a--  <19.00g    0
  /dev/sdb   rootvg lvm2 a--  <20.00g    0
  /dev/sdc1         lvm2 ---    5.00g 5.00g
[root@mysql-node1 ~]# pvremove /dev/sdc1   #移除未使用的PV
  Labels on physical volume "/dev/sdc1" successfully wiped.
[root@mysql-node1 ~]# pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda2  centos lvm2 a--  <19.00g    0
  /dev/sdb   rootvg lvm2 a--  <20.00g    0
[root@mysql-node1 ~]# fdisk /dev/sdc
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d   #表示删除一个分区
Selected partition 1
Partition 1 is deleted

Command (m for help): 1   #默认选择最新的分区
1: unknown command
Command action
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

Command (m for help): w   #保存修改（切记一旦保存就无法取消了！！！）
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
[root@mysql-node1 ~]# lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                 8:0    0   20G  0 disk
├─sda1              8:1    0    1G  0 part /boot
└─sda2              8:2    0   19G  0 part
  ├─centos-root   253:0    0   17G  0 lvm  /
  └─centos-swap   253:1    0    2G  0 lvm  [SWAP]
sdb                 8:16   0   20G  0 disk
└─rootvg-lv_mysql 253:2    0   20G  0 lvm  /mysql
sdc                 8:32   0   10G  0 disk
sr0                11:0    1  4.4G  0 rom
```

# 批量创建和删除分区
准备一个文本文件`sdc.txt`如下：
```
n



+5G
w
```
中间有三个空行，代表3次回车。执行下面的命令来创建一个5G大小的分区：
```bash
[root@mysql-node1 ~]# fdisk /dev/sdc < sdc.txt
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): Using default response p
Partition number (1-4, default 1): First sector (2048-20971519, default 2048): Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): Partition 1 of type Linux and of size 5 GiB is set

Command (m for help): The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
[root@mysql-node1 ~]#
[root@mysql-node1 ~]# lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                 8:0    0   20G  0 disk
├─sda1              8:1    0    1G  0 part /boot
└─sda2              8:2    0   19G  0 part
  ├─centos-root   253:0    0   17G  0 lvm  /
  └─centos-swap   253:1    0    2G  0 lvm  [SWAP]
sdb                 8:16   0   20G  0 disk
└─rootvg-lv_mysql 253:2    0   20G  0 lvm  /mysql
sdc                 8:32   0   10G  0 disk
└─sdc1              8:33   0    5G  0 part
sr0                11:0    1  4.4G  0 rom
```

配合ansible可以实现批量创建对多台主机创建新分区：
```bash
[root@mysql-node1 ~]# ansible -i myhosts all -m copy -a "src=sdc.txt dest=/root/sdc.txt"
[root@mysql-node1 ~]#
[root@mysql-node1 ~]# ansible -i myhosts all -m shell -a "fdisk /dev/sdc < /root/sdc.txt"
[root@mysql-node1 ~]#
```

同理，自动批量删除脚本可以这样实现：
```bash
[root@mysql-node1 ~]# cat sdc-rm.txt
d
1
w
[root@mysql-node1 ~]#
[root@mysql-node1 ~]# fdisk /dev/sdc < sdc-rm.txt
[root@mysql-node1 ~]# lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                 8:0    0   20G  0 disk
├─sda1              8:1    0    1G  0 part /boot
└─sda2              8:2    0   19G  0 part
  ├─centos-root   253:0    0   17G  0 lvm  /
  └─centos-swap   253:1    0    2G  0 lvm  [SWAP]
sdb                 8:16   0   20G  0 disk
└─rootvg-lv_mysql 253:2    0   20G  0 lvm  /mysql
sdc                 8:32   0   10G  0 disk
sr0                11:0    1  4.4G  0 rom
```

