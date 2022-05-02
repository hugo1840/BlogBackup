@[TOC](【MySQL基础篇】启动选项与配置文件)

# mysqld启动选项
`mysqld` 即 MySQL Server，是在MySQL数据库服务安装初始化时执行的程序。执行mysqld初始化数据库时，可以指定服务器启动选项（**server options**）。

```bash
# 查看所有的启动选项
$ mysqld --verbose --help
abort-slave-event-count           0
allow-suspicious-udfs             FALSE
archive                           ON
auto-increment-increment          1
auto-increment-offset             1
autocommit                        TRUE
automatic-sp-privileges           TRUE
avoid-temporal-upgrade            FALSE
back-log                          80
basedir                           /home/jon/bin/mysql-5.7/
...
tmpdir                            /tmp
transaction-alloc-block-size      8192
transaction-isolation             REPEATABLE-READ
transaction-prealloc-size         4096
transaction-read-only             FALSE
transaction-write-set-extraction  OFF
updatable-views-with-limit        YES
validate-user-plugins             TRUE
verbose                           TRUE
wait-timeout                      28800
```


# 配置文件：my.cnf
由于MySQL Server启动选项很多，一般不直接写在命令行里，而是统一写到配置文件里，然后在初始化时指定该文件读取启动选项。

```bash
$ mysqld --defaults-file=/etc/my.cnf --initialize
```

## 配置文件的优先级
在类Unix操作系统中，MySQL会按照下表所示顺序由上至下依次寻找启动时所需的配置文件。

<center><font color=red> 表1 MySQL配置文件的优先级</font></center>

| 文件名 | 作用 |
|--|--|
| /etc/my.cnf | 全局启动项 |
| /etc/mysql/my.cnf	 | 全局启动项 |
| SYSCONFDIR/my.cnf	 | 全局启动项 |
| $MYSQL_HOME/my.cnf	 | 服务端的启动项（仅限服务端） |
| defaults-extra-file	 | 命令行中-\-defaults-extra-file后指定的额外配置文件 |
| ~/.my.cnf	 | 特定于当前用户的启动项 |
| ~/.mylogin.cnf	 | 客户端登录路径启动项（仅限客户端） |


其中，SYSCONFDIR是指在用CMake build MySQL时该参数指定的路径，默认是已编译安装目录下的`etc`目录。

>**注意**：如果一个启动项在以上配置文件中被重复定义了多次，服务启动后生效的值会是MySQL**最后**找到的值。唯一的例外是**user**这个启动项，出于安全考虑，MySQL找到的第一个user的值会生效。


例如，假设 `/etc/my.cnf` 和 `~/.my.cnf` 中都定义了存储引擎的启动项，那么将以后一个中定义的值为准。

```bash
# vim /etc/my.cnf
default-storage-engine=InnoDB

# vim ~/.my.cnf
default-storage-engine=MyISAM
```

即MySQL服务启动后默认的存储引擎为MyISAM。


## 配置文件的格式
配置文件中的启动项每行只能写一个。有的启动项需要指定值，有的不需要。其格式与命令行中相比，一般只需要去掉前面的两个短横线 `-- `。配置文件中用 `#` 注释。

使用 `[group]` 可以单独为不同的组或程序指定启动项。

```bash
[client]
port=3306
socket=/tmp/mysql.sock

[mysqld]
port=3306
socket=/tmp/mysql.sock
key_buffer_size=16M
max_allowed_packet=8M

[mysqldump]
quick
```

下面的表格给出了不同的MySQL程序能够读取的组的启动项。

<center><font color=red> 表2 MySQL程序可读取的配置文件组</font></center>

| 程序名 | 类别 | 能读取的组 |
|--|--|--|
| mysqld | 服务端 | [mysqld], [server] |
| mysqld_safe | 服务端 | [mysqld], [server], [mysqld_safe] |
| mysql.server | 服务端 | [mysqld], [server], [mysql.server] |
| mysql | 客户端 | [mysql], [client] |
| mysqladmin | 客户端 | [mysqladmin], [client] |
| mysqldump | 客户端 | [mysqldump], [client] |


我们也可以在配置文件组中加上版本后来限定启动项应用的数据库版本。

```bash
[mysqld-5.7]
sql_mode=TRADITIONAL
```







**References**
【1】https://dev.mysql.com/doc/refman/5.7/en/mysqld.html
【2】https://dev.mysql.com/doc/refman/5.7/en/mysqld-server.html
【3】https://dev.mysql.com/doc/refman/5.7/en/server-configuration.html
【4】https://dev.mysql.com/doc/refman/5.7/en/option-files.html
【5】https://dev.mysql.com/doc/refman/8.0/en/server-option-variable-reference.html
【6】https://dev.mysql.com/doc/refman/5.7/en/binary-log.html
