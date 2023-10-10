---
tags: [postgresql]
title: PostgreSQL单机编译安装手册
created: '2023-10-10T11:47:18.312Z'
modified: '2023-10-10T11:49:36.732Z'
---

PostgreSQL单机编译安装手册

>:sunflower:下载二进制安装包：https://www.postgresql.org/ftp/source/

# 准备工作

```bash
# 检查是否已安装
rpm -qa | grep postgres

# 创建安装用户
groupadd postgres
useradd -g postgres -d /home/postgres -s /bin/bash postgres

# 创建安装目录
mkdir /opt/postgres
mv postgresql-9.2.4.tar.gz /opt/postgres
chown -R postgres:postgres /opt/postgres
```

# 编译安装

编译安装：
```bash
yum install -y zlib-devel gcc make

cd /opt/postgres
tar -xvf postgresql-9.2.4.tar.gz

cd postgresql-9.2.4/
./configure

make && make install
```


添加环境变量：
```bash
chown -R postgres:postgres /opt/postgres
su - postgres

echo 'export PATH=$PATH:/usr/local/pgsql/bin' >> .bash_profile
echo 'export PGDATA=/pgdata' >> .bash_profile
echo 'export PGLOG=/pgdata/server.log' >> .bash_profile

source .bash_profile
```

# 初始化数据库

创建数据目录，初始化数据库：
```bash
# 创建数据目录
mkdir /pgdata
chown -R postgres:postgres /pgdata

# 以postgres用户初始化数据库
su - postgres
initdb -D /pgdata -U postgres

# 启动数据库
pg_ctl -D /pgdata -l /pgdata/server.log start
```

检查数据库日志、端口和进程：
```bash
[postgres@dbhost ~]$ tail -f /pgdata/server.log 
LOG:  could not bind IPv4 socket: Address already in use
HINT:  Is another postmaster already running on port 5432? If not, wait a few seconds and retry.
LOG:  database system was shut down at 2023-10-10 11:03:49 CST
LOG:  database system is ready to accept connections
LOG:  autovacuum launcher started
^C
[postgres@dbhost ~]$ ss -antpl | grep 5432
LISTEN     0      208    127.0.0.1:5432                     *:*                   users:(("postgres",pid=68688,fd=4))
LISTEN     0      208      [::1]:5432                  [::]:*                   users:(("postgres",pid=68688,fd=3))

[postgres@dbhost ~]$ ps -ef | grep postgres
root      68464  53923  0 11:03 pts/0    00:00:00 su - postgres
postgres  68465  68464  0 11:03 pts/0    00:00:00 -bash
postgres  68688      1  0 11:05 pts/0    00:00:00 /usr/local/pgsql/bin/postgres -D /pgdata
postgres  68702  68688  0 11:05 ?        00:00:00 postgres: checkpointer process   
postgres  68703  68688  0 11:05 ?        00:00:00 postgres: writer process   
postgres  68704  68688  0 11:05 ?        00:00:00 postgres: wal writer process   
postgres  68705  68688  0 11:05 ?        00:00:00 postgres: autovacuum launcher process   
postgres  68706  68688  0 11:05 ?        00:00:00 postgres: stats collector process   
postgres  68772  68465  0 11:06 pts/0    00:00:00 ps -ef
postgres  68773  68465  0 11:06 pts/0    00:00:00 grep --color=auto postgres
```

使用psql命令连接到数据库：
```bash
[postgres@dbhost ~]$ psql
psql (9.2.4)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

postgres=# \c postgres
You are now connected to database "postgres" as user "postgres".
postgres=# \d
No relations found.
postgres=# \dS
                        List of relations
   Schema   |              Name               | Type  |  Owner   
------------+---------------------------------+-------+----------
 pg_catalog | pg_aggregate                    | table | postgres
 pg_catalog | pg_am                           | table | postgres
 pg_catalog | pg_amop                         | table | postgres
 pg_catalog | pg_amproc                       | table | postgres
 pg_catalog | pg_attrdef                      | table | postgres
 pg_catalog | pg_authid                       | table | postgres
 pg_catalog | pg_available_extension_versions | view  | postgres
 pg_catalog | pg_available_extensions         | view  | postgres
...
postgres=# 
postgres=# \q
```

# 修改配置文件

PostgreSQL的数据库配置文件是位于数据目录下的`postgresql.conf`和`pg_hba.conf`。
```bash
# 修改共享内存为服务器物理内存的25%
[postgres@dbhost ~]$ cat /pgdata/postgresql.conf | grep shared_buffer
shared_buffers = 32MB            # min 128kB
#wal_buffers = -1            # min 32kB, -1 sets based on shared_buffers

[postgres@dbhost ~]$ vi /pgdata/postgresql.conf 
[postgres@dbhost ~]$ cat /pgdata/postgresql.conf | grep shared_buffer
shared_buffers = 1024MB            
#shared_buffers = 32MB            # min 128kB
#wal_buffers = -1            # min 32kB, -1 sets based on shared_buffers

# 修改监听地址
[postgres@dbhost ~]$ cat /pgdata/postgresql.conf | grep -n listen
59:#listen_addresses = 'localhost'        # what IP address(es) to listen on;

[postgres@dbhost ~]$ vi /pgdata/postgresql.conf 
[postgres@dbhost ~]$ cat /pgdata/postgresql.conf | grep -n listen
59:listen_addresses = '*'                # 添加此行
60:#listen_addresses = 'localhost'        # what IP address(es) to listen on;

# 添加允许连接的客户端地址
[postgres@dbhost ~]$ cat /pgdata/pg_hba.conf | grep -A2 'IPv4 local connections'
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:

[postgres@dbhost ~]$ vi /pgdata/pg_hba.conf 
[postgres@dbhost ~]$ cat /pgdata/pg_hba.conf | grep -A2 'IPv4 local connections'
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all             all             0.0.0.0/0               trust      # 添加此行
```

重启数据库：
```bash
echo $PGDATA
pg_ctl stop
pg_ctl -D $PGDATA -l $PGLOG start
```

检查配置参数：
```sql
postgres=# show data_directory;
 data_directory 
----------------
 /pgdata
(1 row)

postgres=# show shared_buffers;
 shared_buffers 
----------------
 1GB
(1 row)

postgres=# select name,setting from pg_settings where name='data_directory';
      name      | setting 
----------------+---------
 data_directory | /pgdata
(1 row)
```



