@[TOC](Oracle数据库实例：基础知识)

# 基本概念
## 数据库实例结构

## 数据库实例配置

## 可读写实例 vs 只读实例

## Oracle SID
SID（system identifier）是Oracle数据库实例在某一特定主机上的唯一标识符。在 Unix 和 Linux 平台上，Oracle数据库默认利用 `SID` 来定位初始化参数文件。参数文件可以用来定位数据库控制文件。在大多数平台上，SID的值通过环境变量 `ORACLE_SID` 来设定。

# 数据库实例启停
数据库实例为用户提供了访问数据库的途径。数据库实例和数据库本身可以处于几种不同的状态。

## 数据库和实例启动
一般来说，你需要手动启动（**start**）一个数据库实例，然后挂载（**mount**）并打开（**open**）数据库。这些步骤可以通过 SQL*PLUS 的 `STARTUP` 命令、Oracle Enterprise Manager、或者 `SRVCTL` 功能来实现。

通过 Oracle Net 来启动数据库实例必须满足以下条件：

- 数据库已经向一个 Oracle Net 监听器（listener）静态注册；
- 当前客户端使用具有 **SYSDBA** 权限的角色连接到数据库。

监听器会创建一个专门的服务器，用来启动数据库实例。下图展示了数据库从关闭状态到打开状态的过程。

>**图3 数据库实例启动流程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/img_convert/4b3b87022419fe315b1001b806a94fd8.gif#pic_center)



## 数据库和实例停止


# 检查点

# 实例恢复





References
[1\] https://docs.oracle.com/en/database/oracle/oracle-database/12.2/cncpt/oracle-database-instance.html#GUID-67247052-CE3F-44D2-BA3E-7067DEF4B6D5
[2\] https://www.cnblogs.com/xqzt/p/4832597.html
[3\] https://www.cnblogs.com/kerrycode/p/3254154.html

