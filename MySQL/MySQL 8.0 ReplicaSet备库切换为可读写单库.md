---
tags: [mysql]
title: MySQL 8.0 ReplicaSet备库切换为可读写单库
created: '2024-01-01T12:26:12.498Z'
modified: '2024-01-01T12:59:47.769Z'
---

MySQL 8.0 ReplicaSet备库切换为可读写单库

# 方法一

1. **从集群中删除备库**（不会改变备库只读状态）

```bash
# 检查备库标识
var rs = dba.getReplicaSet()
rs.status()

# 移除备库同步
rs.removeInstance("MYSQL_REPLICA_IDENTIFIER:3306")
#或者 rs.removeInstance("MYSQL_REPLICA_IDENTIFIER:3306", {force:true})
rs.status()
```

2. **关闭备库只读模式**

```sql
--检查备库可读写状态
show variables like '%read_only%';

--关闭备库只读
set global super_read_only=0;
set global read_only=0;

show variables like '%read_only%';
```

# 方法二

1. **关闭备库同步**

```sql
--检查备库同步状态
show replica status\G

stop replica;       --关闭同步
reset replica all;  --清除主库信息

show replica status\G
```

2. **关闭备库只读模式**

```sql
show variables like '%read_only%';

set global super_read_only=0;
set global read_only=0;

show variables like '%read_only%';
```

使用此种方法，集群中还会残留备库的拓扑信息，如果使用`removeInstance`删除备库实例可能会导致备库变成只读。









