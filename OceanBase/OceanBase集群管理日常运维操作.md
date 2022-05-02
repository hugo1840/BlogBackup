@[TOC](OceanBase集群管理日常运维操作)

# 时钟同步
检查NTP时间是否同步，OceanBase能够容忍的集群内部时钟偏差最大为**100ms**。

执行`ntpq -q`，输出的offset应该小于**50ms**。 

# 查看、启停、修改zone
启停zone实际上是在切换提供leader服务的zone，并不是真的在启停OS中的服务进程。

```sql
select * from __all_zone;
alter system {start|stop|force stop} zone [zone_name];
alter system {alter|change|modify} zone [zone_name] set [zone_option_list];
```

# 查看、管理observer
停止observer同样也不表示进程退出，仅表示不提供leader服务。

```sql
select * from __all_server;
select * from __all_server_event_history;
alter system {start|stop} server 'IP:port'[,'IP:port',...] [zone='zone'];
```

# observer服务进程管理
查看observer进程
```bash
ps -ef | grep observer
```
启动进程（admin用户）
```bash
cd /home/admin/oceanbase
./bin/observer [启动参数]
```
停止进程
```bash
kill -15 `pgrep observer`
kill -9 `pgrep observer`
```

# observer服务启动恢复
由于增删改数据在内存中进行，Observer进程启动后，需要：

- 与其他副本同步，将clog或ssd基线数据进行同步（**补齐**）；
- 将上一次合并之后的内存数据恢复出来（**clog回放**），才能提供服务。

为了加快OceanBase的服务恢复过程，可以在停止observer服务之前，执行一次合并（**major freeze**）。

## 服务停止（停机运维）
1. 如果停机维护时长大于1小时但小于1天，需要设置永久下线时间：
```sql
alter system set server_permanent_offline_time='86400s';
```
2. 将服务从当前Observer迁走：
```sql
alter system stop server 'IP_address:2882';
```
3. 检查主副本都已经切走，返回值为0：
```sql
select count(*) from __all_virtual_table t, __all_virtual_meta_table m
where t.table_id=m.table_id and role=1 and m.svr_ip='IP_address';
```
4. 停止进程：
```bash
kill -15 <observer_pid>
```

## 服务恢复（停机运维结束）
1. 机器上电；
2. 检查机器**ntp同步状态**和服务运行情况；
3. admin用户启动observer进程；
```bash
cd /home/admin/oceanbase
./bin/observer [启动参数]
```
4. 系统租户登录，启动server：
```sql
alter system start server 'IP_address:2882';
```
5. 检查`__all_server`表，查看`status='active'`且`start_service_time` **不为Null**，则表示observer正常启动并开始提供服务；
6. 将永久下线时间改回默认值**3600s**：
```sql
alter system set server_permanent_offline_time='3600s';
```

# 故障节点替换
为确保集群中有足够的冗余资源，需要及时对故障节点进行替换。

1. 系统租户登录故障节点，停止observer，确保主副本都切走；
2. 为目标zone添加新的server：
```sql
alter system add server 'IP_address:2882' zone 'zone1';
```
3. 将故障server下线：
```sql
alter system delete server 'IP_address:2882' zone 'zone1';
```
OceanBase会自动将被下线observer上的unit迁移到新增的observer上。

4. 检查`__all_server`表中的server状态，旧的observer的信息已经消失。


# 容量不足
## 内存不足
内存空间不足时，可以有以下两种处理思路：

- 扩容：调大租户内存；
- 释放已用内存：触发转储、合并。

## 存储不足
日志盘满时，根据日志类型处理：

- observer运行日志：清理旧日志；
- clog事务日志：查看`__all_virtual_server_clog_stat`，清理旧日志，再合并。

数据盘满时，可以扩容，将旧数据迁走，再合并。
