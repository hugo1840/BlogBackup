---
tags: [ELK]
title: Elasticsearch重要配置参数
created: '2022-11-21T06:27:35.275Z'
modified: '2022-11-24T08:57:56.654Z'
---

Elasticsearch重要配置参数


# ES重要配置
**ES版本：7.9**

## 数据目录&日志目录
使用tar包安装ES时，data和logs目录默认会在`$ES_HOME`路径下创建。在升级的过程中，`$ES_HOME`下的文件有被删除的风险。因此，官方强烈建议**将ES数据目录和日志目录存放在`$ES_HOME`以外的位置**。

在配置文件`$ES_HOME/config/elasticsearch.yml`中定义数据目录和日志目录位置：
```yml
path:
  data: /var/data/elasticsearch
  logs: /var/log/elasticsearch
```

其中，数据目录支持多条路径，但是每个分片的数据只会存储在一个单独的路径下。
```yml
path:
  data:
    - /mnt/elasticsearch_1
    - /mnt/elasticsearch_2
    - /mnt/elasticsearch_3
```


## 集群名称
新的节点加入集群时会通过集群名称来识别要加入的集群。

在配置文件`$ES_HOME/config/elasticsearch.yml`中定义要创建/加入的集群名称：
```yml
cluster.name: es-cluster1-name
```

## 节点名称
节点名称在ES的许多API响应中都会用到，默认为服务器的主机名。

在配置文件`$ES_HOME/config/elasticsearch.yml`中自定义节点名称：
```yml
node.name: prod-app1-data1
```

## 网络地址绑定`network.host`
默认情况下，Elasticsearch只绑定到环回地址，即`127.0.0.1`和`[::1]`，适用于在单台服务器上运行单个ES节点（实例）的场景。

>:watermelon: 实际上，可以从单台服务器上同一个的`$ES_HOME`路径启动多个ES节点（需指定不同的数据目录和日志目录），并且这些节点会自动加入同一个集群。这种部署方式可以用于测试ES集群，但不建议在生产环境中使用这种配置。

为了与其他服务器上的节点形成集群，节点需要绑定到一个非环回地址。ES节点将绑定到该地址，并将其发布到集群中的其他节点。一般只需要配置`elasticsearch.yml`中的`network.host`参数，例如:

```yml
network.host: 192.168.1.10
```

除了**IP和主机名**以外，`network.host`参数还接受某些特殊值，例如：
- `_local_`：默认值，即环回地址`127.0.0.1`；
- `_site_`：系统上的任何本地站点地址，例如`192.168.0.1`；
- `_global_`：系统上的任何广域网地址，例如`8.8.8.8`；
- `_[networkInterface]_`：网络接口地址，例如`_en0_`；
- `0`：即`0.0.0.0`，绑定到所有网络接口。

**注意**：包含**冒号**的地址需要用引号包围起来，因为冒号是YAML中的特殊字符。


只要提供了自定义设置的`network.host`，Elasticsearch就会假定用户正在从开发模式转移到**生产模式**，并将许多系统启动检查从warning升级为exception。


## 集群发现配置
ES集群发现配置主要涉及到以下两个参数：

1. `discovery.seed_hosts`

当需要与其他主机上的节点组成集群时，应该使用静态的`discovery.seed_hosts`设置，来提供可以参与集群组建和选主的节点列表。这个设置值应该是一个包含集群中所有可以参与选主（*master-eligible*）的节点地址的YAML序列或数组。每个节点地址可以是IP地址，也可以是能够通过DNS解析为一个或多个IP地址的主机名。

```yml
discovery.seed_hosts:
   - 192.168.1.10:9300     # 端口可选，默认为9300
   - 192.168.1.11 
   - seeds.mydomain.com    # 当前节点会尝试发现该主机名解析出来的所有IP地址
   - [0:0:0:0:0:ffff:c0a8:10c]:9301     # IPv6地址必须用方括号包围起来
```

2. `cluster.initial_master_nodes`

当第一次启动一个全新的ES集群时，会有一个集群引导（bootstrap）步骤，用于确定在第一次选举中可以参与选主投票的节点集合。在开发模式中，无需集群发现配置，该步骤由节点本身自动执行。在**生产模式**下启动一个全新的集群时，必须通过配置`elasticsearch.yml`中的`cluster.initial_master_nodes`参数显式指定在第一次选举中参与选主投票的节点。在重启集群或者向现有集群添加新节点时，不应使用此设置。

```yml
discovery.seed_hosts:
   - 192.168.1.10:9300
   - 192.168.1.11
   - seeds.mydomain.com
   - [0:0:0:0:0:ffff:c0a8:10c]:9301
cluster.initial_master_nodes:     # 只需在第一个启动的节点上配置
   - master-node-a     # 与各节点的node.name保持一致
   - master-node-b
   - master-node-c
```


## 堆大小（Heap size）
默认情况下，ES会告诉JVM使用最小和最大大小为**1 GB**的堆。当迁移到生产环境时，配置堆大小以确保ES有足够的堆可用。ES将分配`jvm.options`中通过**Xms**（最小堆大小）和**Xmx**（最大堆大小）指定大小的整个堆。**Xms**和**Xmx**两个设置必须相等。

堆大小配置取决于服务器上可用的RAM:

- 将Xmx和Xms设置为**不超过物理RAM的50%**。ES还会将内存用于JVM堆以外的其他目的，因此需要预留空间。例如，ES使用堆外缓冲区进行高效的网络通信，依靠操作系统的文件系统缓存进行高效的文件访问，并且JVM本身也需要一些内存。因此，ES进程使用的内存超过Xmx设置所配置的限制是正常的。

- 将Xmx和Xms设置为不超过JVM用于压缩对象指针（oops）的阈值；确切的阈值会有所差异，但接近**32 GB**。可以通过在日志中查看形似如下的一行来验证是否在阈值以下:
```bash
heap size [1.9gb], compressed ordinary object pointers [true]
```

- 理想情况下，将Xmx和Xms设置为不超过基于零的压缩oops的阈值；确切的阈值会有所差异，但在大多数系统上**26 GB**是安全的，在某些系统上可能大到30 GB。可以通过使用JVM选项`-XX:+ UnlockDiagnosticsVMOptions -XX:+PrintCompressedOopsMode`启动ES来验证是否在这个阈值之下，并查找形似如下所示的行:
```bash
heap address: 0x000000011be00000, size: 27648 MB, zero based Compressed Oops
```
则显示启用了基于零的压缩oop。如果没有启用基于零的压缩oops，那么会看到如下一行:
```bash
heap address: 0x0000000118400000, size: 28672 MB, Compressed Oops with base: 0x00000001183ff000
```

总的来说，ES可用的堆越多，它可以用于内部缓存的内存就越多，但是留给操作系统用于文件系统缓存的内存就越少。而且，更大的堆会导致垃圾回收的时间更长。

ES的JVM配置文件包括`$ES_HOME/config/jvm.options`以及`$ES_HOME/config/jvm.options.d/`下的文件。

下面的配置将堆大小的最小和最大值都设置为2G：
```java
-Xms2g 
-Xmx2g
```

还可以通过环境变量`ES_JAVA_OPTS`来设置ES的堆大小：
```bash
ES_JAVA_OPTS="-Xms2g -Xmx2g" ./bin/elasticsearch 
ES_JAVA_OPTS="-Xms4000m -Xmx4000m" ./bin/elasticsearch
```


## JVM堆转储（Heap dump）路径
默认情况下，ES配置JVM将OOM异常的堆转储到默认数据目录（对于RPM和Debian包发行版，是`/var/lib/elasticsearch`；对于tar和zip归档发行版，为ES安装目录下的数据目录）。如果该路径不适合接收堆转储，则应该修改`jvm.options`中的`-XX:HeapDumpPath=…`。

如果指定了一个目录，JVM将根据运行实例的PID为堆转储生成一个文件名。如果指定的是固定文件名而不是目录，那么当JVM需要对OOM异常执行堆转储时，该文件必须**未**被事先创建，否则堆转储将失败。


## GC日志
ES默认会启用GC日志，相关配置在`jvm.options`中。GC日志与ES其他日志位于相同的默认位置。GC日志是循环写的滚动日志，默认配置每**64 MB**轮转一次，最多可占用**2 GB**的磁盘空间。

我们可以使用JEP 158: Unified JVM logging中描述的命令行选项来重新配置JVM日志。除非直接修改`jvm.options`中的默认配置，ES默认配置和用户配置都会被应用。如果要禁用默认配置，首先通过`-Xlog:disable`选项禁用日志记录，然后再使用命令行选项配置。这将禁用所有的JVM日志，因此一定要检查可用选项并启用所需的所有选项。

下面是ES 7.9的`jvm.options`文件中关于GC的部分配置：
```bash
## GC configuration
8-13:-XX:+UseConcMarkSweepGC
8-13:-XX:CMSInitiatingOccupancyFraction=75
8-13:-XX:+UseCMSInitiatingOccupancyOnly

## JDK 8 GC logging
8:-XX:+PrintGCDetails
8:-XX:+PrintGCDateStamps
8:-XX:+PrintTenuringDistribution
8:-XX:+PrintGCApplicationStoppedTime
8:-Xloggc:logs/gc.log         # GC日志位置
8:-XX:+UseGCLogFileRotation
8:-XX:NumberOfGCLogFiles=32   # GC日志文件数
8:-XX:GCLogFileSize=64m       # GC文件大小
```

## 临时目录`$ES_TMPDIR`
ES的启动脚本默认会在系统临时目录下创建一个私有的临时目录。在某些Linux发行版中，系统会定期清除`/tmp`目录下最近没有被访问过的文件和目录。如果ES长时间没有使用自己的临时目录，该私有临时目录可能会在ES运行时被删除，从而导致在后续尝试使用临时目录时出现问题。

如果使用`.deb`或`.rpm`包安装ES，并通过systemd管理运行ES，那么ES使用的私有临时目录将**不**受到文件系统定期清理的影响。

如果要在Linux上长时间运行`.tar.gz`发行版，那么应该考虑为ES创建一个专用的临时目录，该目录不能位于会定期清除旧文件的路径下。只有运行ES的用户才能访问该临时目录，并且在启动ES之前设置`$ES_TMPDIR`环境变量指向它。


## JVM致命错误日志
ES默认配置JVM将fatal error logs写入默认日志目录（对于RPM和Debian包发行版，是`/var/lib/elasticsearch`；对于tar和zip归档发行版，为ES安装目录下的数据目录）。这些是JVM在遇到特别严重的错误时产生的日志。如果这个路径不适合接收日志，应该修改`jvm.options`中的`-XX:ErrorFile=…`为其他路径。


# 操作系统重要配置
**Linux发行版本：Centos 7**

## 禁用SWAP
启用SWAP内存交换分区会给数据库的性能带来非常消极的影响，因此一般情况下数据库产品所部署的服务器都要永久禁用SWAP分区。

临时禁用SWAP分区（立即生效，重启后失效）：
```bash
swapoff -a
```

通过注释掉`/etc/fstab`中对应的行来永久禁用SWAP分区（需要重启生效）：
```bash
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

检查禁用SWAP分区的效果：
```bash
free -m
```


## 文件描述符nofile
ES会用到大量的文件描述符（file descriptor, FD）或文件句柄（file handle）。FD耗尽极有可能导致数据丢失。因此我们需要将ES用户可以打开的文件描述符的限制增大到**65536**或者更高。

检查ES用户的nofile配置：
```bash
su - es -c 'ulimit -Hn'   # 硬限制
su - es -c 'ulimit -Sn'   # 软限制
```

在系统配置文件`/etc/security/limits.conf`中为ES用户修改nofile配置：
```bash
es   soft   nofile   65535
es   hard   nofile   65535
```

## 虚拟内存
ES默认使用mmapfs目录来存储索引。Linux操作系统对mmapfs计数的默认限制太低，可能导致内存不足。

在系统配置文件`/etc/sysctl.conf`中添加以下配置来调大该限制：
```bash
# 修改配置文件
echo 'vm.max_map_count = 262144' >> /etc/sysctl.conf
# 使新增配置生效
sysctl -p
```

检查生效后的配置：
```bash
sysctl -a | grep max_map_count
```

## 线程数nproc
确保ES用户可以创建的线程数量至少为**4096**。

检查ES用户的nproc配置：
```bash
su - es -c 'ulimit -Hu'   # 硬限制
su - es -c 'ulimit -Su'   # 软限制
```

在系统配置文件`/etc/security/limits.conf`中为ES用户修改nproc配置：
```bash
es   soft   nproc   4096
es   hard   nproc   14989
```

## TCP重传超时
ES集群中的节点通过TCP连接进行通信，这些TCP连接一直保持打开状态，直到其中一个节点关闭或节点之间的通信因底层基础设施故障而中断。通过向通信的应用程序隐藏临时的网络中断，TCP可以在偶尔不可靠的网络上提供可靠的通信。在通知发送方任何问题之前，操作系统将多次重传丢失的消息。

大多数Linux发行版默认将丢失的包重传**15次**。重传的间隔时间以指数方式增长，所以这15次重传需要900多秒才能完成。也就是说Linux用这种方法检测网络分区或故障节点需要花费较长时间。而Windows默认只有5次重传，相当于约6秒的超时。

Linux默认设置允许在可能经历很长时间丢包的网络上进行通信，但是对于单个数据中心内部的生产网络来说，这种默认设置是不合理的。高可用性集群必须能够快速检测到节点故障，以便通过重新分配丢失的分片、重新路由搜索、以及主节点重新选举来迅速做出反应。因此Linux用户应该减少TCP重传的最大次数。

在系统配置文件`/etc/sysctl.conf`中添加以下配置来调大该限制：
```bash
# 修改配置文件
echo 'net.ipv4.tcp_retries2=5' >> /etc/sysctl.conf
# 使新增配置生效
sysctl -p
```

检查生效后的配置：
```bash
sysctl -a | grep tcp_retries2
```


**References**
【1】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/discovery-settings.html
【2】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/path-settings.html
【3】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/network.host.html
【4】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/jvm-options.html
【5】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/modules-network.html
【6】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/file-descriptors.html
【7】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/vm-max-map-count.html
【8】https://openjdk.org/jeps/158
【9】https://docs.oracle.com/en/java/javase/13/docs/specs/man/java.html#enable-logging-with-the-jvm-unified-logging-framework





