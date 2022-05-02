@[TOC](TDSQL MySQL云数据库DCN同步技术)

# 概述
![在这里插入图片描述](https://img-blog.csdnimg.cn/72efe56a9d0c40468ec361c202df12a1.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAU2ViYXN0aWVuMjM=,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


利用**主从复制**+**GTID**的特性实现**异地数据同步与读写分离**。下面是实现细节与不同于常规方案的特性。

>**实现背景**
是为了将分属两个不同集群的实例，建立同步关系。备实例会自动选择主实例中延迟较小的**备机**建立同步，当该主实例备机发生故障时，会自动与另一个备机建立同步关系。DCN（Data Communication Network）同步建立后，主实例可写，备实例只读。这可作为一种异地容灾方案，也可作为一种异地读写方案，参照图1。

>**图1**
![在这里插入图片描述](https://img-blog.csdnimg.cn/a77d003414534041a9a2c519b0815ec1.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAU2ViYXN0aWVuMjM=,size_11,color_FFFFFF,t_70,g_se,x_16)


# 技术原理
## 级联复制
DCN技术分为两步：
第一部分：在主机房一主两备中，master提交事务后，写入binlog，通过mysql主从复制协议，master机将binlog传输到任意slave机，然后slave机回放relay log，最终完成主从复制。

第二部分：如图1，主备机房完成建立DCN同步后，备机房的master机会从主机房中**主备延迟最小**的slave机上的拉取binlog，随后回放binlog。

>**图2**
![在这里插入图片描述](https://img-blog.csdnimg.cn/1bbc1fa6fa5c401488031226a7c0051c.png)

如果主机房当前已经建立DCN同步的slave机器故障了，会自动与另一个备机建立同步关系，如图2。

主从同步的方式实现异地容灾方案比较成熟，但仍需要解决一些核心问题。

1. 由于存在“**级联复制**”的情况，那么如何准备的计算延迟？

2. 如果实例需要进行**扩容**时，同步关系是否收到影响，作为异地读写分离的场景，级联节点数据延迟扩大如何解决？

针对上述问题：

## 计算延迟

a) 不采用 `Seconds_Behind_Master` 的值作为延迟依据，主机agent不停地向主机数据库写入带有当前**时间戳**的记录，这些记录会同步到备机数据库中备机的agent根据数据库中最新的记录与机器当前时间戳，就可以计算出实时延迟时间了，然后备机agent再将这些信息（包括实时延迟与延迟的主机信息）写入到 **zk** 中，告知其它模块，而这些信也息作为 **scheduler** 仲裁扩容的依据。

b) 如下图延迟的计算过程，在扩容的同步数据步骤中M每写入一条时间戳记录，目标实例中的所有节点都会同步到该条记录，然后上报到zk中，当scheduler发现所有节点的延迟小于**5秒**，且 **delayip** 都是M（这点主要是防止异常）时，进入到下一个流程。

>**图3**
![在这里插入图片描述](https://img-blog.csdnimg.cn/681a7c7f1ab54f74a028c34f26d56160.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAU2ViYXN0aWVuMjM=,size_11,color_FFFFFF,t_70,g_se,x_16)

## 实例扩容
实例进行扩容的时候，集群具备**自动更新DCN同步关系**的功能，并且在扩容过程中不需要人为介入，减少了延迟得影响。由程序介入，并分拆为三步：

a) 建立原实例与目标实例（此处指要扩容到的目标实例）的同步关系；

b) 检测目标实例与原实例之间的延迟，当延迟小于5s时，设置原实例只读，拒绝掉新的写入；

c) 检测目标实例与原实例之间的**GTID**。当GTID无差别时，断开原实例与目标实例之间的同步关系，并将Proxy路由切换到目标实例。

下图表示利用DCN进行集群扩容的流程：

MM表示主实例的主机，MS表示主实例备机，MSET表示主实例。SM表示备实例主机，SS表示备实例备机，SSET表示备实例。

EMM表示主实例扩容的目标实例的主机，EMS表示主实例扩容的目标实例的备机，EMSET表示主实例扩容的目标实例。ESM表示备实例扩容的目标机器的主机，ESS表示备实例扩容的目标机器的备机，ESSET表示备实例扩容的目标实例。

>**图4**
![在这里插入图片描述](https://img-blog.csdnimg.cn/1d5d3f896c594806a602d04f13ac742e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAU2ViYXN0aWVuMjM=,size_11,color_FFFFFF,t_70,g_se,x_16)

扩容流程：

（1）建立EMSET与MSET之间的扩容同步关系。

（2）检测到EMSET与MSET之间的延迟小于5秒时，断开SSET与MSET之间的DCN同步关系。(**提前断开**是为了防止后续SSET的GTID的信息不被EMSET的GTID列表包含，这里的差异信息可能是MM节点与EMSET断开后新写入ZK的时间戳记录)。

（3）确认SSET与MSET之间的DCN同步关系断开后，设置MSET为**隔离**状态（此时网关会拒绝掉所有新的连接），并向EMM节点的agent发送扩容处理任务。

（4）当EMM节点的agent接收到scheduler发送的扩容处理后，设置MM节点**只读**，并计算EMM与MM节点gtid的差值，当gtid无差值时，反馈scheduler任务处理成功。如果期间设置只读失败，反馈sheduler处理失败。

（5）Scheduler根据agent反馈进行区分处理，反馈成功进入步骤6，反馈失败进入步骤7。

（6）断开EMSET与MSET之间的扩容同步关系，返回扩容成功，并建立EMSET与SSET之间的DCN同步关系。

（7）把MSET设置为正常状态（网关正常接收新连接），返回扩容失败，并重建MSET与SSET之间的DCN关系。


# 实施前提
建立DCN同步关系前的主备集群注意事项：

- 主备集群创建实例时，应确保主、备实例的数据库版本、字符集编码、排序规则、大小写敏感、innodb_page_size等参数项一致。
- 确保主备集群中实例数量一一对应。
- 确保在主备集群所有分布式实例（Groupshrad）中实例的分片（Set）数量都有对应关系。例如，在IDC1主机房中，添加IDC2备机房部署的TDSQL集群，并建立DCN同步关系。

# 操作步骤
## 建立DCN同步
 1. 将IDC2（备）机房部署的TDSQL集群添加到IDC1（主）机房中。
 2. 在赤兔管理台主界面，点击【集群管理】>【集群设置】>【接入新集群+】，系统弹出【接入新集群】对话框。
 3. 输入IDC2备机房集群相关信息。
    -【集群名称】：输入集群名称。
    -【集群Key】：为系统自动分配。
    -【集群操作接口】：为集群OSS组件部署地址。
    -【集群类型】：选择集群部署架构方式，选择Proxy和DB合并部署在一台物理机中，或分别部署在不同物理机中。
    -【监控数据库源配置】：填写实际安装信息。
    -【监控数据库存储DB配置】：填写实际安装信息。
4. 信息填写完成后，点击【保存】，系统进入接入新集群详情页面，可以查看集群接入进度和结果。
5. 在集群中创建实例。
6. 主备实例建立DCN同步关系。 将IDC1（主）机房、IDC2（备）机房集群中，对应的主、备实例创建DCN同步关系。
   - 在赤兔管理台主界面，点击【数据同步】>【DCN同步】>【创建DCN同步】，系统弹出【创建DCN同步】对话框。
   - 选择需建立DCN同步关系的主、备集群和主、备实例。
   - 点击【确定】退出对话框，进入任务流程页面，查看DCN同步进度和结果。

## 故障切换
当IDC1（主）机房故障后，可以手动取消DCN同步关系，并且需要业务层更改连接到IDC2（备）机房中的备集群，或者自行维护更改DNS映射关系。

如果主机房源实例重启或者网络中断了，只要在网络中断期间，源实例未被传输的binlog没有被删除，待网络恢复后，DCN同步会正常进行，不需要人工干预。

主机房中建立DCN同步关系的备库故障后，会自动切换到另一台正常工作的备库作为新的目标端来进行DCN数据同步。

# OSS接口
## 取消DCN同步
**接口描述**
接口请求域名： dcdb.tencentcloudapi.com 。
取消DCN同步
默认接口请求频率限制：20次/秒。

**输入示例**

```html
https://dcdb.tencentcloudapi.com/?Action=CancelDcnJob
&InstanceId=tdsql-2rn9lmpx
&<公共请求参数>
```

**输出示例**

```html
{
  "Response": {
    "FlowId": 11211,
    "RequestId": "cbca1c8e-e123-11ea-a777-525400542aa6"
  }
}
```

## 获取实例灾备详情
**接口描述**
接口请求域名： dcdb.tencentcloudapi.com 。
获取实例灾备详情
默认接口请求频率限制：20次/秒。

**输入示例**

```html
https://dcdb.tencentcloudapi.com/?Action=DescribeDcnDetail
&InstanceId=tdsql-2rn9lmpx
&<公共请求参数>
```

**输出示例**

```html
{
  "Response": {
    "DcnDetails": [
      {
        "DcnFlag": 1,
        "DcnStatus": 0,
        "InstanceId": "tdsql-2rn9lmpx",
        "InstanceName": "tdsql-2rn9lmpx",
        "Region": "ap-guangzhou",
        "Status": 2,
        "StatusDesc": "运行中",
        "Vip": "10.0.0.24",
        "Vipv6": "",
        "Vport": 3306,
        "Zone": "ap-guangzhou-4"
      },
      {
        "DcnFlag": 2,
        "DcnStatus": 2,
        "InstanceId": "tdsql-cbr0g5v1",
        "InstanceName": "tdsql-cbr0g5v1",
        "Region": "ap-guangzhou",
        "Status": 2,
        "StatusDesc": "运行中",
        "Vip": "10.0.0.29",
        "Vipv6": "",
        "Vport": 3306,
        "Zone": "ap-guangzhou-4"
      }
    ],
    "RequestId": "cbca1c8e-e123-11ea-a777-525400542aa6"
  }
}
```



**References**
[1\] https://cloud.tencent.com/developer/article/1727253
[2\] https://cloud.tencent.com/document/product/1515/62356
[3\] https://cloud.tencent.com/document/api/557/58910
