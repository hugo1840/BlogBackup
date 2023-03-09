---
tags: [oracle]
title: Oracle配置多个TNSNAMES连接串
created: '2023-03-09T08:48:25.244Z'
modified: '2023-03-09T10:51:30.966Z'
---

Oracle配置多个TNSNAMES连接串

# 检查服务名与TNS配置
检查服务名：
```sql
SQL> show parameter service_names

NAME           TYPE     VALUE
----           ----     -----
service_names  string   bangkok
```

检查`tnsnames.ora`配置：
```bash
[oracle@primarydb dbs]$ cat $ORACLE_HOME/network/admin/tnsnames.ora
BANGKOK = 
(DESCRIPTION = 
  (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521))
  (CONNECT_DATA = 
    (SERVER = DEDICATED) 
    (SERVICE_NAME = bangkok) 
  ) 
)
```

可以通过以下TNS连接串连接到数据库：
```bash
sqlplus username/password@BANGKOK
sqlplus username/password@bangkok
```


# 修改服务名
修改服务名（一般不建议修改）：
```sql
SQL> alter system set service_names=BANGKOK_SVC scope=both;

System altered.

SQL> show parameter service_names

NAME           TYPE     VALUE
----           ----     -----
service_names  string   BANGKOK_SVC
```

重新通过TNS连接：
```bash
[oracle@primarydb dbs]$ sqlplus miguel/XXXXX@bangkok

ERROR: 
ORA-01034: ORACLE not available 
ORA-27101: shared memory realm does not exist 
Linux-x86_64 Error: 2: No such file or directory 
Additional information: 4376 
Additional information: -545674901 
Process ID: 0 Session ID: 0 Serial number: 0

[oracle@primarydb dbs]$ sqlplus miguel/XXXXX@BANGKOK_SVC

ERROR: 
ORA-12154: TNS:could not resolve the connect identifier specified
```

# 修改TNS配置
将`tnsnames.ora`中的**SERVICE_NAME**修改为正确的服务名：
```bash
vim $ORACLE_HOME/network/admin/tnsnames.ora
BANGKOK = 
(DESCRIPTION = 
  (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521))
  (CONNECT_DATA = 
    (SERVER = DEDICATED) 
    (SERVICE_NAME = BANGKOK_SVC) 
  ) 
)
```
修改后立即生效，无需重启监听。

检查TNS连接：
```bash
[oracle@primarydb admin]$ sqlplus miguel/XXXXX@bangkok

SQL*Plus: Release 19.0.0.0.0 - Production on Thu Mar 9 14:31:11 2023 Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle. All rights reserved.

Last Successful login time: Thu Mar 09 2023 14:16:00 +08:00

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production Version 19.3.0.0.0
```
连接成功。


# 配置多个TNS连接串
定义多个TNS连接串：
```bash
[oracle@primarydb admin]$ cat tnsnames.ora 
BANGKOK = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521)) 
    (CONNECT_DATA = 
      (SERVER = DEDICATED) 
      (SERVICE_NAME = BANGKOK_SVC)  
    ) 
  )

BANGKOKpri = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521)) 
    (CONNECT_DATA = 
      (SERVER = DEDICATED) 
      (SERVICE_NAME = BANGKOK_SVC) 
    )
  )
```

使用`tnsping`命令解析tnsnames：
```bash
[oracle@primarydb admin]$ tnsping bangkok

Used TNSNAMES adapter to resolve the alias Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = BANGKOK_SVC))) OK (0 msec)

[oracle@primarydb admin]$ tnsping bangkokpri

Used TNSNAMES adapter to resolve the alias Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = BANGKOK_SVC))) OK (0 msec)

[oracle@primarydb admin]$ tnsping BANGKOKpri

Used TNSNAMES adapter to resolve the alias Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = BANGKOK_SVC))) OK (0 msec)
```
都能解析成功。

检查能否正常连接：
```bash
[oracle@primarydb admin]$ sqlplus miguel/XXXXX@bangkokpri

Last Successful login time: Thu Mar 09 2023 14:31:11 +08:00

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production Version 19.3.0.0.0

SQL> exit

[oracle@primarydb admin]$ sqlplus miguel/XXXXX@BANGKOKpri

Last Successful login time: Thu Mar 09 2023 14:35:23 +08:00

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production Version 19.3.0.0.0

SQL> exit

[oracle@primarydb admin]$ sqlplus miguel/XXXXX@BANGKOKPRI

Last Successful login time: Thu Mar 09 2023 14:35:34 +08:00

Connected to: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production Version 19.3.0.0.0

SQL> exit
```


