@[TOC](【不积跬步无以至千里】MySQL报大量unauthenticated user连接错误)

# 问题背景
应用报MySQL数据库无法连接，经排查发现是连接数满了。通过`max_connections`参数将连接数上限调大到1000后，数据库开始大量报**unauthenticated user**错误。

```
: 343942 : unauthenticated user : 192.168.0.4:49607 : : Connect : : login :
: 343943 : unauthenticated user : 192.168.0.4:49608 : : Connect : : login :
...
...
```

# 问题分析
在网上查了一下，可能是由于数据库未配置`skip-name-resolve`参数，MySQL在验证应用连接时会解析主机名，如果连接太多且没有及时释放，就会导致MySQL“假死”，从而出现上面的情况。

MySQL官方对`skip-name-resolve`参数的描述如下：

>Whether to resolve host names when checking client connections. **If this variable is OFF, mysqld resolves host names when checking client connections. If it is ON, mysqld uses only IP numbers**; in this case, all Host column values in the grant tables must be IP addresses.

即，关闭该参数时，MySQL在检查客户端连接时会解析主机名；开启该参数时，MySQL只会检查IP。

检查了一下出问题的MySQL数据库的配置文件，里面确实没有`skip-name-resolve`这一行。也可以执行下面的语句查询，看看值是不是OFF：
```sql
show variables like 'skip_name_resolve';
```

MySQL官方对于应用连接数据库时DNS解析过程的描述如下：

>For each applicable new client connection, the server uses the client IP address to check whether the client host name is in the host cache. If so, the server refuses or continues to process the connection request depending on whether or not the host is blocked. If the host is not in the cache, the server attempts to resolve the host name. First, **it resolves the IP address to a host name and resolves that host name back to an IP address**. Then it compares the result to the original IP address to ensure that they are the same. The server stores information about the result of this operation in the host cache. If the cache is full, the least recently used entry is discarded.

即，数据库在验证新的应用连接时，会先将IP地址解析为主机名，再将主机名解析为IP，并和原始IP进行比对。如果DNS服务器出现问题，也有可能报unauthenticated user的错误。

# 问题处理
**方法一**
在MySQL配置文件中加上下面这一行，然后重启数据库。
```
skip-name-resolve
```

好处是能从根本上解决问题；缺点是以后给用户授权时host只能使用IP地址，不能使用主机名。

**方法二**
手动为每个应用连接配置hosts解析。比如Linux操作系统中，可以在`/etc/hosts`文件中添加对应的host记录。缺点是不适用于应用连接特别多的情况。

```
192.168.32.33  appserver01
192.168.32.34  appserver02
```



**References**：
【1】https://dev.mysql.com/doc/refman/5.7/en/host-cache.html
【2】https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html
