@[TOC](TiDB数据库schema设计之表结构设计)

MySQL、Oracle、TiDB中支持的数据对象对比如下：

|  | Table | Partition | View | Index | Sequence | User Function | Procedure | Trigger |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| MySQL | 支持 | 支持 | 支持 | 支持 | 不支持 | 支持 | 支持 | 支持 |
| Oracle | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 |
| TiDB | 支持 | 部分支持 | 部分支持 | 支持 | 支持 | 不支持 | 不支持 | 不支持 |


## Schema的KV映射原理

TiDB中的数据在RocksDB中是以KV键值对的方式存储的。

TiDB中的表可以分为**聚簇表**（*clustered table*）和**非聚簇表**（*non-clustered table*）。聚簇是指以某个列为基准，把拥有相同聚簇键值的所有行都存储在相同位置上的物理存储方法。在指定的聚簇中只创建一个表的聚簇结构叫做单表聚簇。聚簇表中的所有数据是按照聚簇索引（主键）的顺序排列的。

- 对于聚簇表，TiDB会将**表的编号**加上**主键**作为Key，其余列作为Value；
```
# 假设Col1是主键列（聚簇索引）
Key: tablePrefix{tableID}_recordPrefixSep{Col1}
Value:{Col2,Col3,Col4}
```

- 对于非聚簇表，TiDB会自动为每行数据隐式生成一个RowID，将**表的编号**加上**RowID**作为Key，Value中包含所有列的数据。
```
Key: tablePrefix{tableID}_recordPrefixSep{_tidb_RowID}
Value:{Col1,Col2,Col3,Col4}
```

在有些文档中，也会把聚簇表称为**聚簇索引表**、或者**索引组织表**（*index-organized table*）。

数据存储管理的基本单位是**Region**。

- 每一个Region的默认大小是**96M**；
- 每一个Region按照左闭右开的区间划分数据存储范围，例如`[a,b)`；
- 每一个Schema会被分配一个唯一的`TableID`。


## 聚簇表和非聚簇表
聚簇表具有以下特点：

- 表中的行数据的存储顺序与主键的存储顺序一致；
- 表的主键是KV映射中Key的一部分；
- 通过主键访问行记录时，可以直接获取行数据。

TiDB中创建聚簇表时，需要**将主键指定为聚簇索引**：

```sql
create table User(
  ID int not null PRIMARY KEY Clustered,
  Name varchar(20),
  Role varchar(20),
  Age int,
  KEY idxAge(Age)
);
```

非聚簇表具有以下特点：

- 表中的行数据存储顺序与主键的存储顺序不一定一致；
- 行数据的Key由TiDB内部隐式分配的`_tidb_rowid`构成，而主键本质上是唯一索引；
- 通过主键访问行记录时，不可以直接获取行数据，需要先从额外存储的主键获取行的`_tidb_rowid`，再通过RowID获取行数据。因此要比聚簇表多一次回表操作。

TiDB中创建非聚簇表的语法如下：

```sql
create table User(
  ID int not null PRIMARY KEY nonClustered,
  Name varchar(20),
  Role varchar(20),
  Age int,
  KEY idxAge(Age)
);
```

查询是否为聚簇表的几种方法如下：

```sql
> show create table User;
> show index from User;
> select table_name, tidb_pk_type from information_schema.tables 
where table_schema='库名' and table_name='表名';
```

非聚簇表中，支持在创建表之后添加或删除非聚簇索引（主键）。此时可以选择显式指定NONCLUSTERED关键字，也可以省略关键字。

```sql
> alter table t1 add PRIMARY KEY(b,a) NONCLUSTERED;
> alter table t2 add PRIMARY KEY(b,a);  --不指定关键字时，为非聚簇索引
> alter table t1 drop PRIMARY KEY;
> alter table t2 drop index `PRIMARY`;
```

目前TiDB不支持在聚簇表中添加或删除聚簇索引，也不支持聚簇索引和非聚簇索引的相互转化。


## 非聚簇表的写热点问题
由于非聚簇表中使用隐式生成的自增RowID，在大量插入数据时，容易出现写热点问题。

在创建表时使用`SHARD_ROW_ID_BITS`参数可以调整生成的`_tidb_rowid`的高位，将数据写入拆分到不同的分片中来打散热点；同时，配合使用`PRE_SPLIT_REGIONS`参数，将表拆分为多个Region。

**示例**：创建一个非聚簇表，并将其拆分为16个分片、4个Region
```sql
create table t3 (
  c int PRIMARY KEY NONCLUSTERED,
  b varchar(20)
) shard_row_id_bits=4 pre_split_regions=2;
```

**示例**：修改表的分片数为32个
```sql
alter table t3 shard_row_id_bits=5;
```

## 分区表
TiDB当前支持的分区类型有Range分区、List分区、List Columns分区、以及Hash分区。

Range分区、List分区、List Columns分区可以用于解决业务中大量删除带来的性能问题。Hash分区可以用于大量写入场景下的数据打散。

创建分区表时，**分区表的每个唯一索引或主键，都必须包含分区表达式中用到的所有列**。

```sql
create table t4(
  col1 int not null,
  col2 date not null,
  col3 int not null,
  UNIQUE KEY (col1, col2)
)
PARTITION BY HASH(col2)
PARTITIONS 4;
```

## TiDB的数据类型
TiDB支持除空间类型（*SPATIAL*）之外的所有MySQL数据类型，包括

- 数值型类型
- 字符串类型
- 时间和日期
- JSON类型

其中，数值型、以及绝大部分字符串类型列的默认值必须是常量。时间和日期类型列的默认值可以是函数，例如`NOW()`、`CURRENT_TIMESTAMP()`、`LOCALTIME()`、`LOCALTIMESTAMP()`。

常见的数据类型有CHAR、VARCHAR、BINARY、VARBINARY、TEXT、BLOB、FLOAT、DOUBLE、INT、BIGINT等。

**注**：Blob、Text以及JSON类型不可以设置默认值。

## TIDB的自增ID
TiDB使用`AUTO_INCREMENT`关键字来标识自增列。

```sql
create table t5 (
  id int PRIMARY KEY AUTO_INCREMENT,
  c int
);
```

TiDB实现分配自增ID的原理如下：

1. 每一个自增列使用一个全局可见的键值对来记录当前已分配的最大ID；
2. 为了降低分布式系统分配自增ID的网络开销，每个TiDB节点会缓存一个不重复的ID段；
3. 当前预分配的ID使用完毕，或者TiDB重启，都会重新申请新的ID段。

**注**：TiDB重启后，缓存中未使用的自增ID即丢失，不会被使用，因此重启后新分配的ID段与表中已使用的最大ID之间是**不连续**的。

TiDB中自增ID有如下使用限制：

- **必须定义在主键或者唯一索引的列上**；
- 只能定义在类型为整数、Float或Double的列上；
- 不支持与列的默认值Default同时指定在同一列上；
- 不支持使用`alter table`来**添加**AUTO_INCREMENT属性；
- 需要通过session变量`@@tidb_allow_remove_auto_inc`来控制是否允许通过`alter table`来**移除**AUTO_INCREMENT属性；默认不允许移除列的自增属性。


## 聚簇表自增ID的写热点问题
由于行数据的存储顺序与主键顺序一致，**聚簇表**中（主键）使用自增ID时，在大量插入时会产生写热点问题。

关键字`AUTO_RANDOM`用于解决大批量写数据时因含有整型自增主键列的表而产生的热点问题。

```sql
create table t6 (a bigint PRIMARY KEY AUTO_RANDOM(3), b varchar(255));
```

AUTO_RANDOM的实现原理如下：

1. AUTO_RANDOM是一个8字节的`bigint`整数，其最高位为符号位；
2. 默认其63~59位为随机位（*shard bits*），每次插入行记录时随机生成一个1~32位的随机数；
3. 若要使用不同长度的随机位可以调整AUTO_RANDOM后面括号中的数字。

AUTO_RANDOM有如下使用限制：

- AUTO_RANDOM列的类型只能为`bigint`；
- 主键类型为**NONCLUSTERED**时，即使是整型主键列，也不支持使用AUTO_RANDOM属性；
- 不支持使用`alter table`来修改AUTO_RANDOM属性，包括添加和移除该属性；
- 不支持修改含有AUTO_RANDOM属性的主键列的类型；
- 不支持与AUTO_INCREMENT同时指定在同一列上；
- 不支持与列的默认值Default同时指定在同一列上；
- 插入数据时，不建议自行显式指定含有AUTO_RANDOM属性的列值。这可能会导致该表提前耗尽用于自动分配的数值。

通过下面的实验可以看出，AUTO_RANDOM不能保证自增属性，只能保证非空唯一属性。

```sql
--无需指定Clustered属性
> create table `t` (a bigint primary key auto_random);

-- 查看建表语句为聚簇表
> show create table `t`;
CREATE TABLE `t` (
  `a` bigint(20) NOT NULL /*T![auto_rand] AUTO_RANDOM(5) */,
  PRIMARY KEY (`a`) /*T![clustered_index] CLUSTERED */             
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4, COLLATE=utf8mb4_bin;

> insert into t values (),();
> select * from t;
+---------------------+
| a                   |
+---------------------+
| 8037454828582738833 |
| 8037454828582738834 |
+---------------------+

> insert into t values (),();
> select * from t;
+---------------------+
| a                   |
+---------------------+
| 1485769282949426453 |
| 1485769282949426454 |
| 8037454828582738833 |
| 8037454828582738834 |
+---------------------+
```

## Schema设计建议
### 高兼容性Schema
高兼容性Schema适合从原来的MySQL业务迁移到TiDB数据库上的表。建表时创建非聚簇表，并为表添加`shard_row_id_bits`和`pre_split_regions`表提示，其他列则保持原有设计。

```sql
create table noncluster_t (
  id bigint(20) unsigned auto_increment not null,
  code varchar(30) not null,
  create_time datetime default null,
  ...,
  primary key (id) nonclustered
) engine=InnoDB SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=3;
```

### 高性能Schema
高性能Schema适用于在TiDB上新创建的表，或迁移的表兼容以下改造；尤其需要注意原业务使用是否要求主键ID的单调连续性。

- 首先，建表时创建聚簇表；
- 主键使用具有较强随机性的列，或者使用自动生成的ID，并使用AUTO_RANDOM代替AUTO_INCREMENT来创建主键；
- 同时，准确选取列的数据类型，能使用整数型或日期类型的列，避免使用字符串类型；
- 最后，避免创建无效的索引。

```sql
create table cluster_t (
  id bigint(20) unsigned auto_random not null,
  code varchar(30) not null,
  create_time datetime default null,
  ...,
  primary key (id) clustered
) engine=InnoDB;
```
