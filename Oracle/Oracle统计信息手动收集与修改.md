---
tags: [oracle]
title: Oracle统计信息手动收集与修改
created: '2023-09-19T12:49:23.690Z'
modified: '2023-09-19T13:07:14.238Z'
---

Oracle统计信息手动收集与修改

# 检查统计信息
查看表统计信息是否过期：
```sql
select owner,table_name,partition_name from dba_tab_statistics 
where STATTYPE_LOCKED is null and STALE_STATS='YES' and owner='XXX';

select table_name,partition_name from user_tab_statistics 
where STATTYPE_LOCKED is null and STALE_STATS='YES';
```
其中，`stattype_locked`表示锁定的统计信息类型（data/cache/all），`stale_stats`表示统计信息是否过期。


# 收集统计信息
## Schema统计信息收集
收集某个用户下所有数据库对象的统计信息：
```sql
BEGIN
  dbms_stats.gather_schema_stats(ownname => 'XXX',      --用户Schema名称
		estimate_percent => 60,                         --取样率（不能超过100）
		method_opt	     => 'FOR ALL COLUMNS SIZE AUTO',
		degree 	         => 32,                         --并行度（对于只能串行的某些内部SQL不生效）
		cascade	         => true,                       --收集表统计信息的同时也收集索引统计信息
		options          => 'GATHER AUTO',              --自动收集必要的统计信息
		no_invalidate	 => FALSE);                     --使shared pool中统计信息相关的游标立即失效
END;
/
```

注意：
- `estimate_percent`：收集表统计信息取样的行数占总行数的比率。取值范围在0.000001到100之间，默认由`DBMS_STATS.AUTO_SAMPLE_SIZE`参数决定。
- `method_opt`：直方图统计信息收集方法。
  - 默认为**FOR ALL COLUMNS SIZE AUTO**，表示Oracle自己决定列直方图的收集方法；
  - 取值为**FOR ALL COLUMNS SIZE REPEAT**时，仅对已有直方图统计信息的列收集直方图。
- `degree`：统计信息收集的并行度，默认值为NULL。
  - 不要超过`parallel_max_servers`参数的值。
  - 对于只能串行的某些内部SQL不生效。
- `options`：收集哪些表的统计信息。
  - 默认为**GATHER**，收集指定Schema下所有对象的统计信息；
  - 取值为**GATHER AUTO**时，自动收集必须的统计信息，除`no_invalidate`以外的绝大多数参数都会被忽略；
  - 取值为**GATHER STALE**时，仅对统计信息已过期的对象收集统计信息；
  - 取值为**GATHER EMPTY**时，仅对没有统计信息的对象收集统计信息。
- `no_invalidate`：是否使shared pool中统计信息相关的游标立即失效。默认为`DBMS_STATS.AUTO_INVALIDATE`，数据库自己决定。
  - 取值为TRUE时，收集完统计信息后，不会使共享池中的游标立即失效，即使共享游标已经不是最优的执行计划。原有的执行计划只有在被age out或者flush out之后，才会生成新的执行计划；
  - 取值为FALSE时，收集完统计信息后，使共享池中的游标立即失效，可能在短时间内造成大量硬解析。


## 表统计信息收集
收集单张表的统计信息：
```sql
BEGIN
  dbms_stats.gather_table_stats(ownname	 => 'XXX',
         tabname          => 'XXX_TABLE_NAME',             --表名
		 partname         => 'P11',                        --分区名（可以省略）
		 estimate_percent => 60,
		 method_opt	      => 'FOR ALL COLUMNS SIZE REPEAT',
		 degree 	      => 32,
		 cascade	      => true,
		 no_invalidate 	  => FALSE);
END;
/
```

示例：
```sql
exec dbms_stats.gather_table_stats('XX_SCHEMA_NAME','XX_TABLE_NAME',cascade=>true,no_invalidate=>false);
```

# 修改统计信息
对于某些无法准确收集统计信息、并且行数基本不变的表，可以手动指定行数统计信息。

手动修改单张表的统计信息：
```sql
BEGIN
  dbms_stats.set_table_stats(ownname => 'XXX',
       tabname  => 'XXX_TABLE_NAME',          --表名
	   partname => 'P11',                     --分区名（可以省略）
       numrows  => 10000,                     --手动指定表或分区中行数的统计信息
	   no_invalidate => FALSE);
END;
/
```

示例：
```sql
exec dbms_stats.set_table_stats('XX_SCHEMA_NAME','XX_TABLE_NAME',numrows=>20000,no_invalidate=>false);
```

类似地，也可以通过**DBMS_STATS.SET_COLUMN_STATS**来手动指定列的统计信息（distcnt、density、nullcnt、avgclen等）。


# 锁定统计信息
手动修改统计信息后，如果不想表的统计信息再发生变化，还可以锁定数据库对象的统计信息。

示例：
```sql
--锁定表的统计信息
exec dbms_stats.lock_table_stats(ownname => 'XX_SCHEMA_NAME',tabname => 'XX_TABLE_NAME');

--锁定整个Schema下所有对象的统计信息
exec dbms_stats.lock_schema_stats(ownname => 'XX_SCHEMA_NAME');
```


**References**
[1] https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_STATS.html#GUID-3B3AE30F-1A34-4BFE-A326-15048F7E904F
[2] http://blog.itpub.net/17203031/viewspace-1067620/

