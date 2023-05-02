---
tags: [监控]
title: MySQL监控告警及可视化：Zabbix+Percona PMP实现（Part II）
created: '2023-05-01T02:51:25.980Z'
modified: '2023-05-02T09:47:01.270Z'
---

MySQL监控告警及可视化：Zabbix+Percona PMP实现（Part II）


服务器清单如下：
| 服务器IP | 配置 | OS版本 | 服务器角色 |
| :--: | :--: | :--: | :--: |
| 172.16.175.x | 4c8g | CentOS 7.7 | MySQL Server |
| 172.16.175.y | 4c8g | CentOS 7.7 | Zabbix Server |


# PMP插件安装
PMP即Percona Monitoring Plugins，支持Nagios、Cacti和Zabbix。下面我们在被监控主机上安装PMP插件。

```bash
# 下载PMP
[root@mysqldb ~]# wget https://www.percona.com/downloads/percona-monitoring-plugins/percona-monitoring-plugins-1.1.8/binary/redhat/7/x86_64/percona-zabbix-templates-1.1.8-1.noarch.rpm

# 安装PMP
[root@mysqldb ~]# rpm -ivh percona-zabbix-templates-1.1.8-1.noarch.rpm
```

安装成功后的目录结构如下：
```bash
[root@mysqldb ~]# tree /var/lib/zabbix/percona/
/var/lib/zabbix/percona/
├── scripts
│   ├── get_mysql_stats_wrapper.sh  # 对ss_get_mysql_stats.php脚本的封装
│   └── ss_get_mysql_stats.php      # 用于采集MySQL监控数据
└── templates
    ├── userparameter_percona_mysql.conf   # 定义监控项
    └── zabbix_agent_template_percona_mysql_server_ht_2.0.9-sver1.1.8.xml   # zabbix模板文件

2 directories, 4 files
```

# PMP监控脚本配置
在被监控的MySQL数据库中创建监控用户：
```sql
mysql> create user 'monitor_user'@'localhost' identified with mysql_native_password by 'monitor_pass';
mysql> grant process,replication client on *.* to 'monitor_user'@'localhost';
```

修改PMP监控脚本中的数据库连接配置：
```bash
[root@mysqldb ~]# cd /var/lib/zabbix/
[root@mysqldb zabbix]# cat percona/scripts/ss_get_mysql_stats.php | grep '^$mysql_'
$mysql_user = 'monitor_user';
$mysql_pass = 'monitor_pass';
$mysql_port = 3306;
$mysql_socket = '/tmp/mysql.sock';
...
```

测试监控脚本：
```bash
# 测试
[root@mysqldb zabbix]# yum install php-cli php-mysql -y
[root@mysqldb zabbix]# php /var/lib/zabbix/percona/scripts/ss_get_mysql_stats.php --host localhost --items iu 

# 检查采集结果：有数据代表采集成功
[root@mysqldb zabbix]# cat /tmp/localhost-mysql_cacti_stats.txt 
gg:0 gh:0 gi:0 gj:0 gk:0 gl:4887 gm:0 gn:3 go:0 gp:0 gq:327680 gr:326443 gs:1202 gt:0 gu:1059 gv:143 gw:225 gx:135 gy:1087 gz:334 hg:51 hh:0 hi:0 hj:0 hk:-1 hl:0 hm:0 hn:-1 ho:0 hp:0 hq:0 hr:0 hs:0 ht:0 hu:0 hv:0 hw:0 hx:0 hy:0 hz:0 ig:0 ih:3 ii:0 ij:3 ik:88 il:583 im:400 in:60000 io:400 ip:0 iq:0 ir:1 is:0 it:0 iu:1 iv:1 iw:2 ix:1024 iy:200 iz:11 jg:-1 jh:-1 ji:-1 jj:-1 jk:0 jl:-1 jm:-1 jn:-1 jo:-1 jp:-1 jq:-1 jr:-1 js:-1 jt:-1 ju:25 jv:0 jw:0 jx:8 jy:0 jz:0 kg:0 kh:0 ki:0 kj:0 kk:0 kl:1 km:0 kn:0 ko:0 kp:9 kq:0 kr:0 ks:0 kt:0 ku:4 kv:0 kw:5 kx:53608 ky:1632 kz:8388608 lg:0 lh:23523577 li:23523577 lj:-1 lk:32768 ll:0 lm:2 ln:6613 lo:0 lp:0 lq:0 lr:0 ls:0 lt:0 lu:1 lv:0 lw:0 lx:0 ly:0 lz:0 mg:0 mh:0 mi:0 mj:0 mk:0 ml:1 mm:624 mn:8 mo:0 mp:8 mq:44 mr:1828 ms:4334 mt:0 mu:0 mv:2522 mw:0 mx:0 my:0 mz:333 ng:1166 nh:0 ni:0 nj:1328213 nk:0 nl:0 nm:0 nn:0 no:1 np:0 nq:2 nr:-1 ns:-1 nt:-1 nu:-1 nv:-1 nw:-1 nx:-1 ny:-1 nz:-1 og:0 oh:1529856 oi:8388608 oj:0 ok:0 ol:-1 om:-1 on:-1 oo:-1 op:-1 oq:-1 or:-1 os:-1 ot:-1 ou:-1 ov:-1 ow:-1 ox:-1 oy:-1 oz:-1 pg:-1 ph:-1 pi:-1 pj:-1 pk:-1 pl:-1 pm:-1 pn:-1 po:-1 pp:-1 pq:-1 pr:-1 ps:-1 pt:-1 pu:-1 pv:-1 pw:-1 px:-1 py:-1 pz:-1 qg:-1 qh:-1 qi:-1 qj:-1 qk:-1 ql:-1 qm:-1 qn:-1 qo:1060 qp:16988

# 记得删除！！！
[root@mysqldb zabbix]# rm -rf /tmp/localhost-mysql_cacti_stats.txt 
```

拷贝监控项文件到Zabbix Agent配置目录下：
```bash
[root@mysqldb zabbix]# cp /var/lib/zabbix/percona/templates/userparameter_percona_mysql.conf /usr/local/zabbix/etc/zabbix_agentd.conf.d/
```

修改Zabbix Agent配置文件，然后重启agent服务。
```bash
[root@mysqldb zabbix]# cat /usr/local/zabbix/etc/zabbix_agentd.conf | grep '^Include='
Include=/usr/local/zabbix/etc/zabbix_agentd.conf.d/

[root@mysqldb zabbix]# systemctl restart zabbix_agentd
```

# Web界面导入PMP模板
在Zabbix Web管理界面 **Configuration - Template - Import** 导入PMP的Zabbix XML模板。

![img1](https://img-blog.csdnimg.cn/ee175af37ee34413ba75c1e9b3341eea.png#pic_center)
![img2](https://img-blog.csdnimg.cn/95ca5faee1e0466b9ce4f501efea0f5e.png#pic_center)

在 **Configuration - Hosts - Create host** 中添加被监控主机，并关联PMP模板。

![img3](https://img-blog.csdnimg.cn/a61dce54035741259eba4d46f822eeb1.png#pic_center)

在 **Monitoring - Latest data** 中可以看到最新的监控数据。

![img4](https://img-blog.csdnimg.cn/f73cccd7ee13474c9642425f318b5e0a.png#pic_center)



