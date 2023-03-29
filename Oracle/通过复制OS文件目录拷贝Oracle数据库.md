---
tags: [oracle]
title: 通过复制OS文件目录拷贝Oracle数据库
created: '2023-03-29T12:24:32.812Z'
modified: '2023-03-29T13:03:30.672Z'
---

通过复制OS文件目录拷贝Oracle数据库

>:cry:缺点：只适用于测试环境中复制数据库（源库复制时数据库不能打开）。
>:smile:优点：比RMAN恢复失败的概率小、复制的速度更快。

**注**：数据库版为11.2，文件管理模式为OMF，数据目录为`/oradatadir/${db_unique_name}`，快速恢复区目录为`/oradatadir/fast_recovery_area/${db_unique_name}`。

# 源库打包数据目录

启动源库状态为MOUNTED： 
```sql
shutdown immediate; 
startup mount;
```

拷贝参数文件和密码文件到目标库：
```bash
scp $ORACLE_HOME/dbs/init${ORACLE_SID}.ora oracle@x.x.x.x:/home/oracle/ 
scp $ORACLE_HOME/dbs/orapw${ORACLE_SID} oracle@x.x.x.x:/home/oracle/
```

打包快速恢复区： 
```bash
cd /oradatadir/fast_recovery_area/ 
tar -zpcv -f /oradatadir/backup/${db_unique_name}_fast.tar.gz ${db_unique_name}/
```

打包数据目录： 
```bash
cd /oradatadir    # 进入到要打包目录的上一级目录
tar -zpcv -f /oradatadir/backup/${db_unique_name}.tar.gz ${db_unique_name}/
```

打开源库：
```sql
alter database open;
```

传输压缩包： 
```sql
scp /oradatadir/backup/${db_unique_name}_fast.tar.gz oracle@x.x.x.x:/oradatadir/backup/ 
scp /oradatadir/backup/${db_unique_name}.tar.gz oracle@x.x.x.x:/oradatadir/backup/
```

# 目标库恢复数据库
## 修改参数文件
如果之前有创建过同名数据库，删除数据目录和快速恢复区目录，并创建pfile： 
```sql
SQL> create pfile from spfile;
```

根据从源库拷贝过来的参数文件，修改目标库参数文件： 
```bash
diff /home/oracle/init${ORACLE_SID}.ora $ORACLE_HOME/dbs/init${ORACLE_SID}.ora
```

修改参数文件的原则:
- 考虑到目标库服务器性能，目标库参数文件的内存参数配置保持不变，如`db_cache_size`、`java_pool_size`、`sga_target`等；
- 移除DG相关参数，比如`dg_broker_start`、`fal_server`等；
- 目标库缺少的重要参数，采用源库的参数文件配置，比如`db_files`、`db_create_file_dest`、`db_recovery_file_dest`等；
- 其余都有的重要参数，改为源库的参数文件配置，比如`control_files`、`db_recovery_file_dest_size`、`log_archive_dest_1`等；

其他注意事项：
- `db_unique_name`参数与数据目录和快速恢复区目录名称对应（`/oradatadir/${db_unique_name}`和`/oradatadir/fast_recovery_area/${db_unique_name}`）；
- `service_names`与`tnsnames.ora`中的`SERVICE_NAME`相同；
- `service_names`与`$ORACLE_SID`、`instance_name`、以及参数文件（`init${ORACLE_SID}.ora`）和密码文件（`orapw${ORACLE_SID}`）名称保持一致。


## 解压到数据目录
```bash
tar -zxv -f /oradatadir/backup/${db_unique_name}_fast.tar.gz -C /oradatadir/fast_recovery_area/ 
tar -zxv -f /oradatadir/backup/${db_unique_name}.tar.gz -C /oradatadir/
```

手动创建审计目录： 
```bash
mkdir -vp $ORACLE_BASE/admin/$ORACLE_SID/adump
```

## 拉起数据库
使用`init${ORACLE_SID}.ora`初始化文件拉起数据库： 
```sql
SQL> startup nomount pfile='/oracle/app/product/11204/dbs/init${ORACLE_SID}.ora'; 
SQL> alter database mount; 
SQL> alter database open;
```

删除旧的spfile，生成新的参数文件： 
```sql
SQL> create spfile from pfile;

ERROR at line 1: 
ORA-01078: failure in processing system parameters 
LRM-00123: invalid character 148 found in the input file
```
参数文件里不能有中文字符，注释里有也不行。如果出现上面的报错，删除中文字符后重新创建即可。

## 修改tnsnames.ora
添加TNS解析：
```bash
${ORACLE_SID} = 
  (DESCRIPTION = 
    (ADDRESS = (PROTOCOL = TCP)(HOST = ${hostname})(PORT = 1521)) 
    (CONNECT_DATA = 
      (SERVER = DEDICATED) 
      (SERVICE_NAME = ${service_names}) 
    ) 
  )
```





