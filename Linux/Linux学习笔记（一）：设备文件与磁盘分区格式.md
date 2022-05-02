@[TOC](Linux学习笔记（一）：设备文件与磁盘分区格式)
# 设备与文件名
在Linux中，设备（**device**）都被当成文件来对待。
| 设备 | Linux文件名  |
| :-- | :-- |
| SCSI/SATA/USB 硬盘 | /dev/sd[a-p] |
| USB快闪碟 | /dev/sd[a-p]  |
| VirtI/O接口 | /dev/vd[a-p]，用于虚拟磁盘  |
| 软盘驱动器 | /dev/fd[0-7] |
| 打印机 | /dev/lp[0-2]，用于25针打印机<br> /dev/usd/lb[0-15]，用于USB接口 |
| 鼠标 | /dev/input/mouse[0-15]，通用<br> /dev/psaux，用于PS/2接口<br> /dev/mouse，当前鼠标 |
| CDROM/DVDROM | /dev/scd[0-1]，通用<br> /dev/sr[0-1]，通用但CentOS较常见<br> /dev/cdrom，当前CDROM |
| 磁带 | /dev/ht0，IDE接口 <br> /dev/st0，SATA/SCSI接口 <br> /dev/tape，当前磁带  | 

**Question**：如果个人PC上有两个SATA磁盘和一个USB磁盘，主板上有六个SATA插槽。两个SATA磁盘分别插在主板的SATA1和SATA5插槽上。请问这三个磁盘在Linux系统中的文件名分别是什么？
**Answer**：SATA1插槽上的磁盘：/dev/sda；SATA5插槽上的磁盘：/dev/sdb；USB磁盘（开机后才能被系统捕捉到）：/dev/sdc。

# 磁盘分区格式
磁盘的一个**盘面**可以划分为不同的**扇区**（**sector**）和**磁道**（**track**）。扇区是最小的物理存储单位。每个磁道包含**相同数目**的扇区。一个盘面包含多条磁道。每个盘面对应一个**磁头**（**head**）。磁盘不同盘面的相同半径的磁道构成了一个**磁柱**（**cylinder**）。磁柱是文件系统的最小单位，也是分区槽的最小单位（目前有的系统的最小分区单位已经改成扇区）。$$存储容量 = 磁头数 \times 磁道（磁柱）数 \times 每道扇区数 \times 每扇区字节数$$

磁盘的分区格式主要分为**MBR**（Master Boot Record）和**GPT**（GUID Partition Table）两种。==大于**2TB**的磁盘只能采用GPT分区格式==。

## MBR分区表格式
MBR分区格式兼容Windows磁盘，开机管理程序记录区和分区表都放在磁盘的第一个扇区。改扇区大小通常为512bytes，包括以下两个数据：
* **主要启动纪录区**（Master Boot Record, MBR）：安装开机管理程序的地方，有446bytes；
* **分区表**（partition table）：纪录整块磁盘分区的状态，有64bytes。

MBR分区表最多仅能有**四组**记录区，每组记录了该区段的起始与结束的磁柱号码。这四个分区被称为**主要分区槽**（primary partition）或者**延伸分区槽**（extended partition）。如果该硬盘设备的文件名为/dev/sda，MBR分区表中记录的四个分区槽的Linux文件名分别为：/dev/sda1，/dev/sda2，/dev/sda3，/dev/sda4。如果操作系统为Windows，四个分区槽的代号分别为C、D、E、F。

==主要分区和延伸分区一共最多只能有四个，而且**延伸分区最多只能有一个**==。但是可以通过延伸分区将硬盘分区成四个以上的分区。延伸分区使用额外的扇区来记录分区信息。由延伸分区继续切分出来的分区槽被称为**逻辑分区槽**（logical partition）。

假设一个硬盘设备被分为一个主分区和一个延伸分区，而延伸分区又被继续分为五个逻辑分区，那么这七个分区的Linux文件名分别为：/dev/sda1，/dev/sda2，**/dev/sda5**，/dev/sda6，/dev/sda7，/dev/sda8，/dev/sda9。逻辑分区的编号从5开始，因为编号1、2、3、4被保留给了主分区和延伸分区。

最后，**延伸分区无法格式化**，但是主分区和逻辑分区可以被格式化。

## GPT分区表格式
GPT使用**逻辑区块地址**（Logical Block Address, LBA）来处理扇区的定义，以兼容所有磁盘。每个LBA大小预设为**512bytes**。GPT使用了**34**个LBA区块（LBA0至LBA33）记录分区信息。

其中：
* **LBA0**为MBR相容区块，包含了446bytes的开机管理程序和一个GPT分区格式标志；
* **LBA1**为GPT表头记录，记录了分区表本身的位置和大小、备份用的GPT分区位置（最后34个LBA区块）、分区表的检验机制码（用于判断GPT是否正确、是否需要从备份区块位置恢复）；
* **LBA2-33**为实际纪录分区信息处。每个LBA都可以记录4笔分区信息，总共可以有128笔分区记录。每笔纪录用到128bytes空间，其中64bits用来记录开始/结束的扇区号码。对于单一分区槽而言，其最大容量限制为$2^{64}\times512bytes = 2^{63}KB = 2^{33}TB$。

## BIOS与UEFI开机检测程序
主机系统在加载硬件驱动方面的程序主要包括BIOS（Basic Input Output System）和UEFI（Unified Extensible Firmware Interface，统一可扩展固件接口）两种机制。

**BIOS搭配MBR/GPT的开机流程**
BIOS是写入到主板上的一个**固件**（**Firmware**）。BIOS会依照设定去识别第一个可开机的存储设备，比如硬盘，然后到该硬盘里读取第一个扇区的MBR位置。MBR内含的**开机管理程序**（**boot loader**）能够读取核心文件，从而使得操作系统开始工作。BIOS也能从GPT分区的LBA0区块读取开机管理程序。

开机管理程序除了可以安装到MBR以外，也可以安装到每个分区槽的**启动扇区**（boot sector），从而实现**多重引导**。如果要安装多重引导，最好先安装Windows再安装Linux，因为Windows的安装程序会主动覆盖MBR以及自己所在分区槽的启动扇区。

**UEFI搭配GPT的开机流程**
BIOS要通过GPT提供兼容模式才能读取磁盘设备。UEFI由C语言开发而成，意在取代BIOS，其本质类似一个低阶的操作系统。

## Linux安装模式下的磁盘分区
**文件系统与目录树的关系：挂载**
**挂载**就是把一个目录当成进入点，将磁盘分区槽的数据放置在该目录下。也就是说，进入该目录就可以读取该分区槽。进入点的目录称为**挂载点**。


References:
[1] [关于磁盘，磁柱，磁头，扇区等](https://blog.csdn.net/varyall/article/details/83018187)
[2] [硬盘基本知识（磁道、扇区、柱面、磁头数、簇、MBR、DBR）](https://blog.csdn.net/fyfcauc/article/details/39576065?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)
