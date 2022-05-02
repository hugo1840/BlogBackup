@[TOC](幻读是什么，幻读有什么问题？)

考虑表

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

查询语句

```sql
begin;
select * from t where d=5 for update;
commit;
```

这个语句会命中 d=5 的这一行，对应的主键 id=5，因此在 select 语句执行完成后，id=5 这一行会加一个写锁，而且由于两阶段锁协议，这个写锁会在执行 commit 语句的时候释放。

由于字段 d 上没有索引，因此这条查询语句会做全表扫描。那么，其他被扫描到的，但是不满足条件的 5 行记录上，会不会被加锁呢？

InnoDB 的默认事务隔离级别是可重复读（RR）。


# 幻读是什么？
如果只在 id=5 这一行加锁，而其他行的不加锁的话，考虑下面的场景

>**图 1**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/bef6bd2fc06794586845ee8e06a5ad51.png#pic_center)

图1中可以看到，session A 里执行了三次查询，分别是 Q1、Q2 和 Q3。三次查询的 SQL 语句相同，都是 `select * from t where d=5 for update`。这个语句的意思是，查所有 d=5 的行，而且使用**当前读**，并且加上写锁。可以看到，三次输出的查询结果都不相同。

 - 在 T2 时刻，session B 把 id=0 这一行的 d 值改成了 5，因此 T3 时刻 Q2 查出来的是 id=0 和 id=5 这两行；
 - 在 T4 时刻，session C 又插入一行（1,1,5），因此 T5 时刻 Q3 查出来的是 id=0、id=1 和 id=5 的这三行。

Q3 读到 id=1 这一行的现象，被称为“幻读”。也就是说，**幻读**指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。需要注意：

 - 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，**幻读在“当前读”下才会出现**。
 - 上面 session B 的修改结果，被 session A 之后的 select 语句用“当前读”看到，不能称为幻读。**幻读仅专指“新插入的行”**。

因为这三个查询都是加了 `for update`，都是当前读。而当前读的规则，就是要能读到所有已经提交的记录的最新值。所以三次查询的输出跟事务的可见性规则并不矛盾。

# 幻读有什么问题？
**语义问题**
首先是语义上的。session A 在 T1 时刻就声明了，“把所有 d=5 的行锁住，不准别的事务进行读写操作”。而实际上这个语义被破坏了。

>**图 2**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/365e7fd4dc13b60dfd984ccdcfd37d60.png#pic_center)

图2中，由于在 T1 时刻，session A 还只是给 id=5 这一行加了行锁， 并没有给 id=0 这行加上锁。因此，session B 在 T2 时刻，是可以执行这两条 update 语句的。这样，就破坏了 session A 里 Q1 语句要锁住所有 d=5 的行的加锁声明。


**数据不一致问题**
其次，是数据一致性的问题。我们知道，锁的设计是为了保证数据的一致性。而这个一致性，不止是数据库内部数据状态在此刻的一致性，还包含了数据和日志在逻辑上的一致性。

图3的场景中，session A 在 T1 时刻再加一个更新语句，即：`update t set d=100 where d=5`。

>**图 3**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/94a9b264b540debcc9533c240f969cc7.png#pic_center)

图 3 执行完成后，数据库里的结果如下：

 1. 经过 T1 时刻，id=5 这一行变成 (5,5,100)，当然这个结果最终是在 T6 时刻正式提交的；
 2. 经过 T2 时刻，id=0 这一行变成 (0,5,5)；
 3. 经过 T4 时刻，表里面多了一行 (1,5,5)；
 4. 其他行跟这个执行序列无关，保持不变。

再来看看这时候 binlog 里面的内容：

 1. T2 时刻，session B 事务提交，写入了两条语句；
 2. T4 时刻，session C 事务提交，写入了两条语句；
 3. T6 时刻，session A 事务提交，写入了 `update t set d=100 where d=5` 这条语句。

对应记录的语句依次为

```sql
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/

insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/
```

这个语句序列，不论是拿到备库去执行，还是以后用 binlog 来克隆一个库，这三行的结果，都变成了 (0,5,100)、(1,5,100) 和 (5,5,100)。也就是说，id=0 和 id=1 这两行，发生了数据不一致。

这个数据不一致到底是怎么引入的？分析一下可以知道，这是我们假设`select * from t where d=5 for update` ==这条语句只给 d=5 这一行，也就是 id=5 的这一行加锁==导致的。所以我们认为，上面的设定不合理，要改。

我们==把扫描过程中碰到的行，也都加上写锁==，再来看看执行效果。

>**图 4**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/e25f1a107eb32b7e7c1f365fa8661a2d.png#pic_center)

由于 session A 把所有的行都加了写锁，所以 session B 在执行第一个 update 语句的时候就被锁住了。需要等到 T6 时刻 session A 提交以后，session B 才能继续执行。

在 binlog 里面，执行序列是这样的：

```sql
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/

update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/
```

可以看到，按照日志顺序执行，id=0 这一行的最终结果也是 (0,5,5)。所以，id=0 这一行的问题解决了。但同时你也可以看到，id=1 这一行，在数据库里面的结果是 (1,5,5)，而根据 binlog 的执行结果是 (1,5,100)，也就是说幻读的问题还是没有解决。

也就是说，即使把所有的记录都加上锁，还是阻止不了新插入的记录，这也是为什么“幻读”会被单独拿出来解决的原因。

# 如何解决幻读？
**间隙锁**
产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB 只好引入新的锁，也就是**间隙锁** (**Gap Lock**)。

间隙锁，锁的就是两个值之间的空隙。比如文章开头的表 t，初始化插入了 6 个记录，这就产生了 7 个间隙。当你执行 `select * from t where d=5 for update` 的时候，就不止是给数据库中已有的 6 个记录加上了行锁，还同时加了 7 个间隙锁。这样就确保了无法再插入新的记录。

数据行是可以加上锁的实体，数据行之间的间隙，也是可以加上锁的实体。但是间隙锁跟我们之前碰到过的锁都不太一样。行锁中，读锁与写锁冲突，写锁与写锁冲突，读锁与读锁兼容。

而跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。**间隙锁之间都不存在冲突关系**。举个例子

>**图 5**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/1c790e104296259193de3d38d552dd17.png#pic_center)

这里 session B 并不会被堵住。因为表 t 里并没有 c=7 这个记录，因此 session A 加的是**间隙锁 (5,10)**。而 session B 也是在这个间隙加的间隙锁。它们有共同的目标，即：保护这个间隙，不允许插入值。但，它们之间是不冲突的。

间隙锁和行锁合称 **next-key lock**，每个 next-key lock 是**前开后闭区间**。也就是说，我们的表 t 初始化以后，如果用 `select * from t for update` 要把整个表所有记录锁起来，就形成了 7 个 next-key lock，分别是 ($-\infty$,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]。因为 $+\infty$ 是开区间，实现上，InnoDB 给每个索引加了一个不存在的最大值 supremum。

**死锁问题**
间隙锁和 next-key lock 的引入，帮我们解决了幻读的问题，但同时也带来了一些“困扰”。

考虑如下的业务逻辑

```sql
begin;
select * from t where id=N for update;

/*如果行不存在*/
insert into t values(N,N,N);
/*如果行存在*/
update t set d=N set id=N;

commit;
```

现在遇到的问题是：这个逻辑一旦有并发，就会碰到死锁。下面使用两个 session 来模拟并发

>**图 6**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/2819b57ee4198d4b79c5abce311c5a7f.png#pic_center)

按语句执行顺序来分析一下：

 1. session A 执行 select … for update 语句，由于 id=9 这一行并不存在，因此会加上间隙锁 (5,10)；
 2. session B 执行 select … for update 语句，同样会加上间隙锁 (5,10)，间隙锁之间不会冲突，因此这个语句可以执行成功；
 3. session B 试图插入一行 (9,9,9)，被 session A 的间隙锁挡住了，只好进入等待；
 4. session A 试图插入一行 (9,9,9)，被 session B 的间隙锁挡住了。

两个 session 进入互相等待状态，形成**死锁**。InnoDB 的死锁检测马上就发现了这对死锁关系，让 session A 的 insert 语句报错返回了。


**小结**
本文章分析的问题都是在可重复读隔离级别下的，**间隙锁是在可重复读隔离级别下才会生效的**。所以，你如果把隔离级别设置为**读提交**的话，就没有间隙锁了。但同时，你要解决可能出现的数据和日志不一致问题，需要把 binlog 格式设置为 **row**。如果读提交隔离级别够用，也就是说，业务不需要可重复读的保证，这样考虑到读提交下操作数据的锁范围更小（没有间隙锁），这个选择是合理的。


# 补充：binlog日志格式
binlog日志格式分为三种：

 - **STATEMENT**：每一条会修改数据的 SQL 都会记录到 master 的 bin-log 中。slave 在复制的时候 SQL 进程会解析成和原来 master 端执行过的相同的 SQL 再次执行。优点是不需要记录每一行数据的变化，减少了 bin-log 日志量，节省 I/O 以及存储资源，提高性能；缺点是如果使用了某些特定的函数或者功能，可能会无法正确复制。
 
 - **ROW**：日志内容会非常清楚的记录下每一行数据修改的细节。优点是不会出现某些特定情况下的存储过程或函数、以及 trigger 的调用和触发无法被正确复制的问题；缺点是可能会产生大量的日志内容，尤其是当执行 alter table 之类的语句的时候，产生的日志量是惊人的。
 
 - **MIXED**：MySQL 会根据执行的每一条具体的 SQL 语句来区分对待记录的日志形式，也就是在 statement 和 row 之间选择一种。

>https://dev.mysql.com/doc/refman/5.7/en/binary-log-formats.html
>https://blog.csdn.net/helloxiaozhe/article/details/86670675

