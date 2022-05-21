@[TOC](TiDB数据库热点问题诊断与处理)

# 为什么要解决热点问题
分布式架构中各个组件间的理想状态是达到资源利用率相对均衡。以TiKV为例，如果出现热点问题，绝大多数Region leader都集中在一台TiKV服务器上，就会出现性能短板问题——单台服务器的读写压力很大，而其他服务器都处于IO空闲状态，导致资源利用的浪费。

# 热点问题产生的原因
## 写热点产生的原因
TiDB数据库中写热点产生的原因主要来自于数据组织模型。

TiDB数据库中，表中的每行数据是以键值对的形式存储在TiKV上的Region中的。每个Region的大小为96M。

参考文章[$\lceil$TiDB数据库schema设计之表结构设计$\rfloor$](http://t.csdn.cn/qrDMp)中Key的构成：

- 对于聚簇表，TiDB会将表的编号加上主键作为Key。由于行数据的存储顺序与主键顺序一致，聚簇表中（主键）使用自增ID时，在大量插入时会产生写热点问题。

- 对于非聚簇表，TiDB会自动为每行数据隐式生成一个RowID，将表的编号加上RowID作为Key。由于非聚簇表中使用隐式生成的自增RowID，在大量插入数据时，容易出现写热点问题。

## 读热点产生的原因
读热点产生的原因主要有以下几种：
- 高频访问的小表
- SQL执行计划不合理（比如缺索引导致的全表扫描）
- 具有顺序增长属性的索引扫描

# 定位热点问题
TiDB数据库中可以借助以下工具来定位热点：
1. TiDB Dashboard
- 流量可视化页面
- 慢查询页面
- SQL语句分析
2. Grafana监控
- TiKV-Trouble-Shooting
- TiKV-Details
- PD

## TiDB Dashboard流量可视化
TiDB Dashborad Key Visualizer流量可视化页面中，以时间为横轴，以**表和索引**为纵轴，显示了整体写入的流量随时间的变化情况。

热力图中某个坐标的详细信息，包含了数据库对象名称、类型、以及Region的start key和end key的范围。

热力图中，**颜色越明亮，表示流量越大**。结合纵轴坐标可以定位热点问题发生的表和索引对象。

## TiDB Dashboard SQL语句执行情况
在TiDB Dashboard的SQL Statements和Slow Queries页面，对选定时间段内的SQL语句按执行次数、内存使用、等待耗时进行排序，可以帮助定位导致热点问题的SQL语句。

Grafana监控中的内容很多，但是无法帮助我们定位到热点的具体Region。

# 热点问题处理
## 写热点打散的几种方法
### #1: SHARD_ROW_ID_BITS和PRE_SPLIT_REGIONS
在创建表时，使用**SHARD_ROW_ID_BITS**和**PRE_SPLIT_REGIONS**参数将数据打散到多个Region分片中。设置`SHARD_ROW_ID_BITS=n`后，TiDB会随机生成ROW_ID的前n位（bits），从而将行数据打散写入到${2^n}$个不同的分片中。

在创建表时设置`PRE_SPLIT_REGIONS=m`，表示预先创建${2^m}$个空Region。

```sql
---将数据打散到2^4=16个Region分片中
> create table t(c int) SHARD_ROW_ID_BITS=4;
> alter table t SHARD_ROW_ID_BITS=4;
```

**注**：SHARD_ROW_ID_BITS的值不应设置得多大，否则会导致RPC写请求放大。

### #2: 关键字AUTO_RANDOM
创建表时，关键字**AUTO_RANDOM**可以用来避免大批量写数据时因为整型自增主键列产生的热点问题。主键设置AUTO_RANDOM后会随机生成，能够保证非空唯一性，但没有自增属性。
```sql
> create table t(a bigint PRIMARY KEY AUTO_RANDOM, b varchar(255));
```

**注**：AUTO_RANDOM列的类型只能为`bigint`。主键类型为非整型时，可以使用上面第一种方法。主键类型为`NONCLUSTERED`时，即使是整型主键列，也不支持使用AUTO_RANDOM属性。

###  #3: 索引打散
通过下面的命令将表上的指定索引打散到多个Region中：
```sql
> SPLIT TABLE table_name INDEX idx_name BETWEEN () AND () REGIONS n;
```

### #4: 系统变量tidb_scatter_region
全局系统变量**tidb_scatter_region**适用于批量建表后批量插入数据的场景下打散Region，默认值为OFF。

TiDB 默认会在建表时为新表分裂 Region。开启该变量后，会在建表语句执行时，同步打散刚分裂出的 Region。适用于批量建表后紧接着批量写入数据，能让刚分裂出的 Region 先在 TiKV 分散而不用等待 PD 进行调度。为了保证后续批量写入数据的稳定性，建表语句会等待打散 Region 完成后再返回建表成功，建表语句执行时间会是该变量关闭时的数倍。

## 业务运行过程中写热点排查处理
**步骤1**：利用TiDB Dashboard流量可视化页面，观测写流量的情况。
判断流量明亮情况，记录有效信息，包括数据库对象的类型、名称以及对应Region的start_key和end_key。

**步骤2**：TiDB Dashboard慢查询 & SQL语句分析页面，确认对应数据库对象DML操作的类型。 

**步骤3**：登录数据库，确认目标对象schema的信息。
```sql
> show create table table_name;
```

**步骤4**：热点打散。
1. 如果是**行数据**，操作类型为**INSERT**
- 无自增主键，打散方式为：
```sql
alter table table_name shard_row_id_bits=n;
```
- 有自增主键，打散方式为：
```sql
alter table t modify a bigint auto_random(5);
```

2. 如果是**行数据**，操作类型为**Update/Delete**

手动打散Region的流程如下：
- 通过流量可视化菜单，可以定位目标Region ID
```bash
pd-ctl region key {start_key}
```

- 分裂Region
```bash
pd-ctl operator add split-region {region_id} --policy=approximate
```

- transfer分裂出的新Region到其他Store
  - 通过pd.log过滤出分裂后的新Region ID
  - transfer到资源利用率相对较低的其他Store
```bash
pd-ctl operator add add-peer {new_region_id} {target_store_id}
pd-ctl operator add transfer-leader {new_region_id} {target_store_id}
pd-ctl operator add remove-leader {new_region_id} {original_store_id}
```

从TiDB v4.0.13起，支持通过修改AUTO_RANDOM关键字打散Region：
```sql
alter table t modify a bigint auto_random(5);
```

3. 如果是**索引**
也可以参照上面利用pd-ctl手动打散Region的方法。还可以通过下面的命令来打散索引的Region分布：
```sql
SPLIT TABLE table_name INDEX idx_name BETWEEN () AND () REGIONS n;
```

## 读热点问题的排查处理
**步骤1**：利用TiDB Dashboard流量可视化页面，观测写流量的情况。
判断流量明亮情况，记录有效信息，包括数据库对象的类型、名称以及对应Region的start_key和end_key。

**步骤2**：TiDB Dashboard慢查询 & SQL语句分析页面，确认问题时间段数据库中SQL的执行频次、执行计划（扫描数据行数）。

### 场景一：小表频繁访问引起热点
**Load Base Split**
目前的Load Base Split的控制参数为**split.qps-threshold**（QPS阈值）和**split.byte-threshold**（流量阈值）。如果**连续10s内**，某个Region每秒的各类读请求之和都超过QPS阈值或流量阈值，那么就对此Region进行拆分。

目前默认开启Load Base Split，但配置相对保守，split.qps-threshold 默认为**3000**，split.byte-threshold默认为**30MB/s**。

方式1：
```sql
set config tikv split.qps-threshold=3000
set config tikv split.byte-threshold=30
```

方式2：
```bash
curl -X POST "http://ip:status_port/config" -H "accept: application/json" -d '{"split.qps-threshold":"3000"}'
curl -X POST "http://ip:status_port/config" -H "accept: application/json" -d '{"split.byte-threshold":"30"}'
```

### 场景二：SQL执行计划不合理
1. 缺少索引，出现不必要的全表扫描
2. 多个索引，但是优化器选择错误
- 检查表的统计信息健康程度
- 通过Hint或者执行计划绑定干预优化器

### 场景三：顺序增长属性字段索引范围扫描
**方法1：Load Base Split**
```sql
set config tikv split.qps-threshold=3000
set config tikv split.byte-threshold=30
```

**方法2：打散索引分布**
```sql
SPLIT TABLE table_name INDEX idx_name BETWEEN () AND () REGIONS n;
```






