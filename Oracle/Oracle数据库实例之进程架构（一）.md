@[TOC](Oracle数据库实例之进程架构（一）)

# 进程的类型
数据库实例包含或与多个进程交互。这些进程可以分为：

- **客户端进程**（client process）运行应用或者Oracle工具代码；
- **Oracle进程**是运行Oracle数据库代码的执行单元。在多线程架构中，一个Oracle进程可以是一个操作系统级别的进程，也可以是操作系统进程内的一个线程。Oracle进程可以分为以下三类：
  * **后台进程**（background process）：随数据库实例启动，负责实例恢复、清理进程、将重做日志缓冲器刷盘等维护任务；
  * **服务器进程**（server process）：应客户端的请求来工作。比如，对SQL查询进行语法分析、然后将其放入共享池，又如创建和执行查询计划，以及从数据库buffer cache或者磁盘读入缓冲器。
  * **Slave进程**：负责后台进程或者服务器进程的附加任务。

Oracle数据库的进程架构取决于操作系统和数据库配置选择。比如，为用户配置专用服务器连接或者共享服务器连接。在使用共享服务器连接时，每个服务器进程都能为多个客户端进程提供服务。

下图展示了使用专用服务器连接时的SGA和后台进程。对于每个用户连接来说，都有一个客户端进程运行应用。每个客户端进程都对应着一个服务器进程，其中包含了它的PGA。

>**图1 Oracle进程和SGA**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/7cbb3abd679f9b77be0e865df7b8d503.gif#pic_center)

视图 `V$PROCESS` 包含了连接到数据库实例的每个Oracle进程的一行信息，比如系统进程ID和线程ID。

```sql
SQL> col SPID format a8
SQL> col STID format a8
SQL> select SPID, SID, PROGRAM from v$PROCESS order by SPID;
```

# 客户端进程
当有用户运行 `Pro*C` 或者 `SQL*Plus` 程序时，操作系统会创建一个客户端进程（也称为用户进程）来运行用户应用。客户端应用会链接到Oracle数据库的程序库（libraries），从而能够使用其中的APIs来与数据库通信。

## 客户端和服务器进程
客户端进程不能直接读写SGA，但是为客户端进程提供服务的Oracle进程可以。客户端进程可以在数据库主机以外的其他主机上运行，但是Oracle进程只能在数据库所在的主机上运行。

假设有一个用户在客户端主机通过SQL Plus连接到另一台主机上的数据库sample（数据库实例未启动）：

```sql
SQL> connect sys@inst1 as sysdba
```

在客户端主机上，只能看到sqlplus的客户端进程：

```sql
% ps -ef | grep -e sample -e sqlplus | grep -v grep
clientuser  29437 29436  0  15:40  pts/1  00:00:00  sqlplus
```

在数据库主机上，只能看到一个非本地连接的服务器进程：

```sql
% ps -ef | grep -e sample -e sqlplus | grep -v grep
serveruser  29441 1  0  15:40  ?  00:00:00  oraclesample (LOCAL=NO)
```

## 连接和会话
数据库连接（**connection**）是一个客户端进程与一个数据库实例之间的物理通信路径。连接的通信路径是由可用的进程间通信机制或者网络软件创建的。连接一般发生在客户端进程与服务器进程或者调度器（**dispatcher**）之间，也可以发生在客户端进程与Oracle连接管理器（Oracle Connection Manager, **CMAN**）之间。

数据库会话（**session**）是数据库实例内存中的一个逻辑实体，代表了一个当前用户登录到数据库的状态。例如，当用户使用密码通过数据库身份认证时，一个会话就会被创建给该用户。会话一直从用户通过数据库认证持续到用户断开连接或者退出数据库应用。

单个连接可以拥有零个、1个或多个会话。单个连接的多个会话之间相互独立，一个会话中提交的事务不会影响其他会话中的事务。

单个用户可以同时拥有与一个数据库的多个连接。在专用服务器连接中，数据库为每个连接都创建一个服务器进程。只有导致专用服务器被创建的客户端进程才能使用该服务器进程。而在共享服务器连接中，多个客户端进程访问同一个共享服务器进程。

>**图2 单个连接对应单个会话**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/e98516c8c8d9cd6634c6792b72a68ba8.gif#pic_center)


>**图3 单个连接对应多个会话**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/c292f9e9d3df89c871f25166f68bc5b5.gif#pic_center)

下面的例子中，用户通过SQL Plus连接到数据库之后开启了跟踪，因此创建了一个新的会话：

```sql
SQL> SELECT SID, SERIAL#, PADDR FROM V$SESSION WHERE USERNAME = USER;
SID SERIAL# PADDR
--- ------- --------
 90      91 3BE2E41C

SQL> SET AUTOTRACE ON STATISTICS;
SQL> SELECT SID, SERIAL#, PADDR FROM V$SESSION WHERE USERNAME = USER;
 
SID SERIAL# PADDR
--- ------- --------
 88      93 3BE2E41C
 90      91 3BE2E41C
...
SQL> DISCONNECT  # 不要用EXIT命令
```

**DISCONNECT**命令实际上==只是关闭了会话，并没有断开连接==。打开一个新的终端并使用另一个用户身份连接到实例，执行下面的查询可以发现地址为 `3BE2E41C` 的连接仍然处于活跃状态。

```sql
SQL> CONNECT dba1@inst1
Password: ********
Connected.
SQL> SELECT PROGRAM FROM V$PROCESS WHERE ADDR = HEXTORAW('3BE2E41C');

PROGRAM
------------------------------------------------
oracle@stbcs09-1 (TNS V1-V3)
```

##  数据库操作
从数据库监控的背景来说，数据库操作（operation）是用户或者应用代码定义的两个时间点之间的会话活动。简单的数据库操作可以是单个SQL语句，也可以是单个PL/SQL存储过程或函数。复合的数据库操作则是多个简单操作的集合。

可以通过 `DBMS_SQL_MONITOR` PL/SQL包来创建和管理数据库操作。可以利用 `V$SQL_MONITOR`、`V$SQL_PLAN_MONITOR` 和 `V$SQL_MONITOR_SESSTAT`视图来监控数据库操作。

# 服务器进程
Oracle数据库创建服务器进程来处理连接到实例的客户端进程的请求。客户端进程始终通过单独的服务器进程来与数据库通信。为数据库应用创建的服务器进程可以进行以下的任务：

- 语法分析并运行应用发来的SQL语句，包括创建和执行查询计划；
- 执行PL/SQL代码；
- 将数据块从数据文件读入数据库buffer cache（后台进程DBW负责将修改的数据块写回磁盘）；
- 以应用能够处理的形式返回结果。

## 专用服务器进程
在专用（dedicated）服务器连接中，一个客户端连接只与一个且仅仅一个服务器进程相关。每个客户端进程直接与自己的服务器进程通信。每个服务器进程在会话持续时间内都专门为自己的客户端进程提供服务。服务器进程中存储了特定进程相关的信息、以及它的PGA中的UGA。

## 共享服务器进程
在共享（shared）服务器连接中，客户端应用通过网络连接到一个进程**调度器**（dispatcher），而不是直接连接到一个服务器进程。

调度器进程接收来自已连接客户端的请求，并将它们放到大型池（large pool）中的请求队列（request queue）中。第一个可用的共享服务器进程会响应队列中的请求并处理它。然后，共享服务器将处理的结果放到调度器的响应队列（response queue）中。调度器进程会监控响应队列，并将结果传输给客户端。

与专用服务器进程类似，共享服务器进程也有自己的PGA。不同的是，会话的UGA存在于SGA中，从而使得任何共享的服务器都能访问会话数据。

## Oracle如何创建服务器进程
Oracle创建服务器进程的方式取决于连接的方法。连接的方法包括：

- **Bequeath**（字面意思为遗赠）：SQL Plus（Oracle调用接口客户端）或者类似的客户端应用程序直接创建服务器进程；
- **Oracle Net listener**（网络监听器）：通过监听器连接到数据库。
- **Dedicated broker**（字面意思为专职经纪人）：是一个创建前台进程（foreground process）的数据库进程。与监听器不同，broker在数据库实例内部。使用broker时，客户端连接到监听器，然后监听器会把连接交给专用的broker。

当连接**不是**使用的bequeath方法时，数据库按下面的步骤创建服务器进程：

1. 客户端应用向监听器或broker请求建立一个新的连接；
2. 监听器或broker发起新进程或线程的创建；
3. 操作系统完成新进程或线程的创建；
4. Oracle数据库初始化各种组件和通知；
5. 数据库交付连接以及连接相关的代码。

如果使用的是dedicated broker方法，可以利用 `DBMS_PROCESS`包预先创建一个服务器进程池，用于之后关联某个用户请求。进程管理器（PMAN）后台进程会监控该进程池。当有连接需要服务器进程时，数据库会跳过上面的步骤2、3和4，直接进行步骤5，从而提升性能。

# 后台进程
>关于后台进程，请参考：Oracle数据库实例之进程架构（二）


**References:**
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/process-architecture.html#GUID-13FE4098-61DF-4D76-882D-551A88E0EBB8

