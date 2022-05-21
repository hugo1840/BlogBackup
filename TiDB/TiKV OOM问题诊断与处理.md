@[TOC](TiKV OOM问题诊断与处理)

# TiKV OOM对业务的影响
- TiKV上的请求失败造成异常退出
- region leader重新选举
  - raft group开始选举新的region leader
  - 新的region leader上报信息给PD Server
- region cache频繁更新
  - 在访问TiDB Server的region cache时，出现TiKV rpc相关报错
  - 后台自动进行Backoff重试
  - PD将最新的region leader信息返回给TiDB Server的region cache

# TiKV OOM的诊断方法
- 日志
  - `dmesg -T | grep tikv-server`结果中有事故发生时间点附近的**OOM-killer**的日志；
  - **tikv.log**中事故发生后附近时间的`"Welcome to TiKV"`的日志，即TiKV Server发生重启。

- Grafana监控
TiKV Details $\rightarrow$ Cluster $\rightarrow$ Memory
内存使用量曲线不连续，可能是发生了宕机。

# 造成TiKV OOM的原因
1. RocksDB的block cache配置不当导致OOM；
2. Coprocessor收到大量大查询，返回的数据量太大，gRPC的发送速度跟不上Coprocessor往外输出数据的速度；
3. 其他进程占用太多内存。

## Block cache参数配置不当
查看Block cache参数配置有以下两种方法：
- Grafana监控：TiKV - Details $\rightarrow$ RocksDB KV $\rightarrow$ Block cache size
- 检查参数配置`storage.block-cache.capacity`，建议在系统总内存的**45%~60%**之间
```sql
show config where name='storage.block-cache.capacity';
```

从TiDB v4.0开始支持在线调整Block cache大小：
```sql
set config tikv storage.block-cache.capacity='8155MB';
```

## gRPC发送速度跟不上Coprocessor处理速度
对比以下两个监控的数据：
- TiKV Details $\rightarrow$ Coprocessor Overview $\rightarrow$ Total Response Size
查看Coprocessor产生的流量。
- Node_exporter $\rightarrow$ Network $\rightarrow$ Network IN/OUT Traffic
查看每个TiKV实例的网卡Outbound出站流量。

如果Coprocessor产出流量很高，同时网卡流量也几乎打满，就可以说明gRPC发送速度跟不上Coprocessor处理速度。

可以从以下两个方面缓解：
- SQL优化：借助TiDB Dashboard找到结果集比较大的SQL语句，从业务逻辑以及执行计划两个方面来进行优化；
- 网卡配置升级。

## 其他原因
1. Raftstore数据写入环境内存占用高；
TiKV Details $\rightarrow$ Service $\rightarrow$ Memory Trace (for TiDB version >= v5.1)
2. 目标服务器混部了其他组件且内存占用高。




