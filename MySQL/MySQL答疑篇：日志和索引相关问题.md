@[TOC](MySQL答疑篇：日志和索引相关问题)

# 日志相关问题
在两阶段提交的不同瞬间，MySQL 如果发生异常重启，是怎么保证数据完整性的？

>**图 1：两阶段提交**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/74d1a8d3ca16aae5586a067a3c3f40d3.png#pic_center)
注意：图中的 commit 步骤是指事务提交过程中的最后一步，而不是指具体执行一个 commit 命令。

如果在图中**时刻 A** 的地方，也就是写入 redo log 处于 prepare 阶段之后、写 binlog 之前，发生了崩溃（crash），由于此时 binlog 还没写，redo log 也还没提交，所以崩溃恢复的时候，这个事务会**回滚**。

如果在**时刻 B**，也就是 binlog 写完，redo log 还没 commit 前发生 crash，那崩溃恢复的时候 MySQL 会怎么处理？先来看一下崩溃恢复时的判断规则：

 - 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
 - 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
   * a.  如果是，则提交事务；
   * b.  否则，回滚事务。

## 追问 1：MySQL 怎么知道 binlog 是完整的?
一个事务的 binlog 是有完整格式的：

 - statement 格式的 binlog，最后会有 **COMMIT**；
 - row 格式的 binlog，最后会有一个 **XID event**。

另外，在 MySQL 5.6.2 版本以后，还引入了 binlog-checksum 参数，用来验证 binlog 内容的正确性。对于 binlog 日志由于磁盘原因，可能会在日志中间出错的情况，MySQL 可以通过校验 **checksum** 的结果来发现。

## 追问 2：redo log 和 binlog 是怎么关联起来的?
它们有一个共同的数据字段，叫 **XID**。崩溃恢复的时候，会按顺序扫描 redo log：

 - 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
 - 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。


## 追问 3：处于 prepare 阶段的 redo log 加上完整 binlog，重启就能恢复，MySQL 为什么要这么设计?
这个问题还是跟主从数据与备份的一致性有关。在时刻 B，也就是 binlog 写完以后 MySQL 发生崩溃，这时候 binlog 已经写入了，之后就会被从库（或者用这个 binlog 恢复出来的库）使用。所以，在主库上也要提交这个事务。采用这个策略，主库和备库的数据就保证了一致性。

## 追问 4：如果这样的话，为什么还要两阶段提交呢？干脆先 redo log 写完，再写 binlog。崩溃恢复的时候，必须得两个日志都完整才可以。是不是一样的逻辑？
两阶段提交是经典的**分布式系统**问题，并不是 MySQL 独有的。如果必须要举一个场景，来说明这么做的必要性的话，那就是事务的持久性问题。

对于 InnoDB 引擎来说，如果 redo log 提交完成了，事务就不能回滚（如果这还允许回滚，就可能覆盖掉别的事务的更新）。而如果 redo log 直接提交，然后 binlog 写入的时候失败，InnoDB 又回滚不了，数据和 binlog 日志又不一致了。两阶段提交就是为了给所有人一个机会，当每个人都说“我 ok”的时候，再一起提交。

## 追问 5：不引入两个日志，也就没有两阶段提交的必要了。只用 binlog 来支持崩溃恢复，又能支持归档，不就可以了？
考虑只有 binlog 的流程

>图 2
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/b070e32f96fa28a7ed73bbb7f2d9575c.png#pic_center)

这样的流程下，binlog 还是不能支持崩溃恢复的。**binlog 没有能力恢复“数据页”**。

如果在图中标的位置，也就是 binlog2 写完了，但是整个事务还没有 commit 的时候，MySQL 发生了 crash。重启后，引擎内部事务 2 会回滚，然后应用 binlog2 可以补回来；但是对于事务 1 来说，系统已经认为提交完成了，不会再应用一次 binlog1。但是，事务 1 也是可能丢失了的，而且是数据页级的丢失。此时，binlog 里面并没有记录数据页的更新细节，是补不回来的。

InnoDB 引擎使用的是 WAL 技术，执行事务的时候，写完内存和日志，事务就算完成了。如果之后崩溃，要依赖于日志来恢复数据页。

## 追问 6：那能不能反过来，只用 redo log，不要 binlog？
如果只从崩溃恢复的角度来讲是可以的。你可以把 binlog 关掉，这样就没有两阶段提交了，但系统依然是 **crash-safe** 的。但是，如果你了解一下业界各个公司的使用场景的话，就会发现在正式的生产库上，binlog 都是开着的。因为 binlog 有着 redo log 无法替代的功能。

一个是**归档**。redo log 是循环写，写到末尾是要回到开头继续写的。这样历史日志没法保留，redo log 也就起不到归档的作用。

一个就是 MySQL 系统依赖于 binlog。binlog 作为 MySQL 一开始就有的功能，被用在了很多地方。其中，MySQL 系统**高可用**的基础，就是 binlog 复制。

还有很多公司有异构系统（比如一些数据分析系统），这些系统就靠消费 MySQL 的 binlog 来更新自己的数据。关掉 binlog 的话，这些下游系统就没法输入了。

## 追问 7：redo log 一般设置多大？
redo log 太小的话，会导致很快就被写满，然后不得不强行刷 redo log，这样 WAL 机制的能力就发挥不出来了。所以，如果是现在常见的几个 TB 的磁盘的话，就不要太小气了，直接将 redo log 设置为 4 个文件、每个文件 1GB 吧。

## 追问 8：正常运行中的实例，数据写入后的最终落盘，是从 redo log 更新过来的还是从 buffer pool 更新过来的呢？
实际上，redo log 并没有记录数据页的完整数据，所以它并没有能力自己去更新磁盘数据页，也就不存在“数据最终落盘，是由 redo log 更新过去”的情况。

如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘（Flush）。这个过程，甚至与 redo log 毫无关系。

在崩溃恢复场景中，InnoDB 如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让 redo log 更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态。

## 追问 9：redo log buffer 是什么？是先修改内存，还是先写 redo log 文件？
在一个事务的更新过程中，日志是要写多次的。比如下面这个事务：

```sql
begin;
insert into t1 ...
insert into t2 ...
commit;
```

这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。**redo log buffer** 就是用来先存 redo 日志的一块内存。也就是说，在执行第一个 insert 的时候，数据的内存被修改了，redo log buffer 也写入了日志。但是，真正把日志写到 redo log 文件（文件名是 **ib_logfile** + 数字），是在执行 commit 语句的时候做的。

## 追问10： 当 MySQL 去更新一行，但是要修改的值跟原来的值是相同的，这时候 MySQL 会真的去执行一次修改吗？还是看到值相同就直接返回呢？
我们创建了一个简单的表 t，并插入一行，然后对这一行做修改。

```sql
mysql> CREATE TABLE `t` (
`id` int(11) NOT NULL primary key auto_increment,
`a` int(11) DEFAULT NULL
) ENGINE=InnoDB;
insert into t values(1,2);
```

这时候，表 t 里有唯一的一行数据 (1,2)。假设，我现在要执行：

```sql
mysql> update t set a=2 where id=1;
```

结果显示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/171cbf3fca1c5c314e040ca8a2e71e41.png#pic_center)

仅从现象上看，MySQL 内部在处理这个命令的时候，可以有以下三种选择：

 1. 更新都是先读后写的，MySQL 读出数据，发现 a 的值本来就是 2，不更新，直接返回，执行结束；
 2. MySQL 调用了 InnoDB 引擎提供的“修改为 (1,2)”这个接口，但是引擎发现值与原来相同，不更新，直接返回；
 3. InnoDB 认真执行了“把这个值修改成 (1,2)"这个操作，该加锁的加锁，该更新的更新。

首先，验证以下实验：session B 中的更新操作被阻塞，加锁这个动作是 InnoDB 才能做的，所以可以排除选项 1。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/37a90d8adc40d3e8b1eab396846bbd6a.png#pic_center)
然后，验证以下实验：session A 中第二个 SELECT 语句返回的结果是 (1,3)，考虑到一致性读，该查询语句看到的该结果集只能是由 session A 本身的 UPDATE 更新操作造成的（而不是 session B 中的更新操作）。所以可以排除选项 2。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/c4a925b3d7cb28175849478b10f42ba1.png#pic_center)
所以，正确的答案应该是选项 3，即：InnoDB 认真执行了“把这个值修改成 (1,2)"这个操作，该加锁的加锁，该更新的更新。

但是，当我们在 WHERE 条件中加上 `a=3` 时，就不会触发更新。此时 session A 中第二次查询返回的值仍是 (1,2)。
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/4e54f7c206780c4d7b123782b3a56653.png#pic_center)

上述实验验证结果都是在 `binlog_format=statement` 格式下进行的。 如果是 `binlog_format=row` 并且 `binlog_row_image=FULL` 的时候，由于 MySQL 需要在 binlog 里面记录所有的字段，所以在读数据的时候就会把所有数据都读出来了。

# 其他问题
示例
```sql
insert into `t`(user_id, liker_id, relation_ship) values(A, B, 1) 
on duplicate key update relation_ship=relation_ship | 1;

insert ignore into `t1`(friend_1_id, friend_2_id) values(A,B);
```

**INSERT INTO ... ON DUPLICATE KEY UPDATE ...**
在MySQL数据库中，如果在 insert 语句后面带上 `ON DUPLICATE KEY UPDATE` 子句，而要插入的行与表中现有记录的**唯一索引**或**主键**中产生重复值，那么就会发生旧行的**更新**；如果插入的行数据与现有表中记录的唯一索引或者主键不重复，则执行新纪录插入**操作**。


**位或运算符：|**
参与 | 运算的两个**二进制位**有一个为 1 时，结果就为 1，两个都为 0 时结果才为 0。例如 `1|1` 结果为 1，`0|0` 结果为0，`1|0` 结果为1。

**INSERT IGNORE INTO ...**
当使用 `INSERT` 语句向表中添加一些行数据并且在处理期间发生错误时，`INSERT` 语句将被中止，并返回错误消息。因此，可能不会向表中插入任何行。但是，如果使用 `INSERT INGORE` 语句，则会**忽略导致错误的行**，并将其余行插入到表中。



