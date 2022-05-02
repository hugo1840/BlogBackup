@[TOC](OceanBase数据库的日志类型)

OceanBase的日志可以分为**事务/存储日志**和**observer日志**。

# 事务/存储日志
事务/存储日志存放在`/data/log1/集群名/`路径下，又可以分为以下三种类型：

1. **clog**：所有分区共用。日志可能是乱序的，记录了事务、*PartitionService*提供的原始日志内容。此目录下的日志**基于paxos协议在多个副本之间同步**；
2. **ilog**：即 *index log*，所有分区共用。单个分区内部日志是有序的，记录了分区内部`log_id > clog(file_id, offset)`的索引信息。**每个副本自行记录**；
3. **slog**：即 *storage log*，指*SSTable*操作日志信息。

其中，*clog* 和 *ilog* 合起来被称为**事务日志**，*slog* 被称为**存储日志**。

## clog
事务提交日志clog包含了以下内容：

- *redo log*：记录了事务的具体操作；
- *prepare log*：记录了事物的准备状态；
- *commit log*：表示事务成功提交，记录了commit信息，比如事务的全局版本号；
- *clear log*：通知事务清理上下文；
- *abort log*：表示事务被回滚。

# observer日志
Observer日志记录了OceanBase的启动和运行状态、告警信息，存放在`/home/admin/oceanbase/log/`路径下。

## 日志级别
按重要程度由大及小，Observer日志内容可以分为以下五种级别：

1. **Error**：严重错误，必须进行故障处理，否则系统不可用；
2. **Warn**：警告，可能会出现的潜在错误；
3. **Info**：提示，系统当前运行状态，为正常信息；
4. **Trace**：与Debug相比，更细致化的事件记录信息；
5. **Debug**：调试信息，用于调试时了解系统运行状态。

## 日志格式
Observer日志内容格式如下：
```bash
[time]log_level[module_name]function_name(filename:file_no)[thread_id]
[Ytrace_id0_trace_id1][log=last_log_print_time]log_data

#time 日志记录时间
#log_level 日志级别
#module_name 模块名
#filename:file_no 文件名:行号
#thread_id 线程id
```

## 日志切换与回收
在`/home/admin/oceanbase/log/`路径下，包含以下文件：

1. **选举模块日志**：文件名为`election.log`、`election.log.wf`；
2. **rootservice模块日志**：文件名为`rootservice.log`、`rootservice.log.wf`；
3. **observer日志**：文件名为`observer.log`、`observer.log.wf`，记录了除选举模块日志和rootservice模块日志以外的信息。

其中，以`.wf`结尾的日志文件单独记录了**Warn**和**Error**级别的日志，由日志文件控制参数`enable_syslog_wf`控制，默认为`True`。

当单个日志写满**256M**时，会发生日志文件切换。OceanBase会在原有的的日志文件名中加上`%Y%m%d%H%M%S`格式的时间（日志切换时间），并生成一个新的observer日志。

OceanBase默认不会自动清理日志，但可以按照日志个数来回收日志。日志回收由下面两个控制参数控制：

- `enable_syslog_recycle`：默认为False；
- `max_syslog_file_count`：默认为0。

## 错误码
MySQL租户下的错误码如果大于4000，则表示是OceanBase所特有的。如果错误码在4000以内，则是MySQL兼容的错误。

- 错误码在1到1000之间，表示是预留错误码；
- 错误码在1000到2000之间，表示是MySQL Server错误码；
- 错误码在2000到3000之间，表示是MySQL Client错误码。
