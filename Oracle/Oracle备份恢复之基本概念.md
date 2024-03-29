---
tags: [OCP, oracle]
title: Oracle备份恢复之基本概念
created: '2023-06-11T01:56:35.008Z'
modified: '2023-06-11T03:02:30.299Z'
---

Oracle备份恢复之基本概念

# 备份
## 全量备份与增量备份
- **全量备份**：一个或多个数据文件的一个完整副本。
- **增量备份**：包含从最近一次备份以来被修改或者添加的数据块。又可以继续划分为0级、1级、2级、3级、...、n级备份。其中，0级备份是后续所有增量备份的基础，是一个完整备份。

>:snake:**注**：虽然全量备份与0级备份内容一样，但是全量备份**不**能作为增量备份中的0级备份使用。

## 差异增量和累计增量
增量备份可以分为如下两种：
- **差异增量备份**：备份上一级备份（比如0级是1级的上一级、1级是2级的上一级）、以及同一级备份以来所有变化的数据库。差异增量备份是默认的增量备份方式。
- **累计增量备份**：备份上一级备份以来所有变化的数据块。

## 脱机备份与联机备份
- **脱机备份**：在数据库关闭期间进行的备份，即**冷备份**。在一致性关闭数据库后，控制文件中记录的SCN与数据文件头部SCN一致，因此又称为**一致性备份**。
- **联机备份**：在数据库打开期间进行的备份，即**热备份**。联机备份只能在归档模式下进行。联机备份的数据文件不会与特定的SCN以及控制文件保持同步，因此又称为**不一致性备份**。

## 镜像备份与备份集
- 镜像备份（**image copy**）：完整拷贝数据文件，不做任何压缩处理，不支持增量备份，也不能备份到磁带。
- 备份集（**backupset**）：RMAN备份时生成的一个或多个由叫做piece的物理文件组成的逻辑结构。Backup piece中可以包含数据文件、控制文件、以及归档日志。备份集支持备份压缩、增量备份。既可以备份到磁盘，也支持备份到磁带。


# 还原 vs 恢复
数据库备份恢复分为**Restore**和**Recovery**两个阶段：
- Restore（还原）：从备份文件中检索需要的内容，并拷贝到原来对应的位置；
- Recovery（恢复）：在Restore的基础上，通过归档日志和联机日志将数据库恢复到最新的SCN。

## 恢复的类型
数据库恢复（Recovery）又可以分为以下三种类型：
- **实例恢复**：通过联机日志来前滚已提交的事务，并回滚未提交的事务。当数据库异常断电重启后、或者通过**SHUTDOWN ABORT**命令关闭数据库后，当DBA试图打开**不一致性关闭**的数据库时，实例恢复会自动发生。
- **完全恢复**：在Restore的基础上，使用归档日志和联机日志将数据库恢复到**最新的时间点**，使数据库保持一致性。
- **不完全恢复**：在Restore的基础上，使用归档日志和联机日志将数据库恢复到**过去的**某个时间点或SCN。

不完全恢复可以通过以下四种方式实现：
- 基于时间点的不完全恢复；
- 基于取消（CANCEL命令）的不完全恢复；
- 基于日志序列的不完全恢复；
- 基于闪回特性的不完全恢复。


