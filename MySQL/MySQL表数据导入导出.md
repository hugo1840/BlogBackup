
@[TOC](MySQL表数据导入导出)

`select into outfile`和`load data infile`是MySQL中常用的表数据导出导入的方法。由于不能导出表结构信息，因此一般不用于备份与恢复。

# :apple: 用法
```sql
---导出数据
SELECT ... INTO OUTFILE 'file_name';

---导入数据
LOAD DATA
    [LOW_PRIORITY | CONCURRENT] [LOCAL]
    INFILE 'file_name'
    [REPLACE | IGNORE]
    INTO TABLE tbl_name
    [PARTITION (partition_name [, partition_name] ...)]
    [CHARACTER SET charset_name]
    [{FIELDS | COLUMNS}
        [TERMINATED BY 'string']
        [[OPTIONALLY] ENCLOSED BY 'char']
        [ESCAPED BY 'char']
    ]
    [LINES
        [STARTING BY 'string']
        [TERMINATED BY 'string']
    ]
    [IGNORE number {LINES | ROWS}]
    [(col_name_or_user_var
        [, col_name_or_user_var] ...)]
    [SET col_name={expr | DEFAULT}
        [, col_name={expr | DEFAULT}] ...];
```

# :snake: 注意事项
1. 用户权限要求

导数的用户必须具有FILE权限。授予该权限的语句为：
```sql
mysql> grant file on *.* to 'appuser'@'%'; 
```

2. `secure_file_priv`参数影响

该参数有以下三种情况：
- `secure_file_priv=NULL`：表示不允许导出和导入数据；
- `secure_file_priv='/tmp'`：表示只允许导出数据到指定目录、且只允许从指定目录导入数据（这里是/tmp目录）；
- `secure_file_priv=''`：表示对数据导出和导入不做任何限制。

需要注意的是，该参数为只读变量，无法动态修改。
```sql
mysql> set global secure_file_priv='/tmp';
ERROR 1238 (HY000): Variable 'secure_file_priv' is a read only variable
```

只能通过修改配置文件后重启MySQL服务来改变。
```bash
[root@mysql-node1 ~]# vim /etc/my.cnf
[mysqld]
secure-file-priv='/tmp'

[root@mysql-node1 ~]# systemctl restart mysqld

[root@mysql-node1 ~]# mysql -uroot -p
mysql> select @@secure_file_priv;
+--------------------+
| @@secure_file_priv |
+--------------------+
| /tmp/              |
+--------------------+
1 row in set (0.00 sec)
```

3. `local_infile`参数影响

使用`load data [local] infile`导入数据时，如果外部数据在MySQL服务端，无需指定**LOCAL**关键字；反之，如果要导入的数据在客户端服务器上，则必须指定**LOCAL**关键字。

但是，使用LOCAL关键字还受到参数`local_infile`的影响。如果`local_infile=0`，则不允许使用LOCAL关键字从MySQL客户端导入数据。

```sql
mysql> select @@local_infile;
+----------------+
| @@local_infile |
+----------------+
|              0 |
+----------------+
1 row in set (0.00 sec)

mysql> load data local infile '/tmp/players_0909.txt' into table players fields terminated by ',';
ERROR 1148 (42000): The used command is not allowed with this MySQL version
```

幸运的是，该参数可以动态修改，无需重启MySQL服务。
```sql
mysql> set global local_infile=1;
Query OK, 0 rows affected (0.00 sec)

mysql> select @@local_infile;
+----------------+
| @@local_infile |
+----------------+
|              1 |
+----------------+
1 row in set (0.00 sec)
```

# :eagle: 示例

- 导出数据
```sql
mysql> select @@secure_file_priv;
+--------------------+
| @@secure_file_priv |
+--------------------+
| /tmp/              |
+--------------------+
1 row in set (0.00 sec)

mysql> select * from players into outfile '/root/players_0909.txt' fields terminated by ',';
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
mysql>
mysql> select * from players into outfile '/tmp/players_0909.txt' fields terminated by ',';
Query OK, 6 rows affected (0.00 sec)
```

- 导入数据
```sql
---truncate清空表的数据，但是会保留表结构
mysql> truncate table players;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from players;
Empty set (0.00 sec)

mysql> load data infile '/tmp/players_0909.txt' into table players fields terminated by ',';
Query OK, 6 rows affected (0.00 sec)
Records: 6  Deleted: 0  Skipped: 0  Warnings: 0

mysql> select count(*) from players;
+----------+
| count(*) |
+----------+
|        6 |
+----------+
1 row in set (0.01 sec)
```


**References**
【1】https://dev.mysql.com/doc/refman/5.7/en/select-into.html
【2】https://dev.mysql.com/doc/refman/5.7/en/load-data.html#load-data-local


