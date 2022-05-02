@[TOC](如何使用TiUP部署一个TiDB v5.0集群)

# 安装环境
部署架构为：3台PD节点 + 1台Server节点 + 3台TiKV节点。
- tidb-pd1
- tidb-pd2
- tidb-pd3
- tidb-server
- tidb-kv1
- tidb-kv2
- tidb-kv3

操作系统为CentOS 7.9, 采用最低资源配置：1C1G Memory + 20G Disk。

选择tidb-pd1为部署节点，部署操作都在该节点进行。建议所有节点采用相同的root密码，或者提前建立tidb-pd1到其他节点的ssh互信。

# 下载TiUP
下载TiUP安装工具

```bash
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
```

声明全局环境变量
```bash
source /root/.bash_profile
```

安装TiUP cluster组件
```bash
tiup cluster
```

更新组件到最新版本
```bash
tiup update --self && tiup update cluster
```

验证TiUP cluster版本信息
```bash
tiup --binary cluster
```

# 集群拓扑文件
根据不同的集群拓扑，编辑集群初始化配置文件
```bash
tiup cluster template > topology.yaml
```

编辑拓扑文件
```bash
vi topology.yaml
```

配置各个角色的服务器IP
```yaml
pd_servers:
  - host: IP.a.b.132
  - host: IP.a.b.133
  - host: IP.a.b.134
tikv_servers:
  - host: IP.a.b.136
  - host: IP.a.b.137
  - host: IP.a.b.138
tidb_servers:
  - host: IP.a.b.135
tiflash_servers:
  #注释该部分的IP，本次不安装TiFlash节点
monitoring_servers:
  - host: IP.a.b.133
grafana_servers:
  - host: IP.a.b.133
alertmanager_servers:
  - host: IP.a.b.133
```

# 部署前环境检查
检查和修复集群存在的潜在风险。该部分时间可能会很长，且可能需要重复执行多次（Take your time!）。
```bash
tiup cluster check ./topology.yaml --apply --user root -p

Node        Check         Result  Message
----        -----         ------  -------
IP.a.b.136  command       Fail    numactl not usable, bash: numactl: command not found, auto fixing not supported
IP.a.b.136  os-version    Pass    OS is CentOS Linux 7 (Core) 7.9.2009
IP.a.b.136  cpu-cores     Pass    number of CPU cores / threads: 1
IP.a.b.136  cpu-governor  Warn    Unable to determine current CPU frequency governor policy, auto fixing not supported
IP.a.b.136  swap          Fail    swap is enabled, please disable it for best performance, auto fixing not supported
IP.a.b.136  memory        Pass    memory size is 1024MB
IP.a.b.136  network       Pass    network speed of ens33 is 1000MB
IP.a.b.136  service       Fail    will try to 'start irqbalance.service'
IP.a.b.136  service       Fail    will try to 'stop firewalld.service'
IP.a.b.136  disk          Warn    mount point / does not have 'noatime' option set, auto fixing not supported
IP.a.b.136  limits        Fail    will try to set 'tidb    soft    nofile    1000000'
IP.a.b.136  limits        Fail    will try to set 'tidb    hard    nofile    1000000'
IP.a.b.136  limits        Fail    will try to set 'tidb    soft    stack    10240'
IP.a.b.136  sysctl        Fail    will try to set 'fs.file-max = 1000000'
IP.a.b.136  sysctl        Fail    will try to set 'net.core.somaxconn = 32768'
IP.a.b.136  sysctl        Fail    will try to set 'net.ipv4.tcp_syncookies = 0'
IP.a.b.136  sysctl        Fail    will try to set 'vm.swappiness = 0'
IP.a.b.136  selinux       Fail    will try to disable SELinux, reboot might be needed
IP.a.b.136  thp           Fail    will try to disable THP, please check again after reboot
...
```

从上面的检查结果可以得出以下结论
- 需要手动修复的问题：
  - 安装numactl命令：`yum install numactl -y`
  - 关闭swap：
```bash
swapoff -a  # 暂时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab  # 永久关闭
```

- 可以自动修复的问题：
  - 启动irqbalance.service（用于提升多核处理器的CPU性能）；
  - 关闭firewalld.service；
  - 修改tidb用户的内核参数，包括soft nofile, hard nofile, soft stack, fs.file-max, somaxconn, tcp_syncookies, swappiness；
  - 禁用SELinux（需要重启OS）；
  - 禁用THP透明大页。


修改完重启所有服务器，再次运行安装前环境检查：
```bash
tiup cluster check ./topology.yaml --apply --user root -p

Node        Check         Result  Message
----        -----         ------  -------
IP.a.b.135  command       Pass    numactl: policy: default
IP.a.b.135  os-version    Pass    OS is CentOS Linux 7 (Core) 7.9.2009
IP.a.b.135  cpu-cores     Pass    number of CPU cores / threads: 1
IP.a.b.135  thp           Fail    will try to disable THP, please check again after reboot
IP.a.b.135  selinux       Pass    SELinux is disabled
IP.a.b.135  service       Fail    will try to 'start irqbalance.service'
IP.a.b.135  service       Fail    will try to 'stop firewalld.service'
IP.a.b.135  cpu-governor  Warn    Unable to determine current CPU frequency governor policy, auto fixing not supported
IP.a.b.135  memory        Pass    memory size is 1024MB
IP.a.b.135  network       Pass    network speed of ens33 is 1000MB
 ```
 
看样子THP、firewalld.service、irqbalance.service的问题在自动修复且重启后又出现了。

永久禁用透明大页THP：
```bash
vi /etc/rc.d/rc.local 
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
:wq  # 保存退出
# 授予可执行权限
chmod +x /etc/rc.d/rc.local
```

关闭开机自启：
```bash
systemctl disable firewalld.service
```

打开开机自启（仅支持多核CPU）：
```bash
systemctl enable irqbalance.service
```

# 集群部署
使用TiUP部署TiDB集群，时间会很长（Wait while drinking your coffee or play with your phone :-）。
```bash
tiup cluster deploy tidb-test v5.0.0 ./topology.yaml --user root -p
```

部署完成后可以对比一下PD、TiDB Server、TiKV节点的文件。TiDB会在每个节点都生成 `/tidb-data`和`/tidb-deploy`两个目录，但是每个节点存放的文件可能不大一样。

```bash
[root@tidb-pd1 ~]# ls /tidb-data
monitor-9100  pd-2379
[root@tidb-pd1 ~]# ls /tidb-deploy/
monitor-9100  pd-2379

# tidb-pd2节点部署了监控告警系统：alertmanager, prometheus, grafana
[root@tidb-pd2 ~]# ls /tidb-data
alertmanager-9093  monitor-9100  pd-2379  prometheus-9090
[root@tidb-pd2 ~]# ls /tidb-deploy
alertmanager-9093  grafana-3000  monitor-9100  pd-2379  prometheus-9090

[root@tidb-server ~]# ls /tidb-data
monitor-9100
[root@tidb-server ~]# ls /tidb-deploy
monitor-9100  tidb-4000

[root@tidb-kv1 ~]# ls /tidb-data
monitor-9100  tikv-20160
[root@tidb-kv1 ~]# ls /tidb-deploy
monitor-9100  tikv-20160
```

# 检查集群状态
查看TiUP管理的集群情况
```bash
tiup cluster list
```
检查tidb-test集群情况
```bash
tiup cluster display tidb-test

tiup is checking updates for component cluster ...
Starting component `cluster`: /root/.tiup/components/cluster/v1.9.3/tiup-cluster /root/.tiup/components/cluster/v1.9.3/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v5.0.0
Deploy user:        tidb
SSH type:           builtin
ID                Role          Host        Ports        OS/Arch       Status  Data Dir                      Deploy Dir
--                ----          ----        -----        -------       ------  --------                      ----------
tidb-pd2:9093     alertmanager  IP.a.b.133  9093/9094    linux/x86_64  Down    /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
tidb-pd2:3000     grafana       IP.a.b.133  3000         linux/x86_64  Down    -                             /tidb-deploy/grafana-3000
tidb-pd1:2379     pd            IP.a.b.132  2379/2380    linux/x86_64  Down    /tidb-data/pd-2379            /tidb-deploy/pd-2379
tidb-pd2:2379     pd            IP.a.b.133  2379/2380    linux/x86_64  Down    /tidb-data/pd-2379            /tidb-deploy/pd-2379
tidb-pd3:2379     pd            IP.a.b.134  2379/2380    linux/x86_64  Down    /tidb-data/pd-2379            /tidb-deploy/pd-2379
tidb-pd2:9090     prometheus    IP.a.b.133  9090         linux/x86_64  Down    /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
tidb-server:4000  tidb          IP.a.b.135  4000/10080   linux/x86_64  Down    -                             /tidb-deploy/tidb-4000
tidb-kv1:20160    tikv          IP.a.b.136  20160/20180  linux/x86_64  N/A     /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
tidb-kv2:20160    tikv          IP.a.b.137  20160/20180  linux/x86_64  N/A     /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
tidb-kv3:20160    tikv          IP.a.b.138  20160/20180  linux/x86_64  N/A     /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 10
```

可以看到，各角色状态都为**Down**，因为我们还没有启动tidb-test集群。

# 启动集群
启动tidb-test集群
```bash
tiup cluster start tidb-test
```

检查tidb-test集群情况
```bash
tiup cluster display tidb-test

tiup is checking updates for component cluster ...
Starting component `cluster`: /root/.tiup/components/cluster/v1.9.3/tiup-cluster /root/.tiup/components/cluster/v1.9.3/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v5.0.0
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://IP.a.b.133:2379/dashboard
ID                Role          Host        Ports        OS/Arch       Status  Data Dir                      Deploy Dir
--                ----          ----        -----        -------       ------  --------                      ----------
tidb-pd2:9093     alertmanager  IP.a.b.133  9093/9094    linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
tidb-pd2:3000     grafana       IP.a.b.133  3000         linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
tidb-pd1:2379     pd            IP.a.b.132  2379/2380    linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
tidb-pd2:2379     pd            IP.a.b.133  2379/2380    linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
tidb-pd3:2379     pd            IP.a.b.134  2379/2380    linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
tidb-pd2:9090     prometheus    IP.a.b.133  9090         linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
tidb-server:4000  tidb          IP.a.b.135  4000/10080   linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
tidb-kv1:20160    tikv          IP.a.b.136  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
tidb-kv3:20160    tikv          IP.a.b.137  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
tidb-kv3:20160    tikv          IP.a.b.138  20160/20180  linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 10
```

启动集群后，TiDB为我们提供了两个对监控和管理集群非常有用的Web工具：
- **Grafana**：`http://tidb-pd2:3000`, 初始账户密码：admin/admin
- **TiDB Dashboard**：`http://tidb-pd2:2379/dashboard`, 初始账户密码：root/数据库密码

# 查看各节点进程和资源消耗
启动集群后可以对比一下PD、TiDB Server、TiKV节点的进程和资源消耗情况
```bash
[root@tidb-pd1 ~]# top
top - 09:29:49 up  1:31,  2 users,  load average: 0.00, 0.01, 0.05
Tasks: 109 total,   2 running, 107 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  4.8 sy,  0.0 ni, 95.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   995676 total,   138444 free,   228580 used,   628652 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   619332 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
  2767 tidb      20   0 9959848  95012  35148 S  0.0  9.5   0:06.32 pd-server
  2828 tidb      20   0  113804  14620   5372 S  0.0  1.5   0:01.36 node_exporter
   910 root      20   0  574280  13468   2160 S  0.0  1.4   0:01.09 tuned
  2879 tidb      20   0   18280  10104   4080 S  0.0  1.0   0:00.43 blackbox_export
  ...
  
[root@tidb-pd2 ~]# top
top - 09:31:30 up  1:33,  1 user,  load average: 0.00, 0.03, 0.05
Tasks: 115 total,   1 running, 114 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.7 us,  0.7 sy,  0.0 ni, 98.3 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :   995676 total,    74936 free,   511512 used,   409228 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   326368 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
  4513 tidb      20   0  359388 241488  18120 S  1.0 24.3   0:15.65 prometheus
  4453 tidb      20   0 9854.4m  99200  37692 S  1.0 10.0   0:09.05 pd-server
  4568 tidb      20   0  815852  55604  14836 S  0.3  5.6   0:05.74 grafana-server
   905 root      20   0  574280  17352   6040 S  0.0  1.7   0:01.07 tuned
  4636 tidb      20   0  122376  15196   8332 S  0.3  1.5   0:00.49 alertmanager
  4689 tidb      20   0  113804  14648   5376 S  0.0  1.5   0:02.00 node_exporter
   662 polkitd   20   0  612232  12128   4684 S  0.0  1.2   0:00.29 polkitd
  4748 tidb      20   0   18280  10420   4124 S  0.0  1.0   0:01.12 blackbox_export
  ...
  
[root@tidb-server ~]# top
top - 09:32:29 up  1:34,  1 user,  load average: 0.05, 0.07, 0.06
Tasks: 108 total,   1 running, 107 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 99.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   995676 total,   532640 free,   203396 used,   259640 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   645984 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
  2712 tidb      20   0  398152  65936  31280 S  1.0  6.6   0:07.46 tidb-server
   904 root      20   0  574280  17464   6152 S  0.0  1.8   0:01.10 tuned
  2812 tidb      20   0  113804  14752   5424 S  0.0  1.5   0:02.06 node_exporter
   653 polkitd   20   0  612232  12148   4680 S  0.0  1.2   0:00.19 polkitd
  2866 tidb      20   0   18280  10272   4072 S  0.7  1.0   0:00.81 blackbox_export
  ...
  
[root@tidb-kv1 ~]# top
top - 09:32:57 up  1:35,  1 user,  load average: 0.02, 0.04, 0.05
Tasks: 108 total,   1 running, 107 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.7 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   995676 total,    77252 free,   757612 used,   160812 buff/cache
KiB Swap:        0 total,        0 free,        0 used.    93048 avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
  2741 tidb      20   0 1605604 599776  16728 S  1.7 60.2   0:14.64 tikv-server
   898 root      20   0  574280  17268   5960 S  0.0  1.7   0:01.12 tuned
  2915 tidb      20   0  113804  14096   5344 S  0.0  1.4   0:02.15 node_exporter
   657 polkitd   20   0  612232  12108   4660 S  0.0  1.2   0:00.19 polkitd
   655 root      20   0  474228  10584   6608 S  0.0  1.1   0:00.46 NetworkManager
  2969 tidb      20   0   18280  10260   4020 S  0.3  1.0   0:00.85 blackbox_export
  ...
```

可以看到TiKV节点的内存消耗最高。PD节点中，tidb-pd2节点由于安装了prometheus、grafana、alertmanager，资源消耗相对另外两个PD节点更高。另外，每个节点都有的`node_exporter`进程是prometheus用来在被监控端采集监控数据用的。

# 连接到TiDB Server
可能需要先安装mysql客户端：`yum install mysql -y`。

```bash
[root@tidb-pd1 ~]# mysql -h${tidb-server_IP} -P4000 -uroot

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 5.7.25-TiDB-v5.0.0 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| METRICS_SCHEMA     |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+
5 rows in set (0.00 sec)

# 修改数据库密码，可用于登录TiDB Dashboard
MySQL> alter user 'root'@'%' identified by 'newPassword';
```


# 关闭集群
关闭当前TiDB集群
```bash
tiup cluster stop tidb-test
```

集群停止后，Grafana页面和TiDB Dashboard都将无法访问。

检查tidb-test集群情况
```bash
tiup cluster display tidb-test
```

重新启动集群
```bash
tiup cluster start tidb-test
```

检查tidb-test集群情况
```bash
tiup cluster display tidb-test
```

