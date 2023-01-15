---
tags: [oracle]
title: Oracle中的双引号与单引号
created: '2023-01-15T06:39:57.954Z'
modified: '2023-01-15T07:49:20.503Z'
---

Oracle中的双引号与单引号

# 场景一：数据库对象名称
创建对象时，**对象名称**可以加双引号，**不能加单引号**。加双引号表示**区分大小写**，不加双引号表示**默认大写**。

## Example 1：创建表空间
```sql
SQL> create tablespace omf_tbs1;
Tablespace created.

SQL> create tablespace "omf_tbs2";
Tablespace created.

SQL> select tablespace_name from dba_tablespaces;

TABLESPACE_NAME
------------------------------
SYSTEM
SYSAUX
UNDOTBS1
TEMP
USERS
OMF_TBS1
omf_tbs2

SQL> create tablespace 'omf_tbs3';
create tablespace 'omf_tbs3'
                  *
ERROR at line 1:
ORA-02216: tablespace name expected
```

## Example 2：创建用户及授权
```sql
SQL> create user miguel identified by "Xqc$689" default tablespace omf_tbs1;
User created.

SQL> create user "pablo" identified by Milf377 default tablespace omf_tbs2;
create user "pablo" identified by Milf377 default tablespace omf_tbs2
*
ERROR at line 1:
ORA-00959: tablespace 'OMF_TBS2' does not exist

SQL> create user "pablo" identified by Milf377 default tablespace "omf_tbs2";
User created.

SQL> select username from dba_users where username like 'MIGUEL';
USERNAME
--------------------------------------------------------------------------------
MIGUEL

SQL> select username from dba_users where username like 'pablo';
USERNAME
--------------------------------------------------------------------------------
pablo

SQL> create user 'Phoebe' identified by "Pwd3457";
create user 'Phoebe' identified by "Pwd3457"
            *
ERROR at line 1:
ORA-01935: missing user or role name
```

给用户授权的情况与上面类似，大写的用户名可以加也可以不加双引号，小写的用户名要加双引号。但是不能给用户名加单引号。
```sql
SQL> grant create session to miguel;
Grant succeeded.

SQL> grant resource,connect to "MIGUEL";
Grant succeeded.

SQL> grant create session to pablo;
grant create session to pablo
                        *
ERROR at line 1:
ORA-01917: user or role 'PABLO' does not exist

SQL> grant create session to "pablo";
Grant succeeded.

SQL> grant resource,connect to 'pablo';
grant resource,connect to 'pablo'
                          *
ERROR at line 1:
ORA-00987: missing or invalid username(s)
```

## Example 3：用户登录
对于小写的用户名，登录时要加双引号。
```sql
SQL> conn miguel/Xqc$689
Connected.

SQL> conn MIGUEL/Xqc$689
Connected.

SQL> conn pablo/Milf377
ERROR:
ORA-01017: invalid username/password; logon denied

Warning: You are no longer connected to ORACLE.
SQL> conn "pablo"/Milf377
Connected.
```

使用SQLPlus在终端登录时，注意对用户名和密码中的特殊字符进行**转义**（比如引号、`$`等）。
```sql
[oracle@oracledb ~]$ sqlplus miguel/Xqc$689

ERROR:
ORA-01017: invalid username/password; logon denied

[oracle@oracledb ~]$ sqlplus miguel/Xqc\$689   -- 这里$前有一个转义符\
Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

[oracle@oracledb ~]$ sqlplus "pablo"/Milf377

ERROR:
ORA-01017: invalid username/password; logon denied

[oracle@oracledb ~]$ sqlplus \"pablo\"/Milf377
Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
```

# 场景二：用户密码
创建用户时，密码可以也可以不加双引号。不管加不加双引号，密码都区分大小写。**密码不能加单引号**。
```sql
SQL> create user miguel identified by "Xqc$689" default tablespace omf_tbs1;
User created.

SQL> create user "pablo" identified by Milf377 default tablespace "omf_tbs2";
User created.

SQL> create user phoebe identified by 'Jojo666';
create user phoebe identified by 'Jojo666'
                                 *
ERROR at line 1:
ORA-00988: missing or invalid password(s)
```


# 场景三：字段（列）名称
对于列名称，不能加双引号；如果加了单引号，字段名称会被转化成**纯字符串**。
```sql
SQL> select tablespace_name from dba_tablespaces;
TABLESPACE_NAME
------------------------------
SYSTEM
SYSAUX
UNDOTBS1
TEMP
USERS
OMF_TBS1
omf_tbs2

7 rows selected.

SQL> select "TBALESPACE_NAME" from dba_tablespaces;
select "TBALESPACE_NAME" from dba_tablespaces
       *
ERROR at line 1:
ORA-00904: "TBALESPACE_NAME": invalid identifier

SQL> select 'TBALESAPCE_NAME' from dba_tablespaces;
'TBALESAPCE_NAME'
----------------
TBALESAPCE_NAME
TBALESAPCE_NAME
TBALESAPCE_NAME
TBALESAPCE_NAME
TBALESAPCE_NAME
TBALESAPCE_NAME
TBALESAPCE_NAME

7 rows selected.

SQL> select sysdate from dual;
SYSDATE
---------
12-JAN-23

SQL> select "SYSDATE" from dual;
ERROR:
ORA-01741: illegal zero-length identifier

SQL> select 'sysdate' from dual;
'SYSDATE'
---------
sysdate
```

# 场景四：字段（列）的值
对于字段的值，**必须加单引号**，并且区分大小写。

**示例1**
```sql
SQL> select username from dba_users where username like "MIGUEL";
select username from dba_users where username like "MIGUEL"
                                                   *
ERROR at line 1:
ORA-00904: "MIGUEL": invalid identifier

SQL> select username from dba_users where username like MIGUEL;
select username from dba_users where username like MIGUEL
                                                   *
ERROR at line 1:
ORA-00904: "MIGUEL": invalid identifier

SQL> select username from dba_users where username like 'MIGUEL';
USERNAME
--------------------------------------------------------------------------------
MIGUEL

SQL> select username from dba_users where username like 'pablo';
USERNAME
--------------------------------------------------------------------------------
pablo
```

**示例2**
```sql
SQL> select username from dba_users where username="pablo";
select username from dba_users where username="pablo"
                                              *
ERROR at line 1:
ORA-00904: "pablo": invalid identifier

SQL> select username from dba_users where username=pablo;
select username from dba_users where username=pablo
                                              *
ERROR at line 1:
ORA-00904: "PABLO": invalid identifier

SQL> select username from dba_users where username='pablo';
USERNAME
--------------------------------------------------------------------------------
pablo

SQL> select username from dba_users where username='PABLO';
no rows selected
```

****
:fish:总的来说，单双引号的使用大致满足以下规则：
- 对于对象名称（例如用户名、表空间名），不能使用单引号（因为会被转化为纯字符串）；可以使用双引号，此时区分大小写。
- 对于用户密码，不能使用单引号；可以使用双引号，不管加不加双引号，都区分大小写。
- 对于字段（列）的名称，不能使用双引号；如果使用单引号，则会被转化为纯字符串。
- 对于字段（列）的值，必须加单引号，并且区分大小写。



