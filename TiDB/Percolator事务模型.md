@[TOC](Percolator事务模型)

Percolator是Google发表在OSDI 2010上的论文*Large-scale Incremental Processing Using Distributed Transactions and Notifications*中介绍的分布式事务模型。

Percolator本身是一个依赖BigTable作为数据存储引擎的分布式Key-Value数据库，是一个典型的通过2PC进行分布式事务协调的实现，简单来说是依靠Timestamp-Order实现了快照级别的事务。创新点在于，依靠对时间戳的使用和BigTable的单行事务，实现了跨行事务，更进一步说，由于某一行数据可以位于不同的BigTable的TabletServer之上，所以该事务也是跨节点的。

事务实现时主要参与者有三个：Percolator客户端、BigTable TabletServer、Timestamp Oracle(TSO)。按照2PC的要求分为两个步骤：PreWrite 和 Commit。

# 快照隔离级别
TiDB数据库中默认的事务隔离级别为**快照隔离级别**（Snapshot Isolation），事务只能看到**早于它开始时间之前提交**的其他事务。

如果事务采用了悲观锁模型，TiDB中的快照隔离级别跟MySQL中的可重复读隔离级别类似。

# 分布式时钟
常见的分布式授时方法有True Time多点物理时钟、HLC多点混合授时、以及TSO混合授时三种方法，代表产品分别有Google Spanner、CockroachDB、TiDB。

# Percolator事务执行流程
Percolator模型中事务实现时主要参与者有三个：Percolator客户端、BigTable TabletServer、Timestamp Oracle(TSO)。按照 2PC 的要求分为两个步骤：PreWrite和Commit。

- Client：客户端即**Coordinator**，负责协调事务的两阶段提交（TiDB Server）；
- TSO：全局授时器（PD）；
- BigTable：存储在各节点上的数据（TiKV）。

## Prewrite
BigTable中的每行数据是由三个列簇（Column Family, **CF**）存储的，即data、lock、write。

- write：数据列，一个写事务最终被成功提交后，相应的数据部分是存储于这个列中的；
- data：临时数据列，MVCC写过程中会将被修改的数据写入该列，视最终事务的结果是成功commit还是roll-backward、roll-forward来决定如何解释data列数据；
- lock：锁所在的列，某个事务在进行修改时会通过写入该列来锁住该行，在目前的实现下只要有发现某行上存在锁（任意时刻的锁）即需要终止本事务。

事务执行流程的**Prewrite**阶段包含以下步骤：
1. 事务在开始时刻，会通过Coordinator向TSO获取一个开始的时间戳`start_ts`。
2. 事务将要修改的第一行设置为主行，并对其加锁。事务会先读取被修改行的**lock列**，如果第一行已经有锁（比如正在被其他事务修改），即返回写写冲突报错。
3. 事务给其他要修改的行各都加上一把指向主行的“锁”。事务会先读取其他行的lock列，如果其他行上已经有锁，也返回写写冲突报错，并清除主行上的锁。
4. 当发现冲突并不存在之后，事务开始真正更新数据，将新的数据版本写入**data列**，之后将该行对应的lock列也更新。

## Commit
事务执行流程的**Commit**阶段包含以下步骤：
1. 事务通过Coordinator向TSO获取一个提交的时间戳`commit_ts`。
2. 事务提交主行修改，将新的数据版本写入**write列**，并清除主行上的锁。一旦主行完成提交，即认为事务提交成功。
3. 事务提交其它行的修改，将新的数据版本写入write列，并清除其他行上指向主行的锁。

如果其他行没有提交完成前客户端或者服务端就发生了crash，Percolator保证在读操作的实现时依然可以看到整个事务的完整性，并执行roll-forward将所有记录恢复以得到数据完整性。

如果在主行commit前服务器发生了crash，服务器恢复正常后整个事务就会rollback回滚。

# Percolator模型的优缺点
优点：
- 实现简单；
- 基于单行的事务基础上，实现了跨行事务；
- 去中心化的锁管理。

缺点：
- 需要管理中心化的版本号；
- 网络交互较多。


**References**
【1】[Percolator分布式事务模型原理与应用](https://zhuanlan.zhihu.com/p/307438297)
