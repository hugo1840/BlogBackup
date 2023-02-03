---
tags: [oracle]
title: Oracle用户授权及ORA-01950报错处理
created: '2023-02-03T10:53:08.319Z'
modified: '2023-02-03T11:21:03.539Z'
---

Oracle用户授权及ORA-01950报错处理


# 查询用户权限
```sql
--查询用户的系统权限
select grantee,privilege from dba_sys_privs where grantee='ETL_USER';

--查询用户对表的权限
select grantee,table_name,privilege from dba_tab_privs where grantee='ETL_USER';

--查询用户的角色
select * from dba_role_privs where grantee='ETL_USER';
```

# 用户授权
授予登陆权限：
```sql
grant create session,connect to etl_user;
```

授予读权限：
```sql
--授予查询任何表的权限
grant select any table to etl_user;

--授予查询任何数据字典（包括视图）的权限
grant select any dictionary to etl_user;
```

授予写权限：
```sql
--授予用户建表权限
grant create table to etl_user;

--授予对其他用户表的增删查改权限
grant insert,delete,update,select on TRADER.MONTHLY_SALES to etl_user;
```

# 表空间权限
检查用户默认表空间：
```sql
select default_tablespace,username from dba_users;
```

修改默认表空间：
```sql
--为用户创建同名表空间
create tablespace etl_user;

--查看用户下是否已经存在表
select owner,table_name,tablespace_name from dba_tables where owner='ETL_USER';

--修改用户默认表空间
alter user etl_user default tablespace etl_user;
```

对报错“ORA-01950：对表空间ETL_USER无权限”的处理办法：
```sql
--方法一：授予resource权限
grant resource to etl_user;

--方法二：修改用户的表空间配额
alter user etl_user quota unlimited on etl_user;
```


