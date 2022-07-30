
@[TOC](Oracle ADG主备切换的一般流程)

定义相关变量
```bash
#原主库
PRIMARY_IP1=
PRIMARY_IP2=
PRIMARY_DB_SID=

#备库
STANDBY1_IP1=
STANDBY1_IP2=
STANDBY1_DB_SID=
STANDBY2_IP1=
STANDBY2_IP2=
STANDBY2_DB_SID=

#待切换为新主库的备库
TARGET_IP1=
TARGET_IP2=
TARGET_DB_SID=
```

# 切换前检查
```bash
ssh - ${PRIMARY_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${PRIMARY_DB_SID}1 
	sqlplus -S / as sysdba <<EFF
	set head off feedback off
	alter database switchover to ${TARGET_DB_SID} verify;
EFF
EOF
```

# 主备切换
执行切换命令
```bash
ssh - ${PRIMARY_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${PRIMARY_DB_SID}1 
	sqlplus / as sysdba <<EFF
	alter database switchover to ${TARGET_DB_SID};
EFF
EOF
```

## 启动新的主库
打开新主库的第一个节点
```bash
ssh - ${TARGET_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${TARGET_DB_SID}1 
	sqlplus / as sysdba <<EFF
	alter database open;
EFF
EOF
```

打开新主库的第二个节点
```bash
ssh - ${TARGET_IP2} -p 22 su - oracle <<EOF
    export ORACLE_SID=${TARGET_DB_SID}2 
	sqlplus / as sysdba <<EFF
	alter database open;
	alter system set log_archive_dest_state_2='enable';
	alter system set log_archive_dest_state_3='enable'; 
EFF
EOF
```

修改新主库的角色
```bash
ssh ${TARGET_IP1} -p 22 su - root <<EOF
    srvctl modify database -db ${TARGET_DB_SID} -role PRIMARY
EOF
```


## 启动原来的主库为备库
启动数据库的两个节点
```bash
sleep 10
ssh ${PRIMARY_IP1} -p 22 su - root <<EOF
		srvctl start database -d ${PRIMARY_DB_SID}
EOF
```

修改数据库角色
```bash
ssh ${PRIMARY_IP1} -p 22 su - root <<EOF
	srvctl modify database -db ${PRIMARY_DB_SID} -role PHYSICAL_STANDBY
EOF
```

启动实时日志应用
```bash
ssh ${PRIMARY_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${PRIMARY_DB_SID}1
	sqlplus / as sysdba <<EFF
	   alter database recover managed standby database disconnect from session;
EFF
EOF
```


## 启动其他备库
启动数据库的两个节点
```bash
sleep 10
ssh ${STANDBY2_IP1} -p 22 su - root <<EOF
		srvctl start database -d ${STANDBY2_DB_SID}
EOF
```

修改数据库角色
```bash
ssh ${STANDBY2_IP1} -p 22 su - root <<EOF
	srvctl modify database -db ${STANDBY2_DB_SID} -role PHYSICAL_STANDBY
EOF
```

启动实时日志应用
```bash
ssh ${STANDBY2_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${STANDBY2_DB_SID}1
	sqlplus / as sysdba <<EFF
	   alter database recover managed standby database disconnect from session;
EFF
EOF
```


## 切换日志文件
切换新主库的日志
```bash
sleep 10
ssh ${TARGET_IP1} -p 22 su - oracle <<EOF
		export ORACLE_SID=${TARGET_DB_SID}1
		sqlplus / as sysdba <<EFF
			alter system set log_archive_dest_state_2='enable';
            alter system set log_archive_dest_state_3='enable';
			alter system switch logfile;
			alter system switch logfile;
			alter system switch logfile;
			alter system switch logfile;
EFF
EOF
```

检查新主库的状态
```bash
ssh ${TARGET_IP1} -p 22 su - oracle <<EOF
		export ORACLE_SID=${TARGET_DB_SID}1
		sqlplus / as sysdba <<EFF
			set lines 200
			select protection_mode, protection_level, database_role, open_mode, switchover_status from v\\\$database;
EFF
EOF
```

检查旧主库的状态
```bash
ssh ${PRIMARY_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${PRIMARY_DB_SID}1
	sqlplus / as sysdba <<EFF
			set lines 200
			select protection_mode, protection_level, database_role, open_mode, switchover_status from v\\\$database;
			select process,client_process,thread#,sequence#,status from v\\\$managed_standby where process like 'MRP%';
EFF
EOF
```

检查其他备库的状态
```bash
ssh ${STANDBY2_IP1} -p 22 su - oracle <<EOF
    export ORACLE_SID=${STANDBY2_DB_SID}1
	sqlplus / as sysdba <<EFF
			set lines 200
			select protection_mode, protection_level, database_role, open_mode, switchover_status from v\\\$database;
			select process,client_process,thread#,sequence#,status from v\\\$managed_standby where process like 'MRP%';
EFF
EOF
```


# 检查数据库保护模式
检查新主库保护模式和归档目标地址
```bash
ssh ${TARGET_IP1} -p 22 su - oracle <<EOF
		export ORACLE_SID=${TARGET_DB_SID}1
		sqlplus -S / as sysdba <<EFF
			set head off feedback off
			select protection_mode from v\\\\\\$database;
			select value from v\\\\\\$parameter where name='log_archive_dest_2';
			select value from v\\\\\\$parameter where name='log_archive_dest_3';
EFF
EOF
```

默认情况下，`log_archive_dest_2`和`log_archive_dest_3`与主库的归档日志复制模式为**异步模式**（**ASYNC**），数据库保护模式为最大性能模式（*max performance*）。

如果`log_archive_dest_2`或者`log_archive_dest_3`与主库的归档日志复制模式为**同步模式**（**SYNC**），将数据库保护模式切换为最大可用模式（*max availability*）。
```bash
ssh ${TARGET_IP1} -p 22 su - oracle <<EOF
		export ORACLE_SID=${TARGET_DB_SID}1
		sqlplus / as sysdba <<EFF
				alter database set standby database to maximize availability;
EFF
EOF
```


# 检查服务状态
检查新主库和旧主库中`failover_type`为`SESSION`或者`SELECT`的服务，一般是TAF（Transparent Application Failover）。
```bash
service_names=`ssh ${TARGET_IP1} -p 22 su - oracle <<EOF
		export ORACLE_SID=${TARGET_DB_SID}1
		sqlplus -S / as sysdba <<EFF
			set head off feedback off
			select name from dba_services where FAILOVER_TYPE in ('SESSION','SELECT');
EFF
EOF` 

for each_srv in ${service_names}
do
	service_status=`ssh ${TARGET_IP1} -p 22 su - root <<EOF
		srvctl status service -d ${TARGET_DB_SID} -service ${each_srv}
EOF`
	exist_status=`echo ${service_status} | sed -r "s/[[:space:]]?Last login:[a-zA-Z0-9[:space:]\:]{29}[[:space:]]?//g" | sed -r "s/on pts\/[0-9]{1}[[:space:]]?//g" | grep "not running"`
	if [ "${exist_status}A" != "A" ]; then
		ssh ${TARGET_IP1} -p 22 su - root <<EOF
			srvctl stop service -d ${TARGET_DB_SID} -service ${each_srv}
EOF
	fi
done
```




