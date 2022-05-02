@[toc](【不积跬步无以至千里】Oracle应用导数Ora-01455报错处理)

# 问题背景
应用在使用exp工具从Oracle中导出数据时收到如下报错：

```bash
EXP-00008: ORACLE error 1455 encountered
ORA-01455: converting column overflows integer datatype
```

# 问题分析
在Oracle官网检索相关报错得到如下信息：


>**Symptoms**:
An ORA-1455 error is raised while attempting to export a table that uses more than 2^32 database blocks of space within the source database via the classic export utility (exp).

>**Cause**:
Numeric overflow for an OCI variable associated to the exp utility used when the SYS.EXU9STO table contents are referenced from within the OCI code. The issue is investigated in bug 15985925 EXP: ORA-01455: CONVERTING COLUMN OVERFLOWS INTEGER DATATYPE.

>**Solution**:
The workaround is to use the DataPump export utility (expdp) to export those tables that exceed 2^32 database blocks of space.

简单来说，就是存在大表的数据量超过了$2^{32}$个数据块，在使用**exp**工具导数时会报ORA-1455错误。解决办法是使用官方最新的导数工具**expdp**。

# 处理办法
让应用使用expdp工具导数。与exp不同的是，在使用expdp时需要先创建目录对象（Directory），并且导出的数据必须存放在该目录对象对应的操作系统目录中。

**创建本地dump目录**
```bash
mkdir -p /app/dump
chown -R scott:scott /app/dump
setfacl -R -m u:oracle:rwx /app/dump
setfacl -R -d -m u:oracle:rwx /app/dump
```

**创建目录对象**
```sql
-- create directory directory_name as 'directory_os_path';
create directory DMP_DIR as '/app/dump';
```

**授予用户权限**
```sql
-- grant read,write on directory directory_name to username;
grant read,write on directory DMP_DIR to SCOTT;
```

**检查用户权限**
```sql
set linesize 120
col grantor format a12
col grantee format a12
col table_schema format a16
col table_name format a16
col privilege format a16
select * from all_tab_privs where table_name='DMP_DIR';
```

**导出数据**
```bash
# expdp username/password DIRECTORY=directory_name DUMPFILE=file_name TABLES=table_name
expdp SCOTT/password DIRECTORY=DMP_DIR DUMPFILE=dump01.dmp TABLES=productinfo
```
