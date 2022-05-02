@[TOC](Oracle数据库实例之内存架构（一）)

本文讨论的内容涉及的数据库版本为 12.2。

# Oracle数据库内存结构
数据库实例启动后，Oracle数据库会为其分配一块内存区域并启动相关后台进程。该内存区域存储了以下信息：

- **程序代码**；
- 已连接**会话**（session）的信息，即使该session当前并未处于活跃状态；
- 程序执行过程中所需的信息，比如**查询**（query）的当前状态；
- 进程之间共享和通信的**锁**（lock data）相关的信息；
- **缓存数据**（cached data），比如数据块、重做记录等从磁盘中读取的信息。

## 基本内存结构
Oracle数据库中的基本内存结构包括：

- **系统全局区**（System global area, **SGA**）
  SGA是一组**共享内存**，其中包含了数据库实例的数据和控制信息。所有的服务器和后台进程都会共享SGA。SGA中的数据包括缓存数据块、共享SQL区等。
  
- **程序全局区**（Program global area, **PGA**）
  PGA是**非共享**的内存区域，其中包含了一个Oracle进程独占使用的数据和控制信息。当有一个Oracle进程启动时，Oracle数据库就会创建一个PGA。对于每个**服务器进程**（server process）和**后台进程**，都会存在一个对应的PGA。所有单独的PGA的集合构成了总的**实例PGA**（instance PGA）。数据库的初始化参数（在参数文件 spfile 中）决定了PGA实例的大小，而不是每个单独的PGA。
  
- **用户全局区**（User global area, UGA）
  UGA是与一个用户**会话**（user session）相关的内存。
  
- **软件代码区**（Software code areas）
  软件代码区是用于存储正在运行或者可以运行的代码。Oracle数据库代码存储在与用户程序不同的地方。

下图展示了以上内存结构之间的关系。

>**图1 Oracle数据库内存结构**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/11e237f09e21c09bb77ade059e14efac.gif#pic_center)


## Oracle内存管理
内存管理（memory management）涉及到在对数据库的需求发生变化时，维护Oracle实例内存结构的最佳大小。Oracle数据库基于与内存相关的初始化参数配置来管理内存。内存管理的基本可选项包括以下内容：

- 自动内存管理（automatic memory management）
  需要明确设置数据库实例内存的目标大小。数据库实例会自动调整到目标内存大小，并按需在SGA和实例PGA之间分配内存。
  
- 自动共享内存管理（automatic shared memory management）
  该模式为部分自动内存管理。你需要为SGA设定目标大小，然后为实例PGA设定总的目标大小，但是你也可以单独管理PGA。
  
- 手动内存管理（manual memory management）
  该模式下，你需要设定许多初始化参数来单独管理SGA和实例PGA。

如果你用数据库配置助手（Database Configuration Assistant, **DBCA**）来创建数据库，并选择了基本的安装选项，那么默认的内存管理方式为自动内存管理。

# 用户全局区：UGA
UGA是一块为会话变量（session variables）分配的内存，比如登录信息等其他数据库会话需要的信息。本质上，UGA存储了session的状态。

>**图2 UGA结构**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/3574de129f421316725fb5d19491e0b7.gif#pic_center)


如果一个会话向内存中载入了**PL/SQL包**（PL/SQL package），那么UGA中就包含了包的状态，即某个时刻该包里所有变量存储的的一组值。当该包的一个子程序修改了变量的值，包的状态也会改变。包的变量默认是唯一的，且在会话的生存时间内存在。

UGA中还存储了**OLAP页池**（OLAP page pool）。这个页池会管理OLAP数据页（数据块）。该页池在OLAP会话启动时被分配，在会话结束时被释放。OLAP会话会在用户查询维度对象（比如数据立方体 cube）时自动打开。

UGA必须在会话生存时间内可用。因此，使用**共享服务器**（shared server）连接时，UGA不能存储在PGA中。因为PGA只与单个进程相关。此时，UGA存储在SGA中，使得任意一个共享服务器进程都能访问它。相反地，当使用**专用服务器**（dedicated server）连接时，UGA就存储在PGA中。

# 程序全局区：PGA
PGA是针对系统中一个非共享的进程（process）或线程（thread）的内存，不会在SGA中分配。PGA是存储了一个**专用或共享服务器进程**所需的会话变量的堆内存（**memory heap**）。服务器进程分配它在PGA中所需的内存结构。

下图展示了一个实例PGA。你可以通过初始化参数来设置实例PGA的最大目标大小。组成实例PGA的所有单个的PGA会按需自动增长到该目标值。

>**图3 PGA实例结构**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/67e32531d4723dcb6e413cf24e74c69e.gif#pic_center)


## PGA的内容
PGA被划分为几个不同的区域。下图展示了一个专用服务器会话对应的PGA内容。

>**图4 PGA结构**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/0d9ab6eadbe053a926d1265bca555f69.gif#pic_center)

**第一部分：私有SQL区**
私有SQL区（private SQL area）存有经过语法分析的SQL语句（parsed SQL statement）的信息、以及其他针对会话的信息。当服务器进程执行SQL或PL/SQL代码时，该进程会使用私有SQL区来存储**绑定变量**（bind variable）的值、查询执行状态、以及查询执行工作区的信息。

切忌不要混淆私有SQL区（在PGA中）和**共享SQL区**（在SGA中，用于存储执行计划）。同一个或者不同的会话中的多个私有SQL区可以指向SGA中的一个执行计划。但是这些私有SQL区彼此之间并不共享，其中包含的值和数据也可能不一样。

**游标**（cursor）是一个具体的私有SQL区的名称或者**句柄**（handle）。可以把游标认为是客户端的一个指针和服务器端的一个状态。考虑到二者联系紧密，有时候游标和私有SQL区这两个名词会互换着使用。

>**图5 游标**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/8d795d9d1530bbcc15eb4b4e8f3e607b.gif#pic_center)

私有SQL区可以分为以下两个部分：

- **运行时区**（run-time area）
  运行时区包含了查询的执行状态信息。比如，运行时区会追踪在全表扫描（full table scan）中截至到当前已获取的行的数目。Oracle数据库在执行请求的第一步就会创建运行时区。对于DML操作而言，运行时区会在SQL语句关闭时释放。
  
- **持久区**（persistent area）
  持久区包含绑定变量的值。绑定变量的值会在执行SQL语句时传递给处于运行时的SQL语句。只有在游标关闭时，持久区才会被释放。
  
==客户端进程负责管理私有SQL区==。私有SQL区的分配和回收很大程度上取决于应用，尽管一个客户端进程能够分配的私有SQL区的数量会受到初始化参数 `OPEN_CURSORS` 的限制。通常来说，应用应该关闭所有不再使用的游标，以释放持久区来最小化应用用户所需的内存。

**第二部分：SQL工作区**
**工作区**（work area）是用于内存密集型操作而分配的一块私有PGA内存。例如，一个**排序**运算子使用排序区（sort area）来给行排序。一个**哈希连接**运算子（hash join operator）使用一块哈希区来创建哈希表。**位图合并**（bitmap merge）则使用位图合并区来合并多次位图索引扫描获取的数据。

下面展示了两个表 `employees` 和 `departments` 的连接操作及其对应的查询计划。运行时区负责跟踪全表扫描的过程。当前会话执行一个哈希连接操作来匹配两个表的行。`ORDER BY` 排序则发生在排序区。

```sql
SQL> SELECT * 
  2  FROM   employees e JOIN departments d 
  3  ON     e.department_id=d.department_id 
  4  ORDER BY last_name;
.
.
.
--------------------------------------------------------------------------------
| Id| Operation           | Name        | Rows  | Bytes | Cost (%CPU)| Time    |
--------------------------------------------------------------------------------
| 0 | SELECT STATEMENT    |             |   106 |  9328 |    7  (29)| 00:00:01 |
| 1 |  SORT ORDER BY      |             |   106 |  9328 |    7  (29)| 00:00:01 |
|*2 |   HASH JOIN         |             |   106 |  9328 |    6  (17)| 00:00:01 |
| 3 |    TABLE ACCESS FULL| DEPARTMENTS |    27 |   540 |    2   (0)| 00:00:01 |
| 4 |    TABLE ACCESS FULL| EMPLOYEES   |   107 |  7276 |    3   (0)| 00:00:01 |
--------------------------------------------------------------------------------
```

如果运算子处理的数据的量超过了工作区的容量，Oracle数据库就会把输入数据分割成更小的部分。这样数据库就能在内存中先处理一部分数据，而把剩余的部分放在临时的磁盘空间中留待之后处理。

当自动PGA内存管理模式开启时，数据库会自动地调整工作区的大小。当然你也可以手动控制调整工作区的大小。通常来说，更大的工作区能够显著提升运算子的性能，同时也会消耗更多的内存。

## 专用和共享服务器模式下的PGA
PGA的内存分配取决于数据库使用的是专用的还是共享的服务器连接。

| 内存区 | 专用服务器 | 共享服务器 |
| :--: | :--: | :--: |
| 会话内存是否共享 | 私有 | 共享 |
| 持久区的位置 | PGA | SGA |
| DML和DDL语句的运行时区的位置 | PGA | PGA |

# 系统全局区：SGA
SGA是一块可读写内存区域，与Oracle后台进程（background processes）一起构成了数据库实例。所有代表用户执行的服务器进程都能读取实例SGA里的信息。有一些进程能在数据库运行时写入SGA。需要注意的是，服务器和后台进程本身并不在SGA中，而是存在于独立的内存空间中。

每个数据库实例都有自己的SGA。Oracle数据库会在实例启动时自动为SGA分配内存，并在实例关闭时回收内存。正如图1，SGA由多个为了满足特定内存分配需求的内存池组成。除了重做日志缓存（redo log buffer）以外，所有其他的内存池都按连续内存单位分配和回收内存空间。这个连续内存单位称之为粒（granule），其大小与平台有关且取决于总的SGA大小。

查询 `V$SGASTAT` 视图可以看到SGA各部分的信息。其中包括数据库缓存、IM区、重做日志缓存、共享池、Large池、Java池、Streams池、固定SGA等。

>SGA更详细的内容请参考：Oracle数据库实例之内存架构（二）。


**References**
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/memory-architecture.html#GUID-0788EAEE-0E93-497B-9ACA-401EC0F7BCA1



