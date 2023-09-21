---
tags: [oracle]
title: Oracle常见的等待事件
created: '2023-09-21T08:20:45.869Z'
modified: '2023-09-21T08:39:57.590Z'
---

Oracle常见的等待事件

# 等待事件分类
等待事件有如下分类：
1. **administrative**：DBA操作导致的用户等待，例如重建索引；
2. **application**：用户程序代码导致的等待，例如行锁或者直接执行LOCK命令导致的锁等待；
3. **cluster**：RAC集群资源相关的等待。例如global cache资源相关的gc cr block busy等待；
4. **commit**：等待事务提交后的redo日志写入确认，即log file sync等待；
5. **concurrency**：等待数据库内部资源，例如latch；
6. **configuration**：由于数据库和实例配置不合理导致的等待，例如log file size和shared pool size配置过小；
7. **idle**：会话处于非活跃状态（可以理解为等待其他会话的输入），例如`SQL*Net message from client`；
8. **network**：与网络数据传输相关的等待，例如`SQL*Net more data to dblink`；
9. **queueing**：在流水线环境中获取额外数据产生的延迟，通常表明流水线存在低效或其他问题。
10. **scheduler**：资源管理器相关的等待，例如`resmgr: become active`；
11. **system I/O**：等待后台进程I/O，例如DBWR进程的db file parallel write等待；
12. **user I/O**：等待用户I/O，例如db file sequential read等待。
13. **other**：非典型的等待事件，例如wait for EMON to spawn；


# 常见等待事件
## buffer等待事件
Buffer等待事件通常发生在多个进程并发访问buffer cache中的同一块缓存区时。几个常见的buffer等待事件：
1. `buffer busy waits`：当前会话试图修改一个数据块，但是这个数据块正在被另一个会话修改。

2. `read by other session`：当前会话视图读取一个数据块，但是这个数据块正在被另一个会话读取到内存中。

上面两个等待事件通常发生在数据库中存在热块的时候，即有多个会话频繁地读取或修改同一个数据块时。RAC模式下还存在类似的gc buffer busy acquire和gc buffer busy release等待事件。

3. `buffer latch`：由于buffer hash chain太长，当前会话在该哈希列表中检索数据块地址话费的时间太长，导致其他会话等待。

数据库内存中数据块的位置记录在一个名为buffer hash chain的哈希列表中。当一个会话需要访问某个数据块时，它首先要搜索buffer hash chain来获得数据块的地址，然后通过这个地址去访问目标数据块。Buffer hash chain会使用一个Latch来保护其完整性。当一个会话需要访问buffer hash chain时，需要获取一个Latch，来保证buffer hash chain在该会话检索过程中不发生变化。

4. `free buffer waits`：当数据块被读入到内存中时，需要在内存中找到空闲的内存空间（free buffer）来存放其数据。如果找不到足够的空闲内存，数据库就会通知DBWR将内存中的脏数据（dirty buffer）落盘，从而产生该等待事件。

DBWR进程刷脏数据等待产生的原因有如下几种：
- 系统I/O出现瓶颈；
- DBWR正在等待某些资源释放，比如Latch；
- Buffer cache太小，空闲内存经常不够，DBWR进程绝大部分时间都在刷脏数据；
- Buffer cache太大，内存中脏数据太多，单个DBWR进程很难及时通过刷脏数据释放足够的内存空间。


## gc等待事件
RAC环境中的buffer等待事件：
1. `gc buffer busy acquire`：本地实例的一个会话试图访问**远程**实例的buffer，但是**本地**实例上已经有另一个会话正在访问相同的buffer（并且尚未释放），从而产生的等待事件。

2. `gc buffer busy release`：本地实例的一个会话试图访问**本地**实例的buffer，但是**远程**实例上已经有另一个会话正在访问相同的buffer（并且尚未释放），从而产生的等待事件。

以上两个等待事件都是由于跨实例节点访问buffer导致的。可能的原因包括：
- 热块（Hot block）争用；
- `gc block busy`或`enq: TX - row lock contention`等其他等待事件的副作用；
- 高网络延迟或者其他网络问题；
- 内存不足导致的服务器繁忙或者内存交换（swapping）；
- 低效的SQL语句；
- 同一数据在不同实例之间交叉访问；
- 数据库内部Bug等。


## library cache等待事件
发生在shared pool中库缓存区域的等待事件。Library cache缓存了已经编译好的可执行SQL和PL/SQL代码，以便在需要再次执行时可以快速获取并重用。

1. `library cache lock`：库缓存锁是Oracle数据库中用于控制不同用户之间并发访问共享对象的机制。当有会话正在操作库缓存中的数据库对象时，库缓存锁就能控制对共享对象的并发访问，以防止其他会话在重新编译视图、过程、包或者修改对象定义时访问该对象。

2. `library cache pin`：当一个会话尝试在library cache中锁定一个对象以对其进行修改或检查时发生的等待事件。一个会话在操作library cache中的共享对象之前需要先锁定该对象（在共享池中pin住），以确保在操作期间它不会被其他会话修改。如果该对象已经被其他会话持有，就会产生library cache pin等待事件。该等待事件通常发生在重新编译视图或PL/SQL代码时。


## cursor等待事件
游标相关的等待事件是SQL解析的等待，通常与将游标加载到Shared Pool、以及在Shared Pool中搜索游标有关。

Cursor等待事件通常有由以下原因导致：
- shared pool中的游标版本数目过多；
- 硬解析或软解析过多；
- 过多的游标失效或重新载入；
- shared pool大小设置不合理；
- 游标资源持有者被操作系统或资源管理器调度出CPU；
- 操作系统内存管理配置缺陷（例如没有开启大页）；
- 代码缺陷（例如内部Bug）。

常见的cursor等待事件：
1. `cursor: mutex S`：当前会话正在以共享模式请求一个mutex锁，但是该mutex正被另一个会话在同一个游标上以独占模式持有。

2. `cursor: mutex X`：当前会话正在以独占模式请求一个mutex锁，但是该mutex或者正被另一个会话以独占模式持有、或者正被一个或多个其他会话以共享模式持有。

以上两种等待事件通常发生在访问父游标、检查游标相关统计信息、创建子游标、捕获SQL绑定变量、更新游标统计信息（`v$sqlstats`）时。

3. `cursor: pin S`：当一个会话试图以共享模式来锁定一个游标时，需要通过一个独占的原子操作来更新游标mutex的reference count。如果此时有其他会话正在修改同一个游标mutex的reference count，就会产生该等待事件。

该等待事件通常是由于某些SQL以超高频繁的频率执行导致的，也可能与系统CPU能力不足有关。最简单的解决方案是将频繁执行的SQL分区拆解，以分散竞争。例如通过Hint注释将同一条SQL分解为2条，分别使用不同的游标。

4. `cursor: pin S wait on X`：当前会话试图以共享模式锁定一个游标，但是该游标已经被另一个会话以独占模式锁定并正在执行加载（parsing）。该等待事件通常与高频率的硬解析、High version counts（子游标过多）、或者内部bug有关。

5. `cursor: pin X`：当前会话试图以独占模式锁定一个游标，但是该游标已经被另一个会话以独占模式锁定、或者已经被一个或多个其他会话以共享模式锁定。


## direct path等待事件
Direct path等待事件涉及到绕过SGA直接将数据从磁盘读入PGA、或直接从PGA写入磁盘的操作：
1. `direct path read`：直接路径读。发生在用户会话直接将数据块从磁盘读入到PGA中（而不是SGA）。该等待事件通常发生在以下几种情况下：
  - 排序操作中的数据量过大，部分参与排序的数据被写到磁盘中，这部分数据后续会通过直接路径读被读入到内存中；
  - 并行扫描数据的操作；
  - Oracle进程读入数据块的速度快于系统I/O的处理速度。通常发生在系统I/O负载很高的时候。

2. `direct path write`：直接路径写。与直接路径读刚好相反，发生在PGA中的数据直接写入到磁盘文件时（而不经过DBWR和buffer cache）。该等待事件通常发生在以下几种情况下：
  - 内存不足时使用临时表空间来进行排序；
  - 并行DML操作；
  - 直接路径插入（Direct Path Insert）；
  - 并行的**CREATE TABLE AS SELECT**语句；
  - 某些LOB大对象操作。


## control file等待事件
控制文件相关的等待事件：
1. `control file parallel write`：控制文件并行写。发生在Oracle将数据并行写入多个控制文件时。控制文件并行写通常发生在控制文件需要频繁更新（例如日志切换太过频繁）、或者系统I/O整体出现瓶颈时。

2. `control file sequential read`：控制文件顺序读。发生在读取控制文件信息时，例如：
  - 备份控制文件；
  - RAC环境中不同实例间共享控制文件信息；
  - 读取控制文件的文件头（header）；
  - 读取控制文件中的其他信息；


## db file等待事件
访问数据文件时发生的等待事件：
1. `db file parallel read`：数据文件并行读。发生在数据库恢复时，需要被恢复的数据块以并行的方式被读到内存中。

2. `db file parallel write`：数据文件并行写。数据库后台进程DBWR向磁盘写入脏数据时发生的等待。DBWR后台进程会批量地将脏数据写入到磁盘上的数据文件中，在当前的批次作业完成前，DBWR进程会出现该等待事件。该等待事件一般对用户进程无影响。通过启用异步I/O可以缓解该等待事件。

3. `db file sequential read`：数据文件顺序读。数据库每次I/O读取单个数据块时发生的等待事件。通常发生在通过索引或ROWID直接访问行记录、重建数据文件、导出数据文件头的时候。

4. `db file scattered read`：数据文件离散读。数据库每次I/O读取多个数据块时发生的等待事件。通常发生在用户发出的SQL导致全表扫描、或者索引快速扫描的时候。此处的**离散**并不是读取数据块的方式，而是指被读取的多个数据块在内存中不是以连续的方式存放的。

5. `db file single write`：数据库更新数据文件头（header）信息时发生的等待，例如发生checkpoint时。


## log file等待事件
日志文件相关的等待事件。

1. `log buffer space`：发生在log buffer中没有足够的空间来存放新产生的redo数据时。通常是因为redo log生成的速度大于LGWR进程将redo脏数据刷盘的速度。该等待事件可以通过扩容log buffer、或者使用I/O性能更好的磁盘存放redo文件来缓解。

2. `log file parallel write`：发生在多个LGWR后台进程将Redo log buffer中的脏数据并行写入磁盘中的日志文件时。该等待事件发生的原因一般包括磁盘I/O性能不足、redo日志文件在磁盘中的分布导致I/O争用、log buffer大小设置不合理等。

3. `log file sequential read`：redo日志顺序读。通常发生在对在线redo日志进行归档时，ARCH进程会读取在线日志信息。由于redo日志是顺序写的，因此读取redo日志是也是顺序读。

4. `log file single write`：发生在更新redo日志的文件头时。向一个日志组中增加新的日志文件成语、或者增加日志文件的sequence号时，LGWR进程都会更新redo日志文件的文件头。

5. `log file switch (archiving needed)`：发生在归档模式下进行日志切换，但是要切换到的日志还没有被ARCH进程归档完毕的时候。发生该等待事件时最好检查ARCH进程是否正常工作。通过增加归档进程、将归档文件存放到I/O性能更好的磁盘上来缓解该等待事件。

6. `log file switch (checkpoint incomplete)`：当一个在线日志切换到下一个在线日志时，必须保证后者记录的信息已经被写到磁盘上（checkpoint）。为了保证能够在数据库宕机后顺利进行实例恢复，必须保证在线日志checkpoint完成后才能被覆写。通过增加redo日志文件大小、或者增加日志组数量可以缓解该等待事件。

7. `log file sync`：日志文件同步，用户会话导致的等待事件。当用户发出COMMIT或者ROLLBACK命令时，LGWR进程会将事务产生的redo数据从log buffer写到磁盘上，并在写入完成后通知用户会话。该等待事件由以下几个过程组成：
  - 用户进程发起COMMIT或ROLLBACK命令；
  - 前台进程通知LGWR进程写日志；
  - LGWR进程收到请求后将日志刷盘（log file parallel write）；
  - LGWR进程写入完成并通知前台进程；
  - 用户完成COMMIT或ROLLBACK命令。


## SQL*Net等待事件

1. `SQL*Net break/reset to client`：服务端在给客户端发送一个断开或重置连接的请求，正在等待客户端的响应。通常是服务端到客户端之间的网络不稳定导致的。

2. `SQL*Net break/reset to dblink`：服务端通过DBLINK向另一个服务端进程发送一个断开或重置连接的请求，正在等待对方的响应。通常是两个服务端之间的网络不稳定导致的。

3. `SQL*Net message from client`：当一个会话建立成功后，客户端会向服务端发送请求，服务端在处理完请求后将结果返回给客户端，并等待客户端的下一步请求。这是一个空闲等待事件。如果该等待事件占据了大部分的等待时间，可能代表客户端和服务端之间的网络通信存在问题、或者客户端进程出现了资源瓶颈。

4. `SQL*Net message from dblink`：与前一个等待事件类似，发生在服务端通过DBLINK访问另一个服务端进程时。这也是一个空闲等待事件。该等待事件产生的原因可能有：
  - 两个服务端之间的网络通信不稳定；
  - 通过DBLINK发送到远端数据库上执行的SQL花费了较长时间；
  - 通过DBLINK往返通信的次数过多，多次通信累计延迟造成的等待。

5. `SQL*Net message to client`：当服务端向客户端发送消息时，由于网络原因、或者客户端繁忙无法接收服务端发来的消息，导致服务端产生的等待事件。

6. `SQL*Net message to dblink`：当服务端通过DBLINK向远端服务进程发送消息时，由于网络原因、或者远端服务进程繁忙无法接收发来的消息，导致的等待事件。

7. `SQL*Net more data from client`：服务端等待客户端进一步发送数据，以便完成操作而产生的等待事件。比如有一个超大的SQL文本，无法通过单个SQL*Net数据包传输完成，服务端需要等待客户端分多次传输，直到把整个SQL文本传输过来才能进行下一步处理。

8. `SQL*Net more data from dblink`：与前一个等待事件类似，服务端等待远端服务进程通过DBLINK进一步发送数据，以便完成操作而产生的等待事件。


## enq: TX等待事件

1. `enq: TX - row lock contention`：在事务处理过程中，多个会话试图同时访问同一行数据时，导致行锁争用产生的等待事件。


**References**
【1】https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/descriptions-of-wait-events.html#GUID-2FDDFAA4-24D0-4B80-A157-A907AF5C68E2
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/instance-tuning-using-performance-views.html#GUID-03401D0F-DB3E-49E5-89E0-2F2A6164A5C0
【3】http://www.eygle.com/archives/2013/01/awr_cursor_pin_share_case.html
【4】https://www.askmac.cn/archives/understanding-oracle-mutex.html
【5】Doc ID 1356828.1
【6】Doc ID 1349387.1
【7】Doc ID 2256780.1
【8】https://www.modb.pro/db/569308


