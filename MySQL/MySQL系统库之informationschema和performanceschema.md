---
tags: [mysql]
title: MySQL系统库之informationschema和performanceschema
created: '2023-07-01T05:33:18.843Z'
modified: '2023-07-01T11:12:13.248Z'
---

MySQL系统库之information_schema和performance_schema

>:dolphin:数据库版本：MySQL 8.0.30

# INFORMATION_SCHEMA
统计实例中数据量最大的十个数据库：
```sql
select table_schema, count(*) tables,
concat(round(sum(table_rows)/1000,2),'千行') table_rows,
concat(round(sum(data_length)/(1024*1024),2),'M') data_size,
concat(round(sum(index_length)/(1024*1024),2),'M') idx_size,
concat(round(sum(data_length+index_length)/(1024*1024*1024),2),'G') total_size
from information_schema.tables
where table_schema not in ('information_schema','performance_schema','mysql','sys')
group by table_schema order by sum(data_length+index_length) desc limit 10;
```
对于InnoDB表而言，实际上index_length是分配给非聚簇索引的空间大小，data_length是分配给聚簇索引的空间大小。


统计指定库下最大的十张表：
```sql
select table_schema,table_name,
concat(round(data_length/(1024*1024),2),'M') data_size,
concat(round(index_length/(1024*1024),2),'M') idx_size,
concat(round((data_length+index_length)/(1024*1024),2),'M') total_size
from information_schema.tables
where table_schema = 'testdb'
order by (data_length+index_length) desc limit 10;
```

找出所有不是InnoDB引擎的表：
```sql
select table_schema,table_name,engine 
from information_schema.tables
where engine != 'innodb' and 
table_schema not in ('information_schema','performance_schema','mysql','sys');
```

找出没有主键或唯一索引的表：
```sql
select t1.table_schema, t1.table_name
from information_schema.columns t1
join information_schema.tables t2
on t1.table_schema = t2.table_schema
and t1.table_name = t2.table_name
where t1.table_schema not in  ('information_schema','performance_schema','mysql','sys')
and t2.table_type = 'BASE TABLE'
group by t1.table_schema, t1.table_name
having group_concat(t1.column_key) not regexp 'PRI|UNI';
``````
 
# PERFORMANCE_SCHEMA
MySQL性能监控涉及到performance_schema下的5张表：
- setup_actors：新前台线程的监控初始化；
- setup_consumers：事件信息发送和存储位置；
- setup_instruments：可以收集事件的计量对象种类；
- setup_objects：可以被监控的数据库对象；
- setup_threads：计量线程的名称和属性。

## setup_actors
默认开启对所有账户的性能监控和历史性能事件收集（仅对前台进程生效）。
```sql
select * from performance_schema.setup_actors;
+------+------+------+---------+---------+
| HOST | USER | ROLE | ENABLED | HISTORY |
+------+------+------+---------+---------+
| %    | %    | %    | YES     | YES     |
+------+------+------+---------+---------+
1 row in set (0.00 sec)
```

可以直接通过INSERT或UPDATE语句更新该表中的配置。
```sql
insert into performance_schema.setup_actors (host,user,role,enabled,history)
values ('localhost','root','%','yes','no');
```

## setup_consumers
在消费阶段的过滤规则主要有12类，命名以events开头。可以大致分为以下四类：
- stages：记录了SQL语句的执行阶段，默认未开启；
- statements：记录了执行的SQL语句相关信息，包括返回行数、执行事件、索引使用情况等；
- transactions：记录了事务的相关信息，包含事务的隔离级别、状态等；
- waits：记录了IO和线程互斥器（mutex）的相关信息，默认未开启。

每个类别按照生命周期又可以继续划分为三个消费者：
- current：线程当前正在执行的事件。对于空闲的线程，为最后执行的事件；
- history：每个线程最近执行的10个事件。如果线程结束，事件会被删除；
- history_long：整个实例最近执行的10000个事件。如果线程结束，事件仍然会保留。

```sql
select * from performance_schema.setup_consumers;
+----------------------------------+---------+
| NAME                             | ENABLED |
+----------------------------------+---------+
| events_stages_current            | YES     |
| events_stages_history            | NO      |
| events_stages_history_long       | NO      |
| events_statements_cpu            | NO      |
| events_statements_current        | YES     |
| events_statements_history        | YES     |
| events_statements_history_long   | NO      |
| events_transactions_current      | YES     |
| events_transactions_history      | YES     |
| events_transactions_history_long | NO      |
| events_waits_current             | YES     |
| events_waits_history             | NO      |
| events_waits_history_long        | NO      |
| global_instrumentation           | YES     |
| thread_instrumentation           | YES     |
| statements_digest                | YES     |
+----------------------------------+---------+
16 rows in set (0.00 sec)
```

>:spider:**注**：不同的消费者之间存在依赖关系。
>- 要开启`events_xx_xx`消费者，必须先开启`global_instrumentation`和`thread_instrumentation`；
>- 要开启`events_xx_history`和`events_xx_history_long`消费者，必须先开启`events_xx_current`消费者。

使用`ps_is_consumer_enabled`函数来判断消费者是否真正启用：
```sql
select sys.ps_is_consumer_enabled('events_statements_history') as is_enabled;
```

## setup_instruments
统计可以监控的不同种类的性能指标数量：
```sql
select substring_index(name,'/',1) type_instrument,count(name) nums 
from performance_schema.setup_instruments 
group by substring_index(name,'/',1);
+-----------------+------+
| type_instrument | nums |
+-----------------+------+
| wait            |  367 |
| idle            |    1 |
| stage           |  132 |
| statement       |  216 |
| transaction     |    1 |
| memory          |  498 |
| error           |    1 |
+-----------------+------+
7 rows in set (0.00 sec)
```

可以直接更新setup_instruments表来控制某个性能指标的事件收集：通过ENABLED字段来控制是否收集性能诊断信息，TIMED字段来控制是否收集该指标的运行时间信息。

```sql
update performance_schema.setup_instruments set enabled='yes' where name like 'wait/io/file/%';
update performance_schema.setup_instruments set enabled = if(name like 'wait/io/file/%', 'no', 'yes');
```

## setup_objects
通过setup_objects来控制数据库对象的性能诊断信息的收集。默认除了系统数据库外，所有数据库对象都收集性能事件。

```sql
select * from performance_schema.setup_objects;
+-------------+--------------------+-------------+---------+-------+
| OBJECT_TYPE | OBJECT_SCHEMA      | OBJECT_NAME | ENABLED | TIMED |
+-------------+--------------------+-------------+---------+-------+
| EVENT       | mysql              | %           | NO      | NO    |
| EVENT       | performance_schema | %           | NO      | NO    |
| EVENT       | information_schema | %           | NO      | NO    |
| EVENT       | %                  | %           | YES     | YES   |
| FUNCTION    | mysql              | %           | NO      | NO    |
| FUNCTION    | performance_schema | %           | NO      | NO    |
| FUNCTION    | information_schema | %           | NO      | NO    |
| FUNCTION    | %                  | %           | YES     | YES   |
| PROCEDURE   | mysql              | %           | NO      | NO    |
| PROCEDURE   | performance_schema | %           | NO      | NO    |
| PROCEDURE   | information_schema | %           | NO      | NO    |
| PROCEDURE   | %                  | %           | YES     | YES   |
| TABLE       | mysql              | %           | NO      | NO    |
| TABLE       | performance_schema | %           | NO      | NO    |
| TABLE       | information_schema | %           | NO      | NO    |
| TABLE       | %                  | %           | YES     | YES   |
| TRIGGER     | mysql              | %           | NO      | NO    |
| TRIGGER     | performance_schema | %           | NO      | NO    |
| TRIGGER     | information_schema | %           | NO      | NO    |
| TRIGGER     | %                  | %           | YES     | YES   |
+-------------+--------------------+-------------+---------+-------+
20 rows in set (0.00 sec)
```

可以直接更新setup_objects表来控制某个数据库对象的性能指标收集。
```sql
insert into setup_objects (object_type,object_schema,object_name,enabled,timed) 
values ('TABLE','testdb','t1','YES','YES');
insert into setup_objects (object_type,object_schema,object_name,enabled,timed) 
values ('TABLE','testdb','%','NO','NO');
```
上面表示在数据库testdb中，仅对表t1收集性能事件信息。





