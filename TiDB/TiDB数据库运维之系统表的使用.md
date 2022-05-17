@[TOC](TiDB数据库运维之系统表的使用)

# 系统表的存放位置

- **mysql**存储TiDB系统表，如`mysql.user`等；
- **information_schema**查看系统元数据：
  - 与MySQL兼容的表，如`tables`、`processlist`、`columns`等；
  - 自定义的表，如`cluster_config`、`cluster_hardware`、`tiflash_replica`等；
- **metrics_schema**是基于Prometheus中TiDB监控指标的一组视图；
- **performance_schema**目前为了与MySQL兼容保留了大部分视图。


# TiDB常用集群信息系统表
## mysql数据库
下面的表中记录了用户账户以及相应的授权信息：

- `mysql.user`：用户账户、全局权限、以及其他一些非权限的列；
- `mysql.db`：数据库级别的权限；
- `mysql.tables_priv`：表级别的权限；
- `mysql.columns_priv`：列级别的权限（TiDB v5.3及之前版本不支持）。

mysql库中的其他重要系统表还包括：
- `mysql.GLOBAL_VARIABLES`：存储了TiDB的全局变量，支持对系统表中VARIABLE_VALUE的修改；
- `mysql.tidb`：以Key-Value形式存储了集群状态，通过修改VARIABLE_VALUE可以调整相关系统行为。

## information_schema数据库
### information_schema.cluster_info
系统表`information_schema.cluster_info`提供了集群当前的拓扑信息、各个节点的版本信息、版本对应的Git Hash、各节点的启动时间、各实例的运行时间。当集群进行升级或者打了Patch版本，确定各组件版本是否打好时，可以查询此表。

- `version`： 对应节点的语义版本号；
- `start_time`：对应节点的启动时间；
- `uptime`：对应节点已经运行的时间。

### information_schema.cluster_config
系统表`information_schema.cluster_config`用于获取集群当前所有组件实例的配置信息。TiDB v4.0后引入该系统表提高了易用性。

- `type`：节点类型，可以为tidb、pd或tikv；
- `instance`：节点的服务地址；
- `key`：配置项名称；
- `value`：配置项的值。

### information_schema.ddl_jobs
系统表`information_schema.ddl_jobs`为`admin show ddl jobs`命令提供了一个information_schema接口，方便我们通过SQL查询和过滤相关表近期的DDL变更情况。


# 运维常用系统表查询
## 系统慢日志查询
慢日志查询主要用到了information_schema中的`slow_query`系统表。

**场景一**：搜索某个用户的Top N慢查询

```sql
select query_time, query, user
from information_schema.slow_query
where is_internal=false       -- 排除TiDB内存的慢查询
and user = "username"
order by query_time desc limit 3;
```

**场景二**：根据**SQL指纹**（*digest*）搜索同类慢查询

1. 先根据最近慢语句List查找特定慢SQL：
```sql
select query_time, query, digest
from information_schema.slow_query
where is_internal=false
and time between '2021-09-01' and '2021-09-02'
order by query_time desc limit 1;
```

2. 再根据SQL指纹搜索同类慢查询：
```sql
select query_time, query, digest
from information_schema.slow_query
where digest="476dnxmjsagsjajk20djsui92";
```

**注**：SQL指纹指将一条SQL中的字面值替换成其他固定符号。可以用来做SQL脱敏或者SQL归类。
```sql
select * from t1 where id = ?;
```

**场景三**：搜索统计信息为pseudo的慢SQL
```sql
select query, query_time, stats
from information_schema.slow_query
where is_internal=false
and stats like '%pseudo%';
```

**注**: 如果慢查询日志中的统计信息被标记为pseudo，往往说明TiDB表的统计信息更新不及时，需要运行`analyze table`手动收集统计信息。

## 系统读写热点查询
读写热点查询主要用到了information_schema中的`tidb_hot_regions`和`tikv_region_peers`系统表。

**场景一**：查询当前读写热点表
```sql
select db_name, table_name, index_name,
       type,   -- 读写热点分类
       sum(FLOW_BYTES),   -- 每分钟流量
       count(1),
       group_concat(h.region_id),
       count(DISTINCT p.store_id),
       group_concat(p.store_id)
from TIDB_HOT_REGIONS h
join TIKV_REGION_PEERS p 
on h.region_id = p.region_id
and p.IS_LEADER = 1
group by db_name, table_name, index_name, type;
```

**场景二**：统计当前读写热点Store
```sql
select p.store_id, 
       sum(FLOW_BYTES),   -- store总流量
       count(1)           -- region数量
from TIDB_HOT_REGIONS h
join TIKV_REGION_PEERS p 
on h.region_id = p.region_id
and p.IS_LEADER = 1
group by p.store_id
order by 2 desc;
```

## SQL阻塞查询
TiDB v5.2 information_schema中新增了以下几个跟锁和阻塞相关的视图。

### DATA_LOCK_WAITS

- 集群上所有TiKV节点上当前**正在发生的**悲观锁等待；
- 仅拥有**PROCESS权限**的用户可以查询；
- 从所有TiKV节点实时获取；
- 如果集群规模很大、负载很高，查看该表时有造成性能抖动的潜在风险。

### DEADLOCKS

- 提供**当前TiDB节点上**最近发生的若干次死锁错误的信息；
- 默认容纳**最近10次**死锁错误的信息。

死锁问题排查：
```sql
select * from information_schema.deadlocks;
```

### TIDB_TRX

- 返回了当前所有TiDB上**正在执行的**事务信息；
- 仅拥有**PROCESS权限**的用户可以查询。

