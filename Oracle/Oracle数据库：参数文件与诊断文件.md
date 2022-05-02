@[TOC](Oracle数据库：参数文件与诊断文件)

# 参数文件
Oracle中的参数文件是一个包含一系列参数以及参数对应值的操作系统文件。它们是在数据库实例启动时候加载的，决定了数据库的物理 结构、内存、数据库的限制及系统大量的默认值、数据库的各种物理属性、指定数据库控制文件名和路径等信息，是进行数据库设计和性能调优的重要文件。

Oracle数据库实例启动时必须通过参数文件（parameter file）读取一系列配置参数。在手动创建数据库时，也必须先利用参数文件启动一个实例，然后使用 `CREATE DATBASE` 语句创建数据库。因此，即使数据库不存在，也可以有数据库实例和参数文件。

## 初始化参数
初始化参数（initialization parameters）是数据库实例启动时从**参数文件**中读取的配置参数。Oracle数据库提供了许多用于在不同环境中优化运行的初始化参数，其中绝大多数都可以采用默认值，仅有少数初始化参数必须显式地设定。

初始化参数基本可以分为以下几组：
1. 用于给文件或目录等实体命名的参数；
2. 用于给某个进程、数据库资源、或者数据库本身设定限制的参数；
3. 用于设定 SGA、PGA 等内存区域大小的参数。这种初始化参数会影响到数据库的性能。

****
Oracle参数文件可以分为文本初始化参数文件（text initialization parameter files, **pfile**）和 服务器参数文件（server parameter file, **spfile**）。

## pfile
Oracle 9i 之前，Oracle 一直采用 pfile 存储初始化参数。pfile 默认的名称为 `init+sid名.ora`。文件路径为 `/data/app/oracle/product/12.1.0/dbhome_1/dbs`。

pfile 有以下特点：

* 启动或关闭数据库时，pfile 必须与连接到数据库的客户端应用位于同一台主机上；
* pfile 是一个**文本文件**，可以用任何文本编辑工具打开；
* Oracle数据库可以读取 pfile，但不能写 pfile；必须通过**文本编辑器**来修改 pfile 中的初始化参数；
* 执行 `ALTER SYSTEM` 对初始化参数所做的修改仅在**当前实例**生效。你必须手动更新 pfile 中的初始化参数并**重启实例**来使更新生效。


## spfile
从 Oracle 9i 开始，Oracle 引入了 spfile 文件。spfile 默认的名称为 `spfile+sid名.ora`，文件路径为 `/data/app/oracle/product/12.1.0/dbhome_1/dbs`。

spfile 有以下特点：

* 只有Oracle数据库能读写 spfile；
* 每个Oracle数据库**仅有一个** spfile，位于数据库主机上；
* spfile 以**二进制**文本形式存在，不能用文本编辑器对其中参数进行修改，只能通过SQL命令在线修改；
* spfile 中存储的初始化参数是持久化的（**persistent**）。当数据库实例运行时，对 spile 中的初始化参数做的任意修改在经历实例启停后仍然会生效。

spfile 能够免去为客户端应用同时维护多个文本初始化参数文件的麻烦。spfile 可以通过执行 `CREATE SPFILE` 利用 pfile 来创建，也可以直接通过数据库配置助手创建。

spfile 中的参数有三种 scope：

- `scope=spfile`：对参数的修改记录在服务器初始化**参数文件**中，修改后的参数在**重启**数据库后生效。适用于动态和静态初始化参数；
- `scope=memory`：对参数的修改记录在**内存**中，对于动态初始化参数的修改立即生效。在重启数据库后会**丢失**，会复原为修改前的参数值；
- `scope=both`：对参数的修改会同时记录在服务器参数文件和内存中，对于动态参数会立即生效。静态参数不能用这个选项。这是使用 spfile 参数文件时的默认 scope 值。

## 相关运维操作
执行下面的来查看其目录位置。

```bash
SQL> select NAME, VALUE, DISPLAY_VALUE from v$PARAMETER where NAME='spfile';

SQL> show parameter pfile
NAME	TYPE	VALUE
spfile	string	/data/app/oracle/product/12.1.0/dbhome_1/dbs/spfileorcl.ora

SQL> show parameter spfile
NAME	TYPE	VALUE
spfile	string	/data/app/oracle/product/12.1.0/dbhome_1/dbs/spfileorcl.ora
```

执行下面的命令来查看Oracle启动时使用的是 spfile 还是 pfile。
```bash
SQL> select decode(count(*),1,'spfile','pfile') from v$spparameter where rownum=1 and
isspecified='TRUE';

DECODE
------
spfile
```

也可以通过执行 `show parameter pfile` 和 `show parameter spfile` 来判断。如果Oracle是从 spfile 启动的，两条命令的输出中都会有 spfile 的文件路径；反之，如果是从 pfile 启动的，两条命令的输出中都**不会**有 pfile 的文件路径。

pfile 和 spfile 可以互相作为来源创建对方。
```bash
SQL> create pfile from spfile;
File created.
```

使用 pfile 和 spfile 来启动数据库时，spfile 优于 pfile。查找文件的顺序是 `spfileSID.ora > spfile.ora > init.ora`。

```bash
SQL> startup pfile='/data/app/oracle/product/12.1.0/dbhome_1/dbs/initorcl.ora';
ORACLE instance started.
Total System Global Area ... bytes
Database Buffers ... bytes
Redo Buffers ... bytes
Databse mounted.
Database opened.
```

# 诊断文件
Oracle数据库提供了预防、监测、诊断和解决数据库问题的一套故障诊断基础设施。覆盖的问题包括代码bug、元数据损坏、用户数据损坏等。多租户容器数据库（Multitenant container databases, **CDBs**）与非CDB数据库存在架构上的差异。除非特别说明，本节内容仅限于**非CDB数据**。

## ADR
自动诊断仓库（Automatic Diagnostic Repository, ADR）是一个文件仓库，存储有用于数据库诊断的文件，比如跟踪文件（trace files）、告警日志（alert log）、DDL日志、以及健康检查报告。ADR位于数据库外部，因此当数据库不可用时，数据库管理员仍然可以访问和管理ADR。数据库实例可以在创建数据库之前就先创建ADR。

**Problem vs Incident**
ADR会主动跟踪数据库中的关键错误，我们称之为问题（**problem**）。关键错误以内部错误形式显现，比如 `ORA-600`、或者其他严重错误。每个问题都有一个键（problem key），即描述问题的一个文本字符串。

当一个问题重复发生多次时，ADR会为每一次问题发生创建一个带时间戳的事件（**incident**）。每个事件通过一个唯一的**事件ID**（incident ID）来标识。事件发生时，ADR会向 Entreprise Manager 发送事件警告（incident alert）。

由于一个问题可以在短时间内产生大量事件，在事件的数量超过某个阈值后，ADR会通过告警量控制（Flood control）来限制新的事件产生。受到告警量控制的事件会生成一条告警日志记录（alert log entry），但是不会生成事件dump文件。

**ADR目录结构**
**ADR Base** 是ADR的根目录。`ADR Base` 中可以有多个`ADR Home` 目录。其中，每个 **ADR Home** 目录都是一个Oracle数据库产品或组件实例的所有诊断数据集合（跟踪文件、告警日志、dump文件等）的根目录。例如，在 Oracle RAC 集群中，每个数据库实例、每个ASM实例都有自己的 ADR Home 目录。

图1展示了一个数据库实例的ADR目录结构。

>图1 ADR目录结构
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/74977813cc26350a91fbd5d834539c92.gif#pic_center)

```sql
SQL> col name format a21
SQL> col name format a60
SQL> select name,value from V$DIAG_INFO;  --定位诊断文件

NAME              VALUE
----------------  ------------------------
Diag Enabled      TRUE
ADR Base           /d1/28937447/oracle/log
ADR Home           /d1/28937447/oracle/log/diag/rdbms/dbn/osi
Diag Trace         /d1/28937447/oracle/log/diag/rdbms/dbn/osi/trace
Diag Alert         /d1/28937447/oracle/log/diag/rdbms/dbn/osi/alert
Diag Incident      /d1/28937447/oracle/log/diag/rdbms/dbn/osi/incident
Diag Cdump         /d1/28937447/oracle/log/diag/rdbms/dbn/osi/cdump
......
```


## Alert log
每个数据库都有一个**告警日志**（alert log）。告警日志是一个按时间顺序记录数据库消息和错误的XML文件。其中包含以下信息：

- 所有**内部错误**（`ORA-600`）、**块数据损坏错误**（`ORA-1578`）、以及**死锁错误**（`ORA-60`）；
- 数据库**管理操作**，比如SQL*Plus命令 `STARTUP`、`SHUTDOWN`、`ARCHIVE LOG`、`RECOVER`；
- 与共享服务器或调度（dispatcher）进程有关函数的消息与错误；
- 实体化视图的自动刷新过程中出现的错误。

当你首次启动数据库实例时，Oracle 数据库会在图1中的alert子目录下创建一个告警日志。同时，trace子目录下会有一个纯文本的告警日志。

## DDL log
DDL日志和告警日志格式和记录行为相同，但只包含 **DDL命令**及相关信息。每条DDL语句对应一条DDL日志记录。DDL日志位于 ADR Home目录下的 `log/ddl` 子目录下。

## Trace files
**跟踪文件**（trace file）包含了用于调查问题的诊断数据。跟踪文件还能用来指导应用和实例调优（tuning）。

**跟踪文件的分类**
每个服务器和后台进程都能周期性地写入关联的跟踪文件。写入的信息包括进程环境、状态、活动、错误信息等。**SQL Trace**工具也能创建跟踪文件，用于提供单独的SQL语句的性能信息。你也可以跟踪客户标识（client identifier）、服务、模块、动作（action）、会话（session）、实例、数据库。比如，通过执行 `DBMS_MONITOR` 程序包中的存储过程可以跟踪用户会话。

**跟踪文件的位置**
ADR将跟踪文件放在 trace子目录下。跟踪文件命名与平台有关，使用 `.trc` 作为文件名后缀。

通常，**数据库后台进程**的跟跟踪文件名称包含有 **Oracle SID**、后台进程名称、以及操作系统进程号。例如，RECO进程的跟踪文件为 `myorcl_reco_10355.trc`。

**服务器进程**的跟踪文件名称包含有 **Oracle SID**、**ora字符串**、以及操作系统进程号。例如 `myorcl_ora_10304.trc`。

有时，跟踪文件有对应的跟踪元数据文件，以 `.trm` 后缀结尾。这些元数据文件存有数据库用来进行搜索和导航的结构化信息，称为跟踪映射（trace maps）。

**跟踪文件分段**
当跟踪文件的大小受限时（通过 `MAX_DUMP_FILE_SIZE` 限制），数据库有可能自动将其分成最多五个**段**（segments）。这里的段是多个单独的文件，其名称是跟踪文件的命名加上一个段编号，例如 `ora_1234_2.trc`。每个段文件的大小一般是 `MAX_DUMP_FILE_SIZE` 的五分之一。如果五个段文件的文件总容量超过了限制，数据库会删除创建时间最早的段文件（但第一个段文件永远不会被删除，因为里面可能包含进程初始状态的相关信息），然后再创建一个新的空文件。

## 诊断转储文件
诊断转储文件（diagnostic dump file）是一种特殊的跟踪文件，其中存有关于跟踪对象的**状态或结构**的**时间点**（point-in-time）信息。跟踪文件通常是诊断数据的连续输出，而诊断转储文件对应于一次事件的诊断数据的一次性输出。

**跟踪dump和事件**
大多数转储文件是由于事件而生成的。当有事件发生时，数据库会向事件的ADR目录下写入一个或多个dump文件。因事件产生的dump文件的名称中也会有对应的事件ID。

发生事件时，应用可能生成一个堆（heap）转储或者系统状态转储文件（system state dump）。这种情况下，数据库会把转储文件名加到事件文件名称中，而不是默认的跟踪文件名。例如，数据库为某个事件创建的默认跟踪文件名为 `prod_ora_90384.trc`。而发生转储时，生成的文件名为 `prod_ora_90384_incidentID.trc`，其中 incidentID 是事件ID；堆转储文件名为 `prod_ora_90384_incidentID_dumpID.trc`，其中 dumpID 是跟踪转储文件ID。

References
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/oracle-database-instance.html#GUID-1CE79EC6-6B04-4BB2-BBF1-77A2D47A1CE6
[2\] https://www.cnblogs.com/xqzt/p/4832597.html
[3\] https://www.cnblogs.com/kerrycode/p/3254154.html
