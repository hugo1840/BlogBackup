---
tags: [mysql]
title: 基于tpcc-mysql的MySQL数据库TPC-C性能测试
created: '2023-05-01T00:18:07.059Z'
modified: '2023-05-14T01:38:02.634Z'
---

基于tpcc-mysql的MySQL数据库TPC-C性能测试

# TPCC测试工具安装

>:dolphin:下载地址：https://github.com/Percona-Lab/tpcc-mysql

解压安装包： 
```bash
[root@mysqlhost opt]# unzip -q tpcc-mysql-master.zip 
[root@mysqlhost opt]# cd tpcc-mysql-master/ 
[root@mysqlhost tpcc-mysql-master]# ls 
add_fkey_idx.sql count.sql create_table.sql Dockerfile drop_cons.sql load_multi_schema.sh load.sh README.md schema2 scripts src
```

编译安装： 
```bash
[root@mysqlhost tpcc-mysql-master]# cd src 
[root@mysqlhost src]# make 
cc -w -O3 -g -I. mysql_config --include -c load.c 
/bin/sh: mysql_config: command not found 
load.c:19:19: fatal error: mysql.h: No such file or directory 
#include <mysql.h> ^ compilation terminated.
make: *** [load.o] Error 1
```

检查缺失的文件：
```bash
[root@mysqlhost src]# find /mysql -name mysql_config 
/mysql/app/8.0/bin/mysql_config 
/mysql/software/mysql-8.0.30-linux-glibc2.12-x86_64/bin/mysql_config

[root@mysqlhost src]# echo $PATH 
/usr/local/sbin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin 

[root@mysqlhost src]# chown -R mysql:mysql /opt/tpcc-mysql-master
```

重新使用mysql用户编译：
```bash
[root@mysqlhost src]# su - mysql 
[mysql@mysqlhost ~]$ echo $PATH 
/mysql/app/8.0/bin:/mysql/util:/mysql/app/mysql-shell/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/mysql/.local/bin:/home/mysql/bin

[mysql@mysqlhost ~]$ cd /opt/tpcc-mysql-master/src 
[mysql@mysqlhost ~]$ make 
cc -w -O3 -g -I. mysql_config --include -c load.c 
cc -w -O3 -g -I. mysql_config --include -c support.c 
cc load.o support.o mysql_config --libs_r -lrt -o ../tpcc_load 
cc -w -O3 -g -I. mysql_config --include -c main.c 
cc -w -O3 -g -I. mysql_config --include -c spt_proc.c 
cc -w -O3 -g -I. mysql_config --include -c driver.c 
cc -w -O3 -g -I. mysql_config --include -c sequence.c 
cc -w -O3 -g -I. mysql_config --include -c rthist.c 
cc -w -O3 -g -I. mysql_config --include -c sb_percentile.c 
cc -w -O3 -g -I. mysql_config --include -c neword.c 
cc -w -O3 -g -I. mysql_config --include -c payment.c 
cc -w -O3 -g -I. mysql_config --include -c ordstat.c 
cc -w -O3 -g -I. mysql_config --include -c delivery.c 
cc -w -O3 -g -I. mysql_config --include -c slev.c 
cc main.o spt_proc.o driver.o support.o sequence.o rthist.o sb_percentile.o neword.o payment.o ordstat.o delivery.o slev.o mysql_config --libs_r -lrt -o ../tpcc_start

[mysql@mysqlhost src]$ ls ..
add_fkey_idx.sql count.sql create_table.sql Dockerfile drop_cons.sql load_multi_schema.sh load.sh README.md schema2 scripts src tpcc_load tpcc_start
```
编译成功后生成了两个文件`tpcc_load`和`tpcc_start`。


# 导入表结构
创建测试用户和库： 
```bash
[mysql@mysqlhost tpcc-mysql-master]$ mysql -h127.0.0.1 -uroot -p -e "create database tpcc" 
[mysql@mysqlhost tpcc-mysql-master]$ mysql -h127.0.0.1 -uroot -p -e "grant all privileges on tpcc.* to tpcc"
```

在tpcc测试库中创建表和外键索引： 
```bash
[mysql@mysqlhost tpcc-mysql-master]$ mysql -h127.0.0.1 -utpcc -pmyPasswd@@ tpcc < create_table.sql
[mysql@mysqlhost tpcc-mysql-master]$ mysql -h127.0.0.1 -utpcc -pmyPasswd@@ tpcc < add_fkey_idx.sql
```

# 加载测试数据
加载测试数据使用的工具是`tpcc_load`命令：
```bash
[mysql@mysqlhost tpcc-mysql-master]$ ./tpcc_load --help

* TPCC-mysql Data Loader *

./tpcc_load: invalid option -- '-' 
Usage: 
tpcc_load -h server_host -P port -d database_name -u mysql_user -p mysql_password -w warehouses -l part -m min_wh -n max_wh

[part]: 1=ITEMS 2=WAREHOUSE 3=CUSTOMER 4=ORDERS
```

如果要并行加载，需要用到`load.sh`脚本。修改并行加载脚本`load.sh`中的库路径与已安装的MySQL数据库一致： 
```bash
[mysql@mysqlhost tpcc-mysql-master]$ env | grep LIBRARY_PATH 
LD_LIBRARY_PATH=/mysql/app/8.0/lib: 

[mysql@mysqlhost tpcc-mysql-master]$ cat load.sh | grep PATH 
export LD_LIBRARY_PATH=/usr/local/mysql/lib/mysql/

[mysql@mysqlhost tpcc-mysql-master]$ vi load.sh 
[mysql@mysqlhost tpcc-mysql-master]$ cat load.sh | grep PATH 
#export LD_LIBRARY_PATH=/usr/local/mysql/lib/mysql/ 
export LD_LIBRARY_PATH=/mysql/app/8.0/lib/
```

将`load.sh`中的数据加载用户修改为已创建的tpcc数据库用户： 
```bash
[mysql@mysqlhost tpcc-mysql-master]$ cat load.sh 
#export LD_LIBRARY_PATH=/usr/local/mysql/lib/mysql/ 
export LD_LIBRARY_PATH=/mysql/app/8.0/lib/ 
DBNAME=$1 WH=$2 HOST=127.0.0.1 STEP=100

# 修改这里
./tpcc_load -h $HOST -d $DBNAME -utpcc -pmyPasswd@@ -w $WH -l 1 -m 1 -n $WH >> 1.out &

x=1

while [ $x -le $WH ] 
do 
  echo $x $(( $x + $STEP - 1 )) 
  # 修改这里
  ./tpcc_load -h $HOST -d $DBNAME -utpcc -pmyPasswd@@ -w $WH -l 2 -m $x -n $(( $x + $STEP - 1 )) >> 2_$x.out & 
  # 修改这里
  ./tpcc_load -h $HOST -d $DBNAME -utpcc -pmyPasswd@@ -w $WH -l 3 -m $x -n $(( $x + $STEP - 1 )) >> 3_$x.out & 
  # 修改这里
  ./tpcc_load -h $HOST -d $DBNAME -utpcc -pmyPasswd@@ -w $WH -l 4 -m $x -n $(( $x + $STEP - 1 )) >> 4_$x.out & 
  x=$(( $x + $STEP )) 
done
```

并行加载脚本的用法为：
```bash
sh load.sh 数据库名 仓库数量
```

并发加载656个仓库的数据（大概需要2到3个小时）：
```bash
[mysql@mysqlhost tpcc-mysql-master]$ sh load.sh tpcc 656
```

加载完成后检查生成的数据量：
```bash
[mysql@mysqlhost tpcc-mysql-master]$ du -sh /mysql/data/tpcc/ 
65G /mysql/data/tpcc/
```
由`65*1024M/656=101M`，即每个仓库初始大小约为**100M**，这与我们之前使用过的BenchMarkSQL工具一致。


# TPC-C测试
测试用的工具为`tpcc_start`命令：
```bash
[mysql@mysqlhost tpcc-mysql-master]$ ./tpcc_start --help

* ###easy### TPC-C Load Generator *

./tpcc_start: invalid option -- '-' 
Usage: 
tpcc_start -h server_host -P port -d database_name -u mysql_user -p mysql_password -w warehouses -c connections -r warmup_time -l running_time -i report_interval -f report_file -t trx_file
```

其中各个参数含义为：
- `w`：仓库数量；
- `c`：并发连接数；
- `r`：预热时间，单位为秒；
- `l`：压测时间，单位为秒；
- `i`：输出报告中的统计时间间隔。


利用前面加载的数据压测一小时（3600秒），每10秒输出一次统计结果: 
```bash
[mysql@mysqlhost tpcc-mysql-master]$ ./tpcc_start -h127.0.0.1 -dtpcc -utpcc -pmyPasswd@@ \
-w656 -c60 -r10 -l3600 -i10 | tee /tmp/tpcc_report_`date +%F`.log

* ###easy### TPC-C Load Generator *

option h with value '127.0.0.1' 
option d with value 'tpcc' 
option u with value 'tpcc' 
option p with value 'myPasswd@@' 
option w with value '656' 
option c with value '60' 
option r with value '10' 
option l with value '3600' 
option i with value '10' 

<Parameters> 
[server]: 127.0.0.1 
[port]: 3306 
[DBname]: tpcc 
[user]: tpcc 
[pass]: myPasswd@@ 
[warehouse]: 656 
[connection]: 60 
[rampup]: 10 (sec.) 
[measure]: 3600 (sec.)

RAMP-UP TIME.(10 sec.)

MEASURING START.

10, trx: 1827, 95%: 275.749, 99%: 359.713, max_rt: 509.220, 1817|592.320, 183|312.321, 181|1110.903, 179|1263.758 
20, trx: 1825, 95%: 212.781, 99%: 486.549, max_rt: 674.548, 1814|560.789, 181|416.762, 183|1224.756, 179|1339.379 
30, trx: 2103, 95%: 180.860, 99%: 220.431, max_rt: 326.622, 2106|199.355, 211|113.943, 211|772.680, 211|1002.302 
...
```

# 压测报告解读
官方对生成压测报告的说明：
```bash
Output

With the defined interval (-i option), the tool will produce the following output:

  10, trx: 12920, 95%: 9.483, 99%: 18.738, max_rt: 213.169, 12919|98.778, 1292|101.096, 1293|443.955, 1293|670.842
  20, trx: 12666, 95%: 7.074, 99%: 15.578, max_rt: 53.733, 12668|50.420, 1267|35.846, 1266|58.292, 1267|37.421
  30, trx: 13269, 95%: 6.806, 99%: 13.126, max_rt: 41.425, 13267|27.968, 1327|32.242, 1327|40.529, 1327|29.580
  40, trx: 12721, 95%: 7.265, 99%: 15.223, max_rt: 60.368, 12721|42.837, 1271|34.567, 1272|64.284, 1272|22.947
  50, trx: 12573, 95%: 7.185, 99%: 14.624, max_rt: 48.607, 12573|45.345, 1258|41.104, 1258|54.022, 1257|26.626

Where:

10 - the seconds from the start of the benchmark
trx: 12920 - New Order transactions executed during the gived interval (in this case, for the previous 10 sec). Basically this is the throughput per interval. The more the better
95%: 9.483: - The 95% Response time of New Order transactions per given interval. In this case it is 9.483 sec
99%: 18.738: - The 99% Response time of New Order transactions per given interval. In this case it is 18.738 sec
max_rt: 213.169: - The Max Response time of New Order transactions per given interval. In this case it is 213.169 sec
the rest: 12919|98.778, 1292|101.096, 1293|443.955, 1293|670.842 is throughput and max response time for the other kind of transactions and can be ignored
```

测试部分每10秒输出一份统计结果，间隔时间由`-i`参数决定。

- `trx:`后面是10秒内的New Order事务数；
- `95%:`后面是95%的New Order事务响应时间；
- `99%:`后面是99%的New Order事务响应时间；
- `max_rt:`后面是New Order最长事务响应时间。
- 最后的四项分别为其他类型事务（Payment、Order-Status、Delivery以及Stock-Level）的吞吐量和最大响应时间。

TPC-C测试中重点关注新订单（New Order）类型事务。

报告汇总部分解释：
```sql
Raw Results  --第一次统计结果 

[0]: New-Order，新订单业务成功(success,简写sc)次数，延迟(late,简写lt)次数，重试(retry,简写rt)次数，失败(failure,简写fl)次数 
[1]: Payment，支付业务统计，其他同上 
[2]: Order-Status，订单状态业务统计，其他同上 
[3]: Delivery，发货业务统计，其他同上 
[4]: Stock-Level，库存业务统计，其他同上 

Raw Results2--第二次统计结果，其他同上 

transaction percentage: 事务成功率，计算方式为上面的统计结果中(sc+lt) 
response time: 响应时间，90%才算合格 
Tpmc: 每分钟事务数
```

如果要多次重新测试，需要手动清理数据：
```bash
[mysql@mysqlhost tpcc-mysql-master]$ cat drop_database.sql 
drop database tpcc; 
create database tpcc; 
use tpcc; 
show tables;

[mysql@mysqlhost tpcc-mysql-master]$ mysql -h127.0.0.1 -uroot -p < drop_database.sql
```



