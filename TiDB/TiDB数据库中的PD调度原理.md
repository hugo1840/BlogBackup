@[TOC](TiDB数据库中的PD调度原理)

# 调度的产生与执行
宏观上来看，调度流程大体可划分为三个部分：

## 信息收集

TiKV节点周期性地向PD上报**StoreHeartbeat**和**RegionHeartbeat**两种心跳消息：
- StoreHeartbeat 包含Store（即TiKV）的基本信息、容量、剩余空间和读写流量等数据；
- RegionHeartbeat 包含Region的范围、副本分布、副本状态、数据量和读写流量等数据。

PD梳理并转存这些信息到**cluster cache**缓存中供调度进行决策。

## 生成调度

不同的调度器（**scheduler**）从自身的逻辑和需求出发，考虑各种限制和约束后生成待执行的**Operator**。这里所说的限制和约束包括但不限于：

- 不往处于异常状态中（如断连、下线、繁忙、空间不足或在大量收发snapshot等）的Store添加副本；
- Balance时不选择状态异常的Region；
- 不尝试把Leader转移给Pending Peer；
- 不尝试直接移除Leader；
- 不破坏Region各种副本的物理隔离；
- 不破坏Label property等约束。

## 执行调度

调度执行的步骤为：

1. Operator先进入一个由**OperatorController**（也叫coordinator）管理的等待队列；
2. OperatorController会根据配置以一定的并发量从等待队列中取出Operator并执行。执行的过程就是依次把每个Operator Step下发给对应Region的Leader；
3. 标记Operator为`finish`或`timeout`状态，然后从执行列表中移除。

# 常见的调度类型
## 负载均衡
Region负载均衡调度主要依赖**balance-leader**和**balance-region**两个调度器。二者的调度目标是将Region均匀地分散在集群中的所有Store上，但它们各有侧重：
- balance-leader关注Region的Leader，目的是分散处理客户端请求的压力；
- balance-region关注Region的各个Peer（即Follower），目的是分散存储的压力，同时避免出现爆盘等状况。

balance-leader与balance-region 有着相似的调度流程：

1. 根据不同Store的对应资源量的情况分别打分；
2. 不断从得分较高的Store选择Leader或Peer迁移到得分较低的Store上。

两者的分数计算上也有一定差异：balance-leader比较简单，使用Store上所有Leader所对应的**Region Size加和**作为得分。因为不同节点存储容量可能不一致，计算balance-region得分会分以下三种情况：

- 当空间富余时使用数据量计算得分（使不同节点数据量基本上均衡）；
- 当空间不足时由使用剩余空间计算得分（使不同节点剩余空间基本均衡）；
- 处于中间态时则同时考虑两个因素做加权和当作得分。

此外，为了应对不同节点可能在性能等方面存在差异的问题，还可为Store设置负载均衡的**权重**。`leader-weight`和`region-weight`分别用于控制Leader权重以及Region权重（**默认值都为1**）。假如把某个Store的leader-weight设为2，调度稳定后，则该节点的Leader数量约为普通节点的2倍；假如把某个Store的region-weight设为0.5，那么调度稳定后该节点的Region数量约为其他节点的一半。

## 热点调度
热点调度对应的调度器是**hot-region-scheduler**。从TiDB v3.0版本开始，统计热点Region的方式为：

1. 根据Store上报的信息，统计出持续一段时间读或写流量超过一定阈值的Region;
2. 用与负载均衡类似的方式把这些Region分散开来。

对于**写热点**，热点调度会同时尝试打散热点Region的Peer和Leader；对于**读热点**，由于只有Leader承载读压力，热点调度会尝试将热点Region的Leader打散。

## 集群拓扑感知
让PD感知不同节点分布的拓扑是为了通过调度使不同Region的各个副本尽可能分散，保证**高可用**和**容灾**。PD会在后台不断扫描所有Region，当发现Region的分布不是当前的最优化状态时，会生成调度以替换Peer，将Region调整至最佳状态。

负责这个检查的组件叫**replicaChecker**（跟Scheduler类似，但是不可关闭）。它依赖于`location-labels`配置项来进行调度。比如配置`[zone,rack,host]`定义了三层的拓扑结构：集群分为多个zone（可用区），每个zone下有多个rack（机架），每个rack下有多个host（主机）。PD在调度时首先会尝试将Region的Peer放置在不同的zone，假如无法满足（比如配置3副本但总共只有2个zone）则保证放置在不同的rack；假如rack的数量也不足以保证隔离，那么再尝试host级别的隔离，以此类推。

## 缩容及故障恢复
**缩容**是指预备将某个Store下线，通过命令将该Store标记为`Offline`状态，此时PD通过调度将待下线节点上的Region迁移至其他节点。

**故障恢复**是指当有Store发生故障且无法恢复时，有Peer分布在对应Store上的Region会产生缺少副本的状况，此时PD需要在其他节点上为这些Region补副本。

这两种情况的处理过程基本上是一样的。**replicaChecker**检查到Region存在异常状态的Peer后，生成调度在健康的Store上创建新副本替换异常的副本。

## Region merge
Region merge指的是为了避免删除数据后大量小甚至空的Region消耗系统资源，通过调度把相邻的小Region合并的过程。Region merge由**mergeChecker**负责，其过程与replicaChecker类似：PD在后台遍历，发现连续的小Region后发起调度。

具体来说，当某个新分裂出来的Region存在的时间超过配置项`split-merge-interval`的值（**默认1h**）后，如果出现以下任意情况，该Region会触发Region merge调度：

- 该Region大小小于配置项`max-merge-region-size`的值（**默认20MiB**）；
- 该Region中key的数量小于配置项`max-merge-region-keys`的值（**默认200000**）。

# 查询调度状态
## Operator状态
**Grafana PD/Operator**页面展示了Operator的相关统计信息。其中比较重要的有：

- *Schedule Operator Create*：Operator的创建情况；
- *Operator finish duration*：Operator执行耗时的情况；
- *Operator Step duration*：不同Operator Step执行耗时的情况。

查询Operator的pd-ctl命令有：

- `operator show`：查询当前调度生成的所有Operator；
- `operator show [admin | leader | region]`：按照类型查询Operator。

## Balance状态
**Grafana PD/Statistics - Balance**页面展示了负载均衡的相关统计信息，其中比较重要的有：

- *Store Leader/Region score*：每个Store的得分；
- *Store Leader/Region count*：每个Store的Leader/Region数量；
- *Store available*：每个Store的剩余空间。

使用pd-ctl的`store`命令可以查询Store的得分、数量、剩余空间和weight等信息。

## 热点调度状态
**Grafana PD/Statistics - hotspot**页面展示了热点Region的相关统计信息，其中比较重要的有：

- *Hot write Region's leader/peer distribution*：写热点Region的Leader/Peer分布情况；
- *Hot read Region's leader distribution*：读热点Region的Leader分布情况。

使用pd-ctl同样可以查询上述信息，可以使用的命令有：

- `hot read`：查询读热点Region信息；
- `hot write`：查询写热点Region信息；
- `hot store`：按Store统计热点分布情况；
- `region topread [limit]`：查询当前读流量最大的Region；
- `region topwrite [limit]`：查询当前写流量最大的Region。

## Region 健康度
**Grafana PD/Cluster/Region health**面板展示了异常Region的相关统计信息，包括Pending Peer、Down Peer、Offline Peer，以及副本数过多或过少的Region。

通过pd-ctl的`region check`命令可以查看具体异常的Region列表：

- `region check miss-peer`：缺副本的Region；
- `region check extra-peer`：多副本的Region；
- `region check down-peer`：有副本状态为Down的Region；
- `region check pending-peer`：有副本状态为Pending的Region。

# 调度策略控制
## 启停调度器
pd-ctl支持动态创建和删除Scheduler，可以通过这些操作来控制PD的调度行为，如：

- `scheduler show`：显示当前系统中的Scheduler；
- `scheduler remove balance-leader-scheduler`：删除（停用）balance region调度器；
- `scheduler add evict-leader-scheduler 1`：添加移除Store 1的所有Leader的调度器。

## 手动添加Operator
PD支持直接通过pd-ctl来创建或删除Operator，如：

- `operator add add-peer 2 5`：在Store 5上为Region 2添加Peer；
- `operator add transfer-leader 2 5`：将Region 2的Leader迁移至Store 5；
- `operator add split-region 2`：将Region 2拆分为2个大小相当的Region；
- `operator remove 2`：取消Region 2前待执行的Operator。

## 调度参数调整
使用pd-ctl执行`config show`命令可以查看所有的调度参数，执行`config set {key} {value}`可以调整对应参数的值。

### 产生速度控制
通过以下参数来限制Operator的产生速度：
- `leader-schedule-limit`：同时进行Leader调度的任务并发数，默认为4；
- `region-schedule-limit`：同时进行Region调度的任务并发数，默认为2048；
- `replica-schedule-limit`：同时进行Replica调度的任务并发数，默认为64;
- `merge-schedule-limit`：同时进行Region merge调度的任务并发数，默认为8。设置为0时，表示关闭；
- `hot-region-schedule-limit`：同时进行Hot region调度的任务并发数，默认为4。该配置项独立于Region调度。

### 消费速度控制
**store limit**可用于限制单个Store的Operator消费速度。通过以下命令来限制：
```bash
pd-ctl -u ip:port store limit <store_id> <value>
```

