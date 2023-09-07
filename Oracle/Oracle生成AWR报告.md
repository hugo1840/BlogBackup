---
tags: [oracle]
title: Oracle生成AWR报告
created: '2023-09-07T09:17:19.052Z'
modified: '2023-09-07T09:19:08.213Z'
---

Oracle生成AWR报告

手动创建一个当前时间点的快照：
```sql
SQL> exec dbms_workload_repository.create_snapshot();
```

生产一个AWR报告：
```sql
SQL> @?/rdbms/admin/awrrpt.sql
```

生成两个时间段的对比报告：
```sql
SQL> @?/rdbms/admin/awrddrpt.sql
```


