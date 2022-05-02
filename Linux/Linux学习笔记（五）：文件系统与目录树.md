@[TOC](Linux学习笔记（五）：文件系统与目录树)
# Linux文件系统
磁盘分区完还需要格式化（format），之后操作系统才能使用其文件系统（filesystem）。格式化的原因在于==每种操作系统所设定的文件属性和权限不相同，所能利用的文件系统格式也不一样==。譬如，Windows 98之前的微软操作系统使用的文件系统是**FAT**或FAT16，Windows 2000之后的版本采用了**NTFS**文件系统。Linux的文件系统一般为**Ext2**（Linux second extended file system, **ext2fs**）。

## EXT2文件系统
Linux的文件系统包含inode、block、superblock三个重要概念：
* **inode**：一个文件占用一个inode，存储文件的权限与属性、文件数据所在的block号码；
* **block**：存储文件数据的内容，文件过大时会占用多个block；
* **superblock**：存储文件系统的整体信息，包括inode/block总量、使用量、剩余量、文件系统的格式有其他信息。

**block group**
为了便于管理，Ext2文件系统在格式化时会划分多个**区块群组**（**block group**），==每个区块群组都有独立的 **inode table**、data block 和 superblock==。整个文件系统最前面有一个启动扇区（boot sector），用于安装开机管理程序。

**block**
block的大小会在文件系统格式化时固定，并且会对文件系统能够支持的最大单一文件容量与最大磁盘容量产生影响。
| Block大小 | 1KB | 2KB | 4KB |
|--|--|--|--|
| 最大单一文件限制 | 16G | 256G | 2TB |
| 最大文件系统总容量 | 2TB | 8TB | 16TB |

block的其他限制还包括：
* 原则上，block的大小与数目只能通过格式化磁盘修改；
* 每个block内只能放置**最多一个**文件的数据，也就是说，文件小于block时会造成磁盘空间浪费；
* 一个文件过大时会占用**多个**block。

**inode**
inode记录的内容至少包括：
* 文件的存取模式（read/write/execute）；
* 文件的拥有者与群组（owner/group）；
* 文件的容量大小；
* 文件创建或状态改变的时间（ctime）；
* 最近一次的读取时间（atime）；
* 最近修改的时间（mtime）；
* 文件特性标志（flag），比如SetUID；
* 文件真正的内容指向（pointer）。

inode的大小与数目也在格式化后固定不变。其限制包括：
* 大小固定为**128bytes**（某些文件系统可设置为**256bytes**）；
* 每个文件仅仅会占用**一个inode**。因此文件系统能够创建的文件数量与inode的数量有关；
* 系统读取文件时需要先找到inode，分析inode中记载的权限与当前用户是否符合，才能决定是否读取block中的数据。

inode记录一个block编号需要**4bytes**容量。为了尽可能增加能够记录的编号数目，==inode将纪录block编号的区域划分为12个直接、1个间接、1个双间接、和1个三间接记录区==。

假定格式化时设定block大小为**1K**（即1024bytes），每个记录区能够指向的block容量为：
* 12个直接记录区：一共可以记录12个block编号，block总容量为 $12\times1K=12K$；
* 1个间接记录区：即用一个block来记录编号，一共可以记录256个block编号（1024bytes / 4bytes = 256），block总容量为 $256\times1K=256K$；
* 1个双间接记录区：即用两层block来记录编号，一共可以记录 $256\times256$ 个block编号，block总容量为 $256\times256\times1K=256^2K$；
* 1个三间接记录区：即用三层block来记录编号，一共可以记录$256^3$个block编号，block总容量为 $256\times256\times256\times1K=256^3K$。

综上，当block的大小为1K时，一个inode可以记录的最大的文件容量为 $12+256+256^2+256^3(K)=16G$。这正好与上文的表格中block大小对最大单一文件容量的限制一致。

**block group中的其它内容**
除了inode、block、superblock以外，每个区块群组中还有的部分包括：
* 文件系统描述（filesystem description）：描述每个区块群组开始与结束的block编号；
* **区块对照表**（**block bitmap**）：记录可以被使用（空的block）或者被释放（记录的文件数据被删除）的block编号；
* **inode对照表**（**inode bitmap**）：与区块对照表类似，纪录未被使用或者已经被使用的inode。

**dumpe2fs**
dumpe2fs指令可以查看Ext文件系统superblock中的详细信息。

## 文件系统与目录树
**目录**
在Linux系统下建立一个目录时，文件系统会分配一个inode与至少一个block给该目录。其中，inode纪录该目录的权限与属性、以及分配给目录的block的编号；而目录分配到的block中记录了该目录下的文件名以及文件名对应的inode号码。使用指令 **ls -i** 可以查看目录下文件的inode。

**目录树读取**
由于目录树是从根目录开始读起的，系统通过挂载信息可以得到挂载点的inode，也就能得到根目录的inode，从而得到根目录的block内的文件名及对应文件的inode。

**挂载点**
每个文件系统都有独立的inode/block/superblock信息，并且要被被链接到目录树才能被用户使用。将文件系统与目录树结合的动作被称为**挂载**。**挂载点一定是目录**，该目录为进入该文件系统的入口。

==**同一个文件系统**的某个inode只会对应到一个文件内容==，因此可以使用inode号码判断不同文件名是否为相同的文件。
```bash
[root@study ~]$ ls -lid / /. /..
128	dr-xr-xr-x.	17	root	root	4096	May 4 17:56	/
128	dr-xr-xr-x.	17	root	root	4096	May 4 17:56	/.
128	dr-xr-xr-x.	17	root	root	4096	May 4 17:56	/..
```
假设 / 、 /boot、/home分别为三个不同的挂载点（XFS文件系统的最顶层目录的inode号码一般为128）。因为是三个不同的文件系统，所以三个目录的inode号码可以相同，但是inode中记录的权限和属性信息通常并不相同。
```bash
[root@study ~]$ ls -lid / /boot /home
128	dr-xr-xr-x.	17	root	root	4096	May  4  17:56	/
128	dr-xr-xr-x.	4	root	root	4096	May  4 17:59	/boot
128	drwxr-xr-x.	5	root	root	  41	Jun 17 00:20	/home
```

## 元数据与日志式文件系统
**文件存取**
在Ext2/Ext3/Ext4文件系统下，新增一个文件时，文件系统：
* 先确定当前用户对目录是否有w和x权限；
* 根据inode bitmap找到未使用的inode号码，将新文件的权限和属性写入；
* 根据block bitmap找到未使用的block号码，写入新文件的数据，并更新inode中的block指向数据；
* 根据刚写入的inode和block更新inode bitmap和block bitmap，并更新superblock的内容。

通常将inode table和data block称为数据存放区域，而==将inode bitmap、block bitmap和superblock称为**metadata**（中介数据、**元数据**、data about data）==，因为这三个地方的数据经常变动而且互相影响。

**日志式文件系统**
如果在数据写入inode table和data block后发生了系统中断，导致metadata没有被更新，就会发生元数据与实际存储数据不一致的情况。日志式文件系统（**journaling filesystem**）旨在解决这种问题。EXT3和EXT4文件系统中会专门划出一个区块纪录写入或修改文件的步骤。在写入文件前，先在日志记录区块中记录文件准备要写入的信息；实际写入后，在日志记录区块中完成该文件的记录。一旦发生问题，就可以根据日志记录快速修复文件系统。

## 文件系统异步处理
如果用户在处理一个大文件，在编辑文件的同时需要系统频繁写入磁盘，由于磁盘写入的速度比内存慢很多，就会导致效率低下。Linux使用**异步处理**（asynchronously）来解决这个问题。

当一个文件被加载到内存后，如果该文件未被修改，则内存中的文件数据会被认定为干净的（**clean**）。如果内存中的文件数据被修改了，此时内存中的这部分数据就会被设定为脏的（**dirty**）。==系统会不定时将内存中脏的数据写回磁盘，以保持磁盘文件与内存中文件数据的一致性==。我们也可以使用 **sync** 指令手动强迫写入磁盘。

## 其他Linux文件系统
Linux的标准文件系统是EXT2，以及增加了日志功能的EXT3和EXT4。Linux支持的文件系统可以分为以下三类：
* 传统文件系统：ext2, minix, MS-DOS, FAT (vfat), iso9660（光盘）；
* 日志式文件系统：ext3, ext4, ReiserFS, Windows' NTFS, IBM's JFS, SGI's XFS, ZFS；
* 网络文件系统：NFS, SMBFS。

Linux使用 Virtual Filesystem Switch (**VFS**) 管理系统能够识别的所有文件系统。

下列指令可用于查看系统支持的文件类型。

```bash
ls -l /lib/modules/$(uname -r)/kernel/fs   # 查看Linux支持的文件系统
cat /proc/filesystems     # 查看已加载到内存中支持的文件系统
```

# 文件系统操作
## 目录的磁盘使用量
**df指令**
df指令可以列出文件系统的整体磁盘使用量。指令格式为 `df [-ahikHTm] [目录或文件名]`。其选项与参数包括：
* -a：列出所有的文件系统，包括系统特有的/proc文件系统；
* -k：以Kbytes为单位显示容量；
* -m：以Mbytes为单位显示容量；
* -h：以Gbytes、Mbytes、Kbytes等格式自行显示（常用）；
* -H：以M=1000K取代M=1024K的进位方式；
* -T：连同该分区的文件系统名称（Type）也列出；
* -i：不用磁盘容量，而以inode数量的方式显示（常用）。

df后不加任何参数会默认将**系统内所有文件系统**（不包括特殊内存文件系统与swap）以1Kbytes的格式列出。

**du指令**
df指令可以列出文件系统的整体磁盘使用量。指令格式为 `du [-ahskm] [目录或文件名]`。其选项与参数包括：
* -a：列出所有的文件与目录容量（默认统计目录下的文件量）；
* -s：列出总量，而不是各个目录占用的容量（常用）；
* -k：以Kbytes为单位显示容量；
* -m：以Mbytes为单位显示容量；
* -h：以Gbytes、Mbytes等格式自行显示。

du后不加任何参数会默认将**当前所在目录下**的文件与子目录（但是通常只显示子目录容量）以1Kbytes的格式列出。

## 实体链接与符号链接
Linux下存在两种链接文件，一种是**实体链接**（hard link），另一种是**符号链接**（symbolic link）。

**hard link**
实体链接也被称为**硬链接**。我们已经知道，文件名只与目录有关，目录的block记录了文件名对应的inode，而inode指向了存储文件数据的data block。简单地说，硬链接只是在某个目录下新增了一个文件名链接到某个inode的关联记录。因此==删除硬链接或者源文件后，对应文件内容的inode和block依然存在==。硬链接存在两个重要的局限性：
* （1）不能跨文件系统；
* （2）不能给目录创建硬链接。

指令ln可以创建链接文件。指令格式为 `ln [-bdinsSvV] 源文件 目标文件`。不使用参数 **-s** 时创建的是硬链接。

**symbolic link**
符号链接也称为**软链接**，可以被看做一种快捷方式。符号链接即创建一个独立文件，该文件会让数据的读取指向它链接的文件名。==源文件与符号链接文件的inode号码不相同==。因此，==当源文件被删除后，符号链接文件就会打不开==（因为找不到链接的文件名）。

ln指令使用参数 **-s** 时创建的是符号链接。可以给目录创建符号链接。
```bash
[root@study ~]$ ls -li /etc/contab
34474855	-rw-r--r--.	1	root	root	451	Jun 10 2014	/etc/contab
[root@study ~]$ ln /etc/contab .     # create hard link
[root@study ~]$ ls -li /etc/crontab crontab     
# inode相同，文件属性和权限相同，文件链接数都为2，仅文件名不同
34474855	-rw-r--r--.	2	root	root	451	Jun 10 2014	/etc/contab
34474855	-rw-r--r--.	2	root	root	451	Jun 10 2014	contab

[root@study ~]$ ln -s /etc/contab crontab2     # create symbolic link
[root@study ~]$ ls -li /etc/crontab /root/crontab2     
# inode不同，文件属性和权限不同，文件链接数不同
# 软链接的大小12 bytes为指向的文件名/etc/crontab的大小
34474855	-rw-r--r--.	2	root	root	451	Jun 10 2014	/etc/contab
53745909	lrwxrwxrwx.	1	root	root	 12	Jun 23 2014	/root/contab2 -> /etc/crontab
```
不管是对硬链接还是软链接文件的内容进行修改，源文件的内容都会随之更新。但是使用 rm 指令删除链接文件本身并不会把源文件一同删除。

**目录的链接数量**
当我们在目录/tmp下新建一个子目录/test时，新目录/tmp/test的链接数等于2，上层目录/tmp的链接数会增加1。这是因为新建的子目录下默认存在 **/tmp/test/.** 和 **/tmp/test/.\.** 两个链接文件，分别指向子目录本身和上层目录。

