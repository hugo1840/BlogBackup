﻿@[TOC](TiDB数据库常用高可用架构)

# 高可用架构设计中考虑的问题
- 网络延迟
  - Raft协议要求写入复制到最少两个节点（三副本）；
  - Leader有可能与发起读写的TiDB Server不在一个区域；
  - 读取要访问PD一次获取TSO，事务要获取两次；
- Raft协议本身
  - Raft协议要求写入复制到最少两个节点（三副本）；
  - 副本数最好为奇数（考虑Leader选举投票）；
  - 副本的分布最好与TiKV节点的分布相结合；
- 其他要求
  - 多活要求。

**注**：TiKV节点越多，并不代表可用性越高；Region副本的数量及其分布，才与可用性有关。

# 同城三中心架构
同城三数据中心方案，即同城存有三个机房部署TiDB集群，同城三数据中心间的数据同步通过集群自身内部Raft协议完成。同城三数据中心可同时对外进行读写服务，任意中心发生故障不影响数据一致性。

## 简易架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/3ad687ad4bdc47e999743a094485f7fc.png#pic_center)


**优点：**
- 所有数据的副本分布在三个数据中心，具备高可用和容灾能力；
- 任何一个数据中心失效后，不会产生任何数据丢失 (**RPO=0**)；
- 任何一个数据中心失效后，其他两个数据中心会自动发起leader election，并在合理长的时间内（通常情况**20s以内**）自动恢复服务。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e60076512a0a4b8eacb76c05d0f11843.png#pic_center)


**缺点：**
性能受网络延迟影响。具体影响如下：
- 对于写入的场景，所有写入的数据需要同步复制到至少2个数据中心，由于TiDB写入过程使用两阶段提交，故写入延迟至少需要2倍数据中心间的延迟。
- 对于读请求来说，如果数据leader与发起读取的TiDB节点不在同一个数据中心，也会受网络延迟影响。
- TiDB中的每个事务都需要向PD leader获取TSO，当TiDB与PD leader不在同一个数据中心时，它上面运行的事务也会因此受网络延迟影响，每个有写入的事务会获取两次TSO。

## 架构优化
如果不需要每个数据中心同时对外提供服务，可以将业务流量全部派发到一个数据中心，并通过调度策略把Region leader和PD leader都迁移到同一个数据中心。这样一来，不管是从PD获取TSO，还是读取Region，都不会受数据中心间网络的影响。当该数据中心失效后，PD leader和Region leader会自动在其它数据中心选出，只需要把业务流量转移至其他存活的数据中心即可。

![在这里插入图片描述](https://img-blog.csdnimg.cn/4970303bb388446487ab88efc8378da3.png#pic_center)


**优点：**
- 集群TSO获取能力以及读取性能有所提升。

**缺点：**
- 写入场景仍受数据中心网络延迟影响，这是因为遵循Raft多数派协议，所有写入的数据需要同步复制到至少2个数据中心；
- TiDB Server数据中心级别单点；
- 业务流量纯走单数据中心，性能受限于单数据中心网络带宽压力；
- TSO获取能力以及读取性能受限于业务流量数据中心集群PD、TiKV组件是否正常，否则仍受跨数据中心网络交互影响。

# 同城两中心架构
同城两中心部署方案，即在一座城市部署两个数据中心，成本更低，同样能满足高可用和容灾要求。该部署方案采用自适应同步模式，即*Data Replication Auto Synchronous*，简称**DR Auto-Sync**。

同城两中心部署方案下，两个数据中心距离在50km以内，通常位于同一个城市或两个相邻城市（例如北京和廊坊），数据中心间的网络连接延迟**小于1.5ms**，带宽**大于10Gbps**。

## 部署架构
以某城市为例，城里有两个数据中心IDC1和IDC2，分为位于城东和城西。

下图为集群部署架构图，具体如下：

- 集群采用同城两种中心部署方案，主数据中心IDC1在城东，从数据中心IDC2在城西。
- 集群采用推荐的4副本模式，其中IDC1中放2个Voter副本，IDC2中放1个Voter副本 + 1个Learner副本；TiKV按机房的实际情况打上合适的Label。
- 副本间通过Raft协议保证数据的一致性和高可用，对用户完全透明。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b11f5066a97241e68ba5bed466d3ba96.png#pic_center)


## 复制模式
该部署方案定义了三种状态来控制和标示集群的同步状态，该状态约束了TiKV的同步方式。集群的复制模式可以自动在三种状态之间自适应切换。

- `sync`：同步复制模式，此时从数据中心 (**DR**) 至少有一个副本与主数据中心 (**PRIMARY**) 进行同步，Raft算法保证每条日志 (log) 按Label同步复制到DR。
- `async`：异步复制模式，此时不保证DR与PRIMARY完全同步，Raft算法使用经典的majority方式复制日志。
- `sync-recover`：恢复同步，此时不保证DR与PRIMARY完全同步，Raft逐步切换成Label复制，切换成功后汇报给PD。

# 两地三中心架构
两地三中心架构，即生产数据中心、同城灾备中心、异地灾备中心的高可用容灾方案。在这种模式下，两个城市的三个数据中心互联互通，如果一个数据中心发生故障或灾难，其他数据中心可以正常运行并对关键业务或全部业务实现接管。相比同城多中心方案，两地三中心具有跨城级高可用能力，可以应对城市级自然灾害。

TiDB分布式数据库通过Raft算法原生支持两地三中心架构的建设，并保证数据库集群数据的一致性和高可用性。而且因同城数据中心网络延迟相对较小，可以把业务流量同时派发到同城两个数据中心，并通过控制Region Leader和PD Leader分布实现同城数据中心共同负载业务流量的设计。

## 部署架构
以北京和西安为例，北京有两个机房IDC1和IDC2，异地西安一个机房IDC3。北京同城两机房之间网络延迟**低于3ms**，北京与西安之间的网络使用ISP专线，延迟约**20ms**。

下图为集群部署架构图，具体如下：

- 集群采用两地三中心部署方式，分别为北京IDC1，北京IDC2，西安IDC3；
- 集群采用5副本模式，其中IDC1和IDC2分别放2个副本，IDC3放1个副本；TiKV按机柜打Label，既每个机柜上有一份副本。
- 副本间通过Raft协议保证数据的一致性和高可用，对用户完全透明。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3087496c21d44c4e9dc830c88f4c8892.png#pic_center)


该架构具备高可用能力，同时通过PD调度限制了Region Leader尽量只出现在同城的两个数据中心，这相比于三数据中心，即Region Leader分布不受限制的方案有以下优缺点：

**优点：**
- Region Leader都在同城低延迟机房，数据写入速度更优；
- 两中心可同时对外提供服务，资源利用率更高；
- 可保证任一数据中心失效后，服务可用并且不发生数据丢失。

**缺点：**
- 因为数据一致性是基于Raft算法实现，当同城两个数据中心同时失效时，因为异地灾备中心只剩下一份副本，不满足Raft算法大多数副本存活的要求。最终将导致集群暂时不可用，需要从一副本恢复集群，只会丢失少部分还没同步的热数据。这种情况出现的概率是比较小的；
- 由于使用到了网络专线，导致该架构下网络设施成本较高；
- 两地三中心需设置5副本，数据冗余度增加，增加空间成本。

# TiDB集群升级
## 异步复制
- 使用TiDB Binlog或者TiCDC组件进行异步复制；
- 会丢失数据（RPO不为0）；
- 有损恢复后，保证一致性；
- 主集群或从集群内部具有高可用功能。

## 集群升级方案
- 搭建一个从集群，将主集群的数据通过异步复制备份到从集群；
- 等数据差不多快追平时，找一个停业时间窗口，在该时间窗口内等从集群追平主集群后，将业务切换到从集群；
- 对主集群进行集群升级（目前TiDB集群升级无法回退）；
- 如果主集群升级成功，将从集群的数据通过异步复制传输到主集群；
- 等数据差不多快追平时，找一个停业时间窗口，在该时间窗口内等主集群追平从集群后，将业务切换到主集群。








