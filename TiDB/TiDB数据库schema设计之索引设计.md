@[TOC](TiDB数据库schema设计之索引设计)

# 索引的KV映射原理
首先回顾一下表的行数据的KV映射原理：

对于聚簇表，TiDB会将**表的编号**加上**主键**作为Key，其余列作为Value。
```
# 假设Col1是主键列（聚簇索引）
Key: tablePrefix{tableID}_recordPrefixSep{Col1}
Value:{Col2,Col3,Col4}
```

对于非聚簇表，TiDB会自动为每行数据隐式生成一个RowID，将**表的编号**加上**RowID**作为Key，Value中包含所有列的数据。
```
Key: tablePrefix{tableID}_recordPrefixSep{_tidb_RowID}
Value:{Col1,Col2,Col3,Col4}
```

**唯一索引&非聚簇表的主键**
```
Key: tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue
Value: RowID
```

**二级索引**
二级索引的Key不一定唯一。如果二级索引是非唯一索引，其Value值为null；如果二级索引是唯一索引，其Value中存储的是主键索引。
```
Key: tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue_{RowID}
Value: null
```

下面是一个示例：
```sql
create table `EldenBoss` (
  ID int not null primary key nonclustered,
  name varchar(20),
  role varchar(20),
  age int,
  KEY idxAge(age)
);

insert into EldenBoss values (100, "Hoarah Loux", "Elden Lord", 56);
insert into EldenBoss values (200, "Mohg", "Hentai Noble", 28);
insert into EldenBoss values (300, "Malenia", "Goddess Warrior", 24);
```

该表中，数据的KV映射关系为：
```
t10_r1 --> [100, "Hoarah Loux", "Elden Lord", 56]
t10_r2 --> [200, "Mohg", "Hentai Noble", 28]
t10_r3 --> [300, "Malenia", "Goddess Warrior", 24]
```
主键的KV映射关系为：
```
t10_i1_100 --> 1
t10_i1_200 --> 2
t10_i1_300 --> 3
```
二级索引的KV映射关系为：
```
t10_i2_56_1 --> null
t10_i2_28_2 --> null
t10_i2_24_3 --> null
```

# 索引的设计
## 索引的创建
TiDB的索引创建语法与MySQL语法兼容。既可以在建表时一起创建，也可以在建表后单独添加。TiDB在创建索引时不会阻塞表的读写。

```sql
> create table `t1` (
  id int not null primary key auto_increment,
  c1 int not null,
  c2 int not null,
  key idx_t1_c1 (c1)
);

> alter table t1 add index idx_t1_c2(c2);
> alter table t1 add unique index uidx_t1_id(`id`);
```

TiDB目前还不支持在一个`alter table`语句中同时创建或修改多个索引。

## 联合索引
联合索引在查询时遵循**最左匹配原则**。
```sql
> create table `t2` (
  id int not null primary key auto_increment,
  c1 int not null,
  c2 int not null,
  c3 int not null,
  key idx_t2 (c1,c2,c3)
);
```

创建一个联合索引`(c1,c2,c3)`，相当于创建了`(c1)`、`(c1,c2)`、`(c1,c2,c3)`三个索引。

可以有效使用联合索引的场景：
```sql
> select * from t2 where c1 between 1 and 20;   -- (c1)
> select * from t2 where c1=20 and c2>5 and c2<10;   -- (c1,c2)
> select * from t2 where c1=5 and c2=10 and c3 between 1 and 50;   -- (c1,c2,c3)
```

可以部分使用联合索引的场景：
```sql
-- 只能使用(c1)，c1不是等值查询所以无法使用(c1,c2)
> select * from t2 where c1 between 1 and 20 and c2=30;   
-- 只能使用(c1,c2)，c2不是等值查询所以无法使用(c1,c2,c3)
> select * from t2 where c1=3 and c2>5 and c2<10 and c3=12;
```

不能使用联合索引的场景：
```sql
-- 查询条件不是以c1开始，不符合最左匹配原则
> select * from t2 where c2=30 and c3=10;
> select * from t2 where c3=100;
```

## 索引覆盖
索引覆盖是指当通过索引可以获取完整的行数据，那么可以直接通过遍历索引来取得数据，无需回表，从而减少IO。索引覆盖是提升数据库查询性能的主要手段之一。

```sql
-- c1,c2,c3列值都存储在联合索引的Key中
> select c1 from t2 where c1=1 and c2-100 and c3 between 10 and 50;
> select c1,c2,c3 from t2 where c1=100 and c2 between 1 and 200;
```

## 表达式索引
表达式索引是一种特殊的索引，能够将索引建立于表达式之上。表达式索引中的表达式需要用小括号包围起来，否则会报语法错误。

```sql
create index idx1 on t1 ((lower(col1)));
```

可以通过查询变量`tidb_allow_function_for_expression_index`知道哪些函数可以用于表达式索引：
```sql
> select @@tidb_allow_function_for_expression_index;
+--------------------------------------------+
| @@tidb_allow_function_for_expression_index |
+--------------------------------------------+
| lower,upper,md5,reverse,vitesse_hash       |
+--------------------------------------------+
```

当查询语句中的表达式与表达式索引一致时，优化器就可以为其选择使用表达式索引：

```sql
> select lower(col1) from t1;
> select * from t1 where lower(col1)="a";
> select * from t1 order by lower(col1);
> select min(col1) from t1 group by lower(col1); 
```

## 不可见索引
不可见索引（*invisible indexes*）可以在不删除索引的前提下对优化器“隐藏”索引。

- 不可见索引不会被查询优化器使用；
- 不可见索引仍然可以被修改或者删除；
- 即使用**SQL Hint USE INDEX**强制使用索引，优化器也**无法使用**不可见索引；
- 不允许将主键设为不可见；
- 通过`alter index`语句来修改索引的可见性，可以把索引设置为Visible或者Invisible。

```sql
> create table t2 (c1 int, c2 int, unique(c2));
> create unique index c1 on t2(c1) VISIBLE;
> alter table t2 alter index c1 INVISIBLE;
```

# 索引使用的注意事项
与MySQL相比，TiDB中的索引使用有如下限制：

- 不支持FULLTEXT、HASH和SPATIAL索引；
- 不支持降序索引（类似MySQL 5.7）；
- 无法向表中添加CLUSTERED类型的PRIMARY KEY；
- 不支持删除CLUSTERED类型的PRIMARY KEY；
- 不支持使用类似MySQL中提供的优化器开关`use_invisible_indexes=ON`来将所有不可见索引重新设置为可见。

# 运维技巧
查看索引的Region分布：
```sql
-- show table [table_name] index [index_name] REGIONS [WhereClauseOption];
> show table t1 index idx_t1_c1 regions;
```

