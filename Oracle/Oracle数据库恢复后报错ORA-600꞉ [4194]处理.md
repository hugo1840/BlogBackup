---
tags: [oracle]
title: 'Oracle数据库恢复后报错ORA-600: [4194]处理'
created: '2023-11-04T01:52:54.251Z'
modified: '2023-11-04T02:01:26.469Z'
---

Oracle数据库恢复后报错ORA-600: [4194]处理

# 故障现象

**现象**：完成NBU带库恢复后，测试库打开后几分钟就会自己宕机挂掉。

告警日志报错如下：
```bash
Errors in file /oracle/app/diag/rdbms/ORCL_0/ORCL/trace/ORCL_smon_201857.trc  (incident=592157):
ORA-00600: internal error code, arguments: [4194], [546.27.149175], [0], [], [], [], [], [], [], [], [], []
Incident details in: /oracle/app/diag/rdbms/ORCL_0/ORCL/incident/incdir_592157/ORCL_smon_201857_i592157.trc
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
Mon Oct 30 09:17:09 2023
PMON (ospid: 201781): terminating the instance due to error 474
System state dump requested by (instance=1, osid=201781 (PMON)), summary=[abnormal instance termination].
System State dumped to trace file /oracle/app/diag/rdbms/ORCL_0/ORCL/trace/ORCL_diag_201796_20231030091710.trc
Errors in file /oracle/app/diag/rdbms/ORCL_0/ORCL/trace/ORCL_ora_203196.trc:
ORA-00474: SMON 进程因错误而终止
ORA-00600: 内部错误代码, 参数: [4194], [u do not have the SHARED lock on this object.], [], [], [], [], [], [], [], [], [], []
ORA-00600: 内部错误代码, 参数: [4194], [ unlock objec], [], [], [], [], [], [], [], [], [], []
Errors in file /oracle/app/diag/rdbms/ORCL_0/ORCL/trace/ORCL_ora_203196.trc:
ORA-00474: SMON 进程因错误而终止
```

在MOS查询`ORA-00600: internal error code, argument: [4194]`这个报错，得到的解释如下（Doc ID 39283.1）：
```bash
A mismatch has been detected between Redo records and rollback (Undo) records.
...
This error may indicate a rollback segment corruption.
...
This may require a recovery from a database backup depending on the situation.
```

>:star: 具体解决办法可以参考 Step by step to resolve ORA-600 4194 4193 4197 on database crash (Doc ID 1428786.1)。

以下是我的处理过程。


# 处理办法
## 重建UNDO表空间

检查控制文件和数据文件头中记录的最新的SCN：
```sql
idle> startup mount;

SQL> col checkpoint_change# for 999999999999999
SQL> select distinct checkpoint_change#  from v$datafile;  --控制文件中记录的最后一次checkpoint时的SCN

CHECKPOINT_CHANGE#
------------------
     1053731346332

SQL> select distinct checkpoint_change# from v$datafile_header;  --数据文件头中记录的SCN

CHECKPOINT_CHANGE#
------------------
     1053731346332
```
发现控制文件和数据文件头中记录的SCN是一致的，考虑重建UNDO表空间即可。

生成一个初始化参数文件：
```sql
SQL> create pfile='initORCL_new.ora' from spfile;

File created.
```

修改pfile，修改UNDO管理为手动模式，存储在SYSTEM表空间中，并设置10513事件禁用事务恢复：
```bash
[oracle@dbhost dbs]$ cat initORCL_new.ora | grep undo
*._optimizer_undo_cost_change='11.2.0.4'
*._undo_autotune=FALSE
*.undo_retention=10800
*.undo_tablespace='UNDOTBS1'

[oracle@dbhost dbs]$ vi initORCL_new.ora 
[oracle@dbhost dbs]$ cat initORCL_new.ora | grep undo
*._optimizer_undo_cost_change='11.2.0.4'
*._undo_autotune=FALSE
*.undo_management='MANUAL'
*.undo_retention=10800
*.undo_tablespace='SYSTEM'
*.event='10513 trace name context forever, level 2'
```
**注**：我自己实际操作过程中没有设置10513事件，可能会导致ORA-600 [4137]报错，后面会提到。


使用pfile启动数据库：
```sql
shutdown immediate;
startup pfile='initORCL_new.ora';
```
不能有报错，否则要单独对报错进行处理。


创建新的UNDO表空间：
```sql
SQL> show parameter undo

NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
_optimizer_undo_cost_change	     string		    11.2.0.4
_undo_autotune			     boolean		    FALSE
undo_management 		     string		    MANUAL
undo_retention			     integer		    10800
undo_tablespace 		     string		    SYSTEM

SQL> CREATE UNDO TABLESPACE UNDOTBS2;

Tablespace created.

SQL> alter tablespace undotbs2 add datafile;
alter tablespace undotbs2 add datafile;
alter tablespace undotbs2 add datafile;
alter tablespace undotbs2 add datafile;
alter tablespace undotbs2 add datafile;
alter tablespace undotbs2 add datafile;
alter tablespace undotbs2 add datafile;

Tablespace altered.

SQL> select file_name,sum(bytes)/1024/1204/1204 from dba_data_files where tablespace_name like 'UNDOTBS%' group by file_name;

FILE_NAME											     SUM(BYTES)/1024/1204/1204
---------------------------------------------------------------------------------------------------- -------------------------
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmskcjmq_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmqyxqqg_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs2_lmy7b0py_.dbf								    .070639397
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmqyo3pk_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lms93876_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs2_lmy7br2f_.dbf								    .070639397
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmrb0qkx_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmqytnn7_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmsb8plf_.dbf								    46.2942131
/oradata/ORCL_0/datafile/o1_mf_undotbs1_lmsl7d0o_.dbf								    46.2942131
...
```

再次修改pfile，将UNDO管理模式设置为自动，UNDO表空间设置为新建的UNDOTBS2：
```bash
[oracle@dbhost dbs]$ vi initORCL_new.ora
[oracle@dbhost dbs]$ cat initORCL_new.ora | grep undo
*._optimizer_undo_cost_change='11.2.0.4'
*._undo_autotune=FALSE
*.undo_management='AUTO'
*.undo_retention=10800
*.undo_tablespace='UNDOTBS2'
```

重启数据库：
```sql
SQL> shutdown immediate;
SQL> create spfile from pfile='initORCL_new.ora';
SQL> startup;   

SQL> show parameter undo

NAME				     TYPE		    VALUE
------------------------------------ ---------------------- ------------------------------
_optimizer_undo_cost_change	     string		    11.2.0.4
_undo_autotune			     boolean		    FALSE
undo_management 		     string		    AUTO
undo_retention			     integer		    10800
undo_tablespace 		     string		    UNDOTBS2
```

检查ALERT日志，发现新的报错ORA-600 [4137]（这里可能是没有设置10513事件才会出现的报错）：
```bash
[oracle@dbhost ~]$ tail -n300 /oracle/app/diag/rdbms/ORCL_0/ORCL/trace/alert_ORCL.log
...
Sweep [inc2][624461]: completed
Sweep [inc2][624460]: completed
ORACLE Instance ORCL (pid = 36) - Error 600 encountered while recovering transaction (546, 27).
Errors in file /oracle/app/diag/rdbms/ORCL_0/ORCL/trace/ORCL_smon_224148.trc  (incident=640172):
ORA-00600: internal error code, arguments: [4137], [546.27.149175], [0], [0], [], [], [], [], [], [], [], []
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
ORACLE Instance ORCL (pid = 36) - Error 600 encountered while recovering transaction (546, 27).
Mon Oct 30 11:14:27 2023
Sweep [inc][640172]: completed
Sweep [inc][624467]: completed
```


## ORA-600 [4137]报错

查询MOS可知报错 **ORA-600 [4137]** 的解释如下：
```bash
There is a mismatch between the XID in the undo segment header and the XID in the undo block
during rollback or transaction recovery.  

This would indicate a corrupted rollback segment
```

尝试删除旧的UNDO：
```sql
SQL> drop tablespace UNDOTBS1 including contents and datafiles;
drop tablespace UNDOTBS1 including contents and datafiles
*
ERROR at line 1:
ORA-01548: active rollback segment '_SYSSMU314_3300756365$' found, terminate dropping tablespace

SQL> select tablespace_name, status, segment_name from dba_rollback_segs where status != 'OFFLINE';

TABLESPACE_NAME 					     STATUS			      SEGMENT_NAME
------------------------------------------------------------ -------------------------------- ------------------------------
SYSTEM							     ONLINE			      SYSTEM
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU546_811175239$
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU360_2198386275$
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU347_654930751$
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU314_3300756365$
UNDOTBS2						     ONLINE			      _SYSSMU1274_2513395007$
UNDOTBS2						     ONLINE			      _SYSSMU1273_1341585299$
UNDOTBS2						     ONLINE			      _SYSSMU1272_34058637$
UNDOTBS2						     ONLINE			      _SYSSMU1271_4040385653$
UNDOTBS2						     ONLINE			      _SYSSMU1270_1270536444$
UNDOTBS2						     ONLINE			      _SYSSMU1269_402143936$
UNDOTBS2						     ONLINE			      _SYSSMU1268_4100704859$
UNDOTBS2						     ONLINE			      _SYSSMU1267_2250107085$
UNDOTBS2						     ONLINE			      _SYSSMU1266_94778785$
UNDOTBS2						     ONLINE			      _SYSSMU1265_4196515074$

15 rows selected.
```
不能删除的原因是UNDOTBS1还有未下线的段，状态为**PARTLY AVAILABLE**。


过了一会儿数据库又宕机了，检查发现是生成了大量trace文件占满了oracle目录。可能是因为没有设置10513事件，大量事务恢复失败的日志不停地刷到trace文件中。
 ```bash
[oracle@dbhost trace]$ du -sh $ORACLE_BASE/diag/rdbms/ORCL_0/${ORACLE_SID}/ 
5.2G	/oracle/app/diag/rdbms/ORCL_0/ORCL/
[oracle@dbhost trace]$ du -sh $ORACLE_BASE/diag/rdbms/ORCL_0/${ORACLE_SID}/trace/ 
31G	/oracle/app/diag/rdbms/ORCL_0/ORCL/trace/
[oracle@dbhost trace]$ df -h | grep oracle
/dev/mapper/VolGroup-lv_oracle    50G   50G  848M  99% /oracle
```


通过隐含参数忽略UNDOTBS1中未下线的回滚段：
```sql
SQL> select tablespace_name, status, segment_name from dba_rollback_segs 
where tablespace_name='UNDOTBS1' and status != 'OFFLINE';

TABLESPACE_NAME 					     STATUS			      SEGMENT_NAME
------------------------------------------------------------ -------------------------------- ------------------------------
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU546_811175239$
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU360_2198386275$
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU347_654930751$
UNDOTBS1						     PARTLY AVAILABLE		      _SYSSMU314_3300756365$
```

修改`initORCL_new.ora`添加隐含参数：
```bash
*._corrupted_rollback_segments='_SYSSMU546_811175239$','_SYSSMU360_2198386275$','_SYSSMU347_654930751$','_SYSSMU314_3300756365$'
```

启动数据库：
```sql
SQL> create spfile from pfile='initORCL_new.ora';
SQL> startup;

--确认是否已忽略
SQL> select segment_name,tablespace_name,status from dba_rollback_segs where tablespace_name='UNDOTBS1' and status != 'OFFLINE';

SEGMENT_NAME		       TABLESPACE_NAME						    STATUS
------------------------------ ------------------------------------------------------------ --------------------------------
_SYSSMU314_3300756365$	       UNDOTBS1 						    NEEDS RECOVERY
_SYSSMU347_654930751$	       UNDOTBS1 						    NEEDS RECOVERY
_SYSSMU360_2198386275$	       UNDOTBS1 						    NEEDS RECOVERY
_SYSSMU546_811175239$	       UNDOTBS1 						    NEEDS RECOVERY
```
这里上面的SQL最好是没有任何输出，但是实际测试发现UNDO段状态变成**NEEDS RECOVERY**也可以删除UNDOTBS1表空间。


删除旧的UNDO表空间：
```sql
SQL> drop tablespace UNDOTBS1 including contents and datafiles;

Tablespace dropped.

SQL> select segment_name,tablespace_name,status from dba_rollback_segs where tablespace_name='UNDOTBS1' and status != 'OFFLINE';

no row selected.
```

检查ALERT日志有无报错。


## 可能的扫尾工作

停止数据库，以便移除掉10513事件和`_corrupted_rollback_segments`隐含参数：
```sql
SQL> shutdown immediate;
SQL> create pfile from spfile;
```

移除pfile中的下列参数：
```bash
##*.event='10513 trace name context forever, level 2'
##*._corrupted_rollback_segments"='_SYSSMU546_811175239$','_SYSSMU360_2198386275$','_SYSSMU347_654930751$','_SYSSMU314_3300756365$'
```

重建spfile并拉起数据库：
```sql
SQL> create spfile from pfile='initORCL.ora';
SQL> startup;
```

检查ALERT日志有无报错。可能遇到的TEMP表空间为空的报错：
```bash
*********************************************************************
WARNING: The following temporary tablespaces contain no files.
         This condition can occur when a backup controlfile has
         been restored.  It may be necessary to add files to these
         tablespaces.  That can be done using the SQL statement:
 
         ALTER TABLESPACE <tablespace_name> ADD TEMPFILE
 
         Alternatively, if these temporary tablespaces are no longer
         needed, then they can be dropped.
           Empty temporary tablespace: TEMP
*********************************************************************
```

为临时表空间添加临时文件即可：
```sql
SQL> alter tablespace temp add tempfile; 
```


**REFs**
【1】https://www.modb.pro/db/48609
【2】https://blog.csdn.net/sinat_36757755/article/details/130333335
【3】https://www.modb.pro/db/45428
【4】Step by step to resolve ORA-600 4194 4193 4197 on database crash (Doc ID 1428786.1)








