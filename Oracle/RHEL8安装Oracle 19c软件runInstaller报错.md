---
tags: [oracle]
title: RHEL8安装Oracle 19c软件runInstaller报错
created: '2024-01-24T11:17:29.040Z'
modified: '2024-01-24T11:19:57.865Z'
---


RHEL8安装Oracle 19c软件runInstaller报错

:duck:操作系统版本：CentOS 8.8

:frog:报错信息如下：
```bash
INFO : install oracle database software 19c in silent mode
Launching Oracle Database Setup Wizard...

[WARNING] [INS-08101] Unexpected error while executing the action at state: 'supportedOSCheck'
   CAUSE: No additional information available.
   ACTION: Contact Oracle Support Services or refer to the software manual.
   SUMMARY:
       - java.lang.NullPointerException
Moved the install session logs to:
 /oracle/oraInventory/logs/InstallActions2024-01-24_02-15-22PM

INFO : now excute /oracle/oraInventory/orainstRoot.sh
sh: /oracle/oraInventory/orainstRoot.sh: No such file or directory

...
```

:lion:MOS检索关键字：INS-08101 supportedOSCheck RHEL8，匹配到如下文档：
- Doc ID 2584365.1
- Doc ID 2668780.1

上述报错涉及到操作系统兼容性问题。

:star:解决办法：
```bash
echo 'export CV_ASSUME_DISTID=OL7' >> /home/oracle/.bash_profile 
source /home/oracle/.bash_profile 
```
然后重新运行runInstaller即可。
