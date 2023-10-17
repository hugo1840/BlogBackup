---
tags: [postgresql]
title: PostgreSQL数据库配置文件
created: '2023-10-17T11:38:12.915Z'
modified: '2023-10-17T11:49:30.607Z'
---

PostgreSQL数据库配置文件

>PostgreSQL版本：10.5

检查数据库参数：

```sql
postgres=# select name,setting,unit from pg_settings 
where name in 
('max_connections','shared_buffers','Effective_cache_size',
'work_mem','maintenance_work_mem','wal_sync_method','wal_buffers',
'synchronous_commit','default_statistics_target',
'checkpoint_timeout','checkpoint_completion_target',
'max_wal_size','min_wal_size');

             name             |  setting  | unit 
------------------------------+-----------+------
 checkpoint_completion_target | 0.5       | 
 checkpoint_timeout           | 300       | s
 default_statistics_target    | 100       | 
 maintenance_work_mem         | 16384     | kB
 max_connections              | 100       | 
 shared_buffers               | 131072    | 8kB
 synchronous_commit           | on        | 
 wal_buffers                  | 2048      | 8kB
 wal_sync_method              | fdatasync | 
 work_mem                     | 1024      | kB
(10 rows)
```

# 配置文件postgresql.conf
## 数据库连接认证参数
与数据库连接和安全认证相关的参数，可以参考`pg_settings`视图中的说明。

- **listen_address**：监听客户端连接的TCP/IP地址。修改后需要重启数据库。支持逗号分隔的多个地址。设置成`*`或者`0.0.0.0`表示监听本机的所有IP地址，并且支持IPv6地址。
- **port**：监听的TCP端口，默认为**5432**。修改后需要重启数据库。
- **max_connections**：数据库的最大并发连接数。默认为100，修改后需要重启数据库。
- **authentication_timeout**：客户端建立连接的超时时间，单位是秒。如果在该时间内没有完成客户端认证操作，则连接将被拒绝。默认为60s。


## 数据库内存参数
与数据库缓存和I/O相关的参数，对数据库性能调优非常重要。

- **shared_buffers**：数据库使用的共享内存缓冲区大小。默认为128MB。建议配置为操作系统内存的**25%~40%**。
- **temp_buffers**：单个会话使用的临时缓冲区大小。默认值为8MB。可以在会话层面进行设置。
- **work_mem**：工作内存，用于SORT操作和HASH操作。默认值为4MB。如果设置过小，排序操作和哈希操作会使用磁盘SWAP，会极大的降低系统性能；如果设置过大，进行排序和哈希操作的并发连接数大时，会耗费大量的系统内存。
- **maintenance_work_mem**：维护工作内存，用于VACUUM、CREATE INDEX、REINDEX和ALTER DATABASE ADD FOREIGN KEY等数据库维护工作。默认为64MB。
- **autovacuum_work_mem**：每个自动清理工作能够使用的最大内存。默认值为-1，表示使用maintenance_work_mem的值。
- **huge_pages**：是否开启大页，可取值包括try、on和off。默认为try，会尝试使用大页启动服务，如果失败将通过普通的内存申请启动数据库服务。

>:dolphin: 内存参数配置须满足： 
> 
> max_connections*work_mem + shared_buffers + temp_buffers + maintenance_work_mem < OS_PHYSIC_MEMORY


为PostgreSQL配置大页内存的方法如下：
```bash
#查看操作系统大页大小
grep Hugepage /proc/meminfo

#计算所需大页数量
num_hpages = shared_buffers/Hugepagesize

#设置大页数量
sed -i '\$ a\vm.nr_hugepages = ${num_hpages}' /etc/sysctl.conf

#在数据库中启用大页
sed -i 's/.*huge_pages.*/huge_pages = on/g' $PGDATA/postgresql.conf

#重启操作系统和数据库
reboot
pg_ctl start

#检查大页
cat /proc/meminfo | grep -i huge
```

## WAL日志参数
WAL日志的作用是保证数据一致性和事务完整性，对于数据库崩溃时进行事务恢复至关重要。

- **wal_level**：WAL日志级别，可取值包括minimal、replica、logical。默认为replica，表示记录WAL日志归档和流复制所需的数据，包括在备库进行的只读操作。设置为minimal时，表示只写入在数据库崩溃后事务恢复所需要的信息。修改该参数后需要重启数据库。
- **fsync**：默认为on，表示数据库将调用fsync函数来确保被更新的数据写入磁盘。该参数用于保证数据库在操作系统或硬件崩溃后能够恢复到一个一致性状态。关闭该参数可以提升数据库性能，但是会带来数据安全问题。
- **synchronous_commit**：用于指定事务提交后何时向客户端返回成功。可取值包括on、remote_apply、remote_write、local、off。默认值为on。


## 错误日志参数
错误日志输出策略相关：

- **log_destination**：指定数据库输出日志的格式，包括stderr、csvlog、syslog。默认为stderr，表示将错误日志重定向到标准错误输出；syslog表示输出到操作系统syslog日志中；csvlog表示生成CSV格式的错误日志。
- **logging_collector**：设置为on时，表示启用日志收集器，会将捕捉到的stderr日志消息重定向到日志文件。
- **log_directory**：日志文件所在的目录。
- **log_filename**：日志文件名称格式。默认是`postgresql-%y-%m-%d_%h%m%s.log`。
- **log_rotation_age**：单个日志文件的最大生命周期，默认为1d。超过该时间后，会生成一个新的日志文件。
- **log_rotation_size**：单个日志文件的最大大小，默认为10MB。超过该大小后，会生成一个新的日志文件。
- **log_truncate_on_rotation**：表示是否开启日志轮转，默认为off。
- **log_file_mode**：日志文件权限。默认为0600，表示只有数据库OS用户才能读写日志。

未开启日志轮转时，生成的日志文件不会过期，需要手动清理，否则可能导致文件系统空间被占满。假如日志文件只需保留7天，可以进行如下设置：
```sql
alter system set logging_collector=on;
alter system set log_filename="server_%a.log";
alter system set log_truncate_on_rotation=on;
```
然后重启数据库。这样每周会依次生成server_Mon.log、server_Tue.log、...、server_Sun.log七个日志文件，并且最新的日志文件会覆盖上一周生成的同名文件。

错误日志输出内容相关：

- **log_connections**：是否记录每一次对数据库的连接尝试以及客户端认证成功的信息。默认为off。
- **log_disconnections**：是否记录会话终止事件。默认为off。
- **log_duration**：是否记录已完成SQL语句的执行时间。默认为off。
- **log_min_duration_statement**：慢SQL执行时长。超过该时间的SQL文本和执行时间会被记录。默认为-1，表示不记录慢SQL。
- **log_lock_waits**：是否记录锁等待信息，默认为off。
- **log_statement**：是否记录特定类型SQL语句，可取值包括none、ddl（只记录DDL语句）、mod（记录DDL和DML语句）和all。默认为none。

>:fish:**postgresql.auto.conf**文件：
> 
> 通过ALTER SYSTEM修改的配置参数会被记录到postgresql.auto.conf文件中。该文件不能手动修改。


# 配置文件pg_hba.conf
HBA即Host-Based Authentication，该配置文件记录了允许哪些IP地址的机器可以访问数据库。
```bash
[postgres@dbhost ~]$ cat $PGDATA/pg_hba.conf | grep -v '^$' | grep -v '^#'
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             0.0.0.0/0               trust 
host    all             all             ::1/128                 trust
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
```

其中：

- 第一列表示允许的数据库访问协议。例如loal（UNIX域套接字连接）、host（TCP/IP连接）、hostssl（使用SSL加密的TCP/IP连接）、hostnossl（不使用SSL加密的TCP/IP连接）。
- 第二列匹配数据库名称。all匹配所有数据库，replication表示允许流复制连接。
- 第三列匹配数据库用户名。all匹配所有用户。
- 第四列匹配客户端服务器地址。可以是主机名或者IP地址范围。
- 第五列匹配认证方式。PostgreSQL支持的认证方式包括trust、reject、md5、ident等。

常用的认证方式：
- **trust**：表示无条件地允许连接，不需要口令和其他任何认证。数据库服务器上的任何操作系统用户都可以使用数据库超级用户连接到数据库，存在安全隐患。
- **md5**：表示在连接数据时需要使用密码验证。该认证方法要求客户端提供一个双重MD5加密口令来进行认证。
- **cert**：表示使用SSL客户端证书进行认证。该认证方法只适用于SSL连接。


**References** 
【1】https://www.postgresql.org/docs/current/runtime-config-resource.html 
【2】https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server 
【3】https://www.percona.com/blog/2018/08/31/tuning-postgresql-database-parameters-to-optimize-performance/ 
【4】https://pgtune.leopard.in.ua/#/




