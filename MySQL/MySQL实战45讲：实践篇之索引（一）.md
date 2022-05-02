@[TOC](MySQL实战45讲：实践篇之索引（一）)

# 基本概念
以 MySQL 8.0 官方语句为例

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    [(create_definition,...)]
    [table_options]
    [partition_options]
    [IGNORE | REPLACE]
    [AS] query_expression

create_definition: {
    col_name column_definition
  | {INDEX | KEY} [index_name] [index_type] (key_part,...)
      [index_option] ...
  | {FULLTEXT | SPATIAL} [INDEX | KEY] [index_name] (key_part,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] PRIMARY KEY
      [index_type] (key_part,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] UNIQUE [INDEX | KEY]
      [index_name] [index_type] (key_part,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] FOREIGN KEY
      [index_name] (col_name,...)
      reference_definition
  | check_constraint_definition
}

key_part: {col_name [(length)] | (expr)} [ASC | DESC]
```

**INDEX vs KEY**
>KEY is normally a synonym for INDEX.

根据 MySQL 8.0 官方文档，索引（index）与键（key）可以认为是同义词，无需区别。

**UNIQUE INDEX / KEY**

>A UNIQUE index creates a constraint such that all values in the index must be distinct.

唯一索引限制了索引中的所有值必须是唯一的，彼此不相等。
 
**PRIMARY KEY**
>A unique index where all key columns must be defined as NOT NULL. 
>A table can have only one PRIMARY KEY. 

主键是一种特殊的唯一索引，除了要求索引值的唯一性之外，还要求不能是 NULL 值。如果没有显式地定义为 `NOT NULL`，MySQL 会自动隐式地将主键定义为 `NOT NULL`。

 - 一张表只能有一个主键。
 - 主键的索引名（index_name）就是 PRIMARY，其他索引都不能使用 PRIMARY 作为索引名。对于普通索引，如果没有指定索引名，MySQL 会将添加索引的第一个列名作为索引名。
 - 如果表没有定义主键，而应用（application）要求表的主键时，MySQL 会返回第一个列值 not null 的唯一索引。
 - 对使用了 InnoDB 存储引擎的表，应该保持主键字段尽可能小，以减少普通索引的存储开销（普通索引的叶节点存储的是主键的值）。
 - 在创建的表中，最先存放主键索引，然后存放唯一索引，最后存放非唯一索引。
 - 主键可以是一个多列索引（multiple-column index），但是不能将主键作为一个列来创建一个多列索引。

**CLUSTERED INDEX vs SECONDARY INDEX**
聚簇索引（Clustered index）和二级索引（Secondary index）是 InnoDB 中的说法。

>Typically, the clustered index is synonymous with the primary key.
>All indexes other than the clustered index are known as secondary indexes.

聚簇索引就是主键索引。二级索引，也即普通索引，就是除主键索引之外的所有索引。

**FOREIGN KEY**
外键（Foreign key）允许在不同表之间交叉引用有相关性的数据。外键涉及一个定义了原始列的父表（Parent table）和一个引用了父表中原始列作为自己的一个列的子表（Child table）。在子表上定义有外键约束（Constraint）。

使用 InnoDB存储引擎的分区表（Partioned table）不支持外键。

# 普通索引和唯一索引怎么选择
## 查询过程
假设执行查询的语句是 `SELECT id FROM T WHERE k=5`。这个查询语句在索引树上查找的过程，先是通过 B+ 树从树根开始，按层搜索到叶节点。

 - 对于普通索引来说，查找到满足条件的第一个记录 (5,500) 后，需要查找下一个记录，直到碰到第一个不满足 k=5 条件的记录；
 - 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。
 
 然而，两种查询过程带来的性能差距微乎其微。InnoDB 存储引擎的数据是按数据页为单位来读写的。也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是**以页为单位**，将其整体读入内存。在 InnoDB 中，每个数据页的大小默认是 **16KB**。

因为引擎是按页读写的，所以说，当找到 k=5 的记录的时候，它所在的数据页就都在内存里了。那么，对于普通索引来说，要多做的那一次“查找和判断下一条记录”的操作，就只需要一次指针寻找和一次计算。

对于整型字段，一个数据页可以放近千个 key，因此出现 “k=5 这个记录刚好是这个数据页的最后一个记录” 这种情况的概率会很低。所以，我们计算平均性能差异时，仍可以认为这个操作成本对于现在的 CPU 来说可以忽略不计。

## 更新过程
>The **change buffer** is a special data structure that caches changes to secondary index pages when those pages are not in the buffer pool.
 
>The **buffer pool** is an area in main memory where InnoDB caches table and index data as it is accessed. The buffer pool permits frequently used data to be accessed directly from memory, which speeds up processing. On dedicated servers, up to 80% of physical memory is often assigned to the buffer pool.

当需要更新一个数据页时，如果数据页**在内存中**就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB 会将这些更新操作缓存在 **change buffer** 中，这样就不需要从磁盘中读入这个数据页了。在**下次查询**需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。

虽然名字叫作 change buffer，实际上它是可以持久化的数据。也就是说，change buffer 在内存中有拷贝，也会被写入到磁盘上。将 change buffer 中的操作应用到原数据页，得到最新结果的过程称为 **merge**。除了访问这个数据页会触发 merge 外，系统有后台线程会定期 merge。在数据库正常关闭（shutdown）的过程中，也会执行 merge 操作。

显然，如果能够将更新操作先记录在 change buffer，减少读磁盘，语句的执行速度会得到明显的提升。而且，数据读入内存是需要占用 **buffer pool** 的，所以这种方式还能够避免占用内存，提高内存利用率。

那什么条件下可以使用 change pool 呢？

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。比如，要插入 (4,400) 这个记录，就要先判断现在表中是否已经存在 k=4 的记录，而这**必须要将数据页读入内存**才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了。因此，唯一索引的更新就不能使用 change buffer，实际上也**只有普通索引可以使用**。

change buffer 用的是 buffer pool 里的内存，因此不能无限增大。change buffer 的大小可以通过 `innodb_change_buffer_max_size` 参数来动态设置。这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。

## change buffer 使用场景
普通索引的所有场景，使用 change buffer 都可以起到加速作用吗？

因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

因此，对于**写多读少**的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。


## 索引选择和实践
综上，唯一索引和普通索引在查询能力上是没差别的，主要考虑的是对更新性能的影响。所以，建议尽量选择普通索引。在实际使用中，普通索引和 change buffer 的配合使用，对于数据量大的表的更新优化还是很明显的。

特别地，在使用机械硬盘时，change buffer 这个机制的收效是非常显著的。所以，当你有一个类似“历史数据”的库，并且出于成本考虑用的是机械硬盘时，那你应该特别关注这些表里的索引，尽量使用普通索引，然后把 change buffer 尽量开大，以确保这个“历史数据”表的数据写入速度。

## innoDB 内存和磁盘存储结构
以后有时间补充。附上 MySQL 8.0版本的 InnoDB 存储架构。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210321211837723.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


# MySQL为什么有时会选错索引
见下一篇。

# 怎么给字符串字段加索引
见下一篇。


References：
[1\] https://time.geekbang.org/column/article/70848
[2\] https://time.geekbang.org/column/article/71173
[3\] https://time.geekbang.org/column/article/71492
[4\] https://www.cnblogs.com/zjfjava/p/6922494.html
[5\] https://dev.mysql.com/doc/refman/8.0/en/create-table.html
[6\] https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html
[7\] https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html
[8\] https://dev.mysql.com/doc/refman/8.0/en/innodb-system-tablespace.html
[9\] https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html
[10\] https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html

