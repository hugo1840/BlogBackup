---
tags: [oracle]
title: 跨Nginx代理搭建Oracle DG
created: '2023-07-05T14:31:12.005Z'
modified: '2023-07-16T08:35:51.786Z'
---

跨Nginx代理搭建Oracle DG

| 服务器角色 | IP |
| :--: | :--: |
| Oracle主库 | 172.16.171.68 |
| Nginx代理 | 172.16.171.69 |
| Oracle备库 | 172.16.171.70 |

**注**：操作系统为CentOS 7.9，数据库版本为19c，Nginx版本为1.24.0。

>- :beer:Nginx安装参见：https://blog.csdn.net/Sebastien23/article/details/131622725
>- :beer:DG搭建参见：https://blog.csdn.net/Sebastien23/article/details/128858710


# Nginx转发代理配置
配置Nginx服务器的hosts文件，添加数据库服务器主机名解析：
```bash
[root@webserver ~]# cat /etc/hosts
::1     localhost       localhost.localdomain   localhost6      localhost6.localdomain6
127.0.0.1       localhost       localhost.localdomain   localhost4      localhost4.localdomain4

172.16.171.69   webserver       webserver
172.16.171.68   primarydb       primarydb
172.16.171.70   standbydb       standbydb
```

## upstream模块配置
检查nginx当前编译的模块，确认包含stream模块：
```bash
[root@webserver nginx]# nginx -V
nginx version: nginx/1.24.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
configure arguments: --prefix=/nginx --with-stream
```

编辑Nginx DB代理配置文件`conf.d/dbstream_proxy.conf`：
```bash
[root@webserver nginx]# mkdir conf.d
[root@webserver nginx]# cat > conf.d/dbstream_proxy.conf << EOF
stream {
    upstream oracle_primary {
        # 如果30s内有3次建立连接超时(proxy_connect_timeout)，则在该30s内不再接受新的连接
        server 172.16.171.68:1521 max_fails=3 fail_timeout=30s;
    }

    upstream oracle_standby {
        server 172.16.171.70:1521 max_fails=3 fail_timeout=30s;
    }

    # 反向代理监听端口，访问nginx_ip:1522就可以访问到oracle_primary
    server {
        listen 1522 so_keepalive=on;       # 开启socket探活，走操作系统默认策略
        proxy_connect_timeout 3s;          # 建立连接的超时时间，默认60s
        #proxy_timeout 60s;                 # 接收后端服务器响应超时时间，默认10min
        proxy_pass oracle_primary;
    }

    server {
        listen 1523 so_keepalive=on;
        proxy_connect_timeout 3s;
        #proxy_timeout 60s;
        proxy_pass oracle_standby;
    }
}
EOF

# 修改配置文件属主为nginx
[root@webserver nginx]# chown -R nginx:nginx conf.d/
```

## 主配置文件修改
编辑主配置文件`nginx.conf`，include上面的stream配置文件，并注释掉http模块中server的部分（仅保留access_log之前的几行即可）：
```bash
[root@webserver nginx]# cat conf/nginx.conf

#user  nobody;
worker_processes  4;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

include ../conf.d/dbstream_proxy.conf;

http {
     include       mime.types;
     default_type  application/octet-stream;

     log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';

     access_log  logs/access.log  main;

 #   sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
 #   keepalive_timeout  65;

    #gzip  on;

    #server {
    #    listen       80;
    #    server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
    #    error_page   500 502 503 504  /50x.html;
    #    location = /50x.html {
    #        root   html;
    #    }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    #}


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
}
```

测试配置文件：
```bash
[root@webserver nginx]# nginx -t
nginx: the configuration file /nginx/conf/nginx.conf syntax is ok
nginx: configuration file /nginx/conf/nginx.conf test is successful
```

启动Nginx服务器：
```bash
[root@webserver nginx]# systemctl restart nginx
```


## 测试数据库代理（可选）
启动nginx后检查监听：
```bash
[root@webserver nginx]# ss -antpl | grep 152
LISTEN     0      128          *:1522                     *:*                   users:(("nginx",pid=3445,fd=6),("nginx",pid=3444,fd=6),("nginx",pid=3443,fd=6),("nginx",pid=3442,fd=6),("nginx",pid=3441,fd=6))
LISTEN     0      128          *:1523                     *:*                   users:(("nginx",pid=3445,fd=7),("nginx",pid=3444,fd=7),("nginx",pid=3443,fd=7),("nginx",pid=3442,fd=7),("nginx",pid=3441,fd=7))
```

从数据库服务器检查到nginx服务器的端口连通性：
```bash
[oracle@dbprimary ~]$ telnet 172.16.171.69 1522
Trying 172.16.171.69...
Connected to 172.16.171.69.
Escape character is '^]'.
^]
telnet> \q
Connection closed.

[oracle@dbprimary ~]$ telnet 172.16.171.69 1523
Trying 172.16.171.69...
Connected to 172.16.171.69.
Escape character is '^]'.
^]
telnet> \q
Connection closed.
```

尝试通过nginx代理登录数据库：
```sql
[oracle@dbprimary ~]$ sqlplus MIGUEL/XXXXXX@172.16.171.69:1522/${ORACLE_SID}

SQL*Plus: Release 19.0.0.0.0 - Production on Mon Jul 10 14:24:46 2023
Version 19.3.0.0.0

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> select sys_context('userenv','current_user') from dual;

SYS_CONTEXT('USERENV','CURRENT_USER')
--------------------------------------------------------------------------------
MIGUEL
```

登陆成功。等待几秒执行SQL，发现连接已经断开：
```sql
SQL> /
select sys_context('userenv','current_user') from dual
*
ERROR at line 1:
ORA-03113: end-of-file on communication channel
Process ID: 4048
Session ID: 1172 Serial number: 50021
```

检查nginx错误日志，发现是nginx反向代理连接超时导致：
```bash
[root@webserver nginx]# tail -f logs/error.log
...
2023/07/10 14:22:19 [error] 3443#0: *3 upstream timed out (110: Connection timed out) while connecting to upstream, 
client: 172.16.171.69, server: 0.0.0.0:1523, upstream: "172.16.171.68:1521", bytes from/to client:0/0, bytes from/to upstream:0/0
```

修改`proxy_timeout`参数为1分钟后重载nginx：
```bash
[root@webserver nginx]# cat conf.d/dbstream_proxy.conf 
stream {    
    upstream oracle_primary {  
        # 如果30s内有3次建立连接超时(proxy_connect_timeout)，则在该30s内不再接受新的连接    
        server 172.16.171.68:1521 max_fails=3 fail_timeout=30s;
    }

    upstream oracle_dg1 {   
        server 172.16.171.70:1521 max_fails=3 fail_timeout=30s;  
    }

    # 反向代理监听端口，访问nginx_ip:1522就可以访问到oracle_primary
    server {
        listen 1522 so_keepalive=on;       # socket会话保持      
        proxy_connect_timeout 3s;          # 建立连接的超时时间，默认60s
        proxy_timeout 60s;                 # 接收后端服务器响应超时时间，默认10min
        proxy_pass oracle_primary;
    }

    server {
        listen 1523 so_keepalive=on;               
        proxy_connect_timeout 3s;
        proxy_timeout 60s;
        proxy_pass oracle_dg1;
    }
}

[root@webserver nginx]# systemctl restart nginx
```

重新通过nginx代理连接数据库，分别在连接成功后间隔10s和1分钟后执行SQL：
```sql
SQL> select sys_context('userenv','current_user') from dual;

SYS_CONTEXT('USERENV','CURRENT_USER')
--------------------------------------------------------------------------------
MIGUEL

SQL> /

SYS_CONTEXT('USERENV','CURRENT_USER')
--------------------------------------------------------------------------------
MIGUEL

SQL> /
select sys_context('userenv','current_user') from dual
*
ERROR at line 1:
ORA-03113: end-of-file on communication channel
Process ID: 5741
Session ID: 1169 Serial number: 35827
```
第二次连接超时，说明是`proxy_timeout`参数导致连接超时。

这里我们注释`proxy_timeout`参数，即采用默认值10min。

## 会话保持配置介绍
1. `conf/nginx.conf`中通过配置`keepalive_timeout`参数来开启http客户端连接保活机制，可以设置得大一些（这里为2分钟）。

```bash
http {
     include       mime.types;
     default_type  application/octet-stream;

     log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';

     access_log  logs/access.log  main;

    #sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  120s;     # nginx与客户端连接在空闲状态下的超时时间
}
```

2. `conf.d/dbstream_proxy.conf`中通过配置`so_keepalive`参数开启TCP层探活机制。

```bash
[root@webserver nginx]# cat conf.d/dbstream_proxy.conf 
stream {
    ...
	
    # 反向代理监听端口，访问nginx_ip:1522就可以访问到oracle_primary
    server {
        listen 1522 so_keepalive=on;       # 开启socket探活，走操作系统默认策略
        proxy_connect_timeout 3s;          # 建立连接的超时时间，默认60s
        #proxy_timeout 60s;                 # 接收后端服务器响应超时时间，默认10min
        proxy_pass oracle_primary;
    }

    server {
        listen 1523 so_keepalive=on;
        proxy_connect_timeout 3s;
        #proxy_timeout 60s;
        proxy_pass oracle_standby;
    }
}
```

操作系统层的TCP keepalive参数查看方法：
```bash
[root@webserver nginx]# sysctl -a | grep keepalive
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200
```

# 跨Nginx搭建DG
## 数据库连接配置
分别配置主备库的hosts解析文件，将远端库的地址解析配置为Nginx服务器；同时编辑主备库的`tnsnames.ora`文件，分别配置Nginx代理的不同端口。

主库配置：
```bash
[oracle@primarydb ~]$ cat /etc/hosts | grep db
172.16.171.68   primarydb       primarydb
172.16.171.69   standbydb       standbydb

[oracle@primarydb ~]$ cat $ORACLE_HOME/network/admin/tnsnames.ora
BANGKOK =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkok)
    )
  )

BANGKOKDG =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = standbydb)(PORT = 1523))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkokdg)
    )
  )
```

备库配置：
```bash
[oracle@standbydb ~]$ cat /etc/hosts | grep db
172.16.171.70   standbydb       standbydb
172.16.171.69   primarydb       primarydb

[oracle@standbydb ~]$ cat $ORACLE_HOME/network/admin/tnsnames.ora
BANGKOK =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = primarydb)(PORT = 1522))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkok)
    )
  )

BANGKOKDG =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = standbydb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = bangkokdg)
    )
  )
```

启动主备库的数据库监听：
```bash
lsnrctl status
lsnrctl start
```

启动主库：
```sql
SQL> startup;
```

测试主备库的连通性：
```sql
# 备库跨Nginx成功连接到主库
[oracle@standbydb ~]$ sqlplus MIGUEL/XXXXXX@172.16.171.69:1522/bangkok

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Jul 16 15:49:12 2023
Version 19.3.0.0.0

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> show parameter db_unique

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_unique_name                       string      bangkok
SQL> exit

# 主库跨Nginx连接备库失败，这里是因为备库还没有打开
[oracle@primarydb ~]$ sqlplus MIGUEL/XXXXXX@172.16.171.69:1523/bangkokdg

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Jul 16 15:45:53 2023
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

ERROR:
ORA-01033: ORACLE initialization or shutdown in progress
Process ID: 0
Session ID: 0 Serial number: 0
```

## 跨Nginx同步DG
启动备库：
```sql
[oracle@standbydb ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Jul 16 15:35:24 2023
Version 19.3.0.0.0

Connected to an idle instance.

SQL> startup nomount;
ORACLE instance started.

Total System Global Area 4915722040 bytes
Fixed Size                  8906552 bytes
Variable Size             889192448 bytes
Database Buffers         4009754624 bytes
Redo Buffers                7868416 bytes
SQL> alter database mount standby database;

Database altered.
```

启动备库同步，并打开备库：
```sql
SQL> alter database recover managed standby database using current logfile disconnect;

Database altered.

SQL> set lines 200
SQL> col name for a30
SQL> select name,value,unit,time_computed from v$dataguard_stats where name in ('transport lag','apply lag');

NAME                           VALUE                          UNIT                           TIME_COMPUTED
------------------------------ ------------------------------ ------------------------------ ------------------------------
transport lag                  +00 00:00:00                   day(2) to second(0) interval   07/16/2023 15:47:41
apply lag                      +00 00:00:00                   day(2) to second(0) interval   07/16/2023 15:47:41

SQL>  select process,status,thread#,sequence#,block# from v$managed_standby where status <> 'IDLE';

PROCESS   STATUS          THREAD#  SEQUENCE#     BLOCK#
--------- ------------ ---------- ---------- ----------
ARCH      CONNECTED             0          0          0
DGRD      ALLOCATED             0          0          0
DGRD      ALLOCATED             0          0          0
ARCH      CONNECTED             0          0          0
ARCH      CLOSING               1         17          1
ARCH      CONNECTED             0          0          0
MRP0      APPLYING_LOG          1         19      16726

7 rows selected.

SQL> alter database recover managed standby database cancel;

Database altered.

SQL> alter database open;

Database altered.

SQL> alter database recover managed standby database using current logfile disconnect from session;

Database altered.
```

再次测试主备跨Nginx连接备库，成功：
```bash
[oracle@primarydb ~]$ sqlplus MIGUEL/XXXXXX@172.16.171.69:1523/bangkokdg

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Jul 16 15:49:27 2023
Version 19.3.0.0.0

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

> show parameter db_unique

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_unique_name                       string      bangkokdg
> exit
```

观察主备告警日志，等待30分钟以上不进行任何操作，观察同步是否会断：
```bash
[oracle@standbydb ~]$ tail -fn 100 $ORACLE_BASE/diag/rdbms/bangkokdg/bangkokdg/trace/alert_bangkokdg.log

[oracle@primarydb ~]$  tail -fn 100 $ORACLE_BASE/diag/rdbms/bangkok/bangkok/trace/alert_bangkok.log
```
如果同步断开，在排除网络因素后，可能还需要调整Nginx代理的超时参数。


**References** 
【1】http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_connect_timeout 
【2】http://nginx.org/en/docs/stream/ngx_stream_core_module.html#proxy_protocol_timeout 
【3】http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive_timeout
【4】https://www.lmlphp.com/user/16836/article/item/538102/
【5】https://www.cnblogs.com/woaiyitiaochai/p/11778291.html
【6】https://www.cnblogs.com/Anker/p/6056540.html





