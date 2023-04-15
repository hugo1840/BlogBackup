---
attachments: [Clipboard_2023-04-15-19-20-35.png]
tags: [oracle]
title: 基于BenchmarkSQL的Oracle数据库tpcc性能测试
created: '2023-04-14T14:54:21.280Z'
modified: '2023-04-15T11:21:55.759Z'
---

基于BenchmarkSQL的Oracle数据库tpcc性能测试

# 安装BenchmarkSQL及其依赖
| 软件包 | 作用 |
| :--: | :--: |
| benchmarksql-5.0.zip | tpcc性能测试 |
| htop-3.0.5.zip | 可视化压测过程 |
| R-3.6.3.tar.gz | 生成压测报告 |
| ojdbc8.jar | Oracle JDBC驱动 |

>:lion:**下载地址**
><br/>- BenchmarkSQL: https://github.com/petergeoghegan/benchmarksql
><br/>- htop: https://github.com/htop-dev/htop/releases
><br/>- R: https://mirror.bjtu.edu.cn/cran/src/base/R-3/R-3.6.3.tar.gz
><br/>- ojdbc8: https://www.oracle.com/database/technologies/maven-central-guide.html


## 安装软件依赖
安装依赖软件包：
```bash
yum install gcc glibc-headers gcc-c++ gcc-gfortran readline-devel  libXt-devel pcre-devel libcurl libcurl-devel -y

yum install ncurses ncurses-devel autoconf automake zlib zlib-devel bzip2 bzip2-devel xz-devel -y

yum install java-1.8.0-openjdk ant -y
```

编译安装R语言：
```bash
yum install pango-devel pango libpng-devel cairo cairo-devel -y

tar -zxf R-3.6.3.tar.gz
cd R-3.6.3
./configure && make && make install
```

编译安装htop：
```bash
unzip htop-3.0.5.zip
cd htop-3.0.5
./autogen.sh && ./configure && make && make install
```

检查安装情况：
```bash
[root@primarydb benchmarksql]# ant -version
Apache Ant(TM) version 1.9.4 compiled on November 5 2018

[root@primarydb benchmarksql]# java -version
openjdk version "1.8.0_362"
OpenJDK Runtime Environment (build 1.8.0_362-b08)
OpenJDK 64-Bit Server VM (build 25.362-b08, mixed mode)

[root@primarydb benchmarksql]# htop --version
htop 3.0.5

[root@primarydb benchmarksql]# R --version
R version 3.6.3 (2020-02-29) -- "Holding the Windsock"
Copyright (C) 2020 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)
```

## 编译BenchmarkSQL
解压安装包：
```bash
[root@primarydb benchmarksql]# unzip benchmarksql-5.0.zip
[root@primarydb benchmarksql]# ls benchmarksql-5.0/lib/
apache-log4j-extras-1.1.jar  firebird  log4j-1.2.17.jar  oracle  postgres
```

拷贝数据库驱动到BenchmarkSQL的`lib/oracle`目录下：
```bash
cp /u01/app/oracle/product/19.0.0/dbhome_1/jdbc/lib/ojdbc8.jar benchmarksql-5.0/lib/oracle/

cp /u01/app/oracle/product/19.0.0/dbhome_1/jlib/orai18n.jar benchmarksql-5.0/lib/oracle/
```
其中，`ojdbc8.jar`用于连接数据库，而`orai18n.jar`驱动可以避免因为字符集不兼容导致的报错。

使用ant编译BenchmarkSQL：
```bash
[root@primarydb benchmarksql]# cd benchmarksql-5.0

[root@primarydb benchmarksql-5.0]# ant
Buildfile: /opt/benchmarksql/benchmarksql-5.0/build.xml

init:
    [mkdir] Created dir: /opt/benchmarksql/benchmarksql-5.0/build

compile:
    [javac] Compiling 11 source files to /opt/benchmarksql/benchmarksql-5.0/build
    [javac] warning: [path] bad path element "/opt/benchmarksql/benchmarksql-5.0/lib/oracle/oraclepki.jar": no such file or directory
    [javac] 1 warning

dist:
    [mkdir] Created dir: /opt/benchmarksql/benchmarksql-5.0/dist
      [jar] Building jar: /opt/benchmarksql/benchmarksql-5.0/dist/BenchmarkSQL-5.0.jar

BUILD SUCCESSFUL
Total time: 0 seconds
```

# BenchmarkSQL props文件配置
`benchmarksql-5.0/run/props.xxx`是使用BenchmarkSQL进行性能测试的主要配置文件。

Oracle数据库对应的文件名为`props.ora`：
```java
db=oracle                                               //测试的数据库类型
driver=oracle.jdbc.driver.OracleDriver                  //数据库驱动
conn=jdbc:oracle:thin:@localhost:1521:ORACLE_SVC_NAME   //数据库连接串
user=BENCHMARKSQL                                       //数据库连接用户
password=benchmarksql                                   //数据库连接用户密码

warehouses=100        //仓库数，控制测试数据量，每个仓库初始大小约为100MB
loadWorkers=8        //初始化测试数据时，往数据库中加载数据的并发进程数

terminals=16            //客户端并发连接数
//To run specified transactions per terminal- runMins must equal zero
runTxnsPerTerminal=10   //每个客户端运行的事务数量。该参数不为0时，runMins必须为0
//To run for specified minutes- runTxnsPerTerminal must equal zero
runMins=0               //测试的时长，单位为分钟。该参数不为0时，runTxnsPerTerminal必须为0
//Number of total transactions per minute
limitTxnsPerMin=300     //每分钟运行的事务数量

//Set to true to run in 4.x compatible mode. Set to false to use the
//entire configured database evenly.
terminalWarehouseFixed=true

//The following five values must add up to 100
//TPC-C规范的默认百分比为45:43:4:4:4
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4

// Directory name to create for collecting detailed result data.
//测试结果数据存储位置和命名规则
resultDirectory=my_result_%tY-%tm-%td_%tH%tM%tS
osCollectorScript=./misc/os_collector_linux.py
osCollectorInterval=1

//收集OS负载信息：eth0和vdb必须匹配服务器网卡和磁盘名称
//osCollectorSSHAddr=user@dbhost
osCollectorDevices=net_eth0 blk_vdb
```

其中几个重要参数建议按如下规则配置：
- **warehouses**：每个仓库初始大小约100M。建议测试数据量为数据库服务器物理内存的**2到5倍**大小；
- **loadWorkers**：建议配置为CPU核数；
- **terminals**：建议配置为CPU核数的**2到6倍**大小。

`benchmarksql-5.0/run/sql.common`路径下是BenchmarkSQL用于创建测试数据的SQL脚本，可以按需调整。
```bash
[root@primarydb benchmarksql-5.0]# ls run/sql.common/
buildFinish.sql  foreignKeys.sql  indexCreates.sql  indexDrops.sql  tableCreates.sql  tableDrops.sql  tableTruncates.sql
```

# 数据库用户配置
创建BenchmarkSQL测试用户：
```sql
SQL> create tablespace benchmarksql;
SQL> create user benchmarksql identified by "benchmarksql" default tablespace benchmarksql;
SQL> grant connect,resource,create session,create view,create job,create synonym to benchmarksql;
SQL> alter user benchmarksql quota unlimited on benchmarksql;

SQL> alter tablespace benchmarksql add datafile;
SQL> alter tablespace benchmarksql add datafile;
SQL> alter tablespace benchmarksql add datafile;

SQL> select file_name, tablespace_name, bytes/1024/1024 total_mb,
maxbytes/1024/1024 max_mb, autoextensible
from dba_data_files where tablespace_name='BENCHMARKSQL';
```

# BenchmarkSQL压测
## 装载测试数据
使用`runDatabaseBuild.sh`脚本生成测试数据：
```bash
[root@primarydb benchmarksql-5.0]# cd run
[root@primarydb run]# cp props.ora ora.properties
[root@primarydb run]# ./runDatabaseBuild.sh ora.properties
```

## TPC-C压测（固定事务数量）
修改`ora.properties`中的测试模式为固定事务数量：
```bash
runTxnsPerTerminal=10
runMins=0
```

使用`runBenchmark.sh`脚本进行测试：
```bash
[root@primarydb run]# ./runBenchmark.sh ora.properties
...
...
18:29:53,870 [Thread-15] INFO   jTPCC : Term-00, Measured tpmC (NewOrders) = 140.32
18:29:53,870 [Thread-15] INFO   jTPCC : Term-00, Measured tpmTOTAL = 289.63
18:29:53,870 [Thread-15] INFO   jTPCC : Term-00, Session Start     = 2023-04-15 18:29:20
18:29:53,870 [Thread-15] INFO   jTPCC : Term-00, Session End       = 2023-04-15 18:29:53
18:29:53,870 [Thread-15] INFO   jTPCC : Term-00, Transaction Count = 160
```
执行事务的总数等于`terminals`与`runTxnsPerTerminal`的乘积。

测试过程中，可以打开另一个终端执行`htop`命令来查看进程的资源消耗情况。

测试结束后，会在当前路径下生成一个以`ora.properties`中参数`resultDirectory`规则命名的结果目录。

## TPC-C压测（固定时长）
修改`ora.properties`中的测试模式为固定时长：
```bash
runTxnsPerTerminal=0
runMins=180
```

销毁测试数据后重新加载测试：
```bash
[root@primarydb run]# ./runDatabaseDestroy.sh ora.properties
[root@primarydb run]# ./runDatabaseBuild.sh ora.properties
[root@primarydb run]# ./runBenchmark.sh ora.properties
```

## 生成测试报告
使用`runBenchmark.sh`脚本来生成测试报告（需要安装R语言）：
```bash
[root@primarydb run]# ls my_result_2023-04-15_182919/
data  run.properties

[root@primarydb run]# ./generateReport.sh my_result_2023-04-15_182919/
Generating my_result_2023-04-15_182919//tpm_nopm.png ... OK
Generating my_result_2023-04-15_182919//latency.png ... OK
Generating my_result_2023-04-15_182919//cpu_utilization.png ... OK
Generating my_result_2023-04-15_182919//dirty_buffers.png ... OK
Generating my_result_2023-04-15_182919//blk_vdb_iops.png ... OK
Generating my_result_2023-04-15_182919//blk_vdb_kbps.png ... OK
Generating my_result_2023-04-15_182919//net_eth0_iops.png ... OK
Generating my_result_2023-04-15_182919//net_eth0_kbps.png ... OK
Generating my_result_2023-04-15_182919//report.html ... OK

[root@primarydb run]# ls my_result_2023-04-15_182919/
blk_vdb_iops.png  cpu_utilization.png  dirty_buffers.png  net_eth0_iops.png  report.html     tpm_nopm.png
blk_vdb_kbps.png  data                 latency.png        net_eth0_kbps.png  run.properties
```
其中，`report.html`是生成的测试报告，png文件是报告中包含的图片。

报告内容大致如下。

![1](https://img-blog.csdnimg.cn/1b5a9955616e4d8c9990ac3e874e97fd.png#pic_center)

![2](https://img-blog.csdnimg.cn/0b66664c6d3643ce8f2ee501fc663d25.png#pic_center)

![3](https://img-blog.csdnimg.cn/27d43563da004935b86fe2cb8c10ce0e.png#pic_center)

![4](https://img-blog.csdnimg.cn/3d770a9cafbf4464a179347f6839c7fd.png#pic_center)




