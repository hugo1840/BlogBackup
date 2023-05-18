---
tags: [oracle]
title: 实用技巧：让SQLPlus变得好用的几个配置
created: '2023-05-18T11:37:27.348Z'
modified: '2023-05-18T12:20:26.008Z'
---

实用技巧：让SQLPlus变得好用的几个配置

# READLINE + RLWRAP
初次使用SQLPlus的人通常都会觉得这个客户端工具很难用，通过配置`readline + rlwrap`可以使我们像使用MySQL客户端一样地使用SQLPlus以及RMAN。不仅支持Backspace删除，同时还支持上下方向键查看执行过的历史SQL。


## 安装RLWRAP
安装readline工具:
```bash
[root@localhost ~]# yum install -y readline*
```

安装rlwrap工具：
```bash
[root@localhost ~]# tar -xvf rlwrap-0.45.2.tar.gz
[root@localhost ~]# cd rlwrap-0.45.2/
[root@localhost ~]# ./configure && make && make install
```

>:dolphin:**rlwrap下载**：https://github.com/hanslub42/rlwrap/releases/tag/v0.45.2


## 使用RLWRAP
为Oracle配置rlwrap工具：
```bash
[oracle@localhost ~]$ cat .bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin
export PATH

# 配置Oracle环境变量
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/version/db_1
export ORACLE_SID=orclcdb
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export PATH=$PATH:$ORACLE_HOME/bin:$JAVA_HOME/bin
export PATH=$JAVA_HOME/bin:$ORACLE_HOME/bin:$PATH

# 为Oracle命令配置配置别名
alias sysdba='rlwrap sqlplus / as sysdba'
alias rman='rlwrap rman'
alias lsnrctl='rlwrap lsnrctl'
alias asmcmd='rlwrap asmcmd'
alias adrci='rlwrap adrci'
```

使配置生效：
```bash
[oracle@localhost ~]$ . .bash_profile
#或者 source .bash_profile
```


# SQLPlus提示符配置
默认登录后，SQLPlus的提示符是下面的样子，没有任何有用信息：
```sql
SQL>
```

修改配置文件`$ORACLE_HOME/sqlplus/admin/glogin.sql`，在末尾添加以下内容：
```sql
define_editor=vi
set timing on
set serveroutput on size 100000
set linesize 100
set trimspool on
set long 5000
set termout off
default gname=idle
column global_name new_value gname
SELECT lower(USER) || '@' ||upper(instance_name)||'('||nvl(UTL_INADDR.GET_HOST_ADDRESS, SYS_CONTEXT('userenv', 'ip_address'))||')' GLOBAL_NAME FROM v$instance;
set sqlprompt '&gname> '
set termout on
```

再次登录SQLPlus试试看：
```sql
[oracle@localhost ~]$ sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Thu May 18 08:09:06 2023
Version 19.3.0.0.0

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

sys@ORCLCDB(127.0.0.1)>
sys@ORCLCDB(127.0.0.1)>
```
SQLPlus的提示符中显示了当前连接的用户、实例名称、以及服务器IP信息。

:smile:是不是很棒呀~




