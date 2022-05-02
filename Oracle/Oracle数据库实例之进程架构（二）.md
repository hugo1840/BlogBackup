@[TOC](Oracle数据库实例之进程架构（二）)

**后台进程**（background processes）进行运行数据库所必需的维护任务来最大化多用户场景下的数据库性能。每个后台进程都有自己独立的任务，但彼此之间又协同工作。比如，LGWR进程负责将重做日志缓冲器中的数据写入到在线重做日志。如果一个写满的重做日志文件可以被归档了，LGWR进程就会发信号给另一个进程来归档这个重做日志文件。

Oracle数据库会在实例启动时自动创建后台进程。一个数据库实例可以有多个后台进程。下面的查询语句可以查看在你的数据库中运行的后台进程：

```sql
SQL> select PNAME from V$PROCESS where PNAME is not null order by PNAME;
```

# 必需的后台进程
有一些后台进程是所有典型的数据库配置都必需的。即使在以最小化配置初始化参数文件启动的**可读写**数据库实例中，这些后台进程也都存在。但是在只读的数据库中，其中有一些进程可能会被禁用。

这些后台进程包括：PMON、PMAN、LREG、SMON、DBW、LGWR、CKPT、MMON & MMNL、RECO。

## PMON
PMON组（Process Monitor Process Group）包括**PMON**、**CLMN**（Cleanup Main Process）和**CLnn**（Cleanup Helper Processes）。这些后台进程负责监控和清理其他进程。

PMON组监督管理buffer cache的清理、以及客户端进程使用过的资源的释放。例如，PMON组负责重置活跃事务表的状态、释放不再需要的锁、以及从活跃进程清单中移除已经结束进程的ID。数据库必须保证已结束进程占有的资源都被释放，以供其他进程使用。否则，进程可能被阻塞（blocked）或者陷入资源争夺。

**\#1 PMON**
**进程监视器**（PMON）负责检测其他后台进程的终止。如果一个服务器或者调度器进程异常终止，PMON组将负责恢复该进程。进程可能因为多种原因而终止，包括操作系统级别的kill指令、或者 `Alter System Kill Session` 等SQL语句。

**\#2 CLMN**
PMON进程把清理进程的工作委托给了清理主进程（CLMN）。CLMN周期性地清理已经终止的进程、会话、事务、网络连接、空闲会话、断开的事务、以及因为超时而断开的网络连接。

**\#3 CLnn**
CLMN进程将清理工作委托给了CLnn助手进程。CLnn进程会协助已终止进程和会话的清理。CLnn助手进程的数量与清理工作量和清理效率成比例。清理进程可能会被阻塞。如果要清理的进程数很多，清理的时间也可能很长。因此，Oracle数据库使用多个助手进程来并行清理，以提高清理性能。

视图 `V$CLEANUP_PROCESS` 和 `V$DEAD_CLEANUP` 包含有CLMN清理的元数据。每个清理进程在 `V$CLEANUP_PROCESS` 视图中都有一行信息。例如，如果 `V$CLEANUP_PROCESS.STATE` 的值为 BUSY，那么对应的进程当前就在进行清理任务。 

**\#4 资源隔离**
如果一个进程或者会话终止了，PMON组会把其占有的资源释放给数据库使用。在某些情况下，PMON组会自动隔离（quarantine）损坏的、无法恢复的资源，以避免数据库实例被强迫终止。PMON组会继续对占有被隔离资源的进程或会话进行清理。`V$QUARANTINE` 视图中保存了资源类型、消耗的内存、导致资源隔离发生的错误等元数据。

## PMAN
**进程管理器**（Process Manager, PMAN）负责监督管理共享服务器、池化服务器（pooled servers）、工作队列（job queue）等几个后台进程。PMAN负责监控、衍生（spawn）、停止以下类型的进程：

- 调度器和共享服务器进程；
- 连接broker和池化服务器进程（数据库内部连接池使用）；
- 工作队列进程；
- 可重启的后台进程。

## LREG
**监听器注册进程**（Listener Registration Process, LREG）负责向Oracle网络监听器（Oracle Net Listener）注册数据库实例和调度器进程的信息。

当实例启动时，如果LREG检测到监听器在运行，就会把相关参数传递给监听器；如果LREG检测到监听器未运行，就会周期性地重试。在Oracle 12c以前的版本中，PMON进程负责监听器注册。

## SMON
**系统监视进程**（System Monitor Process, SMON）负责多种系统层面的清理工作，包括：

- 如有必要，在实例启动时进行实例恢复。在RAC架构中，SMON进程可以对失败的实例进行恢复。
- 恢复在实例恢复过程中由于文件或者表空间离线而被跳过的终止事务。SMON会在表空间或文件重新上线时恢复被终止的事务。
- 清理未使用的临时段（segment）。例如，数据库在创建索引时会分配区（extents）。如果索引创建失败，SMON就会清理分配的临时空间。
- 字典管理的表空间内部的联合相邻自由区（coalescing contiguous free extents）。

SMON进程定期检查是否需要工作。其他进程也可以根据需求调用SMON。

## DBW
**数据库写进程**（Database Writer Process, DBW）负责将数据库缓冲器内被修改的数据写回磁盘的数据文件中。

尽管一个数据写进程（DBW0）对于大多数系统来说是足够的，你也可以根据需求配置多个写进程——从DBW1到DBW9、从DBWa到DBWz、从BW36到BW99——来提高性能。

DBW进程在以下情形时将脏的缓冲器写入磁盘：

- 当服务器进程在扫描了一定数量的缓冲器后仍然找不到干净可复用的缓冲器时，它会发信号给DBW进程来写盘。DBW尽可能将脏数据异步写入磁盘。
- DBW进程周期性地将缓冲器刷盘以推进检查点（checkpoint）。检查点是重做线程开始实例恢复的位置。检查点的日志位置由buffer cache中存在时间最长的脏缓冲器决定。

在许多情况下，DBW写入磁盘的数据块是分散的。因此，DBW写盘的速度比顺序写的LGWR线程要慢。DBW通过多数据块写（multiblock writes）来提升效率。多数据块写中数据块的数目随操作系统而变化。

## LGWR
**日志写进程**（Log Writer Process, LGWR）管理在线重做日志缓冲器。

LGWR将缓冲器缓冲器的一部分写入在线重做日志。通过将修改数据库缓冲器、将脏的缓冲器数据分散写入磁盘、以及将重做日志顺序写入磁盘分开进行，数据库能够提升性能。

在发现下面的情形时，LGWR会将所有从上次写盘以来拷贝到缓冲器的重做记录写入磁盘：

- 有用户提交了事务；
- 发生了在线重做日志切换（log switch）；
- 从上次LGWR写盘已经过了**3秒**；
- redo log buffer已经写满**三分之一**或者缓冲的数据达到了**1MB**；
- DBW必须将被修改的缓冲器写盘。
  在DBW将脏的缓冲器写盘之前，数据库必须将对应的重做记录写盘（即**日志先写原则**，Write-ahead protocol）。如果DBW发现重做日志尚未写盘，就会发信号给LGWR进程将日志写盘，并等到重做日志落盘后才会将缓冲器脏数据刷盘。

**\#1 LGWR和提交**
Oracle数据库提供了一种快速提交机制来提升事务提交的性能。当有用户发出 COMMIT 指令时，对应的事务会被分配一个 **SCN**（system change number）。LGWR将一条提交记录放入redo log buffer，并立即将缓冲器、提交指令的SCN、以及事务的重做记录一起写盘。

Redo log buffer是循环写的。当LGWR将其中的重做记录写到在线重做日志文件后，服务器进程可以复制新的记录来覆写redo log buffer中已经被持久化写盘的日志记录。LGWR通常写的很快，足以保证redo log buffer中一直有可用空间，即使是在对在线重做日志的访问量特别大的时候。

==包含有事务提交记录的重做记录==的**原子写**（atomic write）是决定事务成功提交的唯一事件。Oracle数据库会向提交的事务返回一个成功码（success code），即使缓冲器数据还未写入磁盘。对应的数据块修改操作会被延迟到DBW将脏的数据快写入数据文件。

当事务活动频繁时，LGWR会使用组提交。比如，有一个用户提交事务，导致LGWR将事务的重做记录写盘。与此同时，也有其他用户提交了事务。LGWR需要等到前一个写盘结束后，才能将这些等待的事务清单一起刷盘。这样做可以减少磁盘IO，同时最大化性能。

**\#2 LGWR和不能访问的文件**
LGWR同步地写入在线重做日志文件的活跃镜像组（active mirrored group）。如果有一个日志文件无法访问了，LGWR会继续写入日志组中的另一个文件，同时将错误信息写入LGWR trace文件和 alert日志。如果日志组中的所有文件都损坏了，或者日志组由于未归档而暂时不可用，LGWR就不能继续工作了。

## CKPT
**检查点进程**（Checkpoint Process, CKPT）负责利用检查点信息更新控制文件和数据文件头部（headers），并发信号给DBW将数据块写盘。检查点信息包括检查点位置、SCN、以及在线重做日志中开始恢复的位置。

如下图所示，CKPT进程不会将数据块写入数据文件（由DBW进程负责），也不会将重做日志块写入在线重做日志文件（由LGWR进程负责）。

>**图1 检查点进程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/324a0801bda4d117d37eeb4aa298cfa9.gif#pic_center)


## MMON & MMNL
可管理性监控进程（Manageability Monitor Process, **MMON**）负责与自动工作负载信息库（Automatic Workload Repository, **AWR**）有关的许多任务。

比如，当有度量标准（metric）违反了阈值时，MMON会对最近修改过的SQL对象进行记录、拍快照、抓取统计信息等操作。**MMNL**（Manageability monitor lite process）则负责在SGA中的活跃会话历史（Active Session History, ASH）缓冲器写满时将其中的统计信息写入磁盘。

## RECO
在分布式数据库中，**恢复进程**（Recoverer process, RECO）会自动解决分布式事务（distributed transaction）失败的问题。

一个节点的RECO进程会自动连接到未决（in-doubt）分布式事务涉及的其他数据库。当RECO进程在数据库之间重新建立连接后，它会自动解决所有未决的事务，从每个数据库的等待（pending）事务表中移除对应已解决事务的相关行。

# 可选的后台进程
大多数可选的后台进程都与特定的功能有关。比如，支持ASM的可选后台进程仅仅在ASM开启时可用。

## ARCn
**归档进程**（Archiver Process, ARCn）负责在**重做日志切换**发生后将重做日志文件拷贝到离线存储中。

这些进程也可以搜集事务重做数据，并将它们传输给备库（standby database）。ARCn进程只有在数据库处于归档模式（**ARCHIVELOG** mode）且开启了自动归档后才会存在。

## CJQ0 & Jnnn
**队列进程**（Queue process）负责运行用户任务（jobs），且通常是以批量模式（batch mode）。Job是用户定义可以运行一次或多次的任务。

动态任务队列进程可以同时运行多个任务。事件的顺序如下：

1. 任务调度进程（Job coordinator process, **CJQ0**）自动启动，根据需求被Oracle Scheduler停止。CJQ0周期性地从系统 `JOB$` 表中选取要运行的任务。按时间对新选择的任务排序。
2. CJQ0进程动态地生成任务队列slave进程（Job queue slave process, **Jnnn**）来运行任务。
3. Jnnn进程运行CJQ0进程选择的任务中的一个来执行。每个任务队列进程同一时间只运行一个任务。
4. 在Jnnn完成一个任务以后，会请求其他的任务来执行。如果没有被分配新的任务，它就会进入睡眠状态，但是会周期性地醒来请求新的任务。在经历一段预先设定的时间后，如果仍然没有被分配新的任务，该Jnnn进程就会终止。

初始化参数 `JOB_QUEUE_PROCESSES` 设定了在一个实例中能够同时运行的Jnnn进程的最大数量。

## FBDA
**闪回数据归档进程**（Flashback data archive process, FBDA）负责将被跟踪表（tracked table）的历史记录归档到闪回数据档案。

## SMCO
**空间管理调度进程**（Space Management Coordinator Process, SMCO）负责协调各种空间管理相关任务的执行。典型的任务包括主动空间分配（proactive space allocation）和空间回收（space reclamation）。SMCO动态地生成slave进程（**Wnnn**）来实施任务。

# Slave进程
Slave进程是为其他进程工作的后台进程。Oracle数据库使用的slave进程包括I/O slave进程（**Innn**）、并行执行（PX）服务器进程等。


**References:**
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/process-architecture.html#GUID-13FE4098-61DF-4D76-882D-551A88E0EBB8

