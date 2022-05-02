@[TOC](MySQL一致性读与当前读)

在 MySQL 里，有两个“视图”的概念：

 - 一个是 **view**。它是一个用查询语句定义的虚拟表，在调用的时候执行查询语句并生成结果。创建视图的语法是 `CREATE VIEW` ，而它的查询方法与表一样。
 - 另一个是 InnoDB 在实现 MVCC 时用到的**一致性读视图**，即 **consistent read view**，用于支持 RC（Read Committed，读提交）和 RR（Repeatable Read，可重复读）隔离级别的实现。它没有物理结构，作用是事务执行期间用来定义当前事务“能看到什么数据”。

本文中之后的内容提到的“视图”均指代第二种视图。

# 一个例子
假设我们创建了下表

```sql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `k` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

INSERT INTO t(id, k) VALUES(1,1),(2,2);
```

有三个事务分别按如下顺序执行。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210318142223784.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

其中，事务C没有显式使用 start/commit 命令，说明该语句本身就是一个事务，完成后就会自动提交。

使用 start/begin 启动事务有两种情况：

 - 仅执行 `START TRANSACTION` 时，实际上在执行之后的第一个操作InnoDB表的语句时，事务才真正启动。一致性视图也是在执行第一个快照**读**语句时创建的；
 - 执行 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 时，会立即启动事务，并生成一致性读视图。

在上面的例子中，事务 A 查到的 k 的值是 1，而事务 B 查到的 k 的值是 3（而不是 2）。具体的原因分析如下。

# 视图数组与一致性读
**数据版本**
InnoDB 里面每个事务有一个唯一的**事务ID**，叫作 transaction id。它是在事务开始的时候向 InnoDB 的事务系统申请的，是按申请顺序严格**递增**的。而每行数据也都是有多个版本的。每次事务更新数据的时候，都会生成一个新的数据版本，并且把 transaction id 赋值给这个数据版本的事务 ID，记为 **row trx_id**。同时，旧的数据版本要保留，并且在新的数据版本中，能够有信息可以直接拿到它。即，数据表中的一行记录，可能有多个**版本** (row)，每个版本有自己的 row trx_id。

下图显示了一个记录被多个事务连续更新后的状态。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210319085503872.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

虚线框里是同一行数据的 4 个版本，即V1、V2、V3和V4。当前最新版本是 V4，k 的值是 22，它是被 transaction id 为 25 的事务更新的，因此它的 row trx_id 也是 25。三个虚线箭头U1、U2、U3，代表 undo log；而 V1、V2、V3 并不是物理上真实存在的，而是每次需要的时候根据当前版本（V4）和 undo log 计算出来的。

**一致性视图**
按照可重复读的定义，一个事务启动的时候，能够看到所有已经提交的事务结果。但是之后，这个事务执行期间，其他事务的更新对它不可见。在实现上， InnoDB 为每个事务构造了一个**数组**，用来保存这个事务启动瞬间，当前正在“**活跃**”的所有事务 ID。“活跃”指的就是，启动了但还没提交。==数组里面事务 ID 的最小值记为**低水位**，当前系统里面已经创建过的事务 ID 的最大值加 1 记为**高水位**==。<u>这个**视图数组**和高水位，就组成了当前事务的**一致性视图**（read-view）。而数据版本的可见性规则，就是基于数据的 row trx_id 和这个一致性视图的对比结果得到的。</u>

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210319090215906.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

上图中需要注意的是，由于事务ID是按申请的时间顺序严格递增的，但事务的提交时间却与事务ID没有确定关系。假设在一个事务启动时，已经存在事务ID分别为33、34、35的三个事务，其中35已经提交，但是33和34尚未提交，此时的活跃数组里包含的事务ID为 [33, 34]。按照定义，低水位为33，高水位为36。事务ID等于35的事务应该处于低水位和高水位之间，但却并不在活跃数组中！

对于当前事务的启动瞬间来说，一个数据版本的 row trx_id，有以下几种可能：

 1. 如果落在绿色部分，表示这个版本是已提交的事务或者是当前事务自己生成的，这个数据是可见的；
 2. 如果落在红色部分，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
 3. 如果落在黄色部分，那就包括两种情况：
    (a) 若 row trx_id 在数组中，表示这个版本是由还没提交的事务生成的，不可见；
    (b) 若 row trx_id 不在数组中，表示这个版本是已经提交了的事务生成的，可见。
    
**一致性读**
回到文章开头的例子。不妨假设：事务 A 开始前，系统里面只有一个活跃事务 ID 是 99；事务 A、B、C 的版本号分别是 100、101、102，且当前系统里只有这四个事务；三个事务开始前，(1,1）这一行数据的 row trx_id 是 90。

事务 A 的视图数组就是 [99,100], 事务 B 的视图数组是 [99,100,101], 事务 C 的视图数组是 [99,100,101,102]。事务A的查询逻辑如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210319102410293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

第一个有效更新是事务 C，把数据从 (1,1) 改成了 (1,2)。此时，这个数据的最新版本的 row trx_id 是 102。第二个有效更新是事务 B，把数据从 (1,2) 改成了 (1,3)。此时，这个数据的最新版本（即 row trx_id）是 101。在事务 A 查询的时候，其实事务 B 还没有提交，但是它生成的 (1,3) 这个版本已经变成当前版本了。

事务 A 开始读数据时的视图数组是 [99,100]，查询语句的读数据流程如下：

 1. 找到最新版本 (1,3) ，判断出 row trx_id = 101，比高水位大，处于红色区域，不可见；
 2. 找到上一个历史版本，一看 row trx_id = 102，比高水位大，处于红色区域，不可见；
 3. 再往前找，找到了（1,1)，它的 row trx_id = 90，比低水位小，处于绿色区域，可见。

这样执行下来，虽然期间这一行数据被修改过，但是事务 A 不论在什么时候查询，看到这行数据的结果都是一致的，称之为**一致性读**。

# 更新逻辑：当前读
在文章开头的例子中，如果按照一致性读，事务 B 的查询结果不应该是1吗？其实，之所以得到的结果是3，是因为我们先执行了更新操作。实际上，如果在 UPDATE 操作之前查询，得到的 k 值确实是1。但是，当要更新数据的时候，就不能再在历史版本上更新了，否则事务 C 的更新就丢失了。因此，事务 B 此时的 `set k=k+1` 是在（1,2）的基础上进行的操作。

即，更新数据都是先读后写的，而这个读，只能读当前的值，称为“**当前读**”（current read）。事务 B 在更新的时候，当前读拿到的数据是 (1,2)，更新后生成了新版本的数据 (1,3)，这个新版本的 row trx_id 是 101。随后，在执行查询语句的时候，事务 B 看到自己的版本号是 101，最新数据的版本号也是 101，是自己的更新，可以直接使用，所以查询得到的 k 的值是 3。

除了 update 语句外，select 语句如果加锁，也是当前读。下面这两个 select 语句，就是分别加了读锁（S 锁，共享锁）和写锁（X 锁，排他锁）。

```sql
mysql> select k from t where id=1 lock in share mode;
mysql> select k from t where id=1 for update;
```

假设事务 C 不是马上提交的，而是变成了下面的事务 C’，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210319110338877.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)
事务 C’的不同是，更新后并没有马上提交，在它提交前，事务 B 的更新语句先发起了。此时，两阶段锁协议就起作用了。事务 C’没提交，也就是说 (1,2) 这个版本上的写锁还没释放。而事务 B 是当前读，必须要读最新版本，而且必须加锁，因此就被锁住了，必须等到事务 C’释放这个锁，才能继续它的当前读。

可重复读的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。

# 提交读
提交读的逻辑和可重复读的逻辑类似，它们最主要的区别是：

 - 在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图；
 - 在提交读隔离级别下，每一个语句执行前都会重新算出一个新的视图。因此在此隔离级别下，无法使用 `start transaction with consistent snapshot` 命令。

下面是读提交时的状态图，可以看到这两个查询语句的创建视图数组的时机发生了变化，就是图中的 read view 框。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210319111308577.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)
事务 A 的查询语句的视图数组是在执行这个语句的时候创建的，时序上 (1,2)、(1,3) 的生成时间都在创建这个视图数组的时刻之前。但是，(1,3) 还没提交，属于情况 1，不可见；(1,2) 提交了，属于情况 3，可见。因此，事务 A 查询语句返回的是 k=2，事务 B 查询结果 k=3。

# 总结
InnoDB 的行数据有多个版本，每个数据版本有自己的 row trx_id，每个事务或者语句有自己的一致性视图。普通查询语句是一致性读，一致性读会根据 row trx_id 和一致性视图确定数据版本的可见性。

 - 对于可重复读，查询只承认在事务启动前就已经提交完成的数据；
 - 对于提交读，查询只承认在语句启动前就已经提交完成的数据；
 - 对于当前读，总是读取已经提交完成的最新版本。

目前表结构不支持“可重复读”，这是因为表结构没有对应的行数据，也没有 row trx_id，因此只能遵循当前读的逻辑。


References
[1\] https://time.geekbang.org/column/article/70562
