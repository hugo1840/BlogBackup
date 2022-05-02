@[TOC](MySQL实战篇：为什么表数据删掉一半，表文件大小不变？)

# 表空间
表空间（Tablespaces）是数据库的逻辑划分，一般由多个数据文件构成。MySQL 5.7 中的表空间可以分为系统表空间（**System tablespace**）、独立表空间（**File-per-table tablespace**）、通用表空间（**General tablespace**）、回滚表空间（**Undo tablespace**）、临时表空间（**Temporary tablespace**）。

>**图 1**
![图 1](https://img-blog.csdnimg.cn/20210329171136185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


系统表空间又被称为**共享表空间**。MySQL 5.7中，系统表空间中保存了 InnoDB 数据字典、doublewrite buffer、change buffer 和 undo日志。系统表空间可以由一个或多个数据文件组成，且默认会创建一个名为 `ibdata1` 的数据文件。系统表空间中数据文件的大小和数量由 `innodb_data_file_path` 参数设定。

**独立表空间**包含单个 InnoDB 表的数据和索引，并且在文件系统中有自己独立的数据文件。InnoDB 引擎默认在独立表空间中创建表。禁用 `innodb_file_per_table` 参数会导致 InnoDB 在系统表空间中创建表。独立表空间对应的数据文件后缀为 `.ibd`，命名规则为 `tablename.ibd`。


一个 InnoDB 表包含两部分，即：**表结构**定义和**数据**。在 MySQL 8.0 版本以前，表结构是存在以 `.frm` 为后缀的文件里。而 MySQL 8.0 版本，则已经允许把表结构定义放在系统数据表中了。

# innodb_file_per_table
从 MySQL 5.6.6 版本开始，`innodb_file_per_table` 的默认值就是 ON 了，即每个 InnoDB 表数据存储在一个以 `.ibd` 为后缀的文件中。这么做的原因是，一个表单独存储为一个文件更容易管理，而且在你不需要这个表的时候，通过 `DROP TABLE` 命令，系统就会直接删除这个文件。而==如果是放在共享表空间中，即使表删掉了，空间也是不会回收的==。

我们在删除整个表的时候，可以使用 `DROP TABLE` 命令回收表空间。但是，我们遇到的更多的删除数据的场景是删除某些行时，表中的数据被删除了，但是表空间却没有被回收。

# 数据删除流程
**可复用**
InnoDB 里的数据都是用 B+ 树的结构组织的。假设，我们要删掉某一行记录，InnoDB 引擎只会把这个记录**标记**为删除。如果之后要再插入一个新记录时，可能会**复用**这个位置。但是，磁盘文件的大小并不会缩小。如果我们删掉了一个数据页上的所有记录，整个数据页就可以被复用了。但是，数据页的复用跟记录的复用是不同的。

记录的复用，只限于符合范围条件的数据。假设某个数据页上存在 ID 分别为 300、500、600 的 R3、R4、R5 三条记录，R4 这条记录被删除后，如果插入一个 ID 是 400 的行，可以直接复用这个空间。但如果插入的是一个 ID 是 800 的行，就不能复用这个位置了。而当整个数据页从 B+ 树里面删除以后，可以复用到任何位置。

如果相邻的两个数据页利用率都很小，系统就会把这两个页上的数据合到其中一个页上（**页合并**），另外一个数据页就被标记为可复用。

**空洞的产生**
如果我们用 `DELETE` 命令把整个表的数据删除呢？结果就是，所有的数据页都会被标记为可复用。但是磁盘上，文件不会变小。`DELETE` 命令只是把记录的位置，或者数据页标记为了“可复用”，但磁盘文件的大小是不会变的。也就是说，**通过 DELETE 命令是不能回收表空间的**。这些可以复用，而没有被使用的空间，看起来就像是“空洞”。

不止是删除数据会造成空洞，插入数据也会。如果数据是按照索引递增顺序插入的，那么索引是紧凑的。但如果数据是**随机插入**的，就可能造成索引的数据**页分裂**。

另外，**更新索引上的值**，可以理解为删除一个旧的值，再插入一个新值。不难理解，这也是会造成空洞的。也就是说，经过大量增删改的表，都是可能是存在空洞的。所以，如果能够把这些空洞去掉，就能达到收缩表空间的目的。而重建表，就可以达到这样的目的。

# 重建表
**ALTER TABLE**
如果现在有一个表 A，需要做空间收缩，为了把表中存在的空洞去掉，可以怎么做呢？

你可以新建一个与表 A 结构相同的表 B，然后按照主键 ID 递增的顺序，把数据一行一行地从表 A 里读出来再插入到表 B 中。由于表 B 是新建的表，所以表 A 主键索引上的空洞，在表 B 中就都不存在了。显然地，表 B 的主键索引更紧凑，数据页的利用率也更高。

>**图 2**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/710afa4e05f49aca765a58da114a78c2.png#pic_center)


可以使用 `alter table A engine=InnoDB` 命令来重建表。在 MySQL 5.5 版本之前，这个命令的执行流程跟我们前面描述的差不多，区别只是这个临时表 B 不需要你自己创建，MySQL 会自动完成转存数据、交换表名、删除旧表的操作。

显然，花时间最多的步骤是往临时表插入数据的过程，如果在这个过程中，有新的数据要写入到表 A 的话，就会造成数据丢失。因此，在整个 DDL (Data Definition Language) 过程中，表 A 中不能有更新。也就是说，这个 DDL 不是 Online 的。

**Online DDL**
在 MySQL 5.6 版本开始引入的 Online DDL，对这个操作流程做了优化。其重建表的流程为：

 1. 建立一个临时文件，扫描表 A 主键的所有数据页；
 2. 用数据页中表 A 的记录生成 B+ 树，存储到临时文件中（tmp-file）；
 3. 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中，对应的是图中 state 2 的状态；
 4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同的数据文件，对应的就是图中 state 3 的状态；
 5. 用临时文件替换表 A 的数据文件。

>**图 3**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/0f8d405747cf1c193471d4b964d35b7e.png#pic_center)

可以看到，与图 2 的不同之处在于，由于日志文件记录和重放操作这个功能的存在，这个方案在重建表的过程中，允许对表 A 做增删改操作。这就是 Online DDL 。

我们知道，`ALTER TABLE` 语句在启动的时候需要获取 MDL 写锁，以禁止其他线程对这个表同时做 DDL。同时，为了实现 Online，这个 MDL 写锁在真正拷贝数据之前就退化成读锁了，因为 MDL 读锁不会阻塞增删改操作。

上述的这些重建方法都会扫描原表数据和构建临时文件。对于很大的表来说，这个操作是很消耗 IO 和 CPU 资源的。因此，如果是线上服务，需要很小心地控制操作时间。如果想要比较安全的操作的话，推荐使用 GitHub 开源的 gh-ost 来做。

# online vs inplace
在图 2 中，我们把表 A 中的数据导出来的存放位置叫作 **tmp_table**。这是一个临时表，是在 **server 层**创建的。在图 3 中，根据表 A 重建出来的数据是放在 **tmp_file** 里的，这个临时文件是 InnoDB 在内部创建出来的。整个 DDL 过程都在 **InnoDB 内部**完成。对于 server 层来说，没有把数据挪动到临时表，是一个“原地”操作（但是 tmp_file 也是要占用临时空间的），这就是“**inplace**”名称的来源。

我们重建表的这个语句 `alter table t engine=InnoDB`，其实隐含的意思是：

```sql
alter table t engine=innodb,ALGORITHM=inplace;
```

当你使用 ALGORITHM=copy 的时候，表示的是强制拷贝表，对应的流程就是图 2 的操作过程。

```sql
alter table t engine=innodb,ALGORITHM=copy;
```

Inplace 跟 Online 之间的逻辑关系可以概括为：

 1. DDL 过程如果是 Online 的，就一定是 inplace 的；
 2. 反过来未必，也就是说 inplace 的 DDL，有可能不是 Online 的。截止到 MySQL 8.0，添加**全文索引**（FULLTEXT index）和**空间索引** (SPATIAL index) 就属于这种情况。

最后，比较 `optimize table`、`analyze table` 和 `alter table` 的区别：

 - 从 MySQL 5.6 版本开始，`alter table t engine = InnoDB` 默认的就是上面图 3 的流程了；
 - `analyze table` 只是对表的索引信息做重新统计，没有修改数据，这个过程中加了 MDL 读锁；
 - `optimize table` 等于 `alter table` + `analyze table`。

References
[1\] https://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace.html

