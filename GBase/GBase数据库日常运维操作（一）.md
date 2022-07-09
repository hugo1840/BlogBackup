@[TOC](GBase数据库日常运维操作（一）)

# 检查集群状态
在管理节点执行：
```sql
$ su - gbase
$ gcadmin showcluster
$ gcadmin showcluster vc [vc名称]
```

# 创建用户及授权
GBase的SQL语法高度兼容MySQL。

在管理节点登录数据库：
```bash
$ gccli -ugbase -p
```

创建数据库用户并授权：
```sql
-- 检查已有用户
> use vc [vc名称]
> select user,host from gbase.user;
-- 建库
> show databases;
> create database mydb;
-- 建用户
> create user myuser identified by '密码';
-- 授权
> grant all privileges on [vc名称].[DB名称].* to 'myuser'@'%';
> grant select on [vc名称].[DB名称].* to 'another_user'@'%';
-- 检查权限
> show grants for 'myuser'@'%';
```

# 数据库进程及日志
以V9版本为例，**管理节点**包含以下三个进程：

- `gcware`
- `gclusterd`
- `gcrecover`

**数据节点**包含以下两种进程：  

- `gbased`
- `gc_sync_server`

上面两个进程的个数与集群中gbase实例的个数有关。执行以下命令查看Gbase实例个数：

```bash
$ ps -ef | grep gbased
```

Gbase数据库的集群错误日志在管理节点上，路径一般为：
```bash
/opt/gbase_workspace/scripts/monitor/logs/abnormal.log
```


