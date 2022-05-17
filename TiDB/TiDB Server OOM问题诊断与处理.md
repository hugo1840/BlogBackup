@[TOC](TiDB Server OOM问题诊断与处理)

# TiDB Server OOM对业务的影响
TiDB Server出现OOM问题会对业务造成如下影响：
- TiDB Server上的业务SQL会失败
- 业务响应时间升高
- 前端体验变差

# TiDB Server OOM的诊断方法
诊断TiDB Server是否出现OOM有以下几种方法：

1. 客户端应用报错
```
ERROR 2013(HY000): Lost connection to MySQL Server during query
```

2. 日志
- `dmesg -T | grep tidb-server`结果中有事故发生附近时间点的**OOM-killer**日志；
- **tidb.log**中事故发生后附近时间的`Welcome to TiDB`日志（即TiDB Server发生重启）；
- **tidb_stderr.log**中grep到`fatal error: "runtime: out of memory"`或者`cannot allocate memory`。

3. Grafana监控：TiDB $\rightarrow$ Server $\rightarrow$ Memory Usage
TiDB Server实例呈现内存迅速升高，然后急速下降。

4. Grafana监控：TiDB $\rightarrow$ Server $\rightarrow$ Uptime
和Memory Usage配合一起使用，确认TiDB Server实例的运行情况，观察是否在宕机重启同时有内存使用率飙高的情况。


# 造成TiDB Server OOM的原因
造成TiDB Server OOM的原因主要来自两个方面：
1. SQL语句的执行：比如SQL执行结果单次返回的数据量太大；
2. Golang的内存释放机制

## 诊断TiDB Server OOM的原因
检查TiDB Runtime $\rightarrow$ Memory Usage，确认造成TiDB Server OOM的原因：
- 黄色部分（*estimate-inuse*）是TiDB真正使用的内存；
- 蓝色（*estimate-garbage*）、橙色（*reserved-by-go*）和红色（*used-by-go*）与Go语言有关。

## 定位占用内存大的SQL
在确定OOM是由SQL语句执行造成的后，可以通过下面的方法来定位问题SQL。

- TiDB Dashboard慢查询
在TiDB Dashboard的**Slow Queries**面板中，将时间范围调到TiDB Server OOM前，在Columns中选择按照**Max Memory**进行排序，可以找到集群中消耗内存较大的SQL语句。
**注**：TiDB Dashboard中看到的SQL是已经执行完的语句。

- TiDB Dashboard SQL语句分析
在TiDB Dashboard的**SQL Statements**面板中，将时间范围调到TiDB Server OOM前，在Columns中选择按照**Mean Memory**进行排序，可以找到集群中消耗内存较大的SQL语句。

- TiDB Server日志中的expensive query
当一条语句在执行过程中达到资源使用阈值（执行时间`tidb_expensive_query_time_threshold`或者使用内存量`mem-quota-query`）时，TiDB会即时将这条语句的相关信息写入tidb-server日志。
```bash
more tidb.log | grep expensivequery 
# cost_time: 已经执行的时间
# mem_max: 日志输出时语句的内存使用量
```

# 缓解TiDB Server OOM的措施
可以从以下几个方面来缓解TiDB Server的OOM问题：

- SQL优化，减少非必要返回的数据量
- 减少大事务，将大事务拆分为小事务
- 调整TiDB Server相关参数，限制单条SQL内存的使用

## SQL优化
1. SQL业务逻辑优化
- TiDB Dashboard慢查询
- TiDB Dashboard SQL语句分析
- TiDB Server日志中的expensive query

2. 执行计划优化
- 优化器没有选择最优计划执行
- 未创建必要索引
- 执行计划准确，但内存占用高

## 拆分大事务
建议将大事务拆分为小事务，在一定程度上提高事务提交的成功概率。

## 调整TiDB Server参数
- `mem-quota-query`
单条SQL占用的最大内存，默认为**1G**，超过该值会触发`oom-action`。

- `oom-action`
单条SQL超出最大使用内存后的行为，默认为**cancel**，表示放弃执行。在单条SQL使用的内存达到`mem-quota-query`，且**没有足够的临时磁盘空间可用**时，会触发oom-action。

- `oom-use-tmp-storage`
设置在单条SQL使用内存超过`mem-quota-query`时是否使用临时磁盘空间。

- `tmp-storage-path`
临时磁盘空间的路径。

- `tmp-storage-quota`
临时磁盘空间的使用上限，单位为**byte**。

## 其他减少内存占用的方法
1. 调整Go内存释放策略
- Go语言的Runtime在释放内存返回到内核时，在Linux平台有以下两种策略：
  - MADV_DONTNEED
  - MADV_FREE
- 在TiDB Server启动前添加环境变量`GODEBUG=madvdontneed=1`。

2. 滚动重启TiDB Server





