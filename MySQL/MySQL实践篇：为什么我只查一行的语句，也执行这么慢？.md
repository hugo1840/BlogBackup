@[TOC](为什么我只查一行的语句，也执行这么慢？)

有些情况下，“查一行”，也会执行得特别慢。如果 MySQL 数据库本身就有很大的压力，导致数据库服务器 CPU 占用率很高或 ioutil（IO 利用率）很高，这种情况下所有语句的执行都有可能变慢，不属于本文章的讨论范围。

考虑一个有10万行记录的表

```sql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000) do
    insert into t values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

>可参考 https://www.tutorialspoint.com/create-a-stored-procedure-with-delimiter-in-mysql

# 第一类：查询长时间不返回
在表 t 执行下面的 SQL 语句：

```sql
select * from t where id=1;
```

查询结果长时间不返回。一般碰到这种情况的话，大概率是表 t 被锁住了。接下来分析原因的时候，一般都是首先执行一下 `show processlist` 命令，看看当前语句处于什么状态。然后我们再针对每种状态，去分析它们产生的原因、如何复现，以及如何处理。

## 等MDL锁
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/1173542b56791da8f2b01f533f9d354a.png#pic_center)
 如果 `show processlist` 命令的输出显示对应查询语句的状态为 **Waiting for table metadata lock**，表示的是，现在有一个线程正在表 t 上请求或者持有 MDL 写锁，把 select 语句堵住了。

在 MySQL 5.7 中可以按如下方式复现
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/950252305ebf25497d7dbdd90f1273ca.png#pic_center)
这类问题的处理方式，就是找到谁持有 MDL 写锁，然后把它 kill 掉。通过查询 `sys.schema_table_lock_waits` 这张表，我们就可以直接找出造成阻塞的 PID，把这个连接用 kill 命令断开即可（MySQL 启动时需要设置 `performance_schema=ON`，相比于设置为 off 会有 10% 左右的性能损失)。

```sql
select blocking_pid from sys.schema_table_lock_waits;
```

执行上述语句需要新开一个session，否则可能报如下错误：`ERROR 1100 (HY000): Table 'schema_table_lock_waits' was not locked with LOCK TABLES`。

>If in one session, you locked one table but want to select from another table, you must either lock that table too or unlock all tables.
>https://stackoverflow.com/questions/36467298/mysql-table-my-table-was-not-locked-with-lock-tables

如果上述查询语句输出为空，执行

```sql
select * from performance_schema.setup_instruments where name='wait/lock/metadata/sql/mdl';
```

查看一下 ENABLED 和 TIMED 参数是不是都是 YES，只有两个都是 YES 的时候才能执行文章中说的操作。如果不是，按如下语句修改

```sql
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES' 
where name='wait/lock/metadata/sql/mdl';
```

>具体可以参考官方文档： 
>https://dev.mysql.com/doc/refman/5.7/en/sys-schema-table-lock-waits.html 
>https://dev.mysql.com/doc/refman/5.7/en/metadata-locks-table.html 

## 等Flush
在表 t 上，执行下面的 SQL 语句：

```sql
mysql> select * from information_schema.processlist where id=1;
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/d6852f1969fe863f7db318de9102afc2.png#pic_center)
这个状态表示的是，现在有一个线程正要对表 t 做 flush 操作。MySQL 里面对表做 flush 操作的用法，一般有以下两个：

```sql
flush tables t with read lock; /*Q1*/
flush tables with read lock; /*Q2*/
```

如果指定表 t 的话，代表的是只关闭表 t；如果没有指定具体的表名，则表示关闭 MySQL 里所有打开的表。

但是正常这两个语句执行起来都很快，除非它们也被别的线程堵住了。所以，出现 **Waiting for table flush** 状态的可能情况是：有一个 `flush tables` 命令被别的语句堵住了，然后它又堵住了我们的 select 语句。

复现的方法如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/07976022f68d025e33c8e825ab459efe.png#pic_center)
在 session A 中，我故意每行都调用一次 `sleep(1)`，这样这个语句默认要执行 10 万秒，在这期间表 t 一直是被 session A“打开”着。然后，session B 的 `flush tables t` 命令再要去关闭表 t，就需要等 session A 的查询结束。这样，session C 要再次查询的话，就会被 flush 命令堵住了。

对应 `show processlist` 命令的输出如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/5c358e7bdf9c53b88249e448946a68d4.png#pic_center)

## 等行锁
使用 InnoDB 引擎时，`SELECT ... LOCK IN SHARE MODE` 会给当前 session 正在读的行加上**共享锁**。其他 session 仍然可以读被加了共享锁的行，或者给这些行加共享锁；但是不能修改这些行，也不能对这些行加排他锁，直到加共享锁的事务提交。当多个事务同时对某一行加共享锁时候，必须等待先执行的事务提交后，其他事务才能依次执行（且会使用前一个提交事务更新后的值）。

`SELECT ... FOR UPDATE` 会给当前 session 正在读的行加上**排他锁**。此时，不允许其他事务对这些行加共享锁或者排它锁读取，更加不允许其他事务修改加排他锁的行，直到最先加排他锁的事务提交。

`LOCK IN SHARE MODE` 和 `FOR UPDATE` 对行加的共享锁和排他锁在加锁的事务提交后都会自动释放。

>Locking reads参考 https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html


考虑下面的场景
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/788b79f39612af43c0e3a8ae102cd60a.png#pic_center)


由于访问 id=1 这个记录时要加读锁，如果这时候已经有一个事务在这行记录上持有一个写锁，我们的 select 语句就会被堵住。如果是 MySQL 5.7 版本，可以通过 `sys.innodb_lock_waits` 表查到占用写锁造成阻塞的线程。

```sql
mysql> select * from t sys.innodb_lock_waits where locked_table='`test`.`t`'\G;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/0f890f4b021a65e3265177535d0ee9bd.png#pic_center)
可以看到，4 号线程是造成堵塞的罪魁祸首。而干掉这个罪魁祸首的方式，就是 `KILL QUERY 4` 或 `KILL 4`。不过，这里不应该显示“KILL QUERY 4”。这个命令表示**停止 4 号线程当前正在执行的语句**，而这个方法其实是没有用的。因为占有行锁的是 update 语句，这个语句已经是之前执行完成了的，现在执行 KILL QUERY，无法让这个事务去掉 id=1 上的行锁。实际上，KILL 4 才有效，也就是说**直接断开这个连接**。这里隐含的一个逻辑就是，连接被断开的时候，会自动回滚这个连接里面正在执行的线程，也就释放了 id=1 上的行锁。


# 第二类：查询慢
考虑如下场景
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/8d825aac46c87cdaed5f3828b21029cc.png#pic_center)

session A 中，两条查询语句的输出如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/26f09352b0fbe8458babd8ff04dd7806.png#pic_center)

可以看到，第一条 select 语句

```sql
mysql> select * from t where id=1；
```
虽然扫描行数是 1，但执行时间却长达 800 毫秒。

session A 先用 `start transaction with consistent snapshot` 命令启动了一个事务，之后 session B 才开始执行 update 语句。session B 执行完 100 万次 update 语句后，id=1 这一行处于什么状态呢？

![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/f8a48b6ec60431481b8168904fdac18c.png#pic_center)

session B 更新完 100 万次，生成了 100 万个回滚日志 (**undo log**)。带 `lock in share mode` 的 SQL 语句，是**当前读**，因此会直接读到 1000001 这个结果，所以速度很快；而 `select * from t where id=1` 这个语句，是**一致性读**，因此==需要从 1000001 开始，依次执行 undo log，执行了 100 万次以后，才将 1 这个结果返回==。

>参考《MySQL一致性读与当前读》
>https://blog.csdn.net/Sebastien23/article/details/114975626
