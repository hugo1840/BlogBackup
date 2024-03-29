---
tags: [oracle]
title: Oracle限制单个用户的并发连接数
created: '2023-05-10T13:29:22.658Z'
modified: '2023-05-10T13:53:35.372Z'
---

Oracle限制单个用户的并发连接数

# 开启RESOURCE_LIMIT参数
检查资源限制是否开启： 
```sql
SQL> show parameter resource_limit

NAME TYPE VALUE
---- ---- -----
resource_limit boolean TRUE
```
这个参数一般是默认开启的，如果没有开启就需要通过`ALTER SYSTEM`命令来开启。


# 查看对用户的资源限制
Oracle数据库通过指定用户的Profile来对用户资源进行限制。Profile是对数据库资源使用约束条件的一个集合。

一般用户默认的Profile为**DEFAULT**：
```sql
SQL> select profile from dba_users where username='APPUSER';

PROFILE
-------
DEFAULT
```

Default Profile不会对用户使用数据库资源做任何限制：
```sql
SQL> select resource_name,resource_type,limit from dba_profiles 
where profile='DEFAULT';

RESOURCE_NAME RESOURCE LIMIT
------------- -------- -----
COMPOSITE_LIMIT KERNEL UNLIMITED 
SESSIONS_PER_USER KERNEL UNLIMITED 
CPU_PER_SESSION KERNEL UNLIMITED 
CPU_PER_CALL KERNEL UNLIMITED 
LOGICAL_READS_PER_SESSION KERNEL UNLIMITED 
LOGICAL_READS_PER_CALL KERNEL UNLIMITED 
IDLE_TIME KERNEL UNLIMITED 
CONNECT_TIME KERNEL UNLIMITED 
PRIVATE_SGA KERNEL UNLIMITED 
FAILED_LOGIN_ATTEMPTS PASSWORD UNLIMITED 
PASSWORD_LIFE_TIME PASSWORD UNLIMITED
PASSWORD_REUSE_TIME PASSWORD UNLIMITED 
PASSWORD_REUSE_MAX PASSWORD UNLIMITED 
PASSWORD_VERIFY_FUNCTION PASSWORD NULL 
PASSWORD_LOCK_TIME PASSWORD 1 
PASSWORD_GRACE_TIME PASSWORD UNLIMITED 
INACTIVE_ACCOUNT_TIME PASSWORD UNLIMITED 
PASSWORD_ROLLOVER_TIME PASSWORD -1

18 rows selected.
``` 

# 限制用户的并发连接数
创建一个限制并发连接数上限为500的Profile： 
```sql
SQL> create profile cur_sess_profile limit sessions_per_user 500;
```
未指定限制的其他资源会采用默认的DEFAULT Profile。

查看该Profile对应的资源限制条件：
```sql
SQL> select profile,resource_name,limit from dba_profiles 
where profile='CUR_SESS_PROFILE';

PROFILE RESOURCE_NAME LIMIT
------- ------------- -----
CUR_SESS_PROFILE COMPOSITE_LIMIT DEFAULT 
CUR_SESS_PROFILE SESSIONS_PER_USER 500 
CUR_SESS_PROFILE CPU_PER_SESSION DEFAULT 
CUR_SESS_PROFILE CPU_PER_CALL DEFAULT 
CUR_SESS_PROFILE LOGICAL_READS_PER_SESSION DEFAULT 
CUR_SESS_PROFILE LOGICAL_READS_PER_CALL DEFAULT 
CUR_SESS_PROFILE IDLE_TIME DEFAULT 
CUR_SESS_PROFILE CONNECT_TIME DEFAULT 
CUR_SESS_PROFILE PRIVATE_SGA DEFAULT 
CUR_SESS_PROFILE FAILED_LOGIN_ATTEMPTS DEFAULT 
CUR_SESS_PROFILE PASSWORD_LIFE_TIME DEFAULT
CUR_SESS_PROFILE PASSWORD_REUSE_TIME DEFAULT 
CUR_SESS_PROFILE PASSWORD_REUSE_MAX DEFAULT 
CUR_SESS_PROFILE PASSWORD_VERIFY_FUNCTION DEFAULT 
CUR_SESS_PROFILE PASSWORD_LOCK_TIME DEFAULT 
CUR_SESS_PROFILE PASSWORD_GRACE_TIME DEFAULT 
CUR_SESS_PROFILE INACTIVE_ACCOUNT_TIME DEFAULT 
CUR_SESS_PROFILE PASSWORD_ROLLOVER_TIME DEFAULT

18 rows selected.
```

将该Profile定义的资源限制应用到指定用户： 
```sql
SQL> alter user APPUSER profile cur_sess_profile;
SQL> select profile from dba_users where username='APPUSER';
```




