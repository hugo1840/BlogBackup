---
tags: [redis]
title: Redis持久化之AOF
created: '2022-11-10T06:57:55.161Z'
modified: '2022-11-14T15:42:08.331Z'
---

Redis持久化之AOF

持久化（Persistence）是指将数据写到比如固态硬盘等持久化的存储中。Redis支持以下几种持久化方式：
- **RDB**：即*Redis Database*，是指通过在不同的时间点，将数据库的快照（snapshot）以二进制格式保存到文件中，这样Redis重启后直接加载数据。
- **AOF**：即*Append Only File*，是指在Redis运行期间，不断将Redis执行的写命令（修改数据的命令）写到文件中，Redis重启后，只要将这些写命令重复执行一遍就可以恢复数据。
- **RDB+AOF**：同时使用RDB和AOF两种持久化方式。


# AOF常用配置
AOF持久化相关的配置如下：
- `appendonly`：默认为no，配置为yes才能启用AOF持久化机制。
- `appendfilename`：默认为`appendonly.aof`，AOF文件名。
- `dir`：AOF文件的保存路径，默认为`./`。
- `appendfsync`：默认为`everysec`，代表每秒执行一次AOF缓冲区数据磁盘同步操作。
- `no-appendfsync-on-rewrite`：默认为no，代表存在RDB或者AOF子进程时，仍允许AOF缓冲区数据磁盘同步。
- `auto-aof-rewrite-percentage`和`auto-aof-rewrite-min-size`：默认为100、64MB，代表上次AOF重写后，AOF文件大小的增长率超过100并且当前AOF文件大小大于64MB时，执行AOF重写操作。
- `aof-use-rdb-preamble`：默认为yes，表示开启AOF混合持久化。


# AOF持久化的优势
AOF持久化主要有以下几点优势：
- AOF的数据持久化更好，它支持三种fsync策略：`no`表示不调用fsync函数（由操作系统自行决定何时将数据变化落盘），`everysec`表示每秒调用一次fsync函数刷盘，`always`表示在写命令每次被写入AOF后都调用fsync函数。fsync函数是由后台线程调用的，在默认策略（everysec）下，只会丢失一秒内的数据。
- AOF日志是一种仅支持**追加写入**的日志文件，即使发生断电时已经写入的日志内容也不会受到影响。如果由于磁盘写满等其他原因导致某个命令只写了一半，也可以使用`redis-check-aof`工具进行修复。
- 当AOF文件变得太大时，可以自动在后台进行**AOF重写**。AOF重写会剔除掉冗余的AOF日志内容，在能够生成相同数据集的前提下缩小日志大小。AOF重写是安全的，因为重写时Redis依然会把新的写命令写入旧的AOF文件。等到重写的新AOF文件就绪时，Redis通过日志切换将后续发生的写命令都写入新的AOF文件。
- AOF文件以易于理解和解析的方式按先后顺序依次包含所有写命令。AOF可以轻松地导出。假设你不小心执行`FLUSHALL`命令删除了所有数据库的所有键，只要AOF日志还没有进行重写，你就可以通过停止Redis服务、然后在AOF文件中移除`FLUSHALL`命令、最后重启Redis的方法来挽救你的数据。


# AOF持久化的缺点
AOF持久化主要有以下劣势：
- 对于同一个数据集而言，它对应的AOF文件通常要比RDB文件更大。
- 取决于具体的fsync策略，AOF可能会比RDB更慢。一般来说，在默认的fsync策略下，AOF的性能还是很高的；禁用fysnc时，即使在高负载的情况下，理论上AOF也应该和RDB一样快。

对于**7.0**之前的版本，AOF持久化还存在以下缺点：
- 如果在AOF重写期间有对数据库的写操作，AOF可能会占用较多内存，因为重写期间的写命令都被缓存在内存中，并且会在重写结束时被写入到新的AOF文件中。
- 重写期间执行的写命令都会被写入磁盘两次。
- Redis可能会在AOF重写结束时“冻结”写操作并将这些写命令写入新的AOF日志。


# RDB还是AOF？
如果想要达到与PostgreSQL相当的数据安全级别，应该同时使用RDB和AOF两种持久化方法。

如果能够容忍**分钟级别**的数据量丢失，就可以单独使用RDB持久化。

官方不建议单独使用AOF持久化，因为RDB快照在**数据备份**和**快速重启**方面有很大的优势，也能在AOF引擎故障时提供额外的数据安全保障。


# 仅追加文件
快照文件的数据安全性不及AOF文件。如果服务器发生故障、断电或者你不小心执行`kill -9`命令杀掉了Redis实例进程，最近几分钟内被修改的数据就可能丢失。

AOF文件最早于Redis 1.1开始支持。在配置文件中添加`appendonly yes`来开启AOF持久化。从Redis 7.0开始，原来单个的AOF文件被划分为一个base文件和多个增量文件。其中base是在重写AOF文件时对数据集的一个初始快照（可以是RDB格式也可以是AOF格式）。增量文件则记录了自上一个base文件被创建以来数据发生的变化。所有这些文件都存放在一个单独的目录中，并且由一个清单文件（manifest file）进行跟踪。


# 日志重写
AOF文件随着写命令的执行会变得越来越大。比如当我们讲一个计数器递增100次，最终我们只会得到一个包含最终值的键，但是在AOF文件中会写入100条记录。

AOF重写是安全的，因为AOF重写的过程中Redis依然会把新的写命令写入旧的AOF文件，直到重写的新AOF文件就绪时，Redis就会通过日志切换将后续发生的写命令都写入新的AOF文件。

Redis在后台对AOF日志进行重写，不会影响对客户端的正常服务。当你执行**BGREWRITEAOF**命令时，Redis都会记录下重建内存中当前数据集所需的最短命令序列。从Redis **2.4**开始支持自动触发AOF日志重写。此前的版本中需要用户不时间断地执行**BGREWRITEAOF**命令来触发AOF重写。

从Redis 7.0.0开始，在进行AOF重写时，Redis父进程会生成一个新的增量AOF文件。`bgrewriteaof`子进程执行重写逻辑并生成一个新的base文件。Redis通过一个临时清单文件来跟踪新生成的base文件和增量文件。当重写的文件就绪后，Redis会执行一个原子性的日志替换操作，使这个临时清单文件生效。为了避免在AOF重写失败并反复重试的情况下创建大量增量文件的问题，Redis引入了AOF重写限制机制，以确保AOF重写失败后重试的频率逐渐降低。

# AOF文件的持久性
AOF文件的fsync支持三种配置策略：
- `appendfsync no`：表示不调用fsync函数，由操作系统自行决定何时将数据变化落盘。性能最好，但是宕机时丢失的数据量可能会很多。通常来说，Linux操作系统每30秒会将内存数据刷盘一次，但具体要取决于内核参数配置。
- `appendfsync everysec`：表示每秒调用一次fsync函数刷盘。性能较好（从Redis 2.4起几乎可以赶上快照的性能），宕机时最多丢失一秒的数据量。
- `appendfsync always`：表示在写命令每次被写入AOF后都调用fsync函数。性能最差，但是数据安全性最高。需要注意的是，每一批来自不同客户端的写命令被执行后会被一起添加到AOF中，而**并不是每执行一次写命令就fsync一次**。该配置支下持组提交（group commit）。

官方建议的默认配置是`appendfsync everysec`。

# AOF文件被截断了怎么办？
Redis服务器可能在写AOF时崩溃，磁盘也有可能在AOF写到一半时满了，发生这两种情况时AOF文件中最后写入的一个命令有可能被截断（truncated）。即使发生截断，Redis也能正常载入AOF文件，只需要丢弃掉文件中最后一个格式不正确的命令。此时Redis会输出下面的日志信息：

```
* Reading RDB preamble from AOF file...
* Reading the remaining AOF tail...
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```
你也可以修改默认配置，让Redis在发现AOF日志截断时停止启动。

老版本的Redis在遇到AOF文件截断时可能无法自动恢复，需要进行以下步骤：
- 对AOF文件进行备份；
- 使用Redis自带的`redis-check-aof`工具修复原始的AOF文件：
```bash
$ redis-check-aof --fix <filename>
```

- 使用`diff -u`命令检查两个文件的差别；
- 利用修复好的AOF文件重启Redis服务。


# AOF文件损坏了怎么办？
如果AOF文件不仅被截断，而且文件中还被无效的字节序列损坏，事情就变得更加复杂了。Redis会在启动时报错并中断：

```
* Reading the remaining AOF tail...
# Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>
```

最好的解决办法是首先使用**不**带`--fix`选项的`redis-check-aof`工具检查AOF文件，在详细了解问题后，跳转到文件中指定的偏移量位置，看看是否可以手动修复文件（AOF文件使用与Redis协议相同的格式，手动修复非常简单）。

如果不能手动修复，就只能退而求其次，使用带`--fix`选项的`redis-check-aof`工具修复AOF文件。但是这种情况下，**从文件中无效的字节开始到文件末尾之间所有的记录都有可能被丢弃**。如果文件损坏发生在文件中比较靠前的部分，就可能导致大量数据的丢失。


# 日志重写工作原理
AOF重写同样也采用了快照中使用到的**写时复制**（*Copy-On-Write, COW*）策略。

在不同版本中AOF重写的流程大致如下：

## Redis 7.0及其以后的版本
- Redis父进程调用fork函数创建一个子进程（称为AOF进程）。
- AOF子进程将重写的**base AOF**日志写入一个临时文件。
- 与此同时，Redis父进程会创建一个新的**增量AOF**文件来保存日志重写期间发生的写操作。如果AOF重写失败了，旧的base AOF、旧的增量AOF文件、再加上重写时新创建的增量AOF文件能够组成一个完整的数据集，因此数据是安全的。
- 如果AOF子进程顺利完成了base AOF日志重写，父进程在收到信号后，会使用重写时新创建的增量文件和AOF子进程新生成的base AOF文件来构建一个临时清单，并进行持久化。
- 最后，Redis通过一个原子性的替换操作，使得通过日志重写新生成的base AOF和增量文件生效，并清除掉旧的base文件和未使用的增量文件。


## Redis 7.0之前的版本
- Redis父进程调用fork函数创建一个子进程（称为AOF进程）。
- AOF子进程将重写的AOF日志写入一个临时文件。
- 父进程负责将日志重写期间发生的写操作写入内存中的**AOF重写缓冲区**。同时，父进程也会将重写期间发生的写操作写入旧的AOF文件，以保证即使AOF重写失败也不会丢失数据。
- 如果AOF子进程顺利完成了AOF日志重写，父进程在收到信号后，会把AOF重写缓冲区中的写命令添加到子进程生成的AOF文件末尾。
- Redis以原子性的方式将新的AOF文件重命名为旧的AOF文件的名字，并将之后发生的写命令到写入到新的AOF文件中。


# RDB与AOF的关联
Redis 2.4及其以后的版本中，会确保在生成RDB快照的过程中不会触发AOF重写，同时在AOF重写的过程中不会允许执行**BGSAVE**命令。这么做是为了避免生成RDB快照和重写AOF的两个后台进程同时进行大量的磁盘I/O。

在正在创建RDB快照时，如果用户通过执行**BGREWRITEAOF**命令显示地要求进行AOF重写时，Redis服务将回复一个OK状态代码，告知用户重写操作已安排，并且一旦快照完成，AOF重写就会开始。

在RDB和AOF同时启用的情况下，Redis发生重启后，**AOF将优先被用来重建原有的数据集**，因为AOF中包含的数据更加完整。


# AOF模式下的备份
仅开启了AOF持久化的Redis实例也可以进行备份。从`Redis 7.0.0`开始，AOF文件被拆分为多个文件，这些文件被保存在由`appenddirname`配置确定的单个目录中。正常情况下，只需复制打包该目录中的文件即可实现备份。但是，如果打包操作是在AOF重写期间完成的，最终可能会得到一个无效的备份。要解决这个问题，必须在备份期间禁用AOF重写：

1. 执行下面的命令来禁用自动重写：
```bash
CONFIG SET auto-aof-rewrite-percentage 0
```
并确保在备份期间不会手动触发重写（通过执行BGREWRITEAOF命令）。

2. 执行下面的命令来确认没有正在进行中的重写：
```bash
INFO persistence
```
并确认`aof_rewrite_in_progress`的值为0。如果该参数的值为1，则需要等待重写过程结束。

3. 复制`appenddirname`配置的路径下的AOF文件。

4. 执行下面的命令来重新启用自动重写：
```bash
CONFIG SET auto-aof-rewrite-percentage <prev-value>
```

如果你想最大限度地减少禁用AOF重写的时间，可以创建指向`appenddirname`中文件的**硬链接**（在上面的步骤3中），然后在创建硬链接后重新启用重写（步骤4）。接着你就可以复制打包硬链接文件，并在完成后将其删除。这样打包的备份文件中就不会包含正在重写的AOF文件。

考虑到备份期间服务器可能会重启，为了确保重启后不会自动开始重写，你也可以修改上面的步骤1，通过**CONFIG REWRITE**命令来持久化被修改的配置。确保在步骤4完成后重新启用自动重写，并再次通过**CONFIG REWRITE**命令来持久化该配置。

在`7.0.0`版本之前，备份AOF文件可以简单地通过复制AOF文件来完成（就像备份RDB快照一样）。该文件可能缺少最后一部分（考虑到日志截断），但Redis仍然能够加载它。


**REFERENCES**
【1】https://redis.io/docs/management/persistence/
【2】https://raw.githubusercontent.com/redis/redis/6.0/redis.conf








