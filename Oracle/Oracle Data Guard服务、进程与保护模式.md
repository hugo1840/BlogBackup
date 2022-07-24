

@[TOC](Oracle Data Guard服务、进程与保护模式)

Oracle Data Guard可以：
- 提供企业级别数据库的高可用、数据保护、灾难恢复；
- 为生产环境中的数据库创建、管理、监视一个或多个备库（**Standby databases**）；
- 与传统的数据库备份、恢复技术一起使用来提供数据保护和数据可用性；
- 在Oracle Streams和Oracle GoldenGate中使用来提供高效可靠的Redo log传输。

# Data Guard主备库
架构上由一个主库（**Primary database**）和多个备库构成。Data Guard架构中的主备库之间通过Oracle Net通信，在地理位置上没有限制，主库和备库可以位于不同的数据中心。

Data Guard管理工具包括SQL命令行以及Data Guard Broker提供的**DGMGRL**命令行/图形化界面。

**主库**
Data Guard中的主库既可以是一个单实例的数据库，也可以是一个RAC集群（*Real Application Clusters*）。

**备库**
在Oracle 12.1中，支持为Data Guard的主库创建多达30个备库。备库创建完成后，Data Guard会自动将主库的Redo数据传输到备库，并在备库上应用。与主库相似，备库既可以是一个单实例的数据库，也可以是一个RAC集群。

Data Guard备库分为以下三种类型：

- 物理备库（**Physical Standby Database**）
  是主库的物理备份，是主库**数据块**（Block）的备份。备库的Schema，包括索引，都与主库完全一致。主库的Redo数据传输到物理备库后，通过**数据库恢复**功能被应用到备库。物理备库是**只读**的。
  
- 逻辑备库（**Logical Standby Database**）
  逻辑备库在磁盘上存储的数据结构可能与主库不同。主库的Redo数据传输到物理备库后，会被转化为**SQL语句**在备库执行。逻辑库是一个开放、独立的、活动的数据库，当重做数据通过SQL进行应用的时候可以进行报表查询。逻辑备库可以成生额外的索引和物化视图以获得更好的查询性能。
  
- 快照备库（**Snapshot Standby Database**）
  快照备库是完全可更新（fully updatable）的。闪照备库从主库接收Redo数据，但是并不应用收到的Redo数据，除非它被转化为一个物理备库。快照备库的应用场景是作为物理备库的一个临时的、可更新的快照副本。
  
# Data Guard服务
Data Guard中的Redo数据传输、Redo数据应用、数据库角色转换分别由不同的数据库服务实现。

## Redo Transport Services
Redo传输服务主要实现以下功能：

- 将Redo数据从主库传输到同一Data Guard配置中的备库；
- 处理由于网络错误导致的归档Redo log文件中的差距问题；
- 自动检测备库上缺失的、或者损坏的归档Redo log文件，并从主库或者其他备库上获取正确的Redo日志文件来替换。

## Apply Services
从主库接收的redo数据会被写入到备库上的备用redo日志文件中。Apply Services自动将接收的redo数据应用到备库，来与主库保持一致。

对于物理备库，Data Guard通过**Redo Apply**方法来使用标准的数据库恢复技术来应用redo数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/07d9f861f3e544099bc249b4712d4602.gif#pic_center)


**注**：Oracle 11g之前，物理备库在应用redo的时候，是不可以打开的，只可以mount。从11g开始，在应用redo的时候，物理备库可以处于read-only模式，这就称为**Active Data Guard (ADG)**。通过Active Data Guard，可以在物理备库进行查询或者导出数据，从而减少对主库的访问和压力。

对于逻辑备库，Data Guard通过**SQL Apply**方法来将接收的Redo数据转化为SQL语句来进行应用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/1a720b7ec3d542458547b6301f7ab2cd.gif#pic_center)



## Role Transitions
Data Guard中的主备数据库角色切换有两种方式：

- **Switchover**
  Switchover切换通常是有计划的切换，能够保证**不丢数据**（no data loss）。
- **Failover**
  Failover可以理解为故障转移切换，只会发生在**主库不可用**时。


# Data Guard中介
Data Guard Broker自动化了Data Guard配置过程中的创建、维护、监控。通过图形化界面或者DGMGRL命令行工具，可以实现：

- DG配置创建，包括redo传输服务和apply服务；
- DG配置管理监控，包括主备库为RAC的情况；
- 简化了Switchover和Failover的切换操作。

使用Broker的好处如下：

- 通过使配置和监视任务自动化，增强了Oracle Data Guard中固有的高可用性、数据保护和灾难保护功能；
- 简化了其中任一备用数据库替换主数据库并接管生产处理职责的过程；
- 实现了轻松配置额外备用数据库；
- 提供了简化的、扩展的集中式管理；
- 使用Oracle Net Services自动在Data Guard配置中的数据库之间进行通信；
- 提供了对配置中所有数据库的健康状况进行监视的内置验证功能。


# Data Guard保护模式
不同的应用场景下，业务对数据库可用性和数据丢失的容忍度有所差异。Data Guard有以下三种保护模式：

## 最大可用（Maximum Availability）
在不牺牲主库可用性的情况下，提供最高的数据保护。在DG中，除非至少有一个处于同步模式的备库接收到了主库的redo数据（写入了备库内存或者写入了备库的redo日志），否则主库事务事务不会提交。

如果没有一个备库能够正常接收主库的redo数据，主库就会以**最大性能模式**运行，以保证数据库的可用性，直到有备库能够重新正常接收主库的redo数据。

最大可用模式能够在绝大多数情况下保证数据零损失，除非在备库挂了之后主库也挂了。

## 最大性能（Maximum Performance）
最大性能模式是DG**默认**的保护模式，能够在不影响主库性能的前提下，提供最高级别的数据保护。在该模式下，事务在redo数据被写入到联机日志（online log）后即可提交。同时，redo数据也会被异步地（asynchronously）写到备库。

最大性能模式的数据保护性略差于最大可用模式，但是对主库性能的影响最小。

## 最大保护（Maximum Protection）
最大保护模式下，即使主库挂了，也能保证**数据零损失**。该模式下，只有在主库的redo数据被写入到联机重做日志（online redo log）后、且同时还被写入到至少一个处于同步模式的备库的redo日志中之后，事务才能提交。

如果没有一个备库能够正常接收主库的redo数据，主库就会关闭（shutdown），停止处理事务。


| 模式 | 数据丢失风险 | 传输 | 是否没有来自备库的确认 |
| :--: | :--: | :--: | :--: |
| 最大保护 | 零数据丢失<br>双重失败保护 | 同步 | 延迟主库，直到确认收到副本 |
| 最大可用 | 零数据丢失<br>单失败保护 | 同步 | 延迟主库，直到确认收到副本、或者发生超时，然后继续处理 |
| 最大性能 | 有少量数据丢失的可能 | 异步 | 主库不会等待备库的确认 |


# Data Guard后台进程
主库节点上，Redo传输服务主要使用以下几个进程：

- **LGWR**和**LNS**(log-write network-server)
  LGWR进程负责更新联机重做日志，LNS进程负责将redo数据传输到备库的RFS进程。同步模式下，LGWR在写Online Redo Log的同时，直接通过LNS进程将redo数据传输到备库的RFS进程；异步模式下，LGWR只管写联机重做日志，不管LNS进程什么时候把redo数据传输到备库。
  
- **ARCH**(archiver)
  在主库中，ARCHn进程可以在归档联机重做日志的同时，传递日志流到备库的RFS进程（ARCH进程和LNS进程不能同时传送redo数据到备库）。该进程还用于前瞻性检测和解决备库的日志不连续问题（Gap）。

- **FAL**(Fetch Archive Log)
  FAL进程用于解决检测在主库产生的连续的归档日志，而在备库接收的归档日志不连续的问题。该进程只有在需要的时候才会启动。


备库节点上，日志应用服务主要依靠如下进程：

- **RFS**(remote file server)
  RFS进程用于接收从主库的LNS进程或者ARCH进程传送过来的Redo数据，并将其写入备库重做日志（Standby Redo Log）。
  
- **ARCH**(archiver)
  备库中ARCH进程只对物理备库归档备库重做日志，这些日志以后将被MPR进程应用到备库。

- **MRP**(Managed Recovery Process)
  MRP进程只针对物理备库，负责将归档重做日志（Archived Redo Log）恢复到备库。如果执行`alter database recover managed standby database`，将作为前台进程来恢复归档日志；如果要在后台进程中恢复，需要额外加上`disconnect from session`。
  
- **LSP**(Logical Standby Process)
  LSP进程只针对逻辑备库，负责将归档日志转化为SQL语句并应用到备库。



**References**
【1】https://www.modb.pro/db/99314
【2】https://docs.oracle.com/database/121/SBYDB/concepts.htm#SBYDB00010



