@[TOC](Linux学习笔记（十二）：进程与进程管理)

在Linux系统中，触发任何一个事件时，系统都会将其定义为一个**进程**（**Process**），并赋予一个进程ID，即**PID**。同时，根据触发这个进程的用户与相关属性关系，给予这个PID一组有效的权限设置。

# 进程是什么
**进程与程序**
**程序**（**Program**）通常为binary program，放在存储介质中（硬盘、光盘、软盘等），以文件的形式存在；程序被触发后，执行者的权限与属性、程序的代码与所需数据都会被加载到**内存**中，操作系统给予这个内存内的单元一个进程标识符（PID）。可以说，**进程**就是一个正在运行中的程序。

**子进程与父进程**
由一个进程中的指令执行后衍生出来的进程被称为前者的一个子进程，前者也被称为是该子进程的父进程。子进程的Parent ID （**PPID**）即是其父进程的PID。父进程通过 **fork-and-exec** 的方法执行子进程。

**服务（daemon）**
Linux系统中有很多常驻在内存中的进程，通常都是负责一些系统提供的功能以服务用户的各项任务，这些常驻在内存中的程序（进程）被称之为服务（daemon）。某些负责网络联机的服务，比如apache、vsftpd等，在运行时都会启动一个负责网络监听的端口（port），以满足外部客户端（client）的连接要求。

**多人多任务环境**
多人环境下，每个用户登录后取得的shell的PID都不同。同时，CPU在不同进程之间的快速切换为多任务提供了支持。

# 任务管理
任务管理是指当用户登录取得bash shell之后，对在**单一终端**接口下同时进行多个工作任务的管理行为。虽然用户也可以同时登录多个终端来分别执行这些任务，但是一方面频繁切换终端比较麻烦，而且另一方面单个用户同时允许登录的最大终端数目也受限于在 `/etc/security/limits.conf`文件的设定值。

假设我们只有一个终端接口，可以出现提示符让用户操作的环境被称为前台（**foreground**），其他任务则被放入后台（**background**）中暂停（**stoppped**）或运行（**running**）。处于后台中的任务无法使用 **Ctrl+z**终止运行。

## job control
**将指令放到后台中运行：&**

```bash
# 在指令后加上&将其放到后台中运行，bash会为其分配一个job number（方括号中的数字）
[root@study ~]$ tar -zpcvf /tmp/etc.tar.gz /etc &
[1] 14432	# [job number] PID
# 由于没有进行数据流重导向，指令的输出内容会显示在终端
[root@study ~]$ tar: Removing leading '/' from memebr names
......
# 后台中的指令执行完成后会在终端中显示Done
[1]+ Done		tar -zpcvf /tmp/etc.tar.gz /etc 

# 为了不影响终端的前台任务，可以将后台任务的输出重定向
[root@study ~]$ tar -zpcvf /tmp/etc.tar.gz /etc >> /tmp/log.txt &  # 覆盖log.txt已有的内容
# 将STDOUT和STDERR同时定向到log.txt并且在原有内容后添加（不覆盖）
[root@study ~]$ tar -zpcvf /tmp/etc.tar.gz /etc &> /tmp/log.txt &  
```

**将指令放到后台中暂停：Ctrl+z**
使用 **Ctrl+z** 即可将前台中正在运行的任务暂停并放到后台中。即**挂起**进程，与 **Ctrl+c** 不同（终止进程）。

**查看后台中的任务状态：jobs**

```bash
[root@study ~]$ jobs [-lrs]
# -l：除了job number和执行的指令外，同时列出PID
# -r：仅列出在后台中running的任务；
# -s：仅列出在后台中stopped的任务。
[root@study ~]$ job -l
[1]-  14566 Stopped	vim ~/.bahsrc  # 减号表示最近倒数第二个被放到后台中的任务
[2]+  14567 Stopped	find / -print  # 加号表示最近被放到后台中的任务
```

**将后台中的任务取回到前台：fg**

```bash
[root@study ~]$ fg %job_number  # %可省略
# 使用fg指令前先使用jobs指令查看job number
```

**让后台中暂停的任务恢复运行：bg**

```bash
[root@study ~]$ bg %job_number 
# 配合使用jobs指令查看暂停任务的job number
```

**管理后台中的任务：kill**

```bash
[root@study ~]$ kill -signal %job_number
[root@study ~]$ kill -signal PID	# 后面将介绍
[root@study ~]$ kill -l
# -l：（小写L）列出目前kill可以使用的signal
# 使用man 7 signal可以查询signal信息，常用的有如下：
# -1：HUP，重新读取一次参数配置文件，相当于reload；
# -2：INT，终止程序，相当于Ctrl+c；
# -9：KILL，立即强制删除一个任务；
# -15：TERM，以正常方式终止一项任务，与-9不同；
# -19：STOP，暂停任务，相当于Ctrl+z。
```

## 脱机管理：nohup
如果用户是以远程方式登录到Linux主机，并将某些任务放到了后台运行，如果在任务尚未运行结束时，用户断开了连接，远程Linux后台中的任务也会**中断**。因为用户登录后放到后台运行的任务实际上是放到了登录bash**终端**的后台，而不是远程Linux主机的系统后台。

为了解决这个问题，一种方法是使用 **at** 指令将任务放到系统后台，另一种方法则是使用 **nohup** 指令。

```bash
[root@study ~]$ nohup [指令与参数]
[root@study ~]$ nohup [指令与参数] &

[root@study ~]$ nohup ./dosomething.sh &  # 假设该指令执行需要耗费一定时间
[2] 14812
# 指令输出被重定向至nohup.out，也可以使用>指定输出重定向文件
[root@study ~]$ nohup: ignoring input and appending out to 'nohup.out'
[root@study ~]$ exit  # 退出登录
# 重新登录后使用pstree指令可以发现./dosomething.sh指令仍然在运行
```

# 进程管理
## 查看进程：ps, top, pstree
**查看某个时间点进程的运行情况：ps**
ps指令的参数很多，下面仅介绍一些常用的组合。其他参数请参见[链接](https://www.man7.org/linux/man-pages/man1/ps.1.html)。

```bash
[root@study ~]$ ps -[AaCdeFfGgjlMmNOopqstUuVwy...]
# -A：列出所有进程，等于-e；
# -a：列出除了session leaders和与终端无关的进程之外的其他所有进程；
# -u：与有效使用者相关的进程
# -l：以long format列出详细信息；
# -f：以full-format列出信息；
# -j：列出job formats。

[root@study ~]$ ps -l	# 仅列出与当前登录bash有关的信息
[root@study ~]$ ps aux	# 列出当前所有在内存中的进程（包括其他登录的用户）
[root@study ~]$ ps axjf	# 以类似进程数的形式展示
[root@study ~]$ ps -ef	# 列出系统中的所有进程
```
其中，`ps aux`指令的输出如下：

```bash
[root@study ~]$ ps aux
USER  PID	%CPU  %MEM  VSZ	   RSS	 TTY   STAT	 START	TIME	COMMAND
root  1	    0.0	  0.2	60636  7948	 ?	   Ss	 Aug04	0:01	/usr/lib/systemd/systemd ...
......
root  1483	0.0	  0.1	210744 3988	 pts/0  S	 Aug04	0:00	sudo su -
```

其中，每一列的含义为

- **USER**：进程属于的账号；
- **PID**：进程号；
- **%CPU**：进程占用的CPU资源百分比；
- **%MEM**：进程占用的物理内存百分比；
- **VSZ**：进程使用的虚拟内存（KB）；
- **RSS**：进程占用的固定内存（KB）；
- **TTY**：进程运行在哪个终端。？表示**与终端无关**，tty1至tty6表示由**本机**登陆的账户进程，形如pts/0则表示由**网络**连接到主机的进程。
- **STAT**：进程目前的状态。**R**表示正在运行（Running），**S**表示处在睡眠状态（Sleep）但可以被唤醒，D表示不可被唤醒的睡眠状态（可能在等待I/O），**T**表示处于后台暂停（Stopped）或除错状态（Traced），**Z**表示进程已经终止但却无法被移出内存（Zombie）。
- **START**：进程被触发启动的时间；
- **TIME**：进程实际使用CPU运行的时间；
- **COMMAND**：进程的实际指令。

由于ps指令的输出较多，可以配合管线与grep指令搜寻指定的进程。

**动态观察进程：top**

```bash
[root@study ~]$ top [-d 数字] [-bnp]
# -d：每次更新间隔的秒数，默认是5秒；
# -b：以批次的方式执行top；
# -n：与-b配合，表示输出几次top的结果；
# -p：指定观察进程的PID。

# 在top指令执行过程中可以使用如下按键：
? # 显示可以使用的按键指令
P # 按CPU使用资源量排序显示
M # 按内存使用量排序显示
N # 以PID排序显示
T # 按进程使用的CPU累计时间排序显示
k # 给于某个PID一个signal
r # 修改某个PID对应的NI值（与Priority有关，NI值越小进程越早被执行）
q # 退出top
```

**查看进程之间的相关性：pstree**
`pstree`可以查看进程之间的依赖关系。

```bash
[root@study ~]$ pstree [-A|U] [-up]
# -A：各进程树之间以ASCII字符连接；
# -U：各进程树之间以Unicode字符连接；
# -u：列出每个进程所属的账户名称；
# -p：列出每个进程的PID。
```

## 进程管理
**kill -signal PID**
假设我们要rsyslogd指令对应的进程重新读取配置文件

```bash
# 找出rsyslogd进程的PID
[root@study ~]$ ps aux | grep 'rsyslogd' | grep -v 'grep' | awk '{print $2}'
[root@study ~]$ kill -1 PID 

# 或者直接执行
[root@study ~]$ kill -1 $(ps aux | grep 'rsyslogd' | grep -v 'grep' | awk '{print $2}')
# 或者
[root@study ~]$ kill -1 `ps aux | grep 'rsyslogd' | grep -v 'grep' | awk '{print $2}'`
```

**killall -signal 指令名称**
killall用于根据指令名称发送signal。

```bash
[root@study ~]$ killall [-iIe] -signal command
# -i：需要删除时提示用户确认；
# -I：指令名称忽略大小写；
# -e：指令名称要一致（不超过15个字符）

[root@study ~]$ killall -9 httpd
```

## 进程的执行顺序
**Priority & Nice**
优先执行序（**Priority, PRI**）是Linux进程的一个属性，其值越小，代表优先权越高。可以通过`ps -l`查看进程的PRI值。PRI的值由Linux核心动态调整，用户无法直接修改。但是用户可以通过修改进程的 **Nice** 属性（**NI**）来调整进程执行的优先顺序。NI的取值范围是 **-20至19**。PRI与NICE的相关性为：$$PRI_{new}=PRI_{old}+NI$$ 即Nice值越小，进程执行的优先权越高。一般用户只能将自己的进程NI值增大（0\~19），root用户可以随意修改所有用户进程的NI值。

**nice**
对于一个新执行的进程，可以直接使用`nice`命令设置NI值。

```bash
[root@study ~]$ nice [-n 数字] command
# -n后接数字的范围是-20~19
[root@study ~]$ nice -n -5 vim &
[1] 19865
[root@study ~]$ ps -l 
```

**renice**
对于一个已经开始执行的进程，可以使用`renice`命令修改NI值。

```bash
[root@study ~]$ renice [数字] PID
# 其中数字为NI值，PID对应要修改的进程
```

## 查看系统资源
**free：观察内存使用**

```bash
[root@study ~]$ free [-b|-k|-m|-g|-h] [-t] [-s 数字 -c 数字]
# -b：使用bytes(b)、kbytes(k)、Mb(m)、Gb(g)来显示单位，默认为Mb。也可以让系统自己指定单位（h）；
# -t：显示物理内存与swap的总量；
# -s：不间断每多少秒输出一次结果；
# -c：与-s结合，让free输出多少次结果。
```

**uname：查阅系统与核心信息**

```bash
[root@study ~]$ uname [-asrmpi]
# -a：与系统相关的所有信息；
# -s：系统核心名称；
# -r：核心的版本；
# -m：系统的硬件名称，如x86_64；
# -p：CPU的类型；
# -i：硬件平台，如ix86。
```

**uptime：系统启动时间与工作负载**

```bash
[root@study ~]$ uptime 
# 显示系统已经开机多久，以及1、5、15分钟的平均负载
```

**netstat：网络监控**

```bash
[root@study ~]$ netstat [-atunlp]
# -a：将目前系统上所有的连接、监听、socket数据都列出来；
# -t：列出tcp网络封装的数据信息；
# -u：列出udp网络封装的数据信息；
# -n：不以进程服务名称显示，而是以数字形式显示；
# -l：列出正在监听（Listen）的服务；
# -p：列出网络服务的进程PID。
```

**dmesg：分析核心产生的信息**

```bash
[root@study ~]$ dmesg | more   # 查看核心的开机信息
[root@study ~]$ dmesg | grep -i vda   # 查看开机时的硬盘信息
```

**vmstat：监视系统资源变化**

```bash
[root@study ~]$ vmstat [-a] [延迟 [总计侦测次数]]  # CPU、内存等信息
[root@study ~]$ vmstat [-fs]  # 开机后的内存变化情况和系统fork的进程数
[root@study ~]$ vmstat [-S 单位]  # 设定显示数据的单位
[root@study ~]$ vmstat [-d]  # 磁盘的读写总量统计信息
[root@study ~]$ vmstat [-p 分区槽]  # 磁盘分区槽的读写总量统计信息
```

# 特殊文件与进程
**/proc/\* 的意义**
主机上的各个进程PID都是以目录的形式存在于`/proc`目录下的。比如开机执行的第一个程序`systemd`的PID等于1。与该进程相关的所有信息都写在`/proc/1/`目录下。此外，该目录下还包括以下与Linux系统相关的参数文件：

| 文件名 | 文件内容 |
|--|--|
| /proc/cmdline | 加载核心时下达的指令与参数 |
| **/proc/cpuinfo** | 本机CPU信息 |
| **/proc/devices** | 系统的主要设备信息 |
| /proc/filesystems | 系统已经加载的文件系统 |
| /proc/interrupts | 系统上IRQ分配状态 |
| /proc/ioports | 系统上各个设备配置的I/O地址 |
| /proc/kcore | 内存大小 |
| /proc/loadavg | top与uptime命令的输出信息 |
| **/proc/meminfo** | 使用free列出的内存信息 |
| /proc/modules | 系统已加载的模块信息，即驱动程序 |
| /proc/mounts | 系统已挂载的数据，即mount指令的输出信息 |
| /proc/swaps | 已使用的分区信息 |
| **/proc/partitions** | 使用fdisk -l列出的磁盘分区信息 |
| /proc/uptime | uptime指令的输出信息 |
| **/proc/version** | 核心版本，uname -a的输出信息 |
| /proc/bus/* | 总线设备，包括USB设备的信息 |

**与进程相关的其他指令**
**fuser**：找出正在使用某个文件的进程

```bash
[root@study ~]$ fuser [-umv] [-k [i] [-signal]] file/dir
# -u：列出文件的拥有者；
# -m：自动关联文件系统最顶层，适用于umount不成功的情况；
# -v：列出文件与进程关联的完整信息；
# -k：找出使用该文件/目录的PID，并试图给予SIGKILL信号；
# -i：与-k结合，在删除PID之前询问用户；
# -signal：信号类型，默认为-9（SIGKILL）。
```

**lsof**：列出被进程开启的文件名

```bash
[root@study ~]$ lsof [-aUu] [+d]
# -a：多项数据需要同时成立时才输出结果；
# -U：仅列出Unix like系统的socket文件类型；
# -u：接用户名，列出该账户相关进程所开启的文件；
# +d：接目录，列出该目录下已开启的文件。
```
**pidof**：找出正在执行程序的PID

```bash
[root@study ~]$ pidof [-sx] 程序名
# -s：仅列出一个PID而不是所有的PID；
# -x：同时列出该进程可能的PPID。
```

