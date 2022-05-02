@[TOC](Oracle数据库实例之内存架构（二）)

# 系统全局区：SGA
SGA是一块可读写内存区域，与Oracle后台进程（background processes）一起构成了数据库实例。所有代表用户执行的服务器进程都能读取实例SGA里的信息。有一些进程能在数据库运行时写入SGA。需要注意的是，服务器和后台进程本身并不在SGA中，而是存在于独立的内存空间中。

每个数据库实例都有自己的SGA。Oracle数据库会在实例启动时自动为SGA分配内存，并在实例关闭时回收内存。正如图1，SGA由多个为了满足特定内存分配需求的内存池组成。除了重做日志缓存（redo log buffer）以外，所有其他的内存池都按连续内存单位分配和回收内存空间。这个连续内存单位称之为粒（granule），其大小与平台有关且取决于总的SGA大小。

查询 `V$SGASTAT` 视图可以看到SGA各部分的信息。其中包括数据库缓存、IM区、重做日志缓存、共享池、Large池、Java池、Streams池、固定SGA等。

## 数据库 buffer cache
数据库缓冲器高速缓冲存储器（Database buffer cache），即**缓冲器高速缓存**，是存储了从数据文件读入的数据块副本的内存区域。**缓冲器**（buffer）是缓冲器管理器临时缓存当前正在使用的数据块的一块主内存地址（main memory address）。所有当前连接到数据库实例的用户都能共享访问 buffer cache。

**\#1 缓冲器高速缓存的用途**
数据库buffer cache的用途包括：

- **改善物理 I/O**：数据库更新**高速缓存**（cache）中的数据块，并存储重做日志缓冲器（redo log buffer）修改操作的元数据。在 commit 之后，数据库将重做日志缓冲器写入在线重做日志，但是并不会立即将数据块写入数据文件。数据库后台写进程 **DBW** (database writer) 通过 **lazy write** 来写入数据文件。
- 将频繁访问的数据块保留在 buffer cache 中，并将不经常访问的数据块写入磁盘。
  当 Database Smart Flash Cache（数据库智能闪存缓存）启用时，可以将部分 buffer cache 放到闪存缓存中。需要通过 `DB_FLASH_CACHE_FILE` 和 `DB_FLASH_CACHE_SIZE` 初始化参数来配置多个闪存设备（仅在 Solaris 和 Oracle Linux系统中支持）。

**\#2 缓冲器状态**
数据库使用内部算法来管理高速缓存中的缓冲器。一个缓冲器可以处于以下任意一种彼此互斥的状态：

- **Unused**：缓冲器可用，且尚未被使用过；
- **Clean**：缓冲器曾被使用过，其中包含的数据是干净的，所以无需刷盘（checkpoint）。可以被数据库重用；
- **Dirty**：缓冲器包含被修改的数据，且未被持久化到磁盘。数据库必须 checkpoint 数据块才能重用。

每个缓冲器有 **pinned** 和 free (unpinned) 两种访问模式。缓冲器被 pinned 在高速缓存中以后，就不会在用户访问时被从内存中置换出去（age out）。一个 pinned 缓冲器不能同时被多个会话修改。

**\#3 缓冲器模式**
当客户端请求数据时，Oracle数据库以 current模式或者 consistent模式从 buffer cache 中获取缓冲器。

- **current模式**
  current mode get，也称为 *db block get*，是指获取当前出现在buffer cache中的数据块。例如，如果一个未提交的事务更新了某个数据块中的两行数据，current mode get 会获取带有未提交行的该数据块。数据库最常在更新语句中使用 *db block get*，且必须只更新数据块的当前版本。 
  
- **consistent模式**
  consistent read get 用于获取数据块的**一致性读版本**（read-consistent version），且有可能用到 undo数据。例如，如果一个未提交的事务更新了某个数据块中的两行数据，并且有一个不相关的会话中的查询要请求访问该数据块时，数据库会使用undo数据来创建该数据块的一个一致性读版本（称为一致性读克隆）。创建的一致性读版本不会包括未提交的更新。


**\#4 缓冲器 IO**
逻辑 IO（**logical I/O**），即缓冲器IO（buffer I/O），是指在缓冲器高速缓存（buffer cache）中读写缓冲器（buffers）。当请求的缓冲器不在内存中时，数据库会进行物理IO从闪存缓存或者磁盘中将缓冲器拷贝到内存中，然后通过逻辑IO读入高数缓存的缓冲器。

**\## 缓冲器写**
数据库写进程（**DBW**）周期性地将脏的缓存器写入磁盘。DBW进程在以下条件下会写缓冲器：

- 服务器进程找不到干净的缓冲器来向数据库 buffer cache读入新的数据块。随着缓冲器变成dirty状态，空闲的缓冲器数量会逐渐下降。如果下降到某个阈值，而这时又需要干净的缓冲器，服务器进程就会发信号给 DBW 来写缓冲器。数据库使用 LRU算法（Least recently used）来决定写哪个脏的缓冲器。
- 数据库必须推进检查点（checkpoint），即redo线程里实例恢复（instance recovery）必须开始的位置。
- 表空间被修改为只读状态或者下线（offline）的时候。


**\## 缓冲器读**
当未使用的缓冲器数量很少时，数据库必须从buffer cache中移除缓冲器。

如果禁用了闪存缓存（flash cache），数据库会按需重用（re-use）每个干净的缓冲器，覆写它们。如果被覆写的缓冲器之后又需要用到，数据库就必须从磁盘中读出来。相反地，如果开启了闪存缓存，DBW可以将干净的缓冲器写入闪存缓存，之后如果再需用到该缓冲器，数据库就可以将它从闪存缓存中读出来（而不是磁盘）。

当客户端进程请求一个缓冲器时，服务器进程从buffer cache中搜索缓冲器。如果数据库在内存中找到了缓冲器，就会发生 **cache hit**（高速缓存命中）。搜索缓冲器的顺序如下：

1. 服务器进程在 buffer cache 中搜寻完整的缓冲器，如果找到了，数据库就会对其进行**逻辑读**。
2. 服务器进程在闪存缓存的 LRU 清单中搜寻缓冲器头部（buffer header）。如果找到了，数据库就会通过**优化的物理读**将缓冲器体（buffer body）从闪存缓存中读入内存中的高速缓存。
3. 如果服务器进程没能在内存中找到缓冲器（称为 **cache miss**），就会进行以下步骤：通过**物理读**将磁盘数据文件中的数据块读入内存；对读入内存的缓冲器进行逻辑读。

下图展示了上面的搜索过程。

>**图1 缓冲器搜索**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/3aaa86b5b740bded7ed7ccc86aba959d.gif#pic_center)


**\#5 缓冲器池**
缓冲器池（Buffer pool）是缓冲器的集合。数据库的buffer cache被分成一个或多个缓冲器池。这些缓冲器以几乎相同的方式管理数据块。你可以手动配置缓冲器池，将数据保留在buffer cache或者在使用完数据块后就将缓冲器设为可用。你也可以将特定的schema对象分配给合适的缓冲器池，来控制数据块被置换出（age out）高速缓存的方式。

可能存在的缓冲器池包括：

- 默认池（**default pool**）
  默认池是数据块被正常高速缓存的位置。除非你手动配置单独的池，默认池就是唯一的缓冲器池。对其他池的可选配置不会对默认池产生影响。
  
- Keep池（**keep pool**）
  keep池用于保留那些频繁被访问、但是由于缺少空间而被置换出默认池的数据块。keep池的目的在于将对象保留在内存中，以避免IO操作。
  
- 回收池（**recycle pool**）
  回收池用于存放不被频繁访问的数据块，其目的在于防止高速缓存中不必要的空间消耗。
  
下图展示了使用多个缓冲器池时buffer cache的状态。默认的数据块大小是 **8K**。高速缓存为使用非标准大小数据块（2KB、4KB、16KB）的表空间准备了独立的池。

>**图2 Buffer cache与缓冲器池**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/93d5a059f69776b4ac5a054235ed27dc.gif#pic_center)

**\#6 缓冲器和全表扫描**
数据库采用非常复杂的算法来处理表扫描。默认情况下，当缓冲器必须被从磁盘读入时，数据库会把缓冲器插入LRU清单中间（middle）。如此以来，经常被访问的数据块（hot blocks）可以被保留在高速缓存中，而无需从磁盘中读入。

全表扫描（**full table scan**）会读入表中**高水位线**（high water mark, **HWM**）以下的所有行。假设一个表段（segment）中数据块的总体大小超过了buffer cache的大小。对该表的全表扫描可能会清空buffer cache，导致数据库不能保留经常访问的数据块的高速缓存。

**\## 全表扫描的默认模式**
默认来说，Oracle数据库对处理全表扫描采取保守的方法，只有在表的大小对于buffer cache来说只占一小部分时，才会把表读入内存。

对于中型的表来说，数据库采用较为复杂的算法来决定是否都将整个表都读入高速缓存，其中考虑的因素包括与上一次表扫描的间隔、buffer cache剩余的空间，等等。

对于非常大的表来说，数据库通常采用直接路径读（**direct path read**），将数据块直接读入PGA，并且分流（bypass）给SGA。

从 Oracle 12.1.0.2 开始，数据库实例的buffer cache会自动进行内部计算，以决定现有内存是否足够容纳数据库完全高速缓存到实例SGA中，以及高速缓存要访问的表是否对性能有影响。

**\## 并行查询执行**
执行全表扫描时，数据库有时候可以通过使用多个并行执行服务器来减少反应时间。在某些案例中，如果数据库内存很大，就可以将并行执行数据高速缓存到SGA中，而不是通过直接路径读入PGA。

##  In-Memory Area
In-Memory区是一个可选的SGA组成部分，包含了**IM列存储**（In-Memory Column Store）。

IM列存储包含了为快速扫描而优化的列格式（columnar format）的表、分区、物化视图（materialized views）的拷贝。IM列存储是以行格式（row format）存储数据的数据库buffer cache的补充。

## 重做日志缓冲器
重做日志缓冲器（redo log buffer）是SGA中的一块循环缓冲器，用于存储描述对数据库修改操作的**重做记录**（redo entries）。重做记录（redo record）包含了重建或重做 DML 或 DDL 操作对数据库所做修改而所必需的信息。数据库恢复过程中就会应用重做记录到数据文件，从而重建丢失的修改内容。

数据库进程从用户内存空间拷贝重做记录到SGA中的重做日志缓冲器。重做记录在缓冲器中占用连续的、有序的空间。后台日志写进程 **LGWR** (log writer process) 将重做日志缓冲器写入磁盘中活跃的在线重做日志组。

>**图3 重做日志缓冲器**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/94719930e411bad8adaf8107d9c37169.gif#pic_center)

LGWR进程将重做记录顺序写入磁盘，与DBW进程分散地将数据块写入磁盘不同。DBW的分散写入比LGWR的顺序写要慢很多。

初始化参数 `LOG_BUFFER` 明确了Oracle数据库使用多少内存来缓冲重做记录。与其他的SGA组成部分不同的是，重做日志缓冲器和后面将介绍的固定SGA缓冲器（fixed SGA buffer）都不会将内存划分成颗粒（granules）。

## 共享池
共享池（shared pool）高速缓存了各种程序数据。比如，经过语法分析的SQL语句（parsed SQL）、PL/SQL代码、系统参数、以及数据字典信息等。共享池几乎与数据库中发生的每个操作都有关。

共享池被划分为以下几个重要部分。

>**图4 共享池**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/5746dd6a62c60196427ba0e51eb102eb.gif#pic_center)


**\#1 Library高速缓存**
Library高速缓存（Library cache）是存储可执行SQL语句和PL/SQL代码的共享池内存结构。它包含了共享的SQL和PL/SQL区、以及锁、library高速缓存句柄等控制结构。在共享的服务器架构中，library高速缓存还会包括私有SQL区。

当要执行一条SQL语句时，数据库会尝试重用之前执行过的代码。如果library高速缓存中有一条经过语法分析的SQL语句，并且可以被共享，数据库就会重用代码，我们称之为软语法分析（**soft parse**）或者library高速缓存命中（**library cache hit**）。否则，数据库就必须为应用代码创建一个新的可执行版本，我们称之为硬语法分析（hard parse）或者library高速缓存未命中（library cache miss）。

**\## 共享SQL区**
数据库使用共SQL区来处理第一次出现的SQL语句。共享SQL区包含了SQL语句的语法分析树和**执行计划**（execution plan），并且所有用户都可以访问。每个要执行SQL语句的会话都在它的PGA中有一个私有SQL区。每个提交了相同SQL语句的用户都会有一个私有SQL区指向同一个共享SQL区。因此，多个独立的PGA中的私有SQL区可以跟同一个共享SQL区联系起来。

数据库会自动决定何时应用提交了相似的SQL语句。用户和应用直接提交的SQL语句、以及其他语句内部提交的递归SQL语句都在考虑范围中。数据库会进行以下步骤：

1. 检查共享池，查看是否存在包含了语法上（syntactically）或者语义上（semantically）相同的SQL语句的共享SQL区。如果存在相同的语句，数据库就会使用该共享SQL区来执行新的语句，以减少内存消耗；如果没有，数据库就会在共享池中分配一块新的共享SQL区。
2. 代表会话分配一块私有SQL区。如果当前会话是通过共享服务器连接的（而不是专用服务器），部分私有SQL区就会留在SGA中。

下图展示了两个会话通过专用服务器连接的架构，可以看到两个会话都在自己PGA中保留了相同SQL语句的副本。

>**图5 私有SQL区和共享SQL区（专用服务器连接）**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/783cbad9095f1fe5ad8ea018eeb3f3e6.gif#pic_center)


**\## 程序单元**
Library高速缓存中保留了PL/SQL程序和Java类的可执行形式，这些内容被共同称为程序单元（program units）。

**\## 共享池内存分配和复用**
数据库在有新的SQL语句被语法分析后分配共享池内存，除非该语句是一个DDL语句（不能共享）。分配的内存大小取决于SQL语句的复杂度。通常，共享池里的项目会保留直到数据库根据LRU算法要移除它的时候。

**\#2 数据字典高速缓存**
**数据字典**（data dictionary）是包含数据库结构和用户信息的一系列表和视图的集合。Oracle数据库在对SQL语句进行语法分析时会频繁访问数据字典。以下的特殊内存位置被指定用来保留字典数据：

- 数据字典高速缓存：包含了数据库对象的信息，也被称为 row cache；
- library高速缓存。

所有服务器进程都共享上述高速缓存来获取数据字典信息。

**\#3 服务器结果高速缓存**
服务器结果高速缓存（server result cache）是共享池内的一个内存池。与缓冲器池不同，服务器结果高速缓存中保留的是结果集（result set）而不是数据块。其中的内容可以分为SQL查询结果高速缓存和PL/SQL函数结果高速缓存。

**\## SQL查询结果高速缓存**
SQL查询结果高速缓存（SQL query result cache）存储了查询和碎片查询的结果。假设一个应用重复运行同一条 SELECT语句，如果查询结果在SQL查询结果高速缓存中，数据库就能立即返回查询结果，从而避免了重复的开销。

**\## PL/SQL函数结果高速缓存**
PL/SQL函数结果高速缓存（PL/SQL function result cache）存储了函数的结果集。PL/SQL函数代码中可以包括高速缓存其结果的请求。在调用该函数时，系统会先检查高速缓存。如果高速缓存中包含了一个以前的带有相同参数值的函数调用，系统就能直接返回高速缓存的结果集，而不用重复执行函数体。

**\#4 保留池**
保留池（**Reserved pool**）是共享池中Oracle数据库用来分配大块相邻的内存块（chunks）的区域。内存大块（Chunking）能够允许超过5KB的大对象无需单个连续的内存区域就能被载入高速缓存。这样数据库就不太可能会因为碎片化（fragmentation）而用完相邻的内存空间。

## Large池
大型池（large pool）用于分配超出共享池合适范围的内存。大型池可以为以下场景提供大型内存分配：

- 共享服务器的UGA、Oracle XA接口（用于事务与多个数据库交互）；
- 语句并行执行中使用的消息缓冲器（message buffers）；
- 恢复管理器 RMAN IO从库（slaves）的缓冲器。

## Java池
Java池（Java pool）是存储了Java虚拟机（JVM）中所有会话特定的Java代码和数据的内存区域。其中包括了在调用结束时迁移到Java会话空间的Java对象。

对于专用服务器连接（dedicated server connections）来说，Java池中包括了每个Java类的共享部分。对于共享服务器连接，Java池包含了每个Java类的共享部分、以及一些用于每个会话状态的UGA。

## Streams池
流池（Streams pool）存储了缓冲的队列消息，并为Oracle Streams捕获进程（capture processes）、应用进程（apply processes）提供内存。流池仅唯一被Oracle Streams使用。

## Fixed SGA
固定SGA（fixed SGA）是一块内部区域，其中包含：

- 后台进程需要访问的数据库和实例状态的通用信息；
- 进程之间的通信信息，比如锁的信息。

固定SGA的大小由Oracle数据库决定，不能手动修改。

**References**
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/memory-architecture.html#GUID-0788EAEE-0E93-497B-9ACA-401EC0F7BCA1

