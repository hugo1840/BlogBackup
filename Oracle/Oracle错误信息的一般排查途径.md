---
tags: [oracle]
title: Oracle错误信息的一般排查途径
created: '2023-03-11T05:04:53.742Z'
modified: '2023-03-11T06:14:21.663Z'
---

Oracle错误信息的一般排查途径

以下面的报错信息为例：
```bash
ORA-03113: end-of-file on communication channel
Process ID: 16770
Session ID: 771 Serial number: 45608
```
我们可以通过以下几种方法来进一步分析该报错。

# oerr命令
使用Oracle提供的`oerr`命令来查看错误信息的含义和大致解决思路。

oerr命令的使用格式为：
```bash
oerr <进程名> <错误编号>
```

例如：
```bash
# 查看ORA-03113报错的含义
[oracle@orahost ~]$ oerr ora 03113
03113, 00000, "end-of-file on communication channel"
// *Cause: The connection between Client and Server process was broken.
// *Action: There was a communication error that requires further investigation.
//          First, check for network problems and review the SQL*Net setup. 
//          Also, look in the alert.log file for any errors. Finally, test to 
//          see whether the server process is dead and whether a trace file
//          was generated at failure time.

# 查看LRM-00109报错的含义
[oracle@orahost ~]$ oerr lrm 00109
109, 0, "could not open parameter file '%.*s'"
// *Cause: The parameter file does not exist.
// *Action: Create an appropriate parameter file.
```

# alert日志
Alert日志记录了数据库在运行过程中的几乎所有告警信息。

Alert日志的位置和命名格式如下：
```bash
$ORACLE_BASE/diag/rdbms/<db_unique_name>/<instance_name>/trace/alert_<instance_name>.log
```

如果数据库运行了较长时间，alert日志通常会非常大（几个G甚至十几个G），不建议使用vim打开。
```bash
less $ORACLE_BASE/diag/rdbms/bangkok/BANGKOK/trace/alert_BANGKOK.log
tail -n 200 $ORACLE_BASE/diag/rdbms/bangkok/BANGKOK/trace/alert_BANGKOK.log

# 从Alert日志中提取指定的行
grep -n 'ORA-03113' $ORACLE_BASE/diag/rdbms/bangkok/BANGKOK/trace/alert_BANGKOK.log
sed -n '20000,20500p' $ORACLE_BASE/diag/rdbms/bangkok/BANGKOK/trace/alert_BANGKOK.log > /tmp/alert_ora_0308.log
```

# trace文件
Trace文件中详细记录了数据库进程在运行过程中产生的重要信息。

Trace跟踪文件的位置和命名格式如下：
```bash
$ORACLE_BASE/diag/rdbms/<db_unique_name>/<instance_name>/trace/<instance_name>_<process_name>_<process_ID>.trc
```

根据报错信息中的进程名和进程编号（Process ID），找到对应的trace文件。
```bash
tail -f $ORACLE_BASE/diag/rdbms/bangkok/BANGKOK/trace/BANGKOK_ora_16770.trc
```




