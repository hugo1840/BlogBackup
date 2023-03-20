---
tags: [oracle]
title: Oracle目录应急清理
created: '2023-03-20T12:38:51.604Z'
modified: '2023-03-20T12:48:08.056Z'
---

Oracle目录应急清理

# 清理错误位置的归档日志
检查`$ORACLE_HOME/dbs`下是否有归档文件：
```bash
ls $ORACLE_HOME/dbs/arch* | wc -l
```

检查和修改归档位置：
```sql
--检查归档位置
SQL> archive log list;

--修改归档位置
SQL> alter system set log_archive_dest_1='location=/oradata/arch' scope=both;
```

移动或清理`$ORACLE_HOME/dbs`下的归档文件： 
```bash
mv $ORACLE_HOME/dbs/arch* /oradata/arch/
```

如果不是归档位置错误，优先进行扩容，无法扩容再考虑清理oracle目录。

# 清理30天前的监听告警日志 
清理`/oracle/app/oracle/diag/tnslsnr/<hostname>/listener/alert`目录。可以清理30天以前的`log_xxxx.xml`，注意**不**能删除`log.xml`。

```bash
# 检查文件个数和占用空间
ls $ORACLE_BASE/diag/tnslsnr/<hostname>/listener/alert | wc -l
du -sh $ORACLE_BASE/diag/tnslsnr/<hostname>/listener/alert

# 清理文件
find $ORACLE_BASE/diag/tnslsnr/<hostname>/listener/alert -mtime +30 -name "log_*.xml" | xargs rm -rf
```

# 清理监听日志 
清理`$ORACLE_BASE/diag/tnslsnr/<hostname>/listener/trace/listener.log`。

```bash
# 检查文件大小
du -sh $ORACLE_BASE/diag/tnslsnr/<hostname>/listener/trace/listener.log

# 备份并压缩监听日志
cd $ORACLE_BASE/diag/tnslsnr/<hostname>/listener/trace/
cp listener.log listener.log.bak
gzip listener.log.bak > listener.log.bak.gz

# 清空监听日志
echo '' > listener.log
```

# 清理30天以前的trace文件
```bash
# 检查文件个数和占用空间
ls $ORACLE_BASE/diag/rdbms/<db_unique_name>/<ORACLE_SID>/trace/ | wc -l
du -sh $ORACLE_BASE/diag/rdbms/<db_unique_name>/<ORACLE_SID>/trace/

# 清理文件
find $ORACLE_BASE/diag/rdbms/<db_unique_name>/<ORACLE_SID>/trace/ -mtime +30 -type f | xargs rm -rf
```

# 清理30天以前的审计日志
```bash
# 检查文件个数和占用空间
du -sh $ORACLE_BASE/admin/<ORACLE_SID>/adump
ls $ORACLE_BASE/admin/<ORACLE_SID>/adump | wc -l

# 清理文件
find $ORACLE_BASE/admin/<ORACLE_SID>/adump -type f -mtime +30 | xargs rm -rf
```



