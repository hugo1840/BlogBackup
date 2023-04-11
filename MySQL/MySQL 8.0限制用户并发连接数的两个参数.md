---
tags: [mysql]
title: MySQL 8.0限制用户并发连接数的两个参数
created: '2023-04-11T12:27:08.667Z'
modified: '2023-04-11T13:30:10.375Z'
---

MySQL 8.0限制用户并发连接数的两个参数

# max_connections

针对所有用户总体而言，MySQL Server允许的最大并发客户端连接数，默认值为151，可以根据服务器配置适当调大到1000。

最大并发连接数`max_connections`的实际生效值等于它的配置值与`open_files_limit`配置值两者中的较小值。`open_files_limit`参数定义了mysqld进程能够使用的操作系统文件描述符的数量。

最大并发连接数`max_connections`为全局系统变量。

修改最大并发连接数`max_connections`：
```sql
set global max_connections = 1000;
set @@global.max_connections = 1000;
```

修改最大并发连接数`max_connections`的运行时值，并将其持久化到`mysqld-auto.cnf`文件：
```sql
set persist max_connections = 1000;
set @@persist.max_connections = 1000;
```

# max_user_connections

针对单个MySQL用户而言，所允许的最大并发连接数。默认值为0，表示没有限制。

用户最大并发连接数`max_user_connections`既有全局级别的参数值，也有会话级别的参数值。全局级别的`max_user_connections`可以动态修改。会话级别的`max_user_connections`为只读变量，其生效值通过以下方式来确定：

1. 如果为用户设置的最大并发连接数`max_user_connections`的值不为0，那么该值即为会话级别`max_user_connections`的生效值；

2. 如果为用户设置的最大并发连接数`max_user_connections`的值为0，那么会话级别`max_user_connections`的生效值即为全局级别的配置值。

修改用户的最大并发连接数`max_user_connections`：
```sql
alter user xxx with max_user_connections 0;
```

修改全局级别的用户最大并发连接数`max_user_connections`：
```sql
set global max_user_connections = 0;
```

:sunflower:常用查询命令：
```sql
--查看用户的并发连接数限制
select user,host,max_connections,max_user_connections from mysql.user;

--查看全局级别的用户最大并发连接数
show global variables like 'max_user_connections';

--查看会话级别的用户最大并发连接数
show session variables like 'max_user_connections';
```


**References**

[1] https://dev.mysql.com/doc/refman/8.0/en/user-resources.html

[2] https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_max_user_connections

[3] https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_max_connections

[4] https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_open_files_limit





