@[TOC](Oracle数据库性能分析之常用视图)

# `v$session` & `gv$session`

`v$session` 和 `gv$session` 只在RAC中区别明显：有g是全局的（global），包含RAC的两个实例中的内容；`v$session` 只包含当前本节点实例的数据。从内容来说，`gv$session` 只比 `v$session` 多一个 INST_ID 字段而已。

`v$session` 动态性能视图中记录了每一个连接到数据库实例的会话的信息，包括用户session、以及后台进程、应用进程等等。

下表展示了该视图中一些重要的列。

| 列 | 描述 |
|--|--|
| SADDR | session地址 |
| SID | session标识符 |
| SERIAL# | session序列号，用于唯一地标识一个session所属的对象。<br>保证了session级别的命令不会在session结束后应用到另一个具有相同session ID的会话 |
| PADDR | session所属进程的地址 |
| USER# | Oracle用户标识符 |
| USERNAME | Oracle用户名 |
| COMMAND | 正在运行的命令。该列是一个数字n，执行<br> `select command_name from v$sqlcommand where command_type = n` 可以查询命令名称 |
| TADDR | 事务状态对象的地址 |
| LOCKWAIT | 该session正在等待的锁的地址。NULL表示没有 |
| STATUS | session状态：ACTIVE表示正在执行SQL；INACTIVE表示非活跃session；KILLED表示session被标记为killed；<br>CACHED表示session暂时被高速缓存；SNIPED表示非活跃session超过了某些配置限制，比如资源限制、空闲时间限制等，这样的session将不被允许恢复为活跃状态 |
| SERVER | 服务器类型，包括Dedicated、Shared、Pseudo、Pooled、None |
| SCHEMA# | Schema用户标识符 |
| SCHEMANAME | Schema用户名 |
| OSUSER | 操作系统客户端用户名 |
| PROCESS | 操作系统客户端进程PID |
| PORT | 客户端口号 |
| PROGRAM | 操作系统程序名 |
| TYPE | session类型 |
| SQL_ADDRESS | 与SQL_HASH_VALUE一起可用于确定当前正在执行的SQL语句 |
| SQL_HASH_VALUE | 与SQL_ADDRESS一起可用于确定当前正在执行的SQL语句 |
| SQL_ID | 当前正在执行的SQL语句的标识符 |
| SQL_CHILD_NUMBER | 当前正在执行的SQL语句的子编号 |
| SQL_EXEC_START | 该session当前执行的SQL语句开始执行的时间 |
| PLSQL_ENTRY_OBJECT_ID | 栈最顶端的PL/SQL子程序的对象ID |
| PLSQL_OBJECT_ID | 当前执行的PL/SQL子程序的对象ID |
| ROW_WAIT_OBJ# | 包含了ROW_WAIT_ROW# 指定行的表的对象ID |
| ROW_WAIT_FILE# | 包含了ROW_WAIT_ROW# 指定行的数据文件的标识符 |
| ROW_WAIT_BLOCK# | 包含了ROW_WAIT_ROW# 指定行的数据块的标识符 |
| ROW_WAIT_ROW# | 当前被锁的行，该列只有在对应session正在等待另一个事务提交、且ROW_WAIT_OBJ#的值不等于-1时才有效 |
| LOGON_TIME | 已登录的时间 |
| LAST_CALL_ET | 该session处于活跃/非活跃状态的时间（elapsed time），单位是秒 |
| CLIENT_IDENTIFIER | 该session的用户标识符 |
| BLOCKING_SESSION_STATUS | session阻塞状态：VALID表示session被阻塞；NO HOLDER表示该session未被其他session阻塞；<br>NOT IN WAIT表示session未处于等待状态；UNKNOWN表示未知 |
| BLOCKING_INSTANCE | 造成阻塞的session的实例标识符 |
| BLOCKING_SESSION | 造成阻塞的session的标识符 |
|  SEQ# | 唯一标识当前或者上一个等待事件的序号 |
| EVENT# | 该session正在等待的资源或事件的序号；如果该session未处于等待状态，则表示最近一次等待的资源或事件的序号 |
| EVENT | 该session正在等待的资源或事件；如果该session未处于等待状态，则表示最近一次等待的资源或事件 |
| P1 | 第一个等待时间参数，十进制表示 |
| P1TEXT | 第一个等待事件参数的描述 |
| WAIT_CLASS_ID | 等待事件的类标识符 |
| WAIT_TIME | 如果session正在等待，该值为0；如果session未在等待，且该值为正数，则表示上一次等待的持续时间，单位为百分之一秒 |
| SECONDS_IN_WAIT | 如果session正在等待，表示已经等待的时间；如果session未在等待，则表示上一次等待开始以来经过的时间 |
|  STATE | 等待状态：WAITING表示正在等待；WAITED UNKNOWN TIME表示上一次等待的时间未知；WAITED SHORT TIME表示上一次等待不到百分之一秒；WAITED KNOWN TIME表示上一次等待的时间为 WAIT_TIME |
| SERVICE_NAME | 该session的服务名 |
| SQL_TRACE | SQL Trace是否开启 |


>*For more: https://docs.oracle.com/en/database/oracle/oracle-database/12.2/refrn/V-SESSION.html#GUID-28E2DC75-E157-4C0A-94AB-117C205789B9*

# `v$active_session_history`
动态性能视图 `v$active_session_history` 和 `gv$active_session_history` 会**每秒**钟将数据库所有节点的Active Session采样一次。同样地，以g开头的视图中包含的是RAC集群中所有实例的信息。

下表展示了该视图中一些重要的列，其中大多数列在 `v$session` 视图中都出现过。

| 列 | 描述 |
|--|--|
| SAMPLE_ID | 取样ID |
| SAMPLE_TIME | 取样发生的时间 |
| IS_AWR_SAMPLE | 样本是否已经或将要被刷入到自动负载仓库（AWR）中 |
| SESSION_ID | session标识符，映射到 v$session.SID |
| SESSION_SERIAL# | session序列号，映射到 v$session.SERIAL# |
| SESSION_TYPE | session类型，分为 FOREGROUND 和 BACKGROUND |
| SQL_ID | 在取样时，session正在执行的SQL语句的标识符 |
| SQL_PLAN_HASH_VALUE | 游标的SQL plan的数字表示。v$session中没有该列数据 |
| EVENT | 如果 SESSION_STATE=WAITING，表示在取样时session正在等待的事件；<br>如果 SESSION_STATE=ON CPU，该列值为NULL |
| SEQ# | 唯一标识等待事件的序列号 |
| XID | 取样时，session正在处理的事务ID。v$session中没有该列数据 |
| IN_SQL_EXECUTION | 表示在取样时，session是否在执行SQL语句 |
| ... | ...... |


>*For more: https://docs.oracle.com/en/database/oracle/oracle-database/12.2/refrn/V-ACTIVE_SESSION_HISTORY.html#GUID-69CEA3A1-6C5E-43D6-982C-F353CD4B984C*

# `dba_hist_active_sess_history`
静态数据字典视图 `dba_hist_active_sess_history` 会将 `v$active_session_history` 里的数据**每10秒**拍一次快照（snapshot）并**持久化**保存。

下表展示了该视图中一些重要的列，其中许多列在 `v$active_session_history` 视图中都出现过。

| 列 | 描述 |
|--|--|
| SNAP_ID | 唯一快照ID |
| DBID| 快照的数据库ID |
| INSTANCE_NUMBER | 快照的实例编号 |
| SMAPLE_ID | 取样ID |
| SAMPLE_TIME | 取样的时间 |
| SAMPLE_TIME_UTC | 取样的世界标准时间 |
| USECS_PER_ROW | 上一次活跃session历史取样以来经过的时间，单位是微秒 |
| SESSION_ID | session标识符 |
| USER_ID | Oracle用户标识符 |
| SQL_ID | 当前正在执行的SQL语句的标识符 |
| EVENT | 等待事件，映射到 v$active_session_history.EVENT |
| SEQ# | 等待事件的唯一标识序列号 |
| XID | 取样时，session正在处理的事务ID |
| IN_SQL_EXECUTION | 表示在取样时，session是否在执行SQL语句 |
| ... | ...... |


>*For more: https://docs.oracle.com/en/database/oracle/oracle-database/12.2/refrn/DBA_HIST_ACTIVE_SESS_HISTORY.html#GUID-335EC838-FEA0-4872-9E14-67C5A1908B35*
