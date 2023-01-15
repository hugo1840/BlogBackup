---
tags: [达梦]
title: 达梦数据库更新License
created: '2023-01-15T08:00:58.326Z'
modified: '2023-01-15T08:43:58.329Z'
---

# 达梦数据库更新License


1. **替换License文件**

把新的License文件拷贝到`$DM_HOME/bin`目录下。

2. **重启数据库**

查看实例服务名：
```bash
[dmdba@dmhost ~]$ ls $DM_HOME/bin | grep DmService
DmServiceInstanceName
```

重启数据库：
```bash
[dmdba@dmhost ~]$ systemctl restart DmServiceInstanceName
```

3. **检查License是否生效**

检查License到期时间：
```sql
[dmdba@dmhost ~]$ $DM_HOME/bin/disql SYSDBA/SYSDBA
SQL> select LIC_VERSION,EXPIRED_DATE from v$license;
```







