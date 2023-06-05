---
tags: [OCP, oracle]
title: LISTENER、TNSNAMES和SQLNET配置文件
created: '2023-05-29T12:52:54.894Z'
modified: '2023-06-05T12:07:30.145Z'
---

LISTENER、TNSNAMES和SQLNET配置文件

# 用户连接验证
Oracle数据库中用户有三种常见的登录验证方式：
- 通过操作系统用户验证：必须是在数据库服务器本地登录，并且操作系统用户必须是oracle用户。此方式不会验证用户名和密码，并且登录数据库后的用户身份是**SYS**。
```bash
sqlplus / as sysdba
sqlplus <username>/<wrong_password> as sysdba   # 输错密码也可以登录
```

- 通过EZCONNECT连接方式验证:
```bash
sqlplus <username>/<password>@<listener_hostname>:<listener_port>/<service_name>
```

- 通过TNSNAMES连接方式验证：此方式要求提前在`tnsnames.ora`中配置好TNS连接名。
```bash
sqlplus <username>/<password>@<TNS连接名>
```

# listener.ora
:shark:Oracle监听器的配置文件，位置在数据库服务器的`$ORACLE_HOME/network/admin/listener.ora`。Listener负责监听客户端的连接请求，并管理到连接到数据库服务器的流量。

## 文件配置
```bash
LISTENER =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracle19c)(PORT = 1521))
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = mario)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1/)
      (SID_NAME = mario)
    )
  )

ADR_BASE_LISTENER=/u01/app      # 监听器LISTENER的ADR自动诊断文件的位置
DIAG_ADR_ENABLED_LISTENER=OFF   # 是否开启对监听器LISTENER的ADR tracing
```
其中：
- `HOST`：监听器所在本地服务器的主机名，也可以是IP地址。如果是主机名，需要在`/etc/hosts`文件或者DNS服务器中配置解析。
- `ORACLE_HOME`：与oracle操作系统用户的环境变量`${ORACLE_HOME}`保持一致。
- `GLOBAL_DBNAME`和`SID_NAME`：通常与pfile中的`db_name`和`instance_name`、以及oracle操作系统用户的环境变量`${ORACLE_SID}`保持一致。


## 监听日志
通过Listener监听日志可以协助排查一些客户端的连接问题。
```bash
grep -A5 '2022-04-27' $ORACLE_BASE/diag/tnslsnr/`hostname`/listener/alert/log.xml
```

## local_listener参数
LREG进程只会动态注册（每隔1分钟）端口为1521的监听。如果要给监听器配置**非**1521端口的监听，还需要修改`local_listener`参数。以配置1523监听端口为例，具体配置过程如下。

在Listener配置文件中添加：
```bash
LISTENER2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracle19c)(PORT = 1522))
  )
```

在TNS配置文件中添加：
```bash
LISTENER2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracle19c)(PORT = 1522))
  )
```

修改`local_listener`参数：
```sql
SQL> alter system set local_listener=listener2;
```

检查监听器状态：
```bash
lsnrctl status listener2
lsnrctl status
```

如果要恢复默认配置只需置空`local_listener`参数：
```sql
SQL> alter system set local_listener='';
```


# tnsnames.ora
:shark:TNS配置文件中可以定义一系列连接描述符，用于标识数据库服务的网络位置和服务名。其位置在客户端或者数据库服务端的`$ORACLE_HOME/network/admin/tnsnames.ora`。

## 文件配置
```bash
MARIO =
(DESCRIPTION =
  (ADDRESS = (PROTOCOL = TCP)(HOST = oracle19c)(PORT = 1521))
  (CONNECT_DATA =
    (SERVER = DEDICATED)
    (SERVICE_NAME = mario)
  )
)
```
其中：
- 第一行等号前的字符串（这里是MARIO）为定义的TNS连接描述符，用于通过TNS方式连接到数据库。一般不区分大小写。
- `HOST`：Oracle数据库服务器（本地或者远程服务器都行）的主机名，也可以是IP地址。如果是主机名，需要在`/etc/hosts`文件或者DNS服务器中配置解析。
- `SERVICE_NAME`：与pfile中的`service_names`参数保持一致。可以通过`show parameter names`语句查看。


# sqlnet.ora
:shark:SQLNET配置文件中定义了一组Oracle Net相关参数。其位置在客户端或者数据库服务端的`$ORACLE_HOME/network/admin/sqlnet.ora`。

## 文件配置
SQLNET常见的一般配置如下：
```bash
# 解析客户端连接字符串所使用方法的先后顺序，默认顺序是：tnsnames > ldap > ezconnect
NAMES.DIRECTORY_PATH=(TNSNAMES,EZCONNECT)

ADR_BASE=/u01/app               # ADR自动诊断仓库文件的位置
DIAG_ADR_ENABLED=OFF            # 是否开启ADR tracing

# 每隔几分钟检查一次客户端和服务器连接是否还处于活跃状态
SQLNET.EXPIRE_TIME=10
# 客户端建立数据库连接的超时时间，默认60s
SQLNET.INBOUND_CONNECT_TIMEOUT=180
```









