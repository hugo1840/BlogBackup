---
tags: [oracle]
title: 批量占满TEMP表空间问题处理与排查
created: '2023-02-08T14:17:19.947Z'
modified: '2023-02-08T15:05:05.169Z'
---

批量占满TEMP表空间问题处理与排查

数据库跑批任务占满TEMP表空间时，如果空间资源足够，可以应急扩容TEMP表空间，以避免批量失败。事后可以通过查看占用TEMP表空间高的SQL执行计划，并结合批量的业务逻辑，作进一步分析。

# 应急处置
查看temp表空间容量（OMF模式）：
```sql
--查看临时表空间文件以及是否自动扩展
set linesize 200
col file_name format a60
select tablespace_name, file_name,
bytes/1024/1024 size_mb, maxbytes/1024/1024 max_mb,autoextensible 
from dba_temp_files;

--查看临时表空间大小以及使用率
select tablespace_name, tablespace_size/1024/1024 size_mb,
free_space/1024/1024 free_mb,
round((1 - nvl(free_space,0)/tablespace_size)*100,2) used_percent 
from dba_temp_free_space;
```

扩容temp表空间（OMF模式）：
```sql
--临时表空间扩容
alter tablespace temp add tempfile;
```

# 问题排查
## 查看占用TEMP表空间高的SQL
查看指定时间段内占用TEMP表空间高的SQL：
```sql
-- v$active_session_history中记录了当前活动会话的快照信息（取样频率为每秒一次）。
-- v$sql中记录了SQL语句的子游标信息，对于正在执行的SQL，每5s会更新一次信息。
set lines 200
col sample_time for a30
select *
  from (select t.sample_time,
               s.PARSING_SCHEMA_NAME,
               t.sql_id,
               t.sql_child_number as sql_child,
               round(t.temp_space_allocated/1024/1024/1024, 2) || ' G' as temp_used,
               round(t.temp_space_allocated /
                     (select sum(decode(d.autoextensible, 'YES', d.maxbytes, d.bytes))
                        from dba_temp_files d),
                     2) * 100 || ' %' as temp_pct,
               t.program,
               t.module,
               s.SQL_TEXT
          from v$active_session_history t, v$sql s
         where t.sample_time > to_date('2023-02-08 06:00:00', 'yyyy-mm-dd hh24:mi:ss')
           and t.sample_time < to_date('2023-02-08 07:00:00', 'yyyy-mm-dd hh24:mi:ss')
           and t.temp_space_allocated is not null
           and t.sql_id = s.SQL_ID
         order by t.temp_space_allocated desc)
 where rownum < 20
 order by temp_used desc;
```

利用上面获取到的SQL_ID，可以查看SQL文本：
```sql
select sql_id, sql_fulltext from v$sql where sql_id = 'zhjw76kh3hjs';
```

## 获取目标SQL执行计划
### 方法一：EXPLAIN PLAN FOR
```sql
explain plan for <目标SQL文本>;
--例如：
--explain plan for select empno,ename,dname from scott.emp,scott.dept where emp.deptno=dept.deptno;

select * from table(dbms_xplan.display);
```

### 方法二：DBMS_XPLAN.DISPLAY_CURSOR

查看刚刚执行过的SQL的执行计划：
```sql
select * from table(dbms_xplan.display_cursor(null,null,'advanced'));
```

查看指定`sql_id`的SQL执行计划：
```sql
select * from table(dbms_xplan.display_cursor('sql_id或sql_hash_value',null,'advanced'));
--例如：
--select * from table(dbms_xplan.display_cursor('2zxtkjmt05up',null,'advanced'));
```

查看指定`sql_id`或`sql_hash_value`、以及子游标的SQL执行计划：
```sql
select sql_text,sql_id,hash_value,child_number from v$sql 
where sql_text like 'select empno,ename%';

--代入上面查到的sql_id（或hash_value）、以及child_number
select * from table(dbms_xplan.display_cursor('sql_id或sql_hash_value',
child_cursor_number,'advanced'));
```

### 方法三：DBMS_XPLAN.DISPLAY_AWR

查看指定`SQL_ID`的所有历史执行计划：
```sql
select * from table(dbms_xplan.display_awr('sql_id'));
```

查看指定`SQL_ID`在指定时间段采用的执行计划`PLAN_HASH_VALUE`：
```sql
select distinct b.begin_interval_time, a.sql_id, a.plan_hash_value
from dba_hist_sqlstat a, dba_hist_snapshot b
where sql_id='2zxtkjmt05up' 
and a.snap_id = b.snap_id
and to_char(b.begin_interval_time, 'yyyy-mm-dd hh24:mi:ss') > '2023-02-07 23:59:59';
--查看2月8号以来的执行计划
```

### 方法四：AUTOTRACE
AUTOTRACE命令的使用方法如下：
```sql
set autotrace {off|on|traceonly} [explain] [stattistics]
```

1. **SET AUTOTRACE ON**

AUTOTRACE默认是关闭的，执行`set autotrace on`可以在当前session中开启autotrace。开启autotrace后，当前session中随后所有执行的SQL不仅会显示执行结果，还会显示SQL对应的执行计划和资源消耗情况。执行`set autotrace off`可以关闭autotrace。

2. **SET AUTOTRACE TRACEONLY**

与SET AUTOTRACE ON相比，省略了SQL执行结果的具体内容，只会显示执行结果的行数、SQL执行计划和资源消耗情况。

3. **SET AUTOTRACE TRACEONLY EXPLAIN**

只显示SQL的执行计划，不会显示SQL执行结果和资源消耗情况。

4. **SET AUTOTRACE TRACEONLY STATISTICS**

只显示SQL执行时的资源消耗情况、以及执行结果的行数，不会显示SQL执行结果的具体内容，也不会显示执行计划。


