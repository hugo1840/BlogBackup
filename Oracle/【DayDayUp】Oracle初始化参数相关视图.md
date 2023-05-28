---
tags: [OCP, oracle]
title: 【DayDayUp】Oracle初始化参数相关视图
created: '2023-05-27T11:40:17.774Z'
modified: '2023-05-28T12:46:22.143Z'
---

【DayDayUp】Oracle初始化参数相关视图


下面四个与数据库初始化参数相关的视图，其中包含的参数大都可以出现在`init${ORACLE_SID}.ora`初始化文件中。

# v$parameter
当前**会话**（session）中生效的初始化参数值。
- `isses_modifiable`：是否可以通过**ALTER SESSION**语句改变当前会话中生效的参数值。
- `issys_modifiable`：是否可以通过**ALTER SYSTEM**语句改变参数值以及何时生效。
  - **IMMEDIATE**：可以通过**ALTER SYSTEM**语句改变参数值，并立即生效；
  - **DEFERRED**：可以通过**ALTER SYSTEM**语句改变参数值，并在后续新建的会话中生效（当前会话中不生效）；
  - **FALSE**：不可以通过**ALTER SYSTEM**语句改变参数值。但是如果实例是通过spfile启动的，可以通过**ALTER SYSTEM ... scope=spfile**来修改参数值，并在重启实例后生效。
- `ispdb_modifiable`：是否可以在可插拔数据库中（PDB）中修改参数值。
- `isinstance_modifiable`：同一个参数的值在RAC的两个实例中是否应该保持一致，还是可以不同。


**示例：**
```sql
sys@MARIO> select name,value,isses_modifiable,
issys_modifiable,ispdb_modifiable,isinstance_modifiable 
from v$parameter where name in ('processes','sessions');

NAME                           VALUE      ISSES ISSYS_MOD ISPDB ISINS
------------------------------ ---------- ----- --------- ----- -----
processes                      300        FALSE FALSE     FALSE FALSE
sessions                       472        FALSE IMMEDIATE TRUE  TRUE
```

# v$parameter2
与**v$parameter**包含的参数内容一致。只不过对于参数值为列表的，将该参数的每一个值都分单行列出。

示例：
```sql
sys@MARIO> select name,value from v$parameter where name='control_files';

NAME                           VALUE
------------------------------ ------------------------------------------------------------
control_files                  /u01/app/oracle/oradata/MARIO/control01.ctl, /u01/app/oracle
                               /oradata/MARIO/control02.ctl

sys@MARIO> select name,value from v$parameter2 where name='control_files';

NAME                           VALUE
------------------------------ ------------------------------------------------------------
control_files                  /u01/app/oracle/oradata/MARIO/control01.ctl
control_files                  /u01/app/oracle/oradata/MARIO/control02.ctl
```

# v$system_parameter
当前**实例**（instance）中生效的初始化参数值。新建的会话中会继承实例级别的对应参数默认值。

该视图中也包含`isses_modifiable`、`issys_modifiable`、`ispdb_modifiable`和`isinstance_modifiable`字段，并且其含义与**v$parameter**中相同。

**示例：**
修改当前会话的某个初始化参数值：
```sql
--检查该初始化参数在session级别生效的值
sys@MARIO> select name,value from v$parameter where name='log_archive_dest_state_2';

NAME                                               VALUE
-------------------------------------------------- --------------------
log_archive_dest_state_2                           enable

--检查该初始化参数在实例级别生效的值
sys@MARIO> select name,value from v$system_parameter where name='log_archive_dest_state_2';

NAME                                               VALUE
-------------------------------------------------- --------------------
log_archive_dest_state_2                           enable

--变更该初始化参数在会话级别的生效值
sys@MARIO> alter session set log_archive_dest_state_2='defer';

Session altered.

sys@MARIO> select name,value from v$parameter where name='log_archive_dest_state_2';

NAME                                               VALUE
-------------------------------------------------- --------------------
log_archive_dest_state_2                           defer

sys@MARIO> select name,value from v$system_parameter where name='log_archive_dest_state_2';

NAME                                               VALUE
-------------------------------------------------- --------------------
log_archive_dest_state_2                           enable

sys@MARIO> exit
```

退出当前会话后重新连接，再次检查该初始化参数的值：
```sql
sys@MARIO> select name,value from v$parameter where name='log_archive_dest_state_2';

NAME                                               VALUE
-------------------------------------------------- --------------------
log_archive_dest_state_2                           enable

sys@MARIO> select name,value from v$system_parameter where name='log_archive_dest_state_2';

NAME                                               VALUE
-------------------------------------------------- --------------------
log_archive_dest_state_2                           enable
```
可以看到，会话生效的值又变回了实例级别的默认值。


# v$system_parameter2
与**v$system_parameter**包含的参数内容一致。只不过对于参数值为列表的，将该参数的每一个值都分单行列出。


****
下面要介绍的视图，其中包含的参数可以出现在`spfile${ORACLE_SID}.ora`参数文件中。

# v$spparameter 
记录了spfile参数文件中的参数值。如果当前实例没有使用spfile启动（而是以pfile启动），那么该视图中，每一行记录的**ISSPECIFIED**列值都会是**FALSE**。

```sql
--检查当前实例是不是用spfile启动的
sys@MARIO> select decode(count(*),1,'spfile','pfile') from v$spparameter 
where rownum=1 and isspecified='TRUE';

--也可以通过查看参数文件位置来判断是不是用spfile启动的
sys@MARIO> show parameter spfile
sys@MARIO> show parameter pfile
```







