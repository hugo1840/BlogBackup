
@[TOC](MySQL慢查询与执行计划分析)

# 慢查询分析
查看是否开启慢日志：
```sql
show variables like 'slow_query_log%';
+---------------------+----------------------------+
| Variable_name       | Value                      |
+---------------------+----------------------------+
| slow_query_log      | ON                         |
| slow_query_log_file | /mysql/log/slow.log        |
+---------------------+----------------------------+
```

查看慢查询的最小执行时长：
```sql
show variables like 'long_query_time';
```

MySQL慢日志中，需要关注的参数有：语句执行时间`Query_time`、锁占用的时间`Lock_time`、查询返回的行数`Rows_sent`、查询扫描的行数`Rows_examined`、以及对应的SQL语句。重点关注**单次执行时间长**以及**执行次数频繁**的SQL。

MySQL官方提供了一个慢查询的统计归类的工具`mysqldumpslow`，该工具会按照SQL进行统计（如果mysql-slow日志很大，可能会执行很长时间）：

```bash
# 按照平均查询时间排序，选取top10返回
$ mysqldumpslow -s at -t 10 mysql-slow.log  
# 按照慢查询个数排序，选取top10返回
$ mysqldumpslow -s c -t 10 mysql-slow.log  
```

mysqldumpslow命令显示的结果形式如下：
```
Count: 2  Time=2.79s (5s)  Lock=0.00s (0s)  Rows=500 (1000), vgos_dba[vgos_dba]@[192.168.219.196]
SELECT * FROM sms_send WHERE service_id=N GROUP BY content LIMIT N, N;
```

可以看到，SQL语句中的数字被替换为了字母N，是因为mysqldumpslow将它们视为同一种类型的语句，并进行了合并显示。

其中，`Count`表示这种类型的语句执行了几次，`Time`表示这种类型的语句执行的最大时间，`Time=2.79s (5s)`中`(5s)`是指这类型的语句执行总共花费的时间。所以上面的意思是：执行了2次，最大时间是2.79s，总共花费时间5s，lock时间0s，单次返回的结果数是500条记录，2次总共返回1000条记录。

# 执行计划分析
找到的慢查询SQL语句可以用explain命令来查看执行计划。查询计划中，重要的参数包括：
- **ID**：id相同，执行顺序由上而下；id不同，id越大优先级越高，越先被执行；
- **Select_type**: 查询类型，包括：
（1）`SIMPLE`：简单查询，不包含子查询和union；
（2）`SUBQUERY`：子查询中的第一个；
（3）`DERIVED`：派生表的SELECT（FROM字句的子查询）；
（4）`PRIMARY`：包含子查询的最外层查询、或者union的第一个查询；
（5）`UNION`：union中的第二个或后面的SELECT；
（6）`DEPENDENT UNION`：union中的第二个或后面的select语句，取决于后面的查询；
（7）`UNION RESULT`：union的结果集。

- **Table**: 查询的表，可能是数据库中的表或视图，也可能是From中的子查询；
-	**Type**: 搜索数据的方法，包括：
（1）`Null`：不需要访问索引和表；
（2）`const`：常量连接，表内最多只有一行匹配，通常用于主键或者唯一索引比较；
（3）`system`：const的一个特例，表内仅有一行满足；
（4）`eq_ref`：每次与之前的表合并行都只在该表读取一行，这是除了system和const以外最好的一种，特点是使用`=`，而且索引的所有部分都参与join且索引是主键或非空唯一键的索引；
（5）`ref`：如果每次只匹配少数行，那就是比较好的一种，使用=或《=》，可以使用左覆盖索引或非主键或非唯一键；
（6）`index`：索引树扫描。如果`Extra`中有`using index`，查询是索引覆盖的，所有数据均可从索引树获取；如果Extra中无using index，查询是以索引顺序从索引中查找数据行的全表扫描；如果Extra中using index与using where同时出现的话，则是利用索引查找键值的意思；如果单独出现，则是用读索引来代替读行，但是不用于查找；
（7）`range`：使用索引进行范围查询；
（8）`all`：全表扫描。

- **Possible_keys**: 可能使用的索引；
-	**Key**: 最终使用到的索引；
-	**Ref**: 查询的列或常量；
-	**Rows**: 需要扫描的行数（估计值）；
-	**Extra**: 额外信息，包括：
（1）`using filesort`：需要进行额外的步骤来发现如何对返回的行排序，无法使用到索引排序。出现using filesort说明sql很烂，亟需优化；
（2）`using temporary`：使用临时表存储中间结果，临时表可能在内存中也可能在磁盘中，常见于排序`order by`和分组查询`group by`，非常危险，应尽量避免；
（3）`using index`：索引中包含查询的所有列而不需要回表，可以加快查询速度；
（4）`using where`：使用了where从句来限制哪些行将与下一章表匹配或者返回给用户，连接类型是all或者index；
（5）`using index condition`：索引条件推送；
（6）`distinct`：查询到匹配的数据后停止继续搜索。


# 表结构分析
查看表上的索引：
```sql
show create table 表名;
show index from 表名;
```

查看索引字段的选择性：
```sql
select count(distinct(title))/count(*) from employess;
select count(distinct(concat(first_name,last_name)))/count(*) from employees;
```
返回值越大，说明索引的区分度越大，索引的选择性也就越好。




