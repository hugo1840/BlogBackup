@[TOC](Linux学习笔记（十一）：磁盘配额、磁盘阵列与逻辑卷)
# 磁盘配额Quota
**quota的一般用途**
磁盘配额（Quota）比较常用的场景有：

- 对WWW server，每个用户的网页空间容量限制；
- 对Mail server，每个用户的邮箱空间限制；
- 对File server，每个用户的最大网络硬盘空间限制。

对Linux系统主机可以配置如下作用：

- 限制某一**用户组**能够使用的最大磁盘配额；
- 限制某一**账户**的最大磁盘配额；
- 限制某一**目录**的最大磁盘配额。

**quota的使用限制**
Quota虽然好用，但却也有如下限制：

- 在**EXT**文件系统中仅能针对**整个**文件系统，而无法针对单一目录设置；
- Linux核心必须支持quota功能并且开启；
- 只对一般身份有效，也就是不能对root用户设置quota；
- 若启用SELinux，并非所有目录都能设置quota（一般只能对/home设置）。

**quota的设定方法**
暂略。

# 磁盘阵列RAID
磁盘阵列（Redundant Arrays of Independent Disks, RAID），即由独立磁盘构成的具有**冗余**能力的阵列。通过软件或硬件技术，RAID可以实现由多个较小的独立磁盘组成一个大容量的磁盘组，从而实现==更好的存储能力和更高的可靠性==。

## RAID level
由于选择的level不同，整合后的RAID也具备不同的功能。下面介绍几种常见的RAID level。

**RAID-0 等量模式 stripe：效能最佳**
当一个文件要写入RAID时，该文件会被切分为不同的chunk（大小一般为4K\~1M），然后依次存入各个磁盘。如果是由两块磁盘组成的RAID-0，写入100M数据时，每块磁盘都会分配到50M的chunk数据。相比直接存入一块磁盘，因为每块磁盘存入的数据都变小了，因此存储速度更快。但缺点是只要有一块RAID-0磁盘损坏，存储数据的完整性就会被破坏。

**RAID-1 镜像模式 mirror：完整备份**
RAID-1模式的目的在于让同一份数据完整地保存在两块磁盘上。如果是由两块磁盘组成的RAID-1，写入100M数据时，每块磁盘都会分配到100M的相同数据。其优点在于存储数据的安全性提高了，而缺点在于磁盘的实际存储空间缩小了至少50%（当两块磁盘大小不相同时，只能存储最小容量磁盘的数据）。

**RAID 0+1**
RAID 0+1结合了前两种模式的特点。假设有四块磁盘，将前两块和后两块分别组成两个RAID-1，然后将两个RAID-1组成一个RAID-0。

**RAID-5：均衡考虑**
RAID-5至少需要三块磁盘，其数据写入方式类似RAID-0，但是在写入过程中会在每块磁盘加入一个同位检查数据（**Parity**），用于记录其他磁盘的备份数据。当有一块磁盘损坏时，可以借助存储在其他磁盘上的Parity数据来恢复其数据。RAID-5的实际存储容量等于**整体磁盘数减去一块磁盘**。当损坏的磁盘数**超过一块**时，RAID-5上存储的数据完整性就被破坏了。

**RAID 6**
RAID-6与RAID-5类似，差别在于使用了两块磁盘存储Parity数据，因此整体的实际存储容量也减少了两块磁盘。但是RAID-6允许损坏的磁盘数量也提高到了两块。

**Spare Disk：备用磁盘**
当RAID中有磁盘损坏时，需要将坏掉的磁盘换成一块新磁盘，使得磁盘阵列可以在新磁盘上重建数据。备用磁盘即是一块或多块尚未加入原本RAID的磁盘，在有磁盘损坏时，备用磁盘会被主动加入RAID用于重建数据。

**磁盘阵列的优点**
|  | RAID-0 | RAID-1 | RAID 0+1 | RAID-5 | RAID-6 |
|--|--|--|--|--|--|
| 最少磁盘数 | 2 | 2 | 4 | 3 | 4 |
| 最大容错磁盘数 | 0 | n-1 | n/2 | 1 | 2 |
| 数据安全性 | 无 | 最佳 | 最佳 | 好 | 比RAID-5好 |
| 理论写入效能 | n | 1 | n/2 | <n-1 | <n-2 |
| 理论读取效能 | n | n | n | <n-1 | <n-2 |
| 可用容量 | n | 1 | n/2 | n-1 | n-2 |
| 一般应用 | 强调效能而不是数据安全性 | 数据备份 | 服务器与云系统 | 数据备份 | 数据备份 |

## 软件磁盘阵列
根据其技术实现，RAID可以分为硬件磁盘阵列（hardware RAID）和软件磁盘阵列（software RAID）。硬件RAID通过磁盘阵列卡来实现阵列，效能较佳，但是成本较高；软件RAID则是通过软件仿真来实现阵列。硬件RAID对应的设备名格式为`/dev/sd[a-p]`，因为使用了SCSI模块；而软件RAID对应的设备名格式一般为`/dev/md[0-9]`。

**软件RAID创建**
CentOS和RHEL中提供磁盘阵列功能的软件是 **mdadm**。

```bash
[root@study ~]$ mdadm --detail /dev/md0
[root@study ~]$ mdadm --create /dev/md[0-9] --auto=yes --level=[015] --chunk=NK \
> --raid-devices=N --spare-devices=N /dev/sdx /dev/hdx

# 选项与参数：
--create # 为建立RAID的选项
--auto=yes # 后面接决定建立的RAID设备，即/dev/md[0-9] 
--level=[015] # 设置RAID level
--chunk=NK # RAID设备的chunk大小，一般为64K或512K
--raid-devices=N # 使用几个磁盘partition作为RAID设备
--spare-devices=N # 使用几个磁盘作为备用设备
--detail # RAID设备的详细信息
```

假设我们要创建一个由4个分区组成的RAID-5。

```bash
# 准备工作：用fdisk或gdisk分出5个大小为1G的partition
# 使用lsblk指令查看5个分区的名字依次为vda[5-9]

# 使用mdadm指令创建RAID
[root@study ~]$ mdadm --create /dev/md0 --auto=yes --level=5 --chunk=256K \
> --raid-devices=4 --spare-devices=1 /dev/vda{5,6,7,8,9}
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
# 查看已创建RAID的详细信息
[root@study ~]$ mdadm --detail /dev/md0
/dev/md0:
	Raid level: raid5
......
[root@study ~]$ cat /proc/mdstat	# 查看RAID设备状态
......
# 格式化RAID
[root@study ~]$ mkfs.xfs -f -d su=256k,sw=3 -r extsize=768k /dev/md0
# su=256k对应chunk的大小；四个分区组成RAID5，容量只有三个分区的大小，因此sw=3，extsize=256k*3=768k

# 挂载RAID设备
[root@study ~]$ mkdir /srv/raid
[root@study ~]$ mount /dev/md0 /srv/raid
[root@study ~]$ df -Th /srv/raid # 查看挂载信息
```

**软件RAID错误修复**

```bash
[root@study ~]$ mdadm --manage /dev/md[0-9] [--add 设备名] [--remove 设备名] [--fail 设备名]
# 选项与参数
--add	# 向md中添加设备
--remove	# 将设备从md中移除
--fail	# 将设备状态设置为故障failure
```

**RAID开机自动挂载**

```bash
[root@study ~]$ mdadm --detail /dev/md0 | grep -i uuid
UUID: xxxxxxxxxxxxxxxxxxxxxxxxxxx
# 修改配置文件/etc/mdadm.conf
[root@study ~]$ vim /etc/mdadm
ARRAY	/dev/md0	UUID=xxxxxxxxxxxxxxxxxxxxxxxxxxx
[root@study ~]$ blkid /dev/md0
/dev/md0: UUID="xxxxxxxxxxxxxxxxxxx" TYPE="xfs"
# 设置开机自动挂载
[root@study ~]$ vim /etc/fstab
UUID=xxxxxxxxxxxxxxxxxxx   /srv/raid   xfs   defaults   0   0
[root@study ~]$ umount /dev/md0; mount -a
[root@study ~]$ df -Th /srv/raid
# 可以重启系统测试是否能自动挂载
```

**正确关闭软件RAID**

```bash
# 取消挂载与开机自动挂载
[root@study ~]$ umount /srv/raid
[root@study ~]$ vim /etc/fstab
# UUID=xxxxxxxxxxxxxxxxxxx   /srv/raid   xfs   defaults   0   0

# 覆盖RAID设备的metadata和xfs的superblock
[root@study ~]$ dd if=/dev/zero of=/dev/md0 bs=1M count=50
# 关闭RAID设备
[root@study ~]$ mdadm --stop /dev/md0
mdadm: stoppped /dev/md0
# 覆盖RAID设备用过的分区（用无穷的0字符初始化）
[root@study ~]$ dd if=/dev/zero of=/dev/vda5 bs=1M count=10
[root@study ~]$ dd if=/dev/zero of=/dev/vda6 bs=1M count=10
[root@study ~]$ dd if=/dev/zero of=/dev/vda7 bs=1M count=10
[root@study ~]$ dd if=/dev/zero of=/dev/vda8 bs=1M count=10
[root@study ~]$ dd if=/dev/zero of=/dev/vda9 bs=1M count=10

[root@study ~]$ cat /proc/mdstat
unused devices: <none>
# 删除配置文件中相应的行
[root@study ~]$ vim /etc/mdadm.conf
# ARRAY	/dev/md0	UUID=xxxxxxxxxxxxxxxxxxxxxxxxxxx
```

# 逻辑卷管理器LVM
每个Linux使用者在安装Linux时都会遇到这样的困境：在为系统分区时，如何精确评估和分配各个硬盘分区的容量。系统管理员不但要考虑到当前某个分区需要的容量，还要预见该分区以后可能需要的容量的最大值。如果估计不准确，当遇到某个分区不够用时管理员可能甚至要备份整个系统、清除硬盘、重新对硬盘分区，然后恢复数据到新分区。逻辑卷管理（**Logical Volume Manager**, LVM）可以弹性地调整文件系统容量。LVM可以整合多个分区，使它们看上去就像一个“磁盘”一样，并且可以在之后向这个“磁盘”中新增或移除分区。

## 基本概念：PV, VG, LV
LVM中需要掌握的基本概念包括PV, PE, VG, LV, LE。

**物理卷（Physical Volume, PV）**
PV是LVM的基本存储逻辑块。和基本的物理存储介质（如分区、磁盘等）相比，PV包含有与LVM相关的管理参数。实体的磁盘或者分区在通过`fdisk`或`gdisk`指令把系统标识符（System ID）修改为`8e`（LVM标识符）后，就可以使用`pvcreate`创建为LVM最底层的PV。

**物理块（Physical Extent, PE）**
PE是PV的基本划分单元，具有唯一编号的PE是可以被LVM寻址的最小单元。PE的大小是可配置的，默认为4MB。所以PV由大小等同的基本单元PE组成。

**卷组（Volume Group, VG）**
VG类似于非LVM系统中的物理磁盘，其由一个或多个物理卷PV组成。可以在卷组上创建一个或多个LV。VG设备的名字格式为`/dev/vg_name`。

**逻辑卷（Logical Volume, LV）**
类似于非LVM系统中的磁盘分区，LV建立在VG之上。在逻辑卷LV之上可以建立文件系统。LV的容量大小受限于LV中的PE总数。LV设备的名字格式为`/dev/vg_name/lv_name`。


**逻辑块（Logical Extent, LE）**
LV也被划分为可被寻址的基本单位，称为LE。在同一个卷组中，LE的大小和PE是相同的，并且一一对应。

由于LVM的重点在于可以弹性调整文件系统容量而不是读写性能，其默认的读写模式是**线性模式**。

## 逻辑卷创建与管理
**创建LVM**
假设我们有4个大小为1G的分区可以整合成一个VG，并创建一个大小为2G的LV。

```bash
# 检查分区状况和系统标识符
[root@study ~]$ fdisk -l /dev/vda
Number	Start (sector)	End (sector)	Size	Code	Name
     1	2048          	6143        2.0 MiB 	EF02	
     ...
     4	65026048      	67123199    1024.0 MiB	8300	Linux Filesystem
     5	67123200     	69220351    1024.0 MiB	8E00	Linux LVM
     6	...           	...         1024.0 MiB	8E00	Linux LVM
     7	...           	...         1024.0 MiB	8E00	Linux LVM
     8	...           	...         1024.0 MiB	8E00	Linux LVM
     9	...           	...         1024.0 MiB	8E00	Linux LVM

# 检查已存在的PV
[root@study ~]$ pvscan
PV	/dev/vda3	...
[root@study ~]$ pvdisplay /dev/vda3
......
# 创建PV
[root@study ~]$ pvcreate /dev/vda{5,6,7,8}
PV "/dev/vda5" successfully created
PV "/dev/vda6" successfully created
PV "/dev/vda7" successfully created
PV "/dev/vda8" successfully created
[root@study ~]$ pvscan
......
# 创建VG（选项-s后接PE大小）
[root@study ~]$ vgcreate -s 16M vg_test /dev/vda{5,6,7}
VG "vg_test" successfully created
[root@study ~]$ vgscan	# 检查系统上已经创建的VG
[root@study ~]$ vgdisplay vg_test	# 查看某个VG的具体信息
......
# VG扩容
[root@study ~]$ vgextend vg_test /dev/vda8
VG "vg_test" successfully extended
[root@study ~]$ vgdisplay vg_test	
......
# 创建LV（选项-L后接LV容量，-n后接LV名称）
[root@study ~]$ lvcreate -L 2G -n lv_test vg_test
LV "lv_test" successfully created
[root@study ~]$ lvscan	# 检查系统上已经创建的LV
[root@study ~]$ lvdisplay /dev/vg_test/lv_test	# 查看某个LV的具体信息
......
# LV挂载
[root@study ~]$ mkfs.xfs /dev/vg_test/lv_test
[root@study ~]$ mkdir /srv/test
[root@study ~]$ mount /dev/vg_test/lv_test /srv/test
[root@study ~]$ df -Th /srv/test
Filesystem	                Type	Size	Used	Avail	Use%	Mounted on
/dev/mapper/vg_test-lv_test	xfs	    2.0G	33M	    2.0G	2%	    /srv/test
[root@study ~]$ cp -a /etc /var/log /srv/test   # 测试是否可用
```

**LV扩容**
LV扩容需要所处的VG中还有剩余的空间。LV扩容后还需要扩容挂载的文件系统。EXT文件系统支持扩容和缩容，但是XFS文件系统仅支持扩容。假设前面挂载的`/srv/test`需要增加500M容量：

```bash
[root@study ~]$ vgdisplay vg_test	# 检查VG是否有剩余空间
Free PE / Size: 124 / 1.94 GiB
...
# LV扩容
[root@study ~]$ lvresize -L +500M /dev/vg_test/lv_test
LV "lv_test" successfully resized
[root@study ~]$ lvscan
[root@study ~]$ df -Th /srv/test   # Size还是2.0G
Filesystem	                Type	Size	Used	Avail	Use%	Mounted on
/dev/mapper/vg_test-lv_test	xfs	    2.0G	111M	1.9G	6%	    /srv/test
...
# 文件系统扩容（EXT文件系统使用resize2fs，XFS使用xfs_growfs）
[root@study ~]$ xfs_info /srv/test   # 检查文件系统superblock信息
..., agcount=4, ..., blocks=524288, ...
[root@study ~]$ xfs_growfs /srv/test   # XFS文件系统扩容
[root@study ~]$ xfs_info /srv/test
..., agcount=5, ..., blocks=655360, ...
[root@study ~]$ df -Th /srv/test   # Size变成了2.5G
Filesystem	                Type	Size	Used	Avail	Use%	Mounted on
/dev/mapper/vg_test-lv_test	xfs	    2.5G	111M	2.4G	5%	    /srv/test
```

**动态调整磁盘**
LVM thin Volume，即建立一个可以随用随取的磁盘容量存储池（thin pool），然后再由该存储池生成一个LV。在实际的使用过程中，该LV设备使用了多少容量时，才会去thin pool中获取相应的磁盘空间。该LV的纪录容量可以大于thin pool，但是实际使用量不能超过thin pool的最大容量。

```bash
# 创建thin pool
[root@study ~]$ lvcreate -L 1G -T vg_test/vg_pool   
[root@study ~]$ lvdisplay /dev/vg_test/vg_pool
...
LV Size: 1.00 GiB
Allocated pool data: 0.00%
Allocated metadata: 0.24%
...
[root@study ~]$ lvs vg_test   # 查看VG中的LV
LV     	VG     	Attr	LSize	Pool	Origin	Data%	Meta%	...
lv_test	vg_test ......
vg_pool	vg_test ......

# 创建LV并绑定thin pool
[root@study ~]$ lvcreate -V 10G -T vg_test/vg_pool -n lv_thin1
[root@study ~]$ lvs vg_test
LV     	    VG     	Attr	LSize	Pool	Origin	Data%	Meta%	...
lv_test	    vg_test ... 	2.50G
lv_thin1	vg_test	... 	10.00G	vg_pool
vg_pool	    vg_test ... 	1.00G

# 格式化与挂载
[root@study ~]$ mkfs.xfs /dev/vg_test/lv_thin1
[root@study ~]$ mkdir /srv/thin
[root@study ~]$ mount /dev/vg_test/lv_thin1 /srv/thin
[root@study ~]$ df -Th /srv/thin
...
```

**LVM磁盘快照**
LV磁盘快照是指将某一时刻的磁盘数据记录下来，将来如果发生数据修改，原始数据就会被移动到快照区。快照区与文件系统共享未变更的LVM存储区域，因此是很好的备份工具。快照区必须与被快照的LV处于同一个VG中。

```bash
[root@study ~]$ vgdisplay vg_test	# 检查VG是否有剩余空间
Free PE / Size: 26 / 416 MiB
...
# 为lv_test创建快照（选项-s代表快照，-l 26代表使用26个PE）
[root@study ~]$ lvcreate -s -l 26 -n lv_snap1 /dev/vg_test/lv_test
LV "lv_snap1" created successfully
[root@study ~]$ lvdisplay /dev/vg_test/lv_snap1
...
LV Size: 2.50 GiB   # lv_test的原始容量
COW-table size: 416.00 MiB   # 该快照能够记录的最大容量
Allocated to snapshot: 0.00%   # 已被使用的容量
...
# 挂载快照
[root@study ~]$ mkdir /srv/snapshot1
[root@study ~]$ mount -o nouuid /dev/vg_test/lv_snap1 /srv/snapshot1
[root@study ~]$ df -Th /srv/test /srv/snapshot1
Filesystem	                    Type	Size	Used	Avail	Use%	Mounted on
/dev/mapper/vg_test-lv_test	    xfs	    2.5G	111M	2.4G	5%	    /srv/test
/dev/mapper/vg_test-lv_snap1	xfs	    2.5G	111M	2.4G	5%	    /srv/snapshot1
# 因为与快照区域共享未修改数据的存储区域，所以磁盘使用状况完全相同
```

我们可以利用LVM磁盘快照进行数据备份和还原。

```bash
# 修改lv_test中的数据
[root@study ~]$ cp -a /usr/share/doc /srv/test
[root@study ~]$ rm -rf /srv/test/log
[root@study ~]$ rm -rf /srv/test/etc/sysconfig
[root@study ~]$ df -Th /srv/test /srv/snapshot1
Filesystem	                    Type	Size	Used	Avail	Use%	Mounted on
/dev/mapper/vg_test-lv_test	    xfs	    2.5G	146M	2.4G	6%	    /srv/test
/dev/mapper/vg_test-lv_snap1	xfs	    2.5G	111M	2.4G	5%	    /srv/snapshot1
[root@study ~]$ lvdisplay /dev/vg_test/lv_snap1
...
Allocated to snapshot: 21.47%   # 已被使用的容量
...

# 利用快照备份原始文件系统（将快照备份为/home/lvm.dump）
[root@study ~]$ xfsdump -l 0 -L lvm1 -M lvm1 -f /home/lvm.dump /srv/snapshot1
[root@study ~]$ umount /srv/snapshot1
[root@study ~]$ lvremove /dev/vg_test/lv_snap1   # 移除快照（因为已经备份了）

# 从备份还原LV
[root@study ~]$ umount /srv/test
[root@study ~]$ mkfs.xfs -f /dev/vg_test/lv_test   # 格式化
[root@study ~]$ mount /dev/vg_test/lv_test /srv/test
[root@study ~]$ xfsrestore -f /home/lvm.dump -L lvm1 /srv/test   # 还原LV
[root@study ~]$ ll /srv/test   # 检查
```

**关闭LVM**
关闭LVM的正确流程为：

 1. 移除LVM文件系统（包括所有快照与LV挂载的文件系统）；
 2. 移除LV；
 3. 取消VG的Active标志；
 4. 移除VG;
 5. 移除PV;
 6. 使用`fdisk`取消`8e`标识符设定。

```bash
[root@study ~]$ umount /srv/test /srv/thin /srv/snapshot1
[root@study ~]$ lvremove /dev/vg_test/lv_thin1 /dev/vg_test/vg_pool
[root@study ~]$ lvremove /dev/vg_test/lv_test
[root@study ~]$ vgchange -a n vg_test   #  取消VG的Active标志
[root@study ~]$ vgremove vg_test
[root@study ~]$ pvremovve /dev/vda{5,6,7,8}
```

References
[1] [Managing RAID Red Hat Entreprise Linux](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_storage_devices/managing-raid_managing-storage-devices)
[2] [LVM](https://baike.baidu.com/item/LVM/6571177)
