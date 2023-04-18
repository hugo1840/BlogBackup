---
tags: [达梦]
title: 基于BenchmarkSQL的达梦数据库TPCC性能测试
created: '2023-04-14T14:59:50.445Z'
modified: '2023-04-18T12:41:35.462Z'
---

基于BenchmarkSQL的达梦数据库TPCC性能测试


# 安装BenchmarkSQL及其依赖
| 软件包 | 作用 |
| :--: | :--: |
| benchmarksql-5.0.zip | tpcc性能测试 |
| R-3.6.3.tar.gz | 生成压测报告 |

>:lion:**下载地址**
>- BenchmarkSQL: https://github.com/petergeoghegan/benchmarksql
>- R: https://mirror.bjtu.edu.cn/cran/src/base/R-3/R-3.6.3.tar.gz

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

检查安装情况：
```bash
[root@primarydb benchmarksql]# ant -version
Apache Ant(TM) version 1.9.4 compiled on November 5 2018

[root@primarydb benchmarksql]# java -version
openjdk version "1.8.0_362"
OpenJDK Runtime Environment (build 1.8.0_362-b08)
OpenJDK 64-Bit Server VM (build 25.362-b08, mixed mode)

[root@primarydb benchmarksql]# R --version
R version 3.6.3 (2020-02-29) -- "Holding the Windsock"
Copyright (C) 2020 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)
```

如果ant命令报如下错误：
```bash
[root@primarydb benchmarksql]# ant -version
Unable to locate tools.jar. 
Expected to find it in /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.242.b08-1.h5.ky10.x86_64/lib/tools.jar
```

解决办法是安装openjdk-devel包：
```bash
yum install java-1.8.0-openjdk-devel -y
```


## 编译BenchmarkSQL
解压安装包：
```bash
[root@primarydb benchmarksql]# unzip benchmarksql-5.0.zip
[root@primarydb benchmarksql]# ls benchmarksql-5.0/lib/
apache-log4j-extras-1.1.jar  firebird  log4j-1.2.17.jar  oracle  postgres
```

拷贝达梦数据库驱动到BenchmarkSQL的`lib/oracle`目录下：
```bash
find /opt/dmdbms -name DmJdbcDriver*
cp /opt/dmdbms/driver/jdbc/DmJdbcDriver18.jar benchmarksql-5.0/lib/oracle/
```

使用ant编译BenchmarkSQL：
```bash
[root@primarydb benchmarksql]# cd benchmarksql-5.0
[root@primarydb benchmarksql-5.0]# ant
```

# BenchmarkSQL props文件配置
`benchmarksql-5.0/run/props.xxx`是使用BenchmarkSQL进行性能测试的主要配置文件。

BenchmarkSQL默认不支持达梦数据库。这里我们让BenchmarkSQL把达梦数据库当作Oracle即可。Oracle数据库对应的测试配置文件为`props.ora`，将其拷贝为`dm.properties`编辑：
```java
db=oracle                                               //benchmarksql默认不支持达梦数据库
driver=dm.jdbc.driver.DmDriver                          //数据库驱动
conn=jdbc:dm://127.0.0.1:5236                           //数据库连接串
user=BENCHMARKSQL                                       //数据库连接用户
password=benchmarksql                                   //数据库连接用户密码

warehouses=500        //仓库数，控制测试数据量，每个仓库初始大小约为100MB
loadWorkers=8         //初始化测试数据时，往数据库中加载数据的并发进程数

terminals=40            //客户端并发连接数
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


# 数据库用户配置
按照`dm.properties`中的用户和密码在达梦数据库中创建测试用户，并保证其表空间足够容纳生成的测试数据。可以按照每个warehouse为100M大小进行估算。如果开启了归档，还要考虑归档日志占用的空间。


# BenchmarkSQL压测
## 装载测试数据
使用`runDatabaseBuild.sh`脚本生成测试数据：
```bash
[root@primarydb benchmarksql-5.0]# cd run
[root@primarydb run]# ./runDatabaseBuild.sh dm.properties
```

## TPC-C压测（固定事务数量）
修改`dm.properties`中的测试模式为固定事务数量：
```bash
runTxnsPerTerminal=10
runMins=0
```

使用`runBenchmark.sh`脚本进行测试：
```bash
[root@primarydb run]# ./runBenchmark.sh dm.properties
```
执行事务的总数等于`terminals`与`runTxnsPerTerminal`的乘积。


测试结束后，会在当前路径下生成一个以`dm.properties`中参数`resultDirectory`规则命名的结果目录。

## TPC-C压测（固定时长）
修改`dm.properties`中的测试模式为固定时长：
```bash
runTxnsPerTerminal=0
runMins=180
```

销毁测试数据后重新加载测试：
```bash
[root@primarydb run]# ./runDatabaseDestroy.sh dm.properties
[root@primarydb run]# ./runDatabaseBuild.sh dm.properties
[root@primarydb run]# ./runBenchmark.sh dm.properties
```

## 生成测试报告
使用`runBenchmark.sh`脚本来生成测试报告（需要安装R语言）：
```bash
[root@primarydb run]# ls my_result_2023-04-15_182919/
data  run.properties

[root@primarydb run]# ./generateReport.sh my_result_2023-04-15_182919/

[root@primarydb run]# ls my_result_2023-04-15_182919/
blk_vdb_iops.png  cpu_utilization.png  dirty_buffers.png  net_eth0_iops.png  report.html     tpm_nopm.png
blk_vdb_kbps.png  data                 latency.png        net_eth0_kbps.png  run.properties

[root@primarydb run]# zip -r tpcc-dm-report.zip my_result_2023-04-15_182919/
```
其中，`report.html`是生成的测试报告，png文件是报告中包含的图片。






