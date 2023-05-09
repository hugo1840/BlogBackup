---
tags: [oracle]
title: 私有DBLINK授予其他用户访问权限的问题
created: '2023-04-29T08:53:03.806Z'
modified: '2023-05-09T14:05:03.401Z'
---

私有DBLINK授予其他用户访问权限的问题

# 检查目标库TNS配置
检查远程目标库的TNS配置：
```bash
[oracle@remotedb ~]$ cat $ORACLE_HOME/network/admin/tnsnames.ora

REMOTEDB = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = remotedb_hostname)(PORT = 1521)) 
    (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = remotedb) ) 
  )
```

在远程目标库上创建测试用户dbtuser3：
```sql
SQL> grant connect,create table,create view,create synonym,create database link to dbtuser3 identified by Password33; 
```

在远程目标库上创建测试表：
```sql
[oracle@remotedb ~]$ sqlplus dbtuser3/Password33 

SQL> create table report ( 
     stuid number(2) not null, 
     sname varchar2(20) not null );

SQL> insert into report (stuid,sname) values (1,'RED Dead 2'); 
insert into report (stuid,sname) values (2,'The Witcher 3'); 
insert into report (stuid,sname) values (3,'Elden Ring');

SQL> select * from dbtuser3.report;

 STUID SNAME
 1 RED Dead 2
 2 The Witcher 3
 3 Elden Ring
```

# 本地库创建DBLINK
在本地库上创建DBLINK测试用户，并授予创建私有DBLINK的权限：
```sql
SQL> grant connect,create view,create synonym,create database link to dbtuser1 identified by Password11; 
SQL> grant connect,create view,create synonym,create database link to dbtuser2 identified by Password22; 
```

在本地库上配置远程库的hosts解析：
```bash
[oracle@localdb ~]$ echo '172.69.x.x remotedb_hostname' >> /etc/hosts 
[oracle@localdb ~]$ ping remotedb_hostname
```

在本地库上配置远程库的TNS解析：
```bash
[oracle@localdb ~]$ cat $ORACLE_HOME/network/admin/tnsnames.ora

LOCALDB = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = localdb_hostname)(PORT = 1521)) 
    (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = localdb) ) 
  )

REMOTEDB = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = remotedb_hostname)(PORT = 1521)) 
    (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = remotedb) ) 
  )

# 检查TNS解析
[oracle@localdb ~]$ tnsping REMOTEDB 
```

使用`dbtuser1`用户创建私有DBLINK：
```sql
[oracle@localdb ~]$ sqlplus dbtuser1/Password11 

SQL> create database link testlink1 
  connect to dbtuser3 identified by "Password33" 
  using '
  REMOTEDB = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = remotedb_hostname)(PORT = 1521)) 
    (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = remotedb) ) 
  )
  ';

--检查DBLINK信息
SQL> select db_link,username,host from user_db_links;

SQL> select * from DBTUSER3.REPORT@TESTLINK1; 
select * from DBTUSER3.REPORT@TESTLINK1 * 
ERROR at line 1: 
ORA-12154: TNS:could not resolve the connect identifier specified
```
使用DBLINK时报ORA-12154 TNS解析错误。

检查`SQLNET.ORA`里面`NAMES.DIRECTORY_PATH`中是否包含了TNSNAMES：
```bash
[oracle@localdb ~]$ cat $ORACLE_HOME/network/admin/sqlnet.ora | grep NAMES

NAMES.DIRECTORY_PATH= (TNSNAMES, EZCONNECT)  # 这里没问题
```

重新创建DBLINK，修改dblink名称与`TNSNAMES.ORA`里一致，并去掉DESCRIPTION前面的连接名:
```sql
[oracle@localdb ~]$ sqlplus dbtuser1/Password11 

SQL> drop database link testlink1;

SQL> create database link REMOTEDB 
  connect to dbtuser3 identified by "Password33" 
  using ' 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = remotedb_hostname)(PORT = 1521)) 
    (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = remotedb) ) 
  )
  ';
```

再次检查DBLINK是否可用：
```sql
SQL> select * from DBTUSER3.REPORT@REMOTEDB;

no rows selected

SQL> insert into DBTUSER3.REPORT@REMOTEDB (stuid,sname) values (4,'GTA V');

1 row created.

SQL> select * from DBTUSER3.REPORT@REMOTEDB;

 STUID SNAME
 4 GTA V
```
看上去DBLINK可以连上了，但是奇怪的是查不到远程库表里的数据。向DBLINK插入数据成功了，并且只能看到在本地库插入的数据。


登录DBLINK远程库remotedb查询，也没有从localdb新插入的行。 
```sql
SQL> select * from DBTUSER3.REPORT;

 STUID SNAME
 1 RED Dead 2
 2 The Witcher 3
 3 Elden Ring
```

过了几分钟后，再次查看本地库，又突然可以查到了！！！
```sql
SQL> select * from DBTUSER3.REPORT@REMOTEDB;

 STUID SNAME
 4 GTA V
 1 RED Dead 2
 2 The Witcher 3
 3 Elden Ring

SQL> commit;

Commit complete.
```

再次登录DBLINK远程库remotedb检查，此时远程库也能看到了！！！
```sql
SQL> select * from report;

 STUID SNAME
 4 GTA V
 1 RED Dead 2
 2 The Witcher 3
 3 Elden Ring
```

# 授予私有DBLINK访问权限给其他用户

在本地库使用用户`dbtuser1`为私有DBLINK创建测试视图和同义词，并将视图的查看权限授予用户`dbtuser2`。
```sql
[oracle@localdb ~]$ sqlplus dbtuser1/Password11

SQL> create view v_testvc1 as 
select sname from DBTUSER3.REPORT@REMOTEDB where stuid < 4;

SQL> create view v_testvc2 as 
select * from DBTUSER3.REPORT@REMOTEDB;

SQL> create synonym report_table for DBTUSER3.REPORT@REMOTEDB;

SQL> grant select on v_testvc1 to dbtuser2;
SQL> grant select on v_testvc2 to dbtuser2;
```

切换用户`dbtuser2`测试：
```sql
[oracle@localdb ~]$ sqlplus dbtuser2/Password22

SQL> select * from DBTUSER3.REPORT@REMOTEDB; 
select * from DBTUSER3.REPORT@REMOTEDB * 
ERROR at line 1:
ORA-02019: connection description for remote database not found
--> 直接访问DBLINK失败

SQL> select * from dbtuser1.report_table; 
select * from dbtuser1.report_table * 
ERROR at line 1: 
ORA-02019: connection description for remote database not found
--> 访问同义词失败

SQL> select * from dbtuser1.v_testvc1;

SNAME
GTA V
RED Dead 2 
The Witcher 3 
Elden Ring

SQL> select * from dbtuser1.v_testvc2;

 STUID SNAME
 4 GTA V
 1 RED Dead 2
 2 The Witcher 3
 3 Elden Ring
``` 

即：可以通过创建视图来将私有DBLINK访问权限授予其他用户，但是不能通过直接为私有DBLINK创建同义词来让其他用户使用。






