@[TOC](安装MySQL主从并配置半同步)

# 安装MySQL
## 安装mysql 5.7
在CentOS7上安装mysql
```bash
$ wget -ic http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
$ yum -y install mysql57-community-release-el7-10.noarch.rpm
$ yum -y install mysql-community-server
```

启动数据库服务

```bash
$ systemctl start mysqld.service
$ systemctl status mysqld.service
```

## 修改密码
查看安装时的临时密码

```bash
$ grep "password" /var/log/mysqld.log
```
修改密码

```sql
$ mysql -uroot -p
PASSWORD:

mysql> set global validate_password_policy=LOW;
-- 限制密码长度为6
mysql> set global validate_password_length=6;

-- 修改root用户登陆密码为123456
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

-- 开启mysql的远程连接权限'
mysql> grant all privileges  on  *.* to 'root'@'%' identified by '123456' with grant option;
mysql> flush privileges;
```

# 配置半同步
查看MySQL服务器是否支持动态增加插件

```sql
mysql> select @@have_dynamic_loading;
-- YES表示支持
```

检查安装目录是否存在所需插件

```bash
$ find / -name semisync_*.so
/usr/lib64/mysql/plugin/semisync_master.so
/usr/lib64/mysql/plugin/semisync_slave.so
```

## 配置主节点
安装半同步插件

```sql
mysql> install plugin rpl_semi_sync_master SONAME 'semisync_master.so';
```

配置全局参数

```sql
mysql> show variables like 'rpl_semi_sync_master%';
...
mysql> set global rpl_semi_sync_master_enabled = 1;
mysql> set global rpl_semi_sync_master_timeout = 20000;
```

创建主从账户

```sql
-- repl为主从复制账户，172.25.223.35为从节点IP，123456为repl账户密码
mysql> grant Replication slave on *.* to 'repl'@'172.25.223.35' identified by '123456';
```

修改数据库配置文件。在`/etc/my.cnf`文件中的`[mysqld]`下加入以下内容，保存然后重启数据库服务。

```bash
port=3306
log-bin=mysql-bin
server-id=1  # 与从节点不能一样

rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_wait_point=AFTER_SYNC
rpl_semi_sync_master_wait_no_slave=ON
```

重启数据库

```bash
$ systemctl restart mysqld.service
```

查看binlog状态信息。记住mysql-bin文件名和Position的值，配置从节点半同步复制时需要用到。

```bash
mysql> show master status;
File                Position
mysql-bin.000002	121
```

## 配置从节点
安装半同步插件

```sql
mysql> install plugin rpl_semi_sync_slave SONAME 'semisync_slave.so';
```

配置全局参数

```sql
mysql> set global rpl_semi_sync_slave_enabled = 1;
```

修改数据库配置文件。在`/etc/my.cnf`文件中的`[mysqld]`下加入以下内容，保存然后重启数据库服务。

```bash
port=3306
log-bin=mysql-bin
server-id=2  # 与主节点不能相同

rpl_semi_sync_slave_enabled=1
```

重启数据库

```bash
$ systemctl restart mysqld.service
```

配置半同步主节点。`MASTER_HOST`为主节点IP，`MASTER_LOG_FILE`和`MASTER_LOG_POS`请查看配置主节点时`show master status`命令的输出信息。

```sql
mysql> CHANGE MASTER TO
> MASTER_HOST='172.25.223.34',
> MASTER_USER='repl',
> MASTER_PASSWORD='123456',
> MASTER_PORT=3306,
> MASTER_LOG_FILE='mysql-bin.000002',
> MASTER_LOG_POS=121,
> MASTER_CONNECT_RETRY=10;
```

开启同步并查看从节点状态

```sql
mysql> start slave;
mysql> show slave status\G;
```

如果从节点上连接主数据失败，可能需要关闭主从服务器上的防火墙。

```bash
$ systemctl stop firewalld  # 暂时关闭
$ systemctl disable firewalld  # 永久关闭
```

## 半同步测试
在主节点创建按数据库和数据表并插入数据

```sql
mysql> create database test_db;
mysql> use test_db;
mysql> create table person(id int not null auto_increment, 
> name varchar(8), 
> height float not null, 
> weight int not null, 
> constraint pk_person primary key(id));

mysql> show tables;
mysql> insert into person(name, height, weight) 
> values('Lebron', 2.06, 113);
mysql> insert into person(name, height, weight) 
> values('Curry', 1.91, 84);
mysql> insert into person(name, height, weight) 
> values('Kyrie', 1.88, 88);
```

登录从节点查看同步的信息

```sql
mysql> use test_db;
mysql> select * from person;
```


References:
[1\] https://blog.csdn.net/hansionz/article/details/106061919
[2\] https://www.cnblogs.com/ding2016/p/9531624.html
