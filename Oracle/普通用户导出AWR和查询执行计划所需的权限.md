---
tags: [oracle]
title: 普通用户导出AWR和查询执行计划所需的权限
created: '2023-06-12T12:17:59.005Z'
modified: '2023-06-12T12:26:24.372Z'
---

普通用户导出AWR和查询执行计划所需的权限

# AWR导出权限
普通用户导出AWR报告需要的权限如下：
```sql
SQL> grant select any dictionary to xxx;
SQL> grant execute on DBMS_WORKLOAD_REPOSITORY to xxx;
```

导出AWR报告：
```sql
SQL> exec dbms_workload_repository.create_snapshot();
SQL> @?/rdbms/admin/awrrpt.sql
```

# 执行计划查询权限
普通用户调用 
```sql
select * from table(dbms_xplan.display_cursor(null,null,'advanced'))
```
查看执行计划至少需要的权限（注意`$`符号前面有下划线）如下：
```sql
SQL> grant select on v_$sql to xxx;
SQL> grant select on v_$sql_plan to xxx;
SQL> grant select on v_$sql_plan_statistics to xxx;
SQL> grant select on v_$session to xxx;
```

也可以直接授予`select any dictionary`权限。


