@[TOC](TiDB Server关键性能参数与优化)

# 操作系统参数
## CPU
**Dynamic Frequency Scaling**
CPU动态节能技术用于降低服务器功耗，通过选择系统空闲状态不同的电源管理策略，可以实现不同程度的降低服务器功耗。更低的功耗策略意味着CPU唤醒更慢、对性能影响更大。

DFS有以下五种模式：`performance|userspace|powersave|ondemand|conservative`。配置命令为：
```bash
$ cpupower frequency-set --governor performance
```
数据库服务器一定要设置为**performance**模式。

**NUMA Binding**
内存直接绑定在CPU上，CPU只有访问自身管理的内存物理地址时，才会有较短的响应时间。NUMA架构的服务器，通过绑定NUMA节点，尽可能地避免跨NUMA访问内存，提升tidb server性能。

可使用`numactl`命令绑定。

## 内存
**Transparent Huge Page**
对于数据库，不建议使用透明大页（THP）。因为数据库往往具有稀疏而不是连续的内存访问模式，且当高阶内存碎片化比较严重时，分配THP会出现比较大的延迟。若开启针对THP的直接内存规整功能，也会出现系统CPU使用率激增的现象，因此建议关闭THP。

**Virtual Memory Parameters**
内存参数`dirty_ratio`和`dirty_background_ratio`通常无需调整。对于高性能SSD，比如NVMe设备来说，降低其值有利于提高内存回收时的效率。

## 磁盘IO
**I/O Scheduler**
I/O调度程序确定IO操作何时在存储设备上运行以及运行多长时间。常用的IO Scheduler包括NOOP、CFQ、DEALINE。

TiDB中使用的是**NOOP**，其IO调度方式类似一个FIFO队列。

**Mount Parameters**
默认方式下，Linux中文件系统在文件被创建、访问、修改时都会记录一些时间戳，比如最近一次被修改的时间`mtime`、最近一次被访问的时间`atime`、最近一次文件状态发生改变的时间`ctime`，这在绝大部分场合都是没有必要的。

可以使用`noatime`和`nodiratime`禁止记录最近一次访问的时间戳。

# TIDB配置参数
## performance性能参数
**max-proc**
控制单个tidb server使用的CPU核数，在单台机器上部署多个tidb server时配置该参数可以限制tidb server使用的资源，避免对其他进程造成影响。

**token-limit**
可以同时执行请求的session数量。用于流量控制，避免并发请求数过多造成tidb server资源耗尽，导致服务无法响应。当session数量达到token-limit配置的值时，新的session仍然可以连进来，但是发送的请求会被阻塞。

**force-priority**
控制tidb server的优先级。设置后，所有请求都是使用该优先级来执行。如果tidb server主要处理响应时间要求比较高的请求，可以配置为`NO_PRIORITY`；反之，如果是主要处理响应时间要求比较低的请求（比如OLAP、导数操作），则可以配置为`LOW_PRIORITY`。

**committer-concurrency**
控制一个事务commit阶段的最大并发数量。对于一个大事务来说，提交事务需要向TiKV发送大量写请求。设置更大的并发量可以让事务提交更快完成，但也可能会造成TiKV瞬时压力过大、请求堆积，无法响应。

## TiKV Client相关参数
**grpc-connection-count**
设置tidb server和每个tikv之间的gRPC连接数量。当大量并发请求发送到一个gRPC连接时，单个gRPC连接串行发送请求，可能会成为瓶颈。当gRPC连接成为瓶颈时，设置更大的gRPC连接数可以提升性能，但代价是消耗更多系统资源。

## Prepared Plan Cache
**enabled**
设置为true表示开启执行计划缓存。开启后可以减少执行计划生成造成的计算开销，让同样类型的语句使用相同的执行计划，提升性能。代价是当数据或查询条件变化时，执行计划可能不会是最好的。

**capacity**
用来限制执行计划缓存的大小（缓存条数），避免占用过多的内存，如果应用使用了大量不同类型的请求，超过了capacity上限，plan cache的效果会大打折扣。

# TiDB系统参数
## Concurrency
系统资源空闲较多时，设置更高的并发度让资源利用更充分，可以提升整体性能。但是更高的并发会消耗更多的系统资源，在资源紧张时反而会降低整体性能。

| 属性 | 含义 |
| :--: | :--: |
| tidb_distsql_scan_concurrency | 控制TableScan和IndexScan算子的并发度 |
| tidb_index_lookup_concurrency | 控制IndexLookUp算子的并发度 |
| tidb_build_stats_concurrency | 控制Analyze执行的并发度，可能会影响在线业务的延迟 |
| tidb_hash_join_concurency | 控制HashJoin算子的并发度 |
| tidb_index_lookup_join_concurrency | 控制IndexLookUpJoin算子的并发度 |
| tidb_ddl_reorg_worker_cnt | 控制DDL加索引的并发度 |

## Batch Size
一个Chunk包含多行数据，SQL语句执行时，执行器以Chunk为单位来执行获取数据、表达式求值和Join等操作。当结果集较大时，更大的Chunk size可以提升性能；如果结果集很小，小于Chunk size，申请过大chunk的内存会造成性能损失。

`tidb_init_chun_size`用于设置chunk中初始的数据行数，默认为32。TiDB在读取结果集时会自适应地调整chunk size，但最大也不会超过`tidb_max_chunk_size`，其值默认为1024。

TiDB中表连接操作时，也是采用了多行数据的批处理，而不是一行一行地匹配。表连接批处理中使用的chunk大小由参数`tidb_inidex_join_batch_size`控制。

## Limit
**tidb_store_limit**
控制同时发往一个tikv节点的请求数量。避免单个tikv因为请求量过大，超过处理能力，导致大量请求超时或者返回错误。

**tidb_retry_limit**
控制乐观事务的重试次数。乐观事务在遇到事务冲突时会进行重试，但是重试次数过多反而会使冲突加剧，造成性能下降；而重试次数过少，则会造成事务成功率下降。

## Backoff
在请求遇到可重试的错误时，在重试前需要等待一段时间，这个时间设置得过大时，会增加延迟；如果设置得太小，会进行许多不必要的尝试，消耗过多的系统资源。

**tidb_backoff_weight**
backoff最大时间的权重，通过这个变量来调整最大重试时间。如果重试时间设置为15s，`tidb_backoff_weight`等于2，那么实际重试时间为30s。

**tidb_backoff_lock_fast**
读请求遇到锁的backoff时间。




