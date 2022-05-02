@[TOC](OceanBase黑屏工具创建租户)

OceanBase支持黑屏和白屏两种客户端工具。**黑屏**工具包括：

- Oceanbase客户端（**OBClient**），同时兼容MySQL和Oracle租户；
- MySQL客户端，兼容MySQL 5.6 和 MySQL 5.7。

OceanBase的**白屏**工具包括：

- OceanBase云平台（**OCP**），支持oceanbase集群和租户的全生命周期管理、资源管理；
- OceanBase开发者中心（**ODC**），企业级的数据库开发平台。 


# 黑屏工具创建租户
OceanBase中，创建租户（**tenant**）时需要为租户分配资源池（**resource pool**），而在创建资源池时又需要指定单元（**unit**）规格。

## 创建unit规格
OceanBase中，通过白屏工具和黑屏工具创建的unit规格分开进行管理，互相看不到。

通过黑屏创建unit的语法如下：

```sql
--创建名称为mini的资源单元
create resource unit mini max_cpu=5, min_cpu=5, max_memory='2G', min_memory='2G', 
max_iops=1000, min_iops=1000, max_session_num=1000000, max_disk_size='10G';
```

虽然在创建unit的语句中指定了iops、session_num等资源限制，但目前实际上只有**cpu**和**内存**资源能起到限制作用。

**注**：OCP 2.5.1版本之后，白屏中默认不会显示内存小于5G的unit。

## 创建资源池
创建unit只是定义了资源规格，实际上在创建资源池时才会真正分配资源。

通过黑屏创建资源池的语法如下：
```sql
--利用定义好的unit规格创建两个资源池
create resource pool mini_pool_t1 unit=mini, unit_num=1;
create resource pool mini_pool_t2 unit=mini, unit_num=1;
```

其中，`unit_num`表示**该资源池在每个zone内的resource unit数目**。**每个resource pool在每台observer上只能有一个resource unit**。因此，`unit_num`的值不会超过一个zone里面的observer数目。

**注**：白屏工具（例如OCP）中，没有创建资源池这一步骤。白屏工具在创建租户时，要选择unit规格和数量。

## 创建租户
黑屏创建MySQL租户的语法如下：
```sql
create tenant obcp_t1 charset='utf8mb4', zone_list=('zone1,zone2,znoe3'), 
primary_zone='zone1,zone2,zone3', resource_pool_list=('mini_pool_t1') 
set ob_tcp_invited_nodes='%';
```

黑屏创建Oracle租户的语法如下：
```sql
create tenant obcp_t2 charset='utf8mb4', zone_list=('zone1,zone2,znoe3'), 
primary_zone='zone1,zone2,zone3', resource_pool_list=('mini_pool_t2') 
set ob_tcp_invited_nodes='%', ob_compatibility_mode='oracle';
```

可以发现，创建租户的默认兼容模式为MySQL租户。在黑屏工具中创建的租户会自动同步到白屏。

**注**：目前，黑屏语法中只支持在`resource_pool_list`中填入一个资源池。也就是说，黑屏模式下创建的租户只有**一个**资源池；在白屏模式下，资源池是自动创建的，一个租户可以对应**多个**资源池。

## 设置密码
通过黑屏创建的租户，第一次登录管理用户时无需密码。登录后需要修改密码。

MySQL租户创建后自动生成的管理用户为**root**，修改用户密码：
```sql
set password = password('MyNewPassword');
```

Oracle租户创建后自动生成的管理用户为**sys**，修改用户密码：
```sql
alter user sys identified by 'MyNewPassword';
```

# 用户登录连接串
通过MySQL客户端登录的连接串如下：
```
mysql -h<OBProxy_IP> -P2883 -u<用户名>@<租户名>#<集群名> -p<用户密码> -D<数据库名>
```
其中，2883表示通过OBProxy访问。不通过OBProxy访问的连接串如下：
```
mysql -h<OBServer_IP> -P2881 -u<用户名>@<租户名>#<集群名> -p<用户密码> -D<数据库名>
```

通过OceanBase客户端登录的连接串如下：
```
obclient -h<OBProxy_IP> -P2883 -u<用户名>@<租户名>#<集群名> -p<用户密码> -c
```

**注**：通过OBProxy登录指定用户、租户和集群名称时，支持以下四种连接串格式：
```
user_name@tenant_name#cluster_name
cluster_name:tenant_name:user_name
cluster_name-tenant_name-user_name
cluster_name.tenant_name.user_name
```

# 查看租户、资源池信息
查看资源池、租户信息
```sql
select * from __all_resource_pool where name like '%mini%';
select * from __all_tenant where tenant_name like '%obcp%';
```

查看Unit规格、资源池、租户信息
```sql
select unit_config_id, name from __all_unit_config;
select tenant_id, tenant_name from __all_tenant;
select unit_config_id, resource_pool_id, name, tenant_id from __all_resource_pool;
```
