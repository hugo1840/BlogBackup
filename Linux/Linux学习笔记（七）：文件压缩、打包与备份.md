@[TOC](Linux学习笔记（七）：文件压缩、打包与备份)
# 常见的压缩指令
## gzip, zcat/zmore/zless/zgrep
gzip指令创建的压缩文件后缀为 **.gz**。gzip同时也可以解压缩由 compress, zip, gzip 等软件压缩的文件。压缩的命令格式为 `gzip [-cdtv#] filename`。其选项与参数包括：
* -c：将压缩后的数据输出到标准输出，并保留原有文件；
* -d：用于**解压**文件（相当于gunzip）；
* -t：检验压缩文件的一致性（看看文件有无错误）；
* -v：显示源文件与压缩文件的压缩比等信息；
* -#：#为数字，代表压缩等级，-1最快但是压缩比最差，-9最慢但是压缩比最快。默认为 **-6**。

默认情况下，gzip将源文件压缩后，源文件就不存在了。如果被压缩的源文件是文本文件，则可以使用 zcat/zmore/zless 指令读取压缩文件。还可以使用 zgrep 指令从压缩文件中查找数据。

```bash
[fang@study ~]$ cd /tmp
[fang@study tmp]$ cp /etc/services .
# 压缩文件
[fang@study tmp]$ gzip -v services
services:	79.7%  -- replaced with services.gz
# 查看压缩文件
[fang@study tmp]$ zcat services.gz
# 解压文件
[fang@study tmp]$ gzip -d services.gz
# 以最佳压缩比压缩，并保留原文件
[fang@study tmp]$  gzip -9 -c services > services.gz
# 在压缩文件中查找关键词
[fang@study tmp]$ zgrep -n 'http' services.gz
```

## bzip2, bzcat/bzmore/bzless/bzgrep
bzip2的目的在于取代gzip并提供更好的压缩比。bzip2指令创建的压缩文件后缀为 **.bz2**。压缩的命令格式为 `bzip2 [-cdkzv#] filename`。其选项与参数包括：
* -c：将压缩后的数据输出到标准输出；
* -d：解压缩；
* -k：保留原文件；
* -z：压缩（默认值）；
* -v：同gzip；
* -#：同gzip。

bzcat/bzmore/bzless/bzgrep与gzip的情况类似。

## xz, xzcat/xzmore/xzless/xzgrep
xz的压缩比相对gzip和bzip2更高，用法也相似。其缺点是压缩时间更长。xz指令创建的压缩文件后缀为 **.xz**。压缩的命令格式为 `xz [-dtlkc#] filename`。其选项与参数包括：
* -d：解压缩；
* -t：测试压缩文件的完整性（是否有错误）；
* -l：列出压缩文件的相关信息；
* -k：保留原文件；
* -c：将压缩后的数据输出到标准输出；
* -#：同gzip和bzip2。

# 打包指令：tar
前面提到的gzip/bzip2/xz只能针对单一文件进行压缩，如果需要将多个文件或目录整合成一个大文件，就需要用到打包指令tar。其指令格式包括：

```bash
tar [-z|-j|-J] [cv] [-f 待创建的文件名] filename...   # 打包与压缩
tar [-z|-j|-J] [tv] [-f 已有的tar文件名]     # 查看文件名
tar [-z|-j|-J] [xv] [-f 已有的tar文件名] [-C 目录]   # 解压缩
```

其选项与参数包括：
* -c：创建打包文件（compress）；
* -t：查看打包文件的内容；
* -x：解压缩。注意==c, t, x不能同时出现==；
* -z：利用gzip压缩或解压缩，建议文件名后缀为 **.tar.gz**；
* -j：利用bzip2压缩或解压缩，建议文件名后缀为 **.tar.bz2**；
* -J：利用xz压缩或解压缩，建议文件名后缀为 **.tar.xz**；
* -v：显示压缩或解压过程中正在处理的文件名；
* -f：后面直接放要处理的文件名；
* -C：后面接解压缩的文件放置的目录；
* -p：保留备份数据的原本权限与属性；
* -P：保留绝对路径（谨慎！**还原时会覆盖现有文件**）；
* --exclude=FILE：不要将FILE文件打包。

使用tar打包备份文件时，会移除绝对路径中的根目录标志 “/”，这样做是出于安全考虑。==如果没有移除根目录标志，解压缩后得到的文件名是绝对路径，就会覆盖原有路径下已经存在的文件==，从而导致错误和安全问题。

```bash
[fang@study ~]$ su -
[root@study ~]$ time tar -zpcv -f /root/etc.tar.gz /etc
tar: Removing leading '/' from member names
real	0m0.799s     # time指令显示其后程序的运行时长
[root@study ~]$ time tar -jpcv -f /root/etc.tar.bz2 /etc
real	0m1.913s
[root@study ~]$ time tar -Jpcv -f /root/etc.tar.xz /etc
real	0m9.023s
```

**解压缩**
tar解压缩时默认在当前工作目录放置解压后的文件。使用参数 **-C** 可以指定解压缩后文件的位置。如果知道压缩包里的文件名，也可以单独解压缩其中的单一文件。指令格式为 `tar -jxv -f 打包文件名 待解压文件名`。

```bash
[root@study ~]$ tar -jxv -f /root/etc.tar.bz2 -C /tmp   # /tmp目录下会出现子目录/etc
[root@study ~]$ tar -jtv -f /root/etc.tar.bz2 | grep 'shadow'   # 利用grep指令查找文件名
----------	root/root	721	2015-06-17 00:20	etc/shadow
[root@study ~]$ tar -jxv -f /root/etc.tar.bz2 etc/shadow     # 注意不是 /etc/shadow
```

**部分打包**
假设我们想打包 /etc和 /root两个目录，但是不想打包 /root/etc*开头的目录，打包后的文件要放在 /root下，路径为 /root/system.tar.bz2，可以使用 **\-\-exclude** 选项来实现。

```bash
[root@study ~]$ tar -jcv -f /root/system.tar.bz2 --exclude=/root/etc* \     # 符号\用于bash换行
> --exclude=/root/system.tar.bz2 /etc /root
```

**部分备份**
使用 **-\-newer** 或 **-\-newer-mtime** 可以备份比某个时刻更新的文件。

```bash
[root@study ~]$ find /etc -newer /etc/passwd
[root@study ~]$ ls -l /etc/passwd     # 查看/etc/passwd的mtime
-rw-r--r--.	root	root	2092	Jun 17 00:20	/etc/passwd
[root@study ~]$ tar -jcv -f /root/etc.newer.than.passwd.tar.bz2 --newer-mtime="2015/06/17" /etc/*
# 列出压缩包内不以/结尾的文件
[root@study ~]$ tar -jtv -f /root/etc.newer.than.passwd.tar.bz2 | grep -v '/$'   
```

# XFS文件系统备份与还原
XFS是CentOS 7的默认文件系统。
## XFS备份：xfsdump
xfsdump既可以对xfs文件系统进行完整备份，也可以用于在初次备份后进行增量备份。备份的记录文件存放在 /var/lib/xfsdump/inventory 中。需要注意的是：
* xfsdump只能备份**已挂载**的文件系统；
* 使用xfsdump必须具备root权限；
* 只能备份xfs文件系统；
* 备份后的数据只能用xfsrestore还原；
* xfsdump通过**UUID**来识别备份文档，因此不能备份两个具有相同UUID的文件系统。

备份命令格式为 `xfsdump [-L S_label] [-M M_label] [-l #] [-f 备份文档] 待备份资料`。其选项与参数包括：
* -L：S_label用于记录每次备份时对文件系统的简易说明；
* -M：M_label用于纪录每次备份时对存储介质的简易说明；
* -l：L的小写（level），#为0\~9的数字，默认为0，表示**完整备份**，1\~9表示不同level的**增量备份**；
* -f：后面接存储备份的文件名或者设备名；
* -I：大写的 i (inventory)，从 /var/lib/xfsdump/inventory 列出目前的备份信息（指令格式为xfsdump -I）。

## XFS还原：xfsrestore
xfsdump的恢复使用的是xfsrestore指令。

```bash
xfsrestore -I	# 查看备份文件
xfsrestore [-f 备份文件] [-L S_label] [-s] 待恢复目录	# 单一文件全系统恢复
xfsrestore [-f 备份文件] -r 待恢复目录	# 通过增量备份文件恢复系统
xfsrestore [-f 备份文件] -i 待恢复目录	# 进入交互模式
```
其选项与参数包括：
* -I：大写的 i (inventory)，与xfsdump相同，用于查询备份数据；
* -f：后面接存储备份的文件名或者设备名；
* -L：后面接用-I查询到的**session label**；
* -s：后面接目录名，用于恢复某个文件或目录；
* -r：如果是一个磁带内有多个文件，需要此选项来完成增量恢复；
* -i：进入交互模式，适用于高级管理员，一般不会用到。

```bash
[root@study ~]$ df -h /boot
Filesystem	...	Mounted on
/dev/vda2	...	/boot     # 独立的文件系统，挂载于/boot
# 完整备份
[root@study ~]$ xfsdump -l 0 -L boot_all -M boot_all -f /srv/boot.dump /boot
...
[root@study ~]$ ls -l /srv/boot.dump     # 查看备份文件属性
[root@study ~]$ ls /var/lib/xfsdump/inventory
...
[root@study ~]$ xfsdump -I     # 查看备份信息
file system 0:
	fs id: ..........
	session 0:
		mount point: study.centos.fang:/boot
		device: study.centos.fang:/dev/vda2
		session label: "boot_all"
		level: 0
		......
xfsdump: Dump status: SUCCESS
# 增量备份
[root@study ~]$ dd if=/dev/zero of=/boot/testing.img bs=1M count=10   # 使用dd制作一个10M大小的文件
[root@study ~]$ xfsdump -l 1 -L boot_l1 -M boot_l1 -f /srv/boot.dump1 /boot
...
[root@study ~]$ ls -l /srv/boot*
-rw-r--r--.	1	root	root	102872168	Jul 18:43	/srv/boot.dump
-rw-r--r--.	1	root	root	 10510952	Jul 18:45	/srv/boot.dump1
[root@study ~]$ xfsdump -I
filesystem 0:
	fs id: ..........
	session 0:
		..........
	session 1:
		mount point: study.centos.fang:/boot
		device: study.centos.fang:/dev/vda2
		session label: "boot_l1"
		level: 1
		......

# 还原备份
[root@study ~]$ xfsrestore -I     # 查看备份信息，找出备份文件对应的挂载点和session label
...
# 还原level 0备份
# 方法一：直接覆盖已有文件（谨慎！）
[root@study ~]$ xfsrestore -f /srv/boot.dump -L boot_all /boot
# 方法二：还原到特定目录下
[root@study ~]$ mkdir /tmp/boot
[root@study ~]$ xfsrestore -f /srv/boot.dump -L boot_all /tmp/boot
# 仅还原备份中的单个文件grub2
[root@study ~]$ xfsrestore -f /srv/boot.dump -L boot_all -s grub2 /tmp/boot

# 还原level 1备份（需要先还原level 0备份）
[root@study ~]$ xfsrestore -f /srv/boot.dump1 /tmp/boot
```

# 光盘写入工具
## 创建镜像文件：mkisofs
制作一般数据光盘镜像文件的指令为mkisofs。命令格式为 `mkisofs [-o 镜像文件] [-Jrv] [-V vol] [-m file] 待备份文件`。选项与参数包括：
* -o：备份要生成的镜像文件；
* -J：产生较兼容Windows的文件名结构；
* -r：产生支持UNIX/Linux的文件数据，可记录较多信息；
* -v：显示创建iso文件的过程信息；
* -V vol：建立volume，类似Windows资源管理器中的CD卷标；
* -m file：排除文件file，不备份到镜像中，可用通配符*；
* -graft-point：定义镜像文件中的目录。

## 刻录光盘：cdrecord
CentOS 7使用**wodim**指令刻录光盘，兼容旧版的cdrecord指令。

```bash
wodim --devices dev=/dev/sr0 ...	# 查询刻录机的bus位置
wodim -v dev=/dev/sr0 blank=[fast|all]	# 抹除重复读写盘
wodim -v dev=/dev/sr0 -format	# 格式化DVD+RW
wodim -v dev=/dev/sr0 [可选选项功能] file.iso	
```
其选项与参数包括：
* -\-devices：扫描磁盘总线并找出可用的刻录机；
* -v：显示运行过程信息；
* dev=/dev/sr0：可以找出此光驱的bus地址；
* blank=[fast|all]：blank为抹除可重复写入的CD/DVD-RW，使用fast较快，all较完整；
* -format：对光盘进行格式化，仅支持DVD+RW；
* 可选 -data：指定后面的文件以数据格式写入；
* 可选 speed=X：指定刻录速度；
* 可选 -eject：刻录完毕后自动推出光盘；
* 可选 fs=Ym：指定缓冲内存大小，默认为4m。

# 其他压缩与备份工具
## dd指令
dd指令可以读取磁盘设备内容（几乎是直接读取扇区），然后将整个设备备份成一个文件（包括没有用到的扇区，不论是否能识别文件系统）。
```bash
dd if="input_file" of="output_file" bs="block_size" count="number"
```
其选项与参数包括：
* if：输入文件或设备名；
* of：输出文件或设备名；
* bs：读取block的大小，默认为一个扇区，即512bytes；
* count：读取bs的数量。

dd可以在旧硬盘分区上将扇区的数据全部复制过来，包括superblock、启动扇区、元数据等等。指令 **dd if=/dev/sda of=/dev/sdb** 可以创建两块一模一样的磁盘。/dev/sdb 甚至不需要分区和格式化。

## cpio指令
cpio可以备份包括设备文件在内的任何东西，但是需要先配合find等查找文件的指令。

```bash
cpio -ovcB > [file|device]	# 备份
cpio -ivcdu < [file|device]	# 还原
cpio -ivct > [file|device]	# 查看
```
其选项与参数包括：
* -o：备份的输出文件或设备；
* -B：备份默认的block可以增加到5120bytes，默认为512bytes；
* -i：从文件或设备中还原数据到系统；
* -d：还原时自动建立目录；
* -u：还原时用较新的文件覆盖旧的版本（update）；
* -t：配合-i选项，可以查看使用cpio指令建立的文件或设备的内容；
* -v：显示备份或还原的过程信息；
* -c：以较新的portable format方式存储。

```bash
# 将/boot下的所有文件备份到/tmp/boot.cpio
[root@study ~]$ cd /
[root@study /]$ find boot -print	# 使用 find /boot 会导致cpio备份绝对路径，还原时会覆盖已有文件（危险！）
...
[root@study /]$ find boot | cpio -ocvB > /tmp/boot.cpio
[root@study /]$ ls -lh /tmp/boot.cpio
```

