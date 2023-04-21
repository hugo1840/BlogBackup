---
tags: [oracle]
title: Oracle数据库查看与修改内存配置
created: '2023-04-21T11:20:38.217Z'
modified: '2023-04-21T12:59:26.165Z'
---

Oracle数据库查看与修改内存配置

# Oracle内存管理模式
Oracle数据库的内存管理模式从自动管理化程度由高到低依次可以分为：

- **自动内存管理**：完全由Oracle自动管理内存分配。DBA只需设置`MEMORY_TARGET`（以及可选初始化参数`MEMORY_MAX_TARGET`），Oracle就会在SGA和PGA之间自动分配内存。
- **自动共享内存管理**：DBA只需设置`SGA_TARGET`和`PGA_AGGREGATE_TARGET`两个初始化参数。Oracle会分别在SGA和PGA中自动分配各组件的内存。
- **手动内存管理**：由DBA为SGA和PGA中的所有组件逐一手动分配内存。

在自动共享内存管理模式下，还可以手动为SGA中的某些重要组件指定**最小**的内存分配值，例如Shared Pool和Buffer Cache。

# 查看Oracle内存分配
检查各内存参数的TARGET配置：
```sql
SQL> show parameter target

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
memory_max_target                    big integer 0
memory_target                        big integer 0
pga_aggregate_target                 big integer 1561M
sga_target                           big integer 4688M
```
其中，`memory_target`和`memory_max_target`都为0，并且`sga_target`和`pga_aggregate_target`不为0，表示当前数据库使用的是自动共享内存管理模式。

检查SGA和PGA相关参数的配置：
```sql
SQL> show parameter sga

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
sga_max_size                         big integer 4688M
sga_min_size                         big integer 0
sga_target                           big integer 4688M

SQL> show parameter pga

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
pga_aggregate_limit                  big integer 3122M
pga_aggregate_target                 big integer 1561M
```

查看SGA中各组件的内存使用情况：
```sql 
SQL> select * from v$sgainfo;

NAME                                  BYTES RESIZEABLE     CON_ID
-------------------------------- ---------- ---------- ----------
Fixed SGA Size                      8906552 No                  0
Redo Buffers                        7868416 No                  0
Buffer Cache Size                3992977408 Yes                 0
In-Memory Area Size                       0 No                  0
Shared Pool Size                  872415232 Yes                 0
Large Pool Size                    33554432 Yes                 0
Java Pool Size                            0 Yes                 0
Streams Pool Size                         0 Yes                 0
Shared IO Pool Size               134217728 Yes                 0
Data Transfer Cache Size                  0 Yes                 0
Granule Size                       16777216 No                  0

NAME                                  BYTES RESIZEABLE     CON_ID
-------------------------------- ---------- ---------- ----------
Maximum SGA Size                 4915722040 No                  0
Startup overhead in Shared Pool   405891224 No                  0
Free SGA Memory Available                 0                     0

14 rows selected.
```
其中，**Buffer Cache Size**和**Shared Pool Size**是需要重点关注的内容。

# 修改Oracle内存分配
如果我们升级了服务器物理内存配置，就需要对Oracle的内存参数进行修改。

在自动共享内存管理模式下，一般按照如下原则配置内存：

- `SGA_TARGET`一般配置为物理内存的30%到70%之间；
- `PGA_AGGREGATE_TARGET`一般配置为物理内存的5%到25%之间；
- `SGA_TARGET`和`PGA_AGGREGATE_TARGET`之和不要超过物理内存的80%；
- **Buffer Cache Size**一般配置为`SGA_TARGET`的 **20%** 左右；
- **Shared Pool Size**一般配置为`SGA_TARGET`的 **10%** 左右。

修改数据库内存配置：
```sql
--备份参数文件
create pfile='/home/oracle/pfile.ora' from spfile;

--禁用自动内存管理
alter system set memory_target=0M scope=spfile;

--设置SGA_TARGET
alter system set sga_max_size=9G scope=spfile;
alter system set sga_target=9G scope=spfile;

--设置Buffer cache、共享池、Java池的最小值
alter system set db_cache_size=2G scope=spfile;
alter system set shared_pool_size=1G scope=spfile;
alter system set java_pool_size=128m scope=spfile;

--设置PGA_AGGREGATE_TARGET
alter system set pga_aggregate_target=1G scope=spfile;
```

然后重启数据库即可生效。
```sql
SQL> show parameter sga_target

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
sga_target                           big integer 9G

SQL> show parameter pga

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
pga_aggregate_limit                  big integer 3000M
pga_aggregate_target                 big integer 1G

SQL> show parameter db_cache_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_cache_size                        big integer 2G

SQL> show parameter shared_pool_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
shared_pool_size                     big integer 1G

SQL> show parameter java_pool_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
java_pool_size                       big integer 128M
```

需要注意的是，服务器物理内存变化通常还涉及内核参数`kernel.shmall`和`kernel.shmmax`的调优。如果数据库使用了大页，还需要调优操作系统的大页配置。
```bash
# 查看是否开启大页
SQL> show parameter use_large_pages

# 查看操作系统大页配置
cat /proc/meminfo | grep HugePage
cat /proc/meminfo | grep Hugepagesize
```




