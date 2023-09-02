---
tags: [oracle]
title: Oracle查看与修改隐藏参数
created: '2023-09-02T06:58:01.634Z'
modified: '2023-09-02T07:26:14.049Z'
---

Oracle查看与修改隐藏参数

# 查看隐藏参数
查看数据库中所有的隐藏参数：
```sql
SELECT a.ksppinm "Parameter", b.KSPPSTDF "Default Value",
       b.ksppstvl "Session Value", 
       c.ksppstvl "Instance Value",
       decode(bitand(a.ksppiflg/256,1),1,'TRUE','FALSE') IS_SESSION_MODIFIABLE,
       decode(bitand(a.ksppiflg/65536,3),1,'IMMEDIATE',2,'DEFERRED',3,'IMMEDIATE','FALSE') IS_SYSTEM_MODIFIABLE
FROM   x$ksppi a,
       x$ksppcv b,
       x$ksppsv c
WHERE  a.indx = b.indx
AND    a.indx = c.indx
AND    a.ksppinm LIKE '/_%' escape '/';
```

或者：
```sql
set pages 10000 lines 180
col name for a60
col value for a15
SELECT x.ksppinm name,
    y.ksppstvl value,
    y.ksppstdf isdefault,
    decode(bitand(y.ksppstvf, 7), 1, 'MODIFIED', 4, 'SYSTEM_MOD', 'FALSE') ismod,
    decode(bitand(y.ksppstvf, 2), 2, 'TRUE', 'FALSE') isadj
FROM 
    sys.x$ksppi x, sys.x$ksppcv y
WHERE 
    x.inst_id = userenv('Instance')
AND y.inst_id = userenv('Instance')
AND x.indx = y.indx
order by translate(x.ksppinm, ' _', ' ');
```

查看指定隐藏参数，在上面SQL的基础上添加条件`x.ksppinm`即可。
```sql
set pages 10000 lines 180
col name for a60
col value for a15
select x.ksppinm name,
    y.ksppstvl value,
    y.ksppstdf isdefault,
    decode(bitand(y.ksppstvf,7),1,'MODIFIED',4,'SYSTEM_MOD','FALSE') ismod,
    decode(bitand(y.ksppstvf,2),2,'TRUE','FALSE') isadj
from
    sys.x$ksppi x,sys.x$ksppcv y
where
    x.inst_id = userenv('Instance')
and y.inst_id = userenv('Instance')
and x.indx = y.indx
and x.ksppinm in (
'deferred_segment_creation',
'_optimizer_use_feedback',
'_time_based_rcv_ckpt_target')
order by translate(x.ksppinm, ' _', ' '); 
```

# 修改隐藏参数
可以通过ALTER SESSION或ALTER SYSTEM语句修改隐藏参数。部分静态参数修改后需要重启数据库生效。
```sql
alter system set "_time_based_rcv_ckpt_target"=0 scope=both;
```

**References**:
【1】https://www.modb.pro/db/610739
【2】https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/TRANSLATE.html#GUID-80F85ACB-092C-4CC7-91F6-B3A585E3A690



