@[TOC](【MySQL基础篇】数据导出导入权限与local_infile参数)

# 问题背景
MySQL高可用集群架构中，应用需要使用`select ... into outfile`和`load data [local] infile`来进行数据导入导出操作。其中，数据导出（只涉及读操作）发生在只读Slave节点，通过localhost连入数据库；数据导入（涉及读写操作）发生在Master节点，通过集群vip连入数据库。

| DB-Master (DB02) | DB-Slave (DB01) | vip |
|--|--|--|
|  A.B.C.120 |  A.B.C.119 |  A.B.C.121 |

涉及数据导入导出的两个参数的当前生效值如下：

```sql
secure_file_priv='' # 表示不限制数据导出的目录
local_infile=OFF  # 表示不允许使用load data local infile从客户端导入数据
```

# 数据导出测试
## 创建测试库（在主库进行）
```sql
--- DB-Master: A.B.C.120
[root@DB02 tmp]# mysql -uroot -p
Password:
mysql> create user 'apptest'@'%' identified by 'appPasswd';

mysql> create database apptest;
mysql> use apptest;
mysql> create table `test01` (id int not null,
    -> name varchar(20),
    -> country varchar(12),
    -> primary key pk_id(`id`)
    -> ) engine=innodb;

--- 此处省略插入数据的过程
mysql> select * from apptest.test01;
+----+---------------+---------+
| id | name          | country |
+----+---------------+---------+
|  1 | Gu Eileen     | CN      |
|  2 | Lebron James  | USA     |
|  3 | Karim Benzema | FR      |
+----+---------------+---------+

--- 此处省略用户授权语句
mysql> show grants for 'apptest'@'%';
+------------------------------------------------------+
| Grants for apptest@%                                 |
+------------------------------------------------------+
| GRANT USAGE ON *.* TO 'apptest'@'%'                  |
| GRANT ALL PRIVILEGES ON `apptest`.* TO 'apptest'@'%' |
+------------------------------------------------------+
```


## 测试数据导出（在从库进行）
登录从库导出数据：
```sql
--- DB-Slave: A.B.C.119 
[root@DB01 tmp]# mysql -uapptest -pappPasswd
mysql> select * from test01 into outfile '/tmp/apptest.txt';
ERROR 1045 (28000): Access denied for user 'apptest'@'%' (using password: YES)
```

可以看到，数据库报错“访问被拒绝”，应该是缺少了数据导出相关的权限。

登录**主库**为应用用户添加相关权限：
```sql
--- DB-Master: A.B.C.120
[root@DB02 tmp]# mysql -hA.B.C.121 -uroot -p
Password:
mysql> grant file on *.* to 'apptest'@'%';
```

再次登录从库导出数据：
```sql
--- DB-Slave: A.B.C.119
[root@DB01 tmp]# mysql -uapptest -pappPasswd
mysql> select * from apptest.test01 into outfile '/tmp/apptest.txt';
Query OK, 3 rows affected (0.01 sec)
```
数据导出成功。

```bash
[root@DB01 tmp]# ll /tmp
total 8
-rw-rw-rw- 1 mysql mysql 53 Feb 17 12:20 apptest.txt

[root@DB01 tmp]# cat /tmp/apptest.txt
1	Gu Eileen	CN
2	Lebron James	USA
3	Karim Benzema	FR
```
可以看到，导出文件的属主为mysql。


假设我们要向应用家目录`/home/apptest`导出数据：
```bash
[root@DB01 tmp]# ll /home | grep apptest
drwxr-xr-x  2 apptest  apptest  62 Feb 17 12:24 apptest
```

```sql
mysql> select * from apptest.test01 into outfile '/home/apptest/apptest.txt';
ERROR 1 (HY000): Can't create/write to file '/home/apptest/apptest.txt' (Errcode: 13 - Permission denied)
```
可以看到，数据库报错“无法创建或写入文件”，原因是mysql用户对应用用户的家目录没有读写权限。

我们尝试给mysql用户添加应用子目录的读写权限：
```bash
[root@DB01 tmp]# setfacl -R -m u:mysql:rwx /home/apptest 
[root@DB01 tmp]# setfacl -R -d -m u:mysql:rwx /home/apptest
```

导出数据到应用子目录：
```sql
mysql> select * from apptest.test01 into outfile '/home/apptest/apptest.txt';
Query OK, 3 rows affected (0.00 sec)
```

## 测试数据导入（在主库进行）
准备好要导入数据库的文件
```bash
# DB-Master: A.B.C.120
[apptest@DB02 tmp]$ ll test02.txt
-rw-r--r-- 1 apptest apptest 52 Feb 17 12:36 test02.txt
[apptest@DB02 tmp]$ cat test02.txt
1	Yao Ming	CN
2	Kobe Bryant	USA
3	Stephen Curry	USA
```

创建要导入的空表：
```sql
[apptest@DB02 tmp]$ mysql -hA.B.C.121 -uroot -p
mysql> use apptest
mysql> create table `test02` (id int not null,
    name varchar(20),
    country varchar(12),
    primary key pk_id(`id`)
    ) engine=innodb;
```

分别尝试在有无**Local**关键字的情况下导入数据：
```sql
mysql> load data local infile '/tmp/test02.txt' into table test02;
ERROR 1148 (42000): The used command is not allowed with this MySQL version

mysql> load data infile '/tmp/test02.txt' into table test02;
Query OK, 3 rows affected (0.00 sec)
Records: 3  Deleted: 0  Skipped: 0  Warnings: 0

mysql> select * from apptest.test02;
+----+---------------+---------+
| id | name          | country |
+----+---------------+---------+
|  1 | Yao Ming      | CN      |
|  2 | Kobe Bryant   | USA     |
|  3 | Stephen Curry | USA     |
+----+---------------+---------+

mysql> show variables like 'local_infile';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| local_infile  | OFF   |
+---------------+-------+
```
可见，由于设置了`local_infile=OFF`，只能使用`load data infile`导入数据，而不能使用`load data local infile`。

假设我们打开 **local_infile** 参数：
```sql
mysql> set global local_infile=ON;

mysql> create table `test03` (id int not null,
    name varchar(20),
    country varchar(12),
    primary key pk_id(`id`)
    ) engine=innodb;
    
mysql> load data local infile '/tmp/test02.txt' into table test03;
Query OK, 3 rows affected (0.00 sec)
Records: 3  Deleted: 0  Skipped: 0  Warnings: 0
 
mysql> set global local_infile=OFF;  # 出于安全考虑一般关闭
```
可见确实是`local_infile`参数影响了数据的导入操作。

MySQL配置文件中也可以指定该参数。
```bash
[root@DB02 ~]# cat /etc/my.cnf | grep local-inf
local-infile=0
```


