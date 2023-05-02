---
tags: [监控]
title: MySQL监控告警及可视化：Zabbix+Percona PMP实现（Part I）
created: '2023-05-01T00:39:10.674Z'
modified: '2023-05-02T02:00:06.177Z'
---

MySQL监控告警及可视化：Zabbix+Percona PMP实现（Part I）

# 准备工作
>:dolphin:**软件下载**：
>- Zabbix下载：https://www.zabbix.com/download_sources#60LTS

服务器清单如下：
| 服务器IP | 配置 | OS版本 | 服务器角色 |
| :--: | :--: | :--: | :--: |
| 172.16.175.x | 4c8g | CentOS 7.7 | MySQL Server |
| 172.16.175.y | 4c8g | CentOS 7.7 | Zabbix Server |

需要提前在MySQL Server和Zabbix Server两台服务器上安装MySQL服务。其中Zabbix Server服务器上的MySQL作用是作为后端数据库存储Zabbix监控数据。

****
Zabbix主要有以下几个功能组件：
- **Zabbix Server**：负责接收监控数据并触发告警，以及将监控数据持久化到后端数据库中。
- **Zabbix Agent**：部署在被监控设备上的客户端，负责采集监控数据并发送到Zabbix服务端。
- **Zabbix Proxy**：在监控数据量较大时，可以作为Zabbix Server的代理接收监控数据并进行预处理，以减轻服务端的压力。
- **Web前端**：用于维护被监控设备的信息、查看监控数据、配置告警等。
- **后端数据库**：负责存储监控数据和被监控设备信息。支持Oracle、MySQL、SQLite3、PostgreSQL。

# Zabbix Server安装
```bash
# 解压安装包
[root@zabsever opt]# tar xvf zabbix-6.0.17.tar.gz
[root@zabsever opt]# ln -s zabbix-6.0.17 zabbix

# 创建系统用户
[root@zabsever opt]# groupadd --system zabbix
[root@zabsever opt]# useradd --system -g zabbix -d /usr/lib/zabbix -s /sbin/nologin zabbix
```

创建后端数据库用户：
```sql
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> create user 'zabbix_admin'@'%' identified with mysql_native_password by 'zabbixnubi666';
mysql> grant all privileges on `zabbix`.* to 'zabbix_admin'@'%' with grant option;
```
:snake:**注意**：这里zabbix数据库的字符集一定要跟上面一样，不能改成utf8mb4，否则后面在配置Web界面时会报数据库连接错误、以及不支持的字符集和排序规则错误。

初始化数据：
```bash
[root@zabsever opt]# cd /opt/zabbix/database/mysql
[root@zabsever mysql]# mysql -h127.0.0.1 -uzabbix_admin -pzabbixnubi666 zabbix < schema.sql 
[root@zabsever mysql]# mysql -h127.0.0.1 -uzabbix_admin -pzabbixnubi666 zabbix < images.sql 
[root@zabsever mysql]# mysql -h127.0.0.1 -uzabbix_admin -pzabbixnubi666 zabbix < data.sql
```

编译安装Zabbix Server：
```bash
# 安装依赖包
[root@zabsever zabbix]# yum install gcc mysql-devel libevent-devel libcurl-devel libxml2-devel net-snmp-devel -y

# 安装Zabbix Server到/usr/local/zabbix目录下
[root@zabsever zabbix]# ./configure --prefix=/usr/local/zabbix --enable-server --enable-agent \
--with-mysql=/mysql/mysql-8.0/bin/mysql_config --enable-ipv6 --with-net-snmp \
--with-libcurl --with-libxml2
[root@zabsever zabbix]# make install | tee /tmp/zabbix_install_`date +%F_%H_%M`.log
```

编译安装成功后的目录结构应该如下所示：
```bash
[root@zabsever zabbix]# tree /usr/local/zabbix/
/usr/local/zabbix/
├── bin
│   ├── zabbix_get
│   ├── zabbix_js
│   └── zabbix_sender
├── etc
│   ├── zabbix_agentd.conf
│   ├── zabbix_agentd.conf.d
│   ├── zabbix_server.conf
│   └── zabbix_server.conf.d
├── lib
│   └── modules
├── sbin
│   ├── zabbix_agentd
│   └── zabbix_server
└── share
    ├── man
    │   ├── man1
    │   │   ├── zabbix_get.1
    │   │   └── zabbix_sender.1
    │   └── man8
    │       ├── zabbix_agentd.8
    │       └── zabbix_server.8
    └── zabbix
        ├── alertscripts
        └── externalscripts

14 directories, 11 files
```

其中，**make install** 过程中的常见报错及解决办法如下：

1. **gcc编译器for循环代码报错**

报错信息：
```bash
parse.c: In function ‘eval_get_last_function_token’:
parse.c:391:2: error: ‘for’ loop initial declarations are only allowed in C99 mode
  for(int i = ctx->ops.values_num - 1; i >= 0; i --)
  ^
parse.c:391:2: note: use option -std=c99 or -std=gnu99 to compile your code
make[3]: *** [parse.o] Error 1
```

解决办法是在configure前配置`-std=gnu99`，然后重新编译安装：
```bash
[root@zabsever zabbix]# export CFLAGS="-std=gnu99"
[root@zabsever zabbix]# ./configure --prefix=/usr/local/zabbix --enable-server --enable-agent \
--with-mysql=/mysql/mysql-8.0/bin/mysql_config --enable-ipv6 --with-net-snmp \
--with-libcurl --with-libxml2
[root@zabsever zabbix]# make install | tee /tmp/zabbix_install_`date +%F_%H_%M`.log
```

2. **找不到libcrypto和libssl模块**

报错信息：
```bash
/usr/bin/ld: warning: libcrypto.so.1.1, needed by /mysql/mysql-8.0/lib/libmysqlclient.so, not found (try using -rpath or -rpath-link)
/usr/bin/ld: warning: libssl.so.1.1, needed by /mysql/mysql-8.0/lib/libmysqlclient.so, not found (try using -rpath or -rpath-link)
/mysql/mysql-8.0/lib/libmysqlclient.so: undefined reference to `SSL_SESSION_free@OPENSSL_1_1_0'
...
collect2: error: ld returned 1 exit status
make[3]: *** [zabbix_server] Error 1
make: *** [install-recursive] Error 1
```

解决办法是建立对应的软链接，然后重新编译安装：
```bash
# 找到模块文件位置
[root@zabsever zabbix]# find / -name libcrypto.so.1.1
/mysql/mysql-8.0/lib/private/libcrypto.so.1.1
[root@zabsever zabbix]# find / -name libssl.so.1.1
/mysql/mysql-8.0/lib/private/libssl.so.1.1

# 建立软链接
[root@zabsever zabbix]# ln -s /mysql/mysql-8.0/lib/private/libcrypto.so.1.1 /usr/lib64
[root@zabsever zabbix]# ln -s /mysql/mysql-8.0/lib/private/libssl.so.1.1 /usr/lib64
# 重新configure ... && make install
```

# Zabbix Server配置
## conf文件配置
修改`zabbix_server.conf`中的后端数据库连接配置：
```bash
[root@zabsever zabbix]# cd /usr/local/zabbix/
[root@zabsever zabbix]# cat etc/zabbix_server.conf | grep -Ev '^$|^#' 
LogFile=/tmp/zabbix_server.log
DBHost=127.0.0.1
DBName=zabbix
DBUser=zabbix_admin
DBPassword=zabbixnubi666
Timeout=4
LogSlowQueries=3000
StatsAllowedIP=127.0.0.1
```

修改`zabbix_agentd.conf`中的数据库连接配置：
```bash
[root@zabsever zabbix]# cat etc/zabbix_agentd.conf | grep -Ev '^$|^#' 
LogFile=/tmp/zabbix_agentd.log
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=Zabbix server
```
这里的配置是因为Zabbix Server本身所在服务器的监控数据也需要Zabbix Agent来采集。

## 系统服务配置
修改Zabbix安装包解压目录下的系统服务配置文件后，拷贝至`/etc/init.d`目录下。

配置zabbix server系统服务：
```bash
# 添加ZABBIX_BIN和CONFIG_FILE
[root@zabsever zabbix]# vi /opt/zabbix/misc/init.d/fedora/core5/zabbix_server
...
ZABBIX_BIN="/usr/local/zabbix/sbin/zabbix_server"
CONFIG_FILE="/usr/local/zabbix/etc/zabbix_server.conf"
...
start() {
        echo -n $"Starting $prog: "
        daemon $ZABBIX_BIN -c $CONFIG_FILE
        ...

# 拷贝到init.d目录下
[root@zabsever zabbix]# cp /opt/zabbix/misc/init.d/fedora/core5/zabbix_server /etc/init.d/
```

配置zabbix agent系统服务：
```bash
[root@zabsever zabbix]# vi /opt/zabbix/misc/init.d/fedora/core5/zabbix_agentd
...
ZABBIX_BIN="/usr/local/zabbix/sbin/zabbix_agentd"
CONFIG_FILE="/usr/local/zabbix/etc/zabbix_agentd.conf"
...
start() {
        echo -n $"Starting $prog: "
        daemon $ZABBIX_BIN -c $CONFIG_FILE
        ...

# 拷贝到init.d目录下
[root@zabsever zabbix]# cp /opt/zabbix/misc/init.d/fedora/core5/zabbix_agentd /etc/init.d/ 
```

启动zabbix server和zabbix agentd服务：
```bash
# 启动服务
#service zabbix_server start
#service zabbix_agentd start
systemctl start zabbix_server
systemctl start zabbix_agentd

# 配置开机自启
systemctl enable zabbix_server
systemctl enable zabbix_agentd
```

启动zabbix系统服务时的常见错误以及解决办法如下：

1. **找不到libmysqlclient模块**

报错信息：
```bash
May 01 20:32:01 zabsever zabbix_server[2636]: Starting Zabbix Server: /usr/local/zabbix/sbin/zabbix_server: error while loading shared libraries: libmysqlclient.so.21: cann...r directory
May 01 20:32:01 zabsever zabbix_server[2636]: [FAILED]
May 01 20:32:01 zabsever systemd[1]: zabbix_server.service: control process exited, code=exited status=1
```

解决办法是创建对应的软链接后再启动服务：
```bash
[root@zabsever zabbix]# find / -name libmysqlclient.so.21
/mysql/mysql-8.0/lib/libmysqlclient.so.21

[root@zabsever zabbix]# ln -s /mysql/mysql-8.0/lib/libmysqlclient.so.21 /usr/lib64
```

## Web服务配置
为了能够使用Web图形化界面管理和查看监控，还需要安装Web服务器组件。Web服务器通常可以使用httpd或者Nginx。Zabbix前端是用PHP语言写的，因此还需要安装PHP。

由于CentOS 7的YUM源中只有PHP 5，而Zabbix从5.0版本开始要求PHP版本不低于7.2，因此需要配置额外的源。
```bash
# 安装第三方源
[root@zabsever zabbix]# rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
[root@zabsever zabbix]# rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
```

安装httpd和PHP语言：
```bash
[root@zabsever zabbix]# yum install httpd -y
[root@zabsever zabbix]# yum install php72w-cli php72w-common php72w-gd \
php72w-ldap php72w-mbstring php72w-mysqlnd \
php72w-xml php72w-bcmath mod_php72w -y
```

将Zabbix源码中的php文件拷贝到httpd的根目录下：
```bash
[root@zabsever zabbix]# mkdir -p /var/www/html/zabbix
[root@zabsever zabbix]# cp -r /opt/zabbix/ui/* /var/www/html/zabbix/
[root@zabsever zabbix]# chown -R apache:apache /var/www/html/
```

修改PHP的配置文件：
```bash
# 修改以下参数
[root@zabsever zabbix]# vi /etc/php.ini
max_execution_time = 300
max_input_time = 300
memory_limit = 128M
post_max_size = 16M
upload_max_filesize = 2M
date.timezone = Asia/Shanghai
```

启动Web服务器并设置开机自启：
```bash
systemctl start httpd
systemctl enable httpd
```
通过访问`http://{ZABBIX_SERVER_IP}/zabbix`来配置Zabbix Server的Web前端页面。

![img1](https://img-blog.csdnimg.cn/90f37ecde41041e4aaf9f413d3ae9f9b.png#pic_center)
![img2](https://img-blog.csdnimg.cn/1e489a09adb44305a2e2ded94659ac6d.png#pic_center)
![img3](https://img-blog.csdnimg.cn/cc26388b9b644116ad40864bdf545a62.png#pic_center)
![img4](https://img-blog.csdnimg.cn/8acedbeb553948948b8b3dcc60b07260.png#pic_center)
![img5](https://img-blog.csdnimg.cn/53c4e4daf71f44c3b937e6402733901a.png#pic_center)

配置完成后，既可以登录Web管理页面。默认管理员用户及登录口令为`Admin/zabbix`。

![img6](https://img-blog.csdnimg.cn/024a7fe86e494d21bd064b72aa3ecb73.png#pic_center)
![img7](https://img-blog.csdnimg.cn/3a8d7081c0b34250abd209146ccf1d98.png#pic_center)

如果有报错，记得检查日志：
```bash
[root@zabsever ~]# tail -f /tmp/zabbix_server.log
```

# Zabbix Agent安装
在被监控端服务器上安装Zabbix Agent。

创建zabbix用户：
```bash
[root@mysqldb ~]# groupadd --system zabbix
[root@mysqldb ~]# useradd --system -g zabbix -d /usr/lib/zabbix -s /sbin/nologin zabbix
```

编译安装zabbix agent：
```bash
# 解压安装包
[root@mysqldb opt]# tar xvf zabbix-6.0.17.tar.gz
[root@mysqldb opt]# ln -s zabbix-6.0.17 zabbix

# 安装依赖包
[root@mysqldb opt]# yum install gcc pcre-devel -y

# 编译安装
[root@mysqldb opt]# cd zabbix
[root@mysqldb zabbix]# ./configure --prefix=/usr/local/zabbix --enable-agent
[root@mysqldb zabbix]# make install
```

编译安装成功后的目录结构如下：
```bash
[root@mysqldb zabbix]# tree /usr/local/zabbix
/usr/local/zabbix
├── bin
│   ├── zabbix_get
│   └── zabbix_sender
├── etc
│   ├── zabbix_agentd.conf
│   └── zabbix_agentd.conf.d
├── lib
│   └── modules
├── sbin
│   └── zabbix_agentd
└── share
    └── man
        ├── man1
        │   ├── zabbix_get.1
        │   └── zabbix_sender.1
        └── man8
            └── zabbix_agentd.8

10 directories, 7 files
```

# Zabbix Agent配置
## conf文件配置
配置文件`zabbix_agentd.conf`有如下几个重要参数：
- `Server`：被动模式下Zabbix Server的地址。被动模式即Pull模式，是Zabbix Agent的默认工作模式。
- `ServerActive`：主动模式下Zabbix Server的地址。主动模式即Push模式，Zabbix Agent会主动把采集到的监控数据发送给Server.
- `Hostname`：主机名，可以是IP地址。要求在Zabbix Server监控的所有主机内具有唯一标识性。Hostname只适用于主动模式。

参照如下配置即可：
```bash
[root@mysqldb zabbix]# cat etc/zabbix_agentd.conf | grep -Ev '^$|^#'
LogFile=/tmp/zabbix_agentd.log
Server=172.16.175.y
ServerActive=127.0.0.1
Hostname=mysqlhost
```

## 系统服务配置
配置zabbix agent系统服务：
```bash
[root@mysqldb zabbix]# vi /opt/zabbix/misc/init.d/fedora/core5/zabbix_agentd
...
ZABBIX_BIN="/usr/local/zabbix/sbin/zabbix_agentd"
CONFIG_FILE="/usr/local/zabbix/etc/zabbix_agentd.conf"
...
start() {
        echo -n $"Starting $prog: "
        daemon $ZABBIX_BIN -c $CONFIG_FILE
        ...

# 拷贝到init.d目录下
[root@mysqldb zabbix]# cp /opt/zabbix/misc/init.d/fedora/core5/zabbix_agentd /etc/init.d/ 
```

启动zabbix agentd服务：
```bash
#service zabbix_agentd start
systemctl start zabbix_agentd
systemctl enable zabbix_agentd
```



