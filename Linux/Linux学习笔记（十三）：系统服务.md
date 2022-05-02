@[TOC](Linux学习笔记（十三）：系统服务)

# daemon & service
A **daemon**, or **system service**, is a background process usually started during the boot sequence. Daemons typically run independent of users, waiting for events to occur and providing services in response. [source1](https://wiki.debian.org/Daemon)

A "**Service**" could refer to either a Daemon or a Service. A **daemon** is a subset of services that always run in memory waiting to service a request. A **non-daemon service** generally is handled by xinetd (eXtended InterNET services daemon). xinetd listens for the request, then starts the required service to handle the request. After the request has been serviced the service is then stopped again. [source2](https://serverfault.com/questions/129055/is-there-a-difference-between-a-daemon-and-a-service#:~:text=A%20daemon%20is%20a%20subset%20of%20services%20that,starts%20the%20required%20service%20to%20handle%20the%20request.)

Typical non-daemon services: rsync vsftpd
Typical daemonized services: MySQL Apache

然而对于一般的Linux使用者来说，并不需要区分daemon与service。

# systemd
systemd即为**system daemon**，是linux下的一种init软件，开发目标是提供更优秀的框架以表示系统服务间的依赖关系，并依此实现系统初始化时服务的并行启动，同时达到降低Shell的系统开销的效果，最终代替常用的System V与BSD风格init程序。[source3](https://baike.baidu.com/item/systemd/18473007)

从CentOS 7.x以后，红帽系列的distribution放弃了沿用多年的System V开机启动服务的流程，改用systemd的启动服务管理机制。==systemd包括常驻内存的**systemd服务**与**systemctl**指令==。与System V的init启动脚本相比，其好处在于：

 - 平行处理所有服务，加速开机流程；
 - 一经要求就响应的on-demand启动方式；
 - 自动检查并处理服务依赖性；
 - 根据功能将**服务单位**（unit）划分为service, socket, target, path, snapshot, timer等不同类型，方便管理；
 - 将多个daemon集合为一个target群组，可以同时执行；
 - 向下兼容旧的init服务脚本。
 
 **systemd的配置文件目录**
systemd将以前的daemon执行脚本通称为一个服务单位（**unit**）。每种服务单位依据功能来区分时，可以分为不同的类型（type）。基本的类型有系统服务、数据监听与交换的套接字服务（**socket**）、存储系统状态的快照（**snapshot**）、提供不同类似执行等级分类的操作环境（**target**），等等。不同类型的服务配置文件都放在如下目录中：
 - `/usr/lib/systemd/system/`：每个服务最主要的启动脚本设定，类似以前`/etc/init.d`中的文件；
 - `/run/systemd/system/`：系统执行过程中所产生的服务脚本，其优先级比`/usr/lib/systemd/system`高；
 - `/etc/systemd/system`：管理员依据主机系统的需求建立的执行脚本，类似以前的`/etc/rc.d/rc5.d/Sxx`，执行的优先级比`/run/systemd/system`高。

因此，决定系统开机是否会执行哪些服务其实要看`/etc/systemd/system`中的设置，但实际中该目录下都是一些链接文件，实际执行的脚本都在`/usr/lib/systemd/system`目录下。

**systemd的unit类型分类说明** 
通过文件扩展名可以区分`/usr/lib/systemd/system/`目录下不同服务的类型。
| 扩展名 | 主要服务功能 |
|--|--|
| .service | 一般服务类型，主要是系统服务，包括服务器本身所需要的本地服务以及网络服务，也是最常见的类型 |
| .socket | 内部程序数据交换的套接字服务，可以看作是为不同进程间提供双向通信功能（Inter-process communication）的端点 |
| .target | 执行环境类型，其实是一群服务单元的集合，执行.target就是执行集合中包含的一堆.service和.socket等服务 |
| .mount<br> .automount | 文件系统挂载相关的服务，例如来自网络的自动挂载、NFS文件系统挂载等等 |
| .path | 侦测特定文件或目录类型，比如打印服务就需要侦测打印队列目录来启动打印功能 |
| .timer | 循环执行的服务，功能与anacrontab类似，但是更加灵活 |

# systemctl指令
相比System V的service, chkconfig, setup, init指令，systemd仅依赖systemctl一个指令来管理服务。

## 管理单一服务（service unit）
通过systemctl管理单一服务的启动、开机启动的指令格式为`systemctl [command] [unit]`。其中，command主要有：

 - start：立刻启动服务单元；
 - stop：立刻停止服务单元；
 - restart：立刻重启服务单元；
 - reload：在不停止服务单元的情况下，重新加载配置文件使设定生效；
 - enable：设定服务单元开机自启动；
 - disable：设定服务单元开始时不会被启动；
 - status：查看服务单元目前的状态；
 - is-active：查看服务单元目前是否在运行中；
 - is-enable：查看服务单元是否设置了开机自启动。

需要注意的是，不要使用kill指令关闭服务，否则systemctl会无法继续监控该服务。`systemctl status`的输出中，常见的服务状态有：

 - active (running)：有一个或多个程序正在系统中执行；
 - active (exited)：仅执行一次就正常结束的服务，目前并没有任何程序在系统中执行；
 - active (waiting)：正在执行中，但是在等待其他的事件才能继续处理；
 - inactive：当前服务没有处于运行状态；
 - enabled：该服务已设置开机启动；
 - disabled：该服务未设置开机启动；
 - static：该服务不能自己启动，但可以被其他服务唤醒；
 - mask：该服务无论如何都不能被启动，因为已经被强制注销。但是可以通过`systemctl unmask`指令改回原来的状态。

## 查看系统上所有的服务
通过systemctl查看系统上所有服务的指令格式为`systemctl [command] [--type=TYPE] [--all]`。其中，command有

 - list-units：根据unit列出目前启动的服务单元，若加上`--all`才会列出未启动的服务；
 - list-unit-files：根据`/usr/lib/systemd/system/`内的文件，将所有文件列表说明。
 
 `--type=TYPE`即是指服务单元的类型，包括service, socket, target等等。

## 管理不同的操作环境（target unit）
列出与操作界面有关的target的指令为`systemctl list-units --type=target --all`。CentOS 7中与操作界面相关性较高的主要有

 - graphical.target：文字加上图形界面，包含了multi-user.target项目；
 - multi-user.target：纯文本模式；
 - rescue.target：在无法使用root登录的情况下，systemd在开机时会多加一个额外的暂时系统，可以用于取得root权限来维护系统；
 - emergency.target：紧急处理系统的错误，在rescue.target无法使用时可以尝试；
 - shutdown.target：关机的流程；
 - getty.target：可以设定需要几个tty。

查看与修改操作界面相关的target的指令格式为`systemctl [command] [unit.target]`。其中，command包括

 - get-default：显示目前的target信息；
 - set-default：设定target为默认的操作模式；
 - isolate：在不关机的情况下，**切换**到后面接的操作模式。

systemctl也提供了其他指令来切换操作模式。

```bash
[root@study ~]$ systemctl get-default
graphical.target
[root@study ~]$ systemctl set-default multi-user.target
[root@study ~]$ systemctl get-default
multi-user.target
[root@study ~]$ systemctl isolate multi-user.target
[root@study ~]$ systemctl isolate graphical.target

[root@study ~]$ systemctl poweroff # 系统关机
[root@study ~]$ systemctl reboot # 重启
[root@study ~]$ systemctl suspend # 暂停
[root@study ~]$ systemctl hibernate # 休眠
[root@study ~]$ systemctl rescue # 强制进入救援模式
[root@study ~]$ systemctl emergency # 进入紧急救援模式
```

其中，暂停模式（suspend）会将系统的状态数据保存到**内存**中，然后关闭大部分系统硬件，但并未关机；休眠模式（hibernate）则是将系统状态保存到**硬盘**当中，然后将计算机关机。暂停模式唤醒的速度通常比休眠模式更快。

## 分析服务间的依赖性
通过systemctl分析服务之间依赖性的指令为`systemctl list-dependencies [unit] [--reverse]`。其中`--reverse`表示反向追踪谁在使用该服务单元。

## 与daemon运行相关的目录
除了之前提到的systemd配置文件目录以外，与系统服务运行有关的目录还包括：

 - `/usr/lib/systemd/system/`
 - `/run/systemd/system/`
 - `/etc/systemd/system/`
 - `/etc/sysconfig/*`：几乎所有服务都会将初始化的一些选项设定写入该目录下；
 - `/var/lib/`：一些会产生数据的服务会将数据写入到该目录下；
 - `/run/`：该目录下存放了很多系统服务的暂存文件，包括lock file和PID file。

## 网络服务与端口
`/etc/services`中设置了网络服务、对应的网路协议（tcp、udp）与端口。一般不建议通过该文件来直接修改服务的端口号。使用`netstat`指令也可以查看打开的端口。如果有不需要开启的网络服务，可以使用`systemctl stop`指令关闭服务（及其使用的端口）。

# systemctl针对service类型的配置文件
## systemctl配置文件目录
以vsftpd服务为例，其对应的配置文件包括：

 - `/usr/lib/systemd/system/vsftpd.service`：官方释出的预设配置文件，建议不要修改；
 - `/etc/systemd/system/vsftpd.service.d/custom.conf`：该文件中的设定会被累加到`/usr/lib/systemd/system/vsftpd.service`文件中；
 - `/etc/systemd/system/vsftpd.service.wants/*`：依赖文件的链接，**wants**表示在vsftpd.service启动**之后**才能启动的其它服务；
 - `/etc/systemd/system/vsftpd.service.requires/*`：依赖文件的链接，**requires**表示在启动vsftpd.service**之前**需要先启动的其它服务。

## systemctl配置文件参数
以sshd服务为例，其配置文件sshd.service的内容如下：

```bash
[root@study ~]$ cat /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
After=network.target sshd-keygen.service
Wants=sshd-keygen.service

[Service]
EnvironmentFile=/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```
注意到，该配置文件内容被分为了三个部分：

 - **[Unit]**：服务单元本身的说明，以及与其他服务的依赖关系；
 - **[Service], [Socket], [Mount], ...**：不同的服务单元类型就要使用对应的项目设定。这里sshd.service当然就是用[Service]设定。该部分内容主要规定服务启动的脚本、环境、配置文件名、重启的方式等等；
 - **[Install]**：表示将该服务单元安装到哪一个target里面。

此外，有几个重要的设定规则如下：
- 配置文件中的设定项目是可以重复的，比如我可以设定多个After参数，但是最后一个After的设定会取代（**覆盖**）前面所有After参数的设定值；
- 如果有的参数的设定方式是布尔值，可以用1, yes, true, on表示启动，用0, no, false, off表示关闭；
- 空白行、以#或;开头的行都表示注释。

[Unit]部分常见的设定参数如下：
| [Unit]参数 | 意义 |
|--|--|
| Description | 使用`systemctl list-units`时会输出的简单说明 |
| Documentation | 提供进一步的文件查询功能，可以是如下内容：<br>Documentation=http://www..., Documentation=man:sshd(8), 或Documentation=file:/etc/ssh/sshd_config |
| After | 说明此Unit在哪些服务启动之后才会启动。仅说明服务的启动顺序，并不是实际上的强制设定（与Requires不同） |
| Before | 与After刚好相反，但也仅仅是说明启动顺序 |
| **Requires** | 明确定义此Unit必须在哪些服务启动后才能启动。如果后面接的服务没有启动，该Unit就不会被启动 |
| Wants | 与Requires刚好相反，定义最好在该Unit启动之后再启动的其他服务。如果后面接的服务没有启动，不会影响到该Unit本身 |
| **Conflicts** | 冲突的服务，即不能与该Unit同时运行的其它服务 |

[Service]部分常见的设定参数如下：
| [Service]参数 | 意义 |
|--|--|
| Type | 该服务启动的方式，会影响到ExecStart。可以分为：<br> **simple**，默认值，邮ExecStart启动，且启动后常驻内存； **forking**，由ExecStart启动的程序通过spawns延伸出其他子程序来作为此服务的主要服务，其原生的父程序在启动结束后就会终止运行；**oneshot**，与simple类似，但该程序在工作完成后就会结束，不会常驻在内存中； **dbus**，与simple类似，但此服务必须在取得一个D-Bus名称后才会继续运行；**idle**，与simple类似，但是必须要等到所有工作都顺利执行完毕后该服务才会执行，通常是开机过程中最后执行的服务。 |
| EnvironmentFile | 指定启动脚本的环境配置文件 |
| **ExecStart** | 实际执行此服务的指令或脚本，与systemctl start有关 |
| **ExecStop** | 关闭此服务时所执行的指令或脚本，与systemctl stop有关|
| **ExecReload** | 重载配置文件，与systemctl reload有关 |
| Restart | 设定为1时，当此服务终止后，会自动再次启动该服务 |
| RemainAfterExit | 设定为1时，当此服务所属的所有程序都终止以后，此服务会自动尝试启动 |
| TimeoutSec | 此服务出现无法正常启动或关闭时，在进入强制结束状态前需要等待的时间 |
| KillMode | 可以是以下三种模式中的一种：<br> **process**，此服务终止时，只会停止主要的程序（ExecStart中的指令）；<br>**control-group**，由该服务产生的其他control-group程序也都会被关闭；<br>**none**，没有程序会被关闭 |
| RestartSec | 此服务被关闭后需要重启时，重启前需要等待的时间，预设为100ms |

[Install]部分常见的设定参数如下：
| [Install]参数 | 意义 |
|--|--|
| WantedBy | 后面接的大部分是*.target unit，表示该服务附挂在哪一个tagrget unit底下。一般是multi-user.target |
| Also | 当目前该服务被设定为开机自启（enabled）时，后面接的服务也会被enable，通常是具有依赖性的其它服务 |
| Alias | 设定别名 |

## 举例：创建备份服务
假设我们要创建一个备份服务。

```bash
# 创建备份脚本
[root@study ~]$ vim /backups/backup.sh
#! /bin/bash

source="/etc /home /root /var/lib /var/spool/{cron.at.mail}"
target="/backups/backup-system-$(data +%Y-%m-%d).tar.gz"
[ ! -d /backups ] && mkdir /backups
tar -zcvf ${target} ${source} &> /backups/backup.log
[root@study ~]$ chmod a+x /backups/backup.sh

# 创建服务配置文件
[root@study ~]$ vim /etc/systemd/system/backup.service
[Unit]
Description=backup my server
Requires=atd.service  # 因为用到了at指令

[Service]
Type=simple
ExecStart=/bin/bash -c " echo /backups/backup.sh | at now"

[Install]
WantedBy=multi-user.target
[root@study ~]$ systemctl daemon-reload  # 重载服务配置文件
[root@study ~]$ systemctl start backup.service  # 运行一次备份服务
[root@study ~]$ systemctl status backup.service
backup.service - backup my server
   Loaded: loaded (/etc/systemd/system/backup.service; disabled)
   Acitve: inactive (dead)  # 备份服务执行完就停止了，并不会常驻内存
```

# systemctl针对timer类型的配置文件
假设上面创建的backup.service服务需要定期执行，可以使用systemd的timer来处理。要使用timer功能，必须具备以下条件：

 - 系统的timer.target必须已经启动；
 - backup.service服务已经创建；
 - backup.timer的时间启动服务已创建，且设置为enabled。

**.timer文件的参数设定**
可以在`/etc/systemd/system/`下建立timer文件，其基本参数包括：

 - OnActiveSec：当timer.target启动多久后才执行此服务；
 - OnBootSec：当开机多久后才执行此服务；
 - OnStartupSec：当systemd第一次启动之后多久才执行此服务；
 - OnUnitActiveSec：此timer配置文件管理的unit服务在最后一次启动后，隔多久后再执行一次；
 - OnUnitInactiveSec：此timer配置文件管理的unit服务在最后一次停止后，隔多久后再执行一次；
 - **OnCalendar**：按实际时间（非循环时间）来启动服务；
 - Unit：一般无需设定，默认为对应的服务名，即name.service对应name.timer；
 - **Persistent**：使用OnCalendar时，是否要持续进行。设定为yes即能满足类似anacron的功能。

**OnCalendar时间格式**
基本的时间格式为“星期几 YYYY-MM-DD HH:MM:SS”，例如`Thu 2020-12-25 14:30:00`。
也可以使用时间间隔，包括毫秒ms、秒s、分钟m、小时h、天数d、周数w、月month(s)、年y。比如，隔5天12小时30分钟可以表示为`30m 12h 5d`，或者`30min 12hours 5day`。
还可以使用口语化的时间表达方式，包括now、today、tomorrow、hourly、daily、weekly、monthly、+3h10m、2020-12-30等。

假设我们需要前面创建的备份服务backup.service每周日凌晨2点运行一次，创建对应的timer文件如下。

```bash
[root@study ~]$ vim /etc/systemd/system/backup.timer
[Unit]
Description=backup my server timer

[Timer]
OnCalendar=Sun *-*-* 02:00:00
Persistent=true
Unit=backup.service

[Install]
WantedBy=multi-user.target
[root@study ~]$ systemctl daemon-reload  # 重载服务配置文件
[root@study ~]$ systemctl enable backup.timer
[root@study ~]$ systemctl start backup.timer
[root@study ~]$ systemctl show backup.timer
NextElapseUSecRealtime=50y 11month 3w 6d 23h 59min  ## 下一次执行需要等待的时间，与1970-01-01 00:00:00比较
```



