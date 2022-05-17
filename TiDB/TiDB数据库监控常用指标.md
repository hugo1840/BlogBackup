@[TOC](TiDB数据库监控常用指标)

# TiDB Server相关监控
## 响应延迟 Duration
- Grafana监控TiDB $\rightarrow$ Query Summary $\rightarrow$ Duration
可以查看整个TiDB集群的所有SQL的平均响应时间。默认可以看到四条曲线，分别代表99.9%、99%、95%、80%四种分位线的SQL平均响应时间。

- Grafana监控TiDB $\rightarrow$ Query Summary $\rightarrow$ 999 Duration/99 Duration/95 Duration/80 Duration
上面监控中99.9%、99%、95%、80%四种分位线的细化，对不同SQL类型进行了区分，如Begin、Commit、Delete、DropTable、Insert、Execute、Replace、Rollback、Select、Show、Update、Use、Set，等等。

- Grafana监控TiDB $\rightarrow$ Query Detail $\rightarrow$ Duration 80/95/99/999 By Instance
上面监控中99.9%、99%、95%、80%四种分位线的细化，对不同实例进行了区分。如果某个TiDB Server的响应时间明显高于其他Server，可能需要排查负载均衡是否出现了问题。

## 每秒查询次数 QPS
- Grafana监控TiDB $\rightarrow$ Query Summary $\rightarrow$ QPS
可以查看不同时段的QPS总量（total）、以及不同SQL类型对应的QPS（Select/Insert/Set/Use/Other）。

- Grafana监控TiDB $\rightarrow$ Query Summary $\rightarrow$ CPS By Instance
可以查看不同TiDB Server实例的CPS（*Command-Per-Second*）。一个Command为一次发送的命令，可能包含多个SQL。

- Grafana监控TiDB $\rightarrow$ Query Detail $\rightarrow$ Internal SQL QPS
数据库内部SQL的QPS，一般不用关注。

## 事务 Transaction
- Grafana监控TiDB $\rightarrow$ Transaction $\rightarrow$ Duration
不同时段事务的平均响应时间，由**事务从应用程序发到TiDB Serve的时延、以及TiDB数据库内部处理事务的时间**两部分组成。该响应时间也会区分悲观事务和乐观事务，因此同一分位会包含两条曲线，如`99-optimistic`、`99-pessimistic`。

- Grafana监控TiDB $\rightarrow$ Transaction $\rightarrow$ Transaction Statement Num
不同时段执行的事务中包含的SQL语句量。不同颜色代表不同的事务。

- Grafana监控TiDB $\rightarrow$ Transaction $\rightarrow$ Transaction Retry Num
**只在乐观事务模式下有用**。显示了乐观事务在提交时因为遇到乐观锁而重试提交的次数。如果乐观事务重试的次数较高，很有可能是出现了写热点。

## 资源相关监控
- Grafana监控TiDB $\rightarrow$ Server $\rightarrow$ CPU Usage
各个TiDB Server在不同时段的CPU使用率。

- Grafana监控TiDB $\rightarrow$ Server $\rightarrow$ Memory Usage
各个TiDB Server在不同时段的内存使用率。

- Grafana监控TiDB $\rightarrow$ Server $\rightarrow$ Connection Count
各个TiDB Server在不同时段的连接数。TiDB Server默认不会限制连接数。

- Grafana监控TiDB $\rightarrow$ Server $\rightarrow$ Get Token Duration
连接到TiDB Server的会话只有在拿到Token对象后才能处理SQL，否则一直处于等待状态。TiDB Server能够同时处理SQL请求的会话的数量由参数`token_limit`控制。如果会话获取Token的等待时间很长，说明TiDB Server很忙，可能需要考虑增加TiDB Server的个数。

## 与PD/TiKV相关
- Grafana监控TiDB $\rightarrow$ PD Client $\rightarrow$ PD TSO Wait/RPC Duration
PD TSO Wait代表TiDB Server通过PD Client获取全局时间戳的等待延迟。PD TSO RPC Duration代表从PD获取全局时间戳的RPC传输时延。 PD TSO Wait中包含了PD TSO RPC Duration。

- Grafana监控TiDB $\rightarrow$ KV Request $\rightarrow$ KV Request Duration 99 By Store/Type
从TiDB Server发送读写请求到TiKV节点的响应时间。**By Store**中显示了不同TiKV节点的响应时间；**By Type**则是区分了不同请求类型的响应时间，比如Cop、Scan、ResolveLock、Commit、CheckTxnStatus、Prewrite、Get，等等。这里的耗时包含了两部分：**TiDB Server与TiKV之间的网络传输耗时、以及TiKV处理请求的耗时**。

- Grafana监控TiDB $\rightarrow$ KV Errors $\rightarrow$ KV Backoff OPS
从TiDB Server到TiKV获取KV数据的重试次数。Backoff重试的原因有很多，比如要访问的Region不在当前TiKV（*Region Miss*）、当前访问的TiKV节点繁忙（*Server Busy*）、PD出现故障（*PD RPC*）、访问TiKV时出现网络错误（*TiKV RPC*）、事务写冲突（*txnLock*）、事务读写冲突（*txnLock fast*）、Region leader发生选主切换（*update leader*），等等。

- Grafana监控TiDB $\rightarrow$ KV Errors $\rightarrow$ KV Backoff Duration
不同时段KV请求Backoff重试的总时间。


# TiKV相关监控
## 资源相关监控
- Grafana监控TiKV-Details $\rightarrow$ Cluster $\rightarrow$ Store/Available/Capacity Size
展示了不同TiKV节点的存储容量信息。Store Size为已使用的容量，Available Size为剩余可用的容量，Capacity Size为总容量。

- Grafana监控TiKV-Details $\rightarrow$ Cluster $\rightarrow$ CPU/Memory/IO Util
展示了不同TiKV节点的CPU、内存和IO使用率。

- Grafana监控TiKV-Details $\rightarrow$ Cluster $\rightarrow$ MBps/QPS
MBps展示了不同TiKV节点每秒钟的读写流量。QPS展示了TiKV节点每秒钟的访问量，可以区分不同的请求类型。

- Grafana监控TiKV-Details $\rightarrow$ Cluster $\rightarrow$ Region/Leader
展示了每个TiKV节点上Region和Leader的数量。如果单个TiKV节点上的Region数量超过了5万个，在发送心跳信号时可能会产生较大的性能压力。

## 线程池
- Grafana监控TiKV-Details $\rightarrow$ Thread CPU $\rightarrow$ **gRPC poll CPU**
展示了不同TiKV节点上负责处理接收到的所有请求的gRPC线程池的CPU利用率。一般不应该超过`server.grpc_concurrency`的80%。gRPC线程池会将接收到的请求分别转发至负责处理读请求和处理写请求的线程池。

- Grafana监控TiKV-Details $\rightarrow$ Thread CPU $\rightarrow$ **Unified read pool CPU**
展示了不同TiKV节点上Unified read线程池的CPU使用率。Unified read pool负责处理读请求。

下面三个线程池用于处理写请求。

- Grafana监控TiKV-Details $\rightarrow$ Thread CPU $\rightarrow$ **Scheduler worker CPU**
展示了不同TiKV节点上Scheduler worker线程池的CPU使用率。一般不应该超过`storage.scheduler_worker_pool_size`的90%。Scheduler worker线程池负责对写请求做冲突检测，把事务的两阶段提交、悲观锁加锁、事务回滚等请求转化为键值对，然后发送给Raft Store线程池。Raft Store线程池进一步将键值对转化为Raft Log。

- Grafana监控TiKV-Details $\rightarrow$ Thread CPU $\rightarrow$ **Raft store CPU**
展示了不同TiKV节点上Raft store线程池的CPU使用率。一般不应该超过`raftstore.store_pool_size`的80%。Raft Store线程池负责生成写请求的Raft log（**Propose**）、将Raft log持久化到本地（**Append**）、分发Raft log给其他的副本（**Replicate**），以及在多数派副本达成一致（**Commit**）后将Raft log发送的Async apply线程池。

- Grafana监控TiKV-Details $\rightarrow$ Thread CPU $\rightarrow$ **Async apply CPU**
展示了不同TiKV节点上Async apply线程池的CPU使用率。一般不应该超过`raftstore.apply_pool_size`的90%。Async apply线程池负责将Raft log转化为键值对并持久化到RocksDB KV中，并在完成写请求后通过回调函数通知gRPC线程池。

下图展示了TiDB写入流程中涉及的三个线程池以及对应的duration耗时阶段。
****
![在这里插入图片描述](https://img-blog.csdnimg.cn/953f31f7c4a642c6a79f6aa2154fafb1.png#pic_center)

****

## TiKV延迟 Duration
- Grafana监控TiKV-Details $\rightarrow$ gRPC $\rightarrow$ 99% gRPC message duration
展示了不同类型请求在TiKV端的总耗时。请求类型包括coprocessor、kv_prewrite、kv_pessimistic_lock、kv_commit、kv_scan、kv_get、kv_batch_get，等等。与TiDB Server监控中的**KV Request Duration 99 By Type**合起来看，可以用于排除是网络传输耗时多还是TiKV处理耗时多。

- Grafana监控TiKV-Details $\rightarrow$ Scheduler-commit $\rightarrow$ Scheduler command duration
Scheduler command duration较大时，写操作耗时长。可能是获取scheduler latch的等待时间长、或者storage async write的耗时过长。

下面的TiKV监控项可以用来排查写请求处理慢的问题。

- Grafana监控TiKV-Details $\rightarrow$ Scheduler-commit $\rightarrow$ Scheduler latch wait duration
Scheduler latch是为了让并发修改同一个Key的多个写请求串行执行。Scheduler latch wait duration较大时，可能是出现了多个并发写请求争抢同一个Key，也可能是Scheduler本身压力较大导致触发了流量控制（schedule writing bytes达到`storage.shceduler_pending_write_threshold`时会触发流控）。

- Grafana监控TiKV-Details $\rightarrow$ Raft propose $\rightarrow$ Propose wait duration
Propose wait duration较大时，可能是Raft store压力较大，导致Raft日志的Propose阶段等待耗时长。

- Grafana监控TiKV-Details $\rightarrow$ Raft IO $\rightarrow$ Append log duration
Append log duration较大时，说明Raft日志落盘耗时过长。

- Grafana监控TiKV-Details $\rightarrow$ Raft IO $\rightarrow$ Commit log duration 
Commit log duration包括了当前TiKV节点将Raft日志落盘的时间、以及将Raft日志分发给其他节点并等待多数节点回应的时间。Commit log如果耗时过长，可以适当调大`raft_max_inflight_msgs`参数。

- Grafana监控TiKV-Details $\rightarrow$ Raft propose $\rightarrow$ Apply wait duration 
Apply wait duration包含了Raft日志从Raft store发送到Async apply线程池，直到被真正应用之前的时间。Apply wait duration如果较大，说明Apply线程池繁忙，可以考虑增加Apply线程池的线程数。 

- Grafana监控TiKV-Details $\rightarrow$ Raft IO $\rightarrow$ Apply log duration
Apply log duration是Raft日志被应用的耗时。此处如果耗时较大，说明RocksDB KV写入慢，或者触发了流量控制。

## Server is busy
- Grafana监控TiKV-Details $\rightarrow$ Errors $\rightarrow$ Server is busy
Server is busy是TiKV自身的一种流控机制，在TiKV压力过大时会报告该错误。

# PD相关监控
## 重要监控项
- Grafana监控PD $\rightarrow$ PD Dashboard
Abnormal Stores中展示了处于异常状态的TiKV节点数目。正常情况下，Disconnect Stores、Unhealth Stores、Down Stores的数目应该为零。Offline Stores表示正在下线的TiKV节点数目；Tombstone Stores为已经下线的TiKV节点数。

- Grafana监控PD $\rightarrow$ Region Health
Region Health中展示了不同时段处于不同健康状态的Region数量。正常调度时可能会出现少量`extra-peer-region`和`learner-peer-region`。在清理大表后，会出现大量`empty-region`。

- Grafana监控PD $\rightarrow$ Statistics $\rightarrow$ Balance
Store region score、Store region size、Store region count展示了不同TiKV节点上的Region分布情况；Store leader score、Store leader size、Store leader count展示了不同TiKV节点上的Leader分布情况。

- Grafana监控PD $\rightarrow$ Statistics $\rightarrow$ Hot write region's leader/peer distribution
展示了不同时段写热点的分布情况。

- Grafana监控PD $\rightarrow$ Statistics $\rightarrow$ Hot read region's leader distribution
展示了不同时段读热点的分布情况。

## 告警级别
TiDB数据库支持三种告警级别：警告级别、严重级别、紧急级别。

