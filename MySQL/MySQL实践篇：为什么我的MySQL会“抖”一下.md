@[TOC](MySQL实践篇：为什么我的MySQL会“抖”一下)

平时的工作中，你有可能遇到过这样的场景，一条 SQL 语句，正常执行的时候特别快，但是有时也不知道怎么回事，它就会变得特别慢，并且这样的场景很难复现，它不只随机，而且持续时间还很短。

# SQL 语句为什么变“慢”了
InnoDB 在处理更新语句的时候，只做了写日志这一个磁盘操作。这个日志叫作 redo log（重做日志），在更新内存写完 redo log 后，就返回给客户端，本次更新成功。这就是MySQL的 **WAL**（*Write-Ahead Logging*）机制。

当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“**脏页**”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“**干净页**”。==把内存里的数据写入磁盘的过程==，术语就是 **Flush**。

## 四种 flush场景
回到文章开头的问题，不难想象，平时执行很快的更新操作，其实就是在写内存和日志，而 MySQL 偶尔“抖”一下的那个瞬间，可能就是在刷脏页（flush）。什么情况会引发数据库的 flush 过程呢？

 - 第一种场景对应的就是 ==InnoDB 的 redo log 写满了==。这时候系统会停止所有更新操作，把 checkpoint 往前推进，redo log 留出空间可以继续写。
 - 第二种场景对应的就是==系统内存不足==。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。
 - 第三种场景对应的就是 MySQL 认为==系统“空闲”== 的时候。
 - 第四种场景对应的就是 MySQL ==正常关闭==的情况。这时候，MySQL 会把内存的脏页都 flush 到磁盘上，这样下次 MySQL 启动的时候，就可以直接从磁盘上读数据，启动速度会很快。

## 对性能的影响
以上的四种场景中，第三种情况是属于 MySQL 空闲时的操作，这时系统没什么压力，而第四种场景是数据库本来就要关闭了。这两种情况下，你不会太关注“性能”问题。

第一种场景是“redo log 写满了，要 flush 脏页”，这种情况是 InnoDB **要尽量避免**的。因为出现这种情况的时候，整个系统就不能再接受更新了，所有的更新都必须堵住。如果你从监控上看，这时候更新数会跌为 0。

第二种场景是“内存不够用了，要先将脏页写到磁盘”，这种情况其实是**常态**。InnoDB 用**缓冲池**（buffer pool）管理内存，缓冲池中的内存页有三种状态：

 1. 尚未使用的内存页；
 2. 使用了并且是干净页；
 3. 使用了并且是脏页。

InnoDB 的策略是==尽量使用内存==，因此对于一个长时间运行的库来说，未被使用的页面很少。而当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页呢，就必须将脏页先刷到磁盘，变成干净页后才能复用。

刷脏页虽然是常态，但是出现以下这两种情况，都是会明显影响性能的：

 1. 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长（影响读操作）；
 2. 日志写满，更新全部堵住，写性能跌为 0，这种情况对敏感业务来说，是不能接受的（阻塞写操作）。

所以，InnoDB 需要有控制脏页比例的机制，来尽量避免上面的这两种情况。

#  InnoDB 刷脏页的控制策略
## innodb_io_capacity
首先，你需要正确地告诉 InnoDB 所在主机的 IO 能力，这样 InnoDB 才能知道需要全力刷脏页的时候，可以刷多快。参数 `innodb_io_capacity` 会告诉 InnoDB 所在主机的磁盘 IO 能力。该参数的默认值为 200，建议设置成磁盘的 **IOPS** (Input/Output Operations Per Second，即磁盘每秒的读写次数)。磁盘的 IOPS 可以通过 fio 这个工具来测试。

```bash
$ fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M 
-numjobs=10 -runtime=10 -group_reporting -name=mytest 
```

我们现在已经定义了“全力刷脏页”的行为，但平时总不能一直是全力刷吧？毕竟磁盘能力不能只用来刷脏页，还需要服务用户请求。所以接下来，我们就一起看看 InnoDB 怎么控制引擎按照“全力”的百分比来刷脏页。

## 按百分比刷脏页
InnoDB 的刷盘速度要参考两个因素：一个是**脏页比例**，一个是 **redo log 写盘速度**。InnoDB 会根据这两个因素先单独算出两个数字。

**脏页比例**
参数 `innodb_max_dirty_pages_pct` 是脏页比例上限，默认值是 **75%**。InnoDB 会根据**当前的脏页比例**（假设为 M），算出一个范围在 0 到 100 之间的数字。

```bash
F1(M)
{
  if M>=innodb_max_dirty_pages_pct then
      return 100;
  return 100*M/innodb_max_dirty_pages_pct;
}
```
可以理解为，当前的脏页比例越大，则 `F1(M)` 值越大。其中，脏页比例是通过 `innodb_buffer_pool_pages_dirty` 除以 `innodb_buffer_pool_pages_total` 得到的，具体的命令参考下面的代码：

```sql
mysql> select VARIABLE_VALUE into @a from global_status 
where VARIABLE_NAME = 'innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'innodb_buffer_pool_pages_total';
select @a/@b;
```

**redo log 写盘速度**
InnoDB 每次写入的日志都有一个序号，当前写入的序号跟 checkpoint 对应的序号之间的差值，我们假设为 `N`（可以理解为已经使用的日志空间）。InnoDB 会根据这个 N 算出一个范围在 0 到 100 之间的数字，记为 F2(N)。 N 越大，算出来的 `F2(N)` 值越大。

然后，根据上述算得的 `F1(M)` 和 `F2(N)` 两个值，取其中较大的值记为 `R`，之后引擎就可以按照 `innodb_io_capacity` 定义的能力乘以 R% 来控制刷脏页的速度。$$R = max (F1(M), F2(N))$$ $$刷脏页速度=innodb\_io\_capacity \times R\% $$


## innodb_flush_neighbors
一旦一个查询请求需要在执行过程中先 flush 掉一个脏页时，这个查询就可能要比平时慢了。而 MySQL 中的一个机制，可能让你的查询会更慢：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个“邻居”也带着一起刷掉；而且这个把“邻居”拖下水的逻辑还可以继续蔓延，也就是对于每个邻居数据页，如果跟它相邻的数据页也还是脏页的话，也会被放到一起刷。

在 InnoDB 中，`innodb_flush_neighbors` 参数就是用来控制这个行为的，值为 1 的时候会有上述的“连坐”机制，值为 0 时表示不找邻居，自己刷自己的。

找“邻居”这个优化在机械硬盘时代是很有意义的，可以减少很多随机 IO。机械硬盘的随机 IOPS 一般只有几百，相同的逻辑操作减少随机 IO 就意味着系统性能的大幅度提升。而如果使用的是 SSD 这类 IOPS 比较高的设备的话，建议把 `innodb_flush_neighbors` 的值设置成 0。在 MySQL 8.0 中，`innodb_flush_neighbors` 参数的默认值已经是 0 了。


References
[1\] https://dev.mysql.com/doc/refman/5.7/en/innodb-configuring-io-capacity.html#:~:text=The%20innodb_io_capacity%20variable%20defines%20the%20overall%20I%2FO%20capacity,for%20background%20tasks%20based%20on%20the%20set%20value.
