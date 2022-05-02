@[TOC](TiDB中的RocksDB读写和Raft日志同步)

# RocksDB存储引擎
TiDB所使用的RocksDB是LSM类储存引擎之一。日志结构合并树（*Log Structured Merge Tree*, **LSM Tree**）类存储引擎的特点是写入的时候是**追加写入**（*append only*）。无论是INSERT、UPDATE、DELETE操作，都会被转化为追加写入操作。

基于这种特性，LSM Tree类存储引擎的**写入速度很快，但是读取速度慢**，且**写放大**和**读放大**现象比较突出。

## RocksDB写
RocksDB写入遵循“日志写先行”的规则，即先将**WAL日志**（*Write-Ahead Log*）落盘，再把数据写入内存中的**MemTable**，以防止宕机时丢失数据。

RocksDB写可以分为以下几个步骤：

1. 将WAL日志写入磁盘（`sync_log=True`）；
2. 将数据追加写入内存中的`MemTable`。如果是删除（或修改）数据的操作，则在MemTable中写入一条删除（或修改）标记，而不用去访问实际的数据，从而大大提高写的速度；
3. 当MemTable的大小达到`write_buffer_size`的大小（典型值是64KB）时，当前的MemTable被转存为内存中的一个**immutable MemTable**。同时，内存中会开辟出一块新的区域作为新的MemTable，供新的数据写入；
4. 后台有专门的进程将`immutable MemTable`刷到磁盘。如果immutable MemTable刷盘的速度明显慢于MemTable写入的速度，导致等待落盘的immutable MemTable积压达到了**5个**，会产生**Write Stall**现象，即TiDB会限制写入MemTable的速度；
5. 当数据成功落盘后，对应的WAL日志就可以被覆盖写了（即WAL日志是循环写）。

**注**：如果没有immutable MemTable，而是直接将MemTable写入磁盘，落盘的过程中可能会造成写阻塞。

RocksDB写入磁盘的数据是分level层次保存的**SSTable**。Level级数越小，表示处于该level的`SSTable`越新。同一level的数据量超出一定大小后会进行合并压缩，并被转化为下一层级，这一过程称为**Compaction**。在Compaction的过程中会对数据进行**去重、排序**。

1. immutable MemTable落盘后存储在Level 0层级的`SSTable`；
2. 当**Level 0**的SSTable数目达到**4个**时，该层的SSTable开始向Level 1做Compaction，被压缩合并为Level 1的一个SSTable，并在此过程中对其中所有的Key进行排序；
3. 当**Level 1**的所有SSTable中的总数据量达到**256M**时，该层的SSTable开始向Level 2做Compaction，被压缩合并为Level 2的一个SSTable，并在此过程中对其中所有的Key进行排序；
4. 当**Level 2**的所有SSTable中的总数据量达到**2.5GB**时，该层的SSTable开始向Level 3做Compaction，被压缩合并为Level 3的一个SSTable，并在此过程中对其中所有的Key进行排序；
5. 当**Level 3**的所有SSTable中的总数据量达到**25GB**时，该层的SSTable开始向Level 4做Compaction，被压缩合并为Level 4的一个SSTable，并在此过程中对其中所有的Key进行排序；
6. 当**Level 4**的所有SSTable中的总数据量达到**250GB**时，该层的SSTable开始向Level 5做Compaction。如此类推，不断向下面的Level推进。


## RocksDB读
相对B+树数据结构的存储引擎来说，RocksDB中的查询操作会慢一些。

RocksDB的内存中有一个叫做`Block Cache`的内存区域，缓存着最近最常被访问的数据。在读数据时，会先访问Block Cache，如果在Block Cache中找到了要读取的数据，这种情况就被称为**Block Cache命中**。

RocksDB读可以分为以下几个步骤：

1. 在`Block Cache`中查询要读取的数据；
2. 如果Block Cache未命中，则到`MemTable`中查询；
3. 如果在MemTable中没有读到数据，则到`immutable MemTable`中查询；
4. 如果所有immutable MemTable中都没有读到数据，则到磁盘的`Level 0`中按SSTable从新到旧的顺序查询；
5. 如果Level 0中没有读到数据，则到磁盘的`Level 1`中按SSTable从新到旧的顺序查询；
6. 如此递归向下，直到读到目标的Key。由于上层Level的数据肯定比下层Level的数据新，我们只要读第一次找到的Key就行。

**注**：由于从Level 1开始，SSTable中的数据是**按Key排好序的**，我们只需要看一个SSTable中的最小的Key和最大的Key，就可以判断想要查询的目标Key是否位于该SSTable中。如果目标Key位于该SSTable中，就可以通过二分查找法、BloomFilter等算法来查找。

# Raft日志同步
每一个Region及其副本构成一个**Raft Group**，其中的leader副本对外提供服务，可以读写。TiKV会将对数据的每个变更操作都转化为一条**Raft Log**，并将Raft日志从leader副本同步到follower副本。

Raft log格式的简单示例如下：
```bash
#Region编号_日志编号,log{操作类型 key=键值,value=变更}
4_1,log{PUT key=1,name='Tom'}
4_2,log{PUT key=2,name='Adny'}
4_3,log{PUT key=1,name='Jack'}
...
4_N,log{DEL key=3}
```

每个TiKV节点中有两个RocksDB实例，存储Raft日志的**RocksDB Raft**实例和存储KV键值对数据的**RocksDB KV**实例。

Raft日志同步可以分为以下几个步骤：

1. **Propose**：TiKV将收到的SQL请求转化为Raft日志；
2. **Append**：Leader副本将Raft日志持久化到本地的`RocksDB Raft`中（RocksDB写）；
3. **Replicate**：Leader副本将Raft日志发送给其他TiKV节点上自己的Follower副本。Follower副本在收到Raft日志后，也要持久化到自己本地的RocksDB Raft中（Append）；
4. **Committed**：Follower副本在将收到的Raft日志持久化到自己的本地存储后，会向Leader副本返回一个确认信息。当**超过半数**的副本（包括Leader副本在内）都完成Append后，Raft日志同步的状态变为Committed；
5. **Apply**：Leader副本将Raft日志应用到本地的`RocksDB KV`中（RocksDB写）。

最后的Apply步骤中，不保证Follower副本也已经将Raft日志应用到本地的RocksDB KV中。

**注**：Raft日志同步中的Committed状态不代表上层事务的Commited状态，也不等同于应用的Committed状态。

**References**
【1】[RocksDB原理及应用](https://zhuanlan.zhihu.com/p/409334218)
【2】[The Log-Structured Merge-Tree](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.44.2782&rep=rep1&type=pdf)

