@[TOC](Linux学习笔记（六）：磁盘分区、格式化与挂载)
# 磁盘的分区、格式化与挂载
## 查看分区状态
**lsblk指令**
lsblk指令（list block）可以列出系统上的所有磁盘列表。指令格式为 `lsblk [-dfimpt] [device]`。其选项与参数包括：
* -d：仅列出磁盘本身，不会列出该磁盘的分区数据；
* -f：同时列出磁盘内的文件系统名称；
* -i：使用ASCII输出，不要使用复杂的编码；
* -m：同时输出该设备在/dev下的权限数据；
* -p：列出设备的完整文件名；
* -t：列出磁盘的详细数据，包括磁盘队列机制、预读写的数据量大小等。

lsblk指令的输出字段一般包括：
* NAME：设备文件名；
* **MAJ:MIN**：主要:次要设备代码；
* RM：是否为可卸载装置（removable），如光盘、USB等；
* SIZE：设备容量；
* RO：是否为只读设备（read-only）；
* TYPE：是磁盘（disk）、分区槽（part）还是只读存储器（rom）等输出；
* MOUNTPOINT：挂载点。

**blkid指令**
blkid指令（block ID）可以列出设备的UUID等参数。Linux系统内的每个设备都会有一个独一无二的全局单一标识符，也叫通用唯一识别码（**Universally Unique Identifier**, **UUID**）。

**parted指令**
parted指令可以列出磁盘的分区表类型与分区信息。指令格式为 `parted device_name print`。

## 磁盘分区
==MBR分区表使用**fdisk**指令分区，而GPT分区表使用**gdisk**指令分区==。一般先通过lsblk和blkid指令找到磁盘，再使用parted指令查看磁盘分区信息，最后再使用fdisk或gdisk指令进行分区操作。

**gdisk新增分区槽**
在root权限下，在终端执行 **gdisk device_name** 可以开始对设备进行分区操作，比如 gdisk /dev/vda。输入 **?** 可以查看能够执行的操作。输入**p**打印已有的分区信息。输入**n**开始新增分区。最后输入**w**写入磁盘分区表。

**gdisk删除分区槽**
gdisk选择一个设备后，输入**d**可以删除一个已有的分区。删除一个分区前需要先将其从文件系统卸除。切记不要删除一个正在使用中的分区。

**partprobe更新分区信息**
使用gdisk新增或删除的分区表并不会马上更新。可以通过重启系统或者 `partprobe [-s]` 指令来更新Linux核心的分区表信息。更新完分区信息后可以使用 `lsblk device_name` 查看磁盘最新的分区状态。也可以使用 `cat /proc/partitions` 指令查看Linux核心的分区记录。

**fdisk指令**
在root权限下，在终端执行 `fdisk device_name` 可以开始对设备进行分区操作。输入 **m** 可以查看能够执行的操作。

## 磁盘格式化
分区结束后就可以进行文件系统的格式化了。对应的指令为 **mkfs**，即make filesystem。终端输入**mkfs并连按两下tab键**可以查看系统支持格式化的文件系统及对应指令。

磁盘格式化的指令格式为 `mkfs [-V] [-t 文件系统] [options] [设备名] [blocks]`。如果要把某一个分区（比如 /dev/vda5）格式化为XFS文件系统，可以使用两个指令：`mkfs -t xfs /dev/vda5` 或者 `mkfs.xfs /dev/vda5`（实际上，前一个指令也是调用了后一个指令来实现格式化的）。

```bash
[root@study ~]$ mkfs -t ext4 /dev/vda6
[root@study ~]$ mkfs.ext4 /dev/vda6
```

## 文件系统检验
系统宕机等其他原因可能导致文件系统发生错误。下面将以xfs和ext4为例介绍如何处理文件系统错误。无论是xfs_repair还是fsck.ext4指令，只有身份为root用户且文件系统发生错误时才能使用，同时被检查的分区一定不能挂载到系统上。

**xfs_repair**
xfs_repair指令可以处理XFS文件系统错误。修复时文件系统不能被挂载。指令格式为 `xfs_repair [-fnd] device_name`。其选项与参数包括：
* -f：后面接的是文件而不是一个实体设备；
* -n：单纯检查而不修改文件系统任何数据；
* -d：在**单用户维护模式**下对**根目录**进行检查与修复（风险操作！切勿随便使用！）。

**fsck.ext4**
fsck与mkfs一样是个综合指令。对于ext4文件系统，可以直接使用fsck.ext4指令。格式为 `fsck.ext4 [-pf] [-b superblock] device_name`。

## 文件系统挂载
文件系统挂载前需要确认以下几点：
* 单一文件系统不应该被重复挂载到不同的挂载点（目录）中；
* 单一目录不应该重复挂载多个文件系统；
* 作为挂载点的目录，在挂载前理论上应该都是空目录（否则目录下原有的内容会被暂时隐藏，新挂载的设备被卸载后才会重新出现）。

**mount指令**
挂载文件系统使用mount指令。指令格式包括：
```bash
[root@study ~]$ mount -a
[root@study ~]$ mount [-l]
# 其中，filesystem为文件系统名，mount_point为挂载点，device_name为设备名
# 使用UUID挂载能避免device_name重名的情况
[root@study ~]$ mount [-t filesystem] LABEL='' mount_point
[root@study ~]$ mount [-t filesystem] UUID='' mount_point
[root@study ~]$ mount [-t filesystem] device_name mount_point
```
仅输入mount指令会显示目前的挂载信息。其可选项与参数包括：
* -a：依照配置文件 **/etc/fstab** 挂载所有未挂载的磁盘；
* -l：显示Label列的信息；
* -t：指定要挂载的文件系统类型。包括xfs, ext3, ext4, reiserfs, vfat, iso9660, nfs, cifs, smbfs；
* -n：禁止将实际的挂载情况实时写入 /etc/mtab 文件（默认会实时写入）；
* -o：支持其他的额外参数，包括
  + async, sync：文件系统同步/异步写入内存（默认为异步写入）；
  + atime, noatime：是否修改文件的读取时间（默认为是）；
  + ro, rw：挂载文件系统为只读/可擦写；
  + auto, noauto：是否允许该文件系统以 mount -a 自动挂载（auto）；
  + dev, nodev：是否允许该文件系统中创建设备文件；
  + suid, nosuid：是否允许该文件系统中包含suid和sgid文件格式；
  + exec, noexec：是否允许该文件系统中包含可执行二进制文件；
  + user, nouser：是否允许该文件系统让任何用户执行mount（默认仅root用户执行mount）；
  + defaults：==默认值为 rw, suid, dev, exec, auto, nouser, atime, async==；
  + remount：重新挂载（系统出错或更新参数时使用）。

**/etc/filesystems** 文件中包含了系统指定的测试挂载文件系统类型的优先级。**/proc/filesystems** 文件中包含了Linux系统已加载的文件系统类型。**/lib/modules/$(uname -r)/kernel/fs/** 文件中记录了Linux系统支持的文件系统的驱动程序。

**挂载xfs/ext4等文件系统**
```bash
# 挂载XFS文件系统
[root@study ~]$ blkid /dev/vda4     # 查看分区的UUID
/dev/vda4: UUID="e061f-s60e-4cb7-a512-826a8bde2" TYPE="xfs"
[root@study ~]$ mount UUID="e061f-s60e-4cb7-a512-826a8bde2" /data/xfs     # 使用UUID挂载文件系统
mount: mount point /data/xfs does not exist
[root@study ~]$  mkdir -p /data/xfs     # 递归创建挂载点目录
[root@study ~]$ mount UUID="e061f-s60e-4cb7-a512-826a8bde2" /data/xfs
[root@study ~]$ df /data/xfs     # 查看磁盘使用量信息

# 挂载EXT4文件系统
[root@study ~]$ blkid /dev/vda5
/dev/vda5: UUID="e03df-5ed3-4cb7-a6b2-926a8bde" TYPE="ext4"
[root@study ~]$ mkdir /data/ext4
[root@study ~]$ mount UUID="e03df-5ed3-4cb7-a6b2-926a8bde" /data/ext4
[root@study ~]$ df /data/ext4
```

**挂载CD或DVD光盘**
光驱挂载后需要先卸载才能退出。
```bash
[root@study ~]$ blkid
/dev/sr0: UUID="2018-04-01-00-21-35-00" LABEL="CentOS 7 x86_64" TYPE="iso9660" PTTYPE="dos"
[root@study ~]$ mkdir /data/cdrom
[root@study ~]$ mount /dev/sr0 /data/cdrom
[root@study ~]$ df /data/cdrom
```

**挂载USB磁盘**
```bash
[root@study ~]$ blkid
/dev/sda1: UUID="35DC-6D6B" TYPE="vfat"
[root@study ~]$ mkdir /data/usb
# codepage=950代表繁体中文，936代表简体中文
[root@study ~]$ mount -o codepage=936,iocharset=utf8 UUID="35DC-6D6B" /data/usb
[root@study ~]$ df /data/usb
```

**重新挂载根目录**
根目录不能被卸载，但是在要修改挂载参数或者根目录出现只读状态时，可以重新挂载根目录。
```bash
[root@study ~]$ mount -o remount,rw,auto /
```
还可以使用mount指令将一个目录暂时挂载到另一个目录下。
```bash
[root@study ~]$ mkdir /data/var
[root@study ~]$ mount --bind /var /data/var
[root@study ~]$ ls -lid /var /data/var     # 属性和权限完全相同
16777346	drwxr-xr-x.	22	root	root	4096	Jun 15 23:43	/data/var
16777346	drwxr-xr-x.	22	root	root	4096	Jun 15 23:43	/var
```

**umount指令**
umount指令可以卸载已挂载的设备文件。格式为 `umount [-fnl] 设备名或挂载点`。其选项与参数包括：
* -f：强制卸载；
* -l：立刻卸载文件系统（比 -f 还强硬）；
* -n：在不更新 /etc/mtab 的情况下卸载。

```bash
[root@study ~]$ mount     # 查看挂载信息
/dev/vda4 on /data/xfs type xfs
/dev/vda5 on /data/ext4 type ext4
/dev/sr0 on /data/cdrom type iso9660
/dev/sda1 on /data/usb type vfat
/dev/mapper/centos-root on /data/var type xfs
......
[root@study ~]$ umount /dev/vda4     # 使用设备名卸载
[root@study ~]$ umount /data/ext4     # 使用挂载点卸载
[root@study ~]$ umount /data/cdrom
[root@study ~]$ umount /data/usb
[root@study ~]$ umount /data/var     # 只能使用挂载点卸载，因为对应设备还有另一个挂载点 /var
```
不要尝试在文件系统的挂载点对应目录下卸载该挂载点（会显示 target is busy）。

## 文件系统参数修改
**mknod**
回忆前文 lsblk 指令输出中的 MAJ:MIN 字段，分别被称为**主要和次要设备代码**。常见的磁盘文件名的设备代码如下：
| 磁盘文件名 | 主要设备代码（MAJ） | 次要设备代码（MIN） |
| -- | -- | -- |
| /dev/sda | 8 | 0-15 |
| /dev/sdb | 8 | 16-31 |
| /dev/loop0 | 7 | 0 |
| /dev/loop1 | 7 | 1 |

某些情况下需要使用 mknod 指令手动处理设备文件。指令格式为 `mknod 设备名 [bcp] [MAJ] [MIN]`。选项b代表磁盘一类的存储设备，选项c代表鼠标和键盘等输入设备，选项p代表FIFO文件（管道）。其中的MAJ和MIN代码需要自己查阅，不能随意设定。

**xfs_admin**
如果在格式化磁盘时没有设置Label和UUID，对于XFS文件系统，可以使用 xfs_admin指令设置，而不需要重新格式化。指令格式为 `xfs_admin [-lu] [-L Label] [-U UUID] 设备文件名`。小写选项l和u用于显示设定好的Label和UUID，而大写选项L和U用于设定Label和UUID。

```bash
[root@study ~]$ xfs_admin -L la_xfs /dev/vda4     # 设定Label
[root@study ~]$ xfs_admin -l /dev/vda4
label = "la_xfs"
[root@study ~]$ mount LABEL=la_xfs /data/xfs
[root@study ~]$ umount /dev/vda4     # 使用该指令修改参数前先卸载设备

[root@study ~]$ uuidgen     # 生成一个UUID
e0fa7252-d56e-987a-406e-3cbf41529e809
[root@study ~]$ xfs_admin -U e0fa7252-d56e-987a-406e-3cbf41529e809 /dev/vda4   # 设备名
[root@study ~]$ mount UUID=e0fa7252-d56e-987a-406e-3cbf41529e809 /data/xfs   # 挂载点
```

**tune2fs**
如果在格式化磁盘时没有设置Label和UUID，对于EXT4文件系统，可以使用 tune2fs指令设置，也不需要重新格式化。指令格式为 `tune2fs [-l] [-L Label] [-U UUID] 装置文件名`。

# 设定开机挂载
## 开机挂载 /etc/fstab 
当系统启动时，将自动读取 **/ect/fstab** 配置文件，将该文件中预先写入的文件系统加载到指定的目录。使用cat指令查看该文件可以发现文件中有六个字段（列），从左至右依次为：
* Device：文件系统或磁盘的设备名，也可以是**UUID**或者**Label**名；
* Mount point：挂载点（目录）；
* filesystem：文件系统类型；
* paramters：mount指令参数，一般是默认（defaults）；
* dump：能否使用dump指令备份；
* fsck：是否用fsck指令检验扇区。

**/ect/fstab** 是开机时的配置文件，实际的文件系统挂载信息会记录在 **/etc/mtab** 和 **/proc/mounts** 这两个文件中。

## 特殊设备loop挂载
如果有光盘镜像文件，或者使用文件作为磁盘，也可使用mount指令挂载，而不需要刻录成光盘才能使用其中的数据。

```bash
[root@study ~]$ ls -lh /tmp/CentOS-7.0-1406-x86_64-DVD.iso
-rw-r--r--.	1	root	root	3.9G	Jul 7 2014	/tmp/CentOS-7.0-1406-x86_64-DVD.iso
[root@study ~]$ mkdir /data/centos_dvd
[root@study ~]$ mount -o loop /tmp/CentOS-7.0-1406-x86_64-DVD.iso /data/centos_dvd
[root@study ~]$ df /data/centos_dvd
[root@study ~]$ ls -l /data/centos_dvd     # 挂载后可以查看镜像文件中的内容
[root@study ~]$ umount /data/centos_dvd
```

# 内存交换空间：swap
Linux系统的swap分区类似于Windows系统的虚拟内存。当物理内存不足时，拿出部分硬盘空间当SWAP分区（虚拟成内存）使用，从而解决内存容量不足的情况。
## 使用实体分区槽创建swap
建立swap分区槽可以分为以下几个步骤：
* 分区：使用gdisk在磁盘中分出一个分区槽作给swap（需要设定system ID）；
* 格式化：使用 `mkswap device_name` 格式化分区槽为swap格式；
* 使用：使用 `swapon device_name` 启用swap设备；
* 观察：使用 `free` 和 `swapon -s` 查看内存使用量。

```bash
[root@study ~]$ gdisk /dev/vda
Command (? for help): n
Last sector (69220352-83886046, default = 83886046) or {+-}size{KMGTP}: +512M     # 设定分区大小
Hex code or GUID (L to show codes, Enter = 8300): 8200     # 设定系统ID
Changed type of partition to 'Linux swap'
Command (? for help): p     # 打印分区信息
Number	Start sector	End sector	Size	Code	Name
6     	69220352    	70268927  	512MB	8200	Linux swap
Command (? for help): w     # 写入分区信息

# 更新并查看分区信息
[root@study ~]$ partprobe
[root@study ~]$ lsblk

# 格式化分区为swap
[root@study ~]$ mkswap /dev/vda6     # 注意数字6别忘了（分区槽Number）
[root@study ~]$ blkid /dev/vda6 

[root@study ~]$ free     # 观察信息
[root@study ~]$ swapon /dev/vda6     # 启用分区
[root@study ~]$ free
[root@study ~]$ swapon -s     # 观察信息
```

## 使用文件创建swap
如果没有实体分区槽，使用前文提到的loop设备方法也可以创建swap。

```bash
# 使用dd指令在/tmp目录下新增一个128M文件
[root@study ~]$ dd if=/dev/zero of=/tmp/swap bs=1M count=128
[root@study ~]$ mkswap /tmp/swap
[root@study ~]$ swapon /tmp/swap
[root@study ~]$ swapon -s
# 设定开机自启动swap
[root@study ~]$ vi /etc/fstab
/tmp/swap	swap	swap	defaults	0	0
# 需要先关闭swap
[root@study ~]$  swapoff /tmp/swap /dev/vda6
[root@study ~]$ swapon -s     # 显示swap设备信息

[root@study ~]$ swapon -a     # 自动启动所有swap设备
[root@study ~]$ swapon -s
```

References
\[1] [Linux mkfs命令](https://www.ctolib.com/docs/sfile/w3school-back-end/linux/170.html)
\[2] [让你知道codepage的重要，关于多语言编码](https://blog.csdn.net/cuoguo1111/article/details/1541860)
\[3] [Linux swap是干嘛的](https://www.cnblogs.com/pipci/p/11399250.html)
