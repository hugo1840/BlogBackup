---
tags: [oracle, SQL优化]
title: Oracle查询转换之视图合并
created: '2022-10-21T17:03:29.480Z'
modified: '2022-10-22T10:08:28.768Z'
---

Oracle查询转换之视图合并

# 视图合并
视图合并（View Merging）是优化器处理带视图的目标SQL的一种优化手段。它是指优化器不再将目标SQL中视图的定义当作一个独立的处理单元来单独执行，而是会将其拆开，把其定义SQL语句中的**基表**拿出来与外部查询中的表合并，这样合并后的SQL将只剩下外部查询中的表和原视图中的基表，不再会有视图出现。

Oracle会确保视图合并的正确性，即合并后的SQL和原SQL在语义上一定是完全等价的。不是所有的视图都能做视图合并，这种情况下Oracle就会把视图的定义语句当作一个独立的处理单元来单独执行。

Oracle里的视图合并分为简单视图合并、外连接视图合并、复杂视图合并三种类型。**对于符合简单视图合并条件的目标SQL，Oracle始终会对其做视图合并**，而不管视图合并后的等价改写SQL的成本值是否小于原SQL。但是在Oracle 10g及其以后的版本中，**对于复杂视图合并，只有等价改写SQL的成本值小于原SQL时**，Oracle才会对目标SQL做复杂视图合并。

## 简单视图合并
简单视图合并（Simple View Merging）是指针对那些**不含外连接**、以及所带的视图的视图定义SQL语句中不含`distinct`、`group by`等聚合函数的目标SQL的视图合并。

```sql
--定义视图
> create or replace view view_1 as
select t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id
and t3.cust_gender = 'MALE';

--目标SQL
> select t1.prod_id, t1.prod_name
from products t1, view_1
where t1.prod_id = view_1.prod_id
and t1.prod_list_price > 1000;
```

如果不对视图`VIEW_1`做视图合并，那么此时只有如下四种表连接顺序：
- 表连接顺序1：(SALES -> CUSTOMERS) -> PRODUCTS
- 表连接顺序2：(CUSTOMERS -> SALES) -> PRODUCTS
- 表连接顺序3：PRODUCTS -> (SALES -> CUSTOMERS) 
- 表连接顺序4：PRODUCTS -> (CUSTOMERS -> SALES)

如果把视图`VIEW_1`拆开做视图合并，可以得到下面的等价改写形式（此处仅为示范，实际为Oracle在执行SQL时自动实现改写）：

```sql
--目标SQL
> select t1.prod_id, t1.prod_name
from products t1, 
(select t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id
and t3.cust_gender = 'MALE') view_1
where t1.prod_id = view_1.prod_id
and t1.prod_list_price > 1000;

--视图合并改写
> select t1.prod_id, t1.prod_name
from products t1, sales t2, customers t3
where t1.prod_id = t2.prod_id
and t2.cust_id = t3.cust_id
and t3.cust_gender = 'MALE'
and t1.prod_list_price > 1000;
```

如果对视图`VIEW_1`做视图合并，那么此时可以在原来四种表连接顺序的基础上新增以下两种表连接顺序：
- 表连接顺序5：CUSTOMERS -> (PRODUCTS -> SALES)
- 表连接顺序6：CUSTOMERS -> (SALES -> PRODUCTS) 

下面查看有无视图合并时目标SQL的执行计划：
```sql
--默认做简单视图合并
> select t1.prod_id, t1.prod_name
from products t1, view_1
where t1.prod_id = view_1.prod_id
and t1.prod_list_price > 1000;

执行计划
 0 | SELECT STATEMENT                | 
*1 |  HASH JOIN                      |
*2 |   VIEW                          | index$_join$_004   --//临时视图
*3 |    HASH JOIN                    |
 4 |     BITMAP CONVERSION TO ROWIDS |
*5 |      BITMAP INDEX SINGLE VALUE  | CUSTOMERS_GENDER_BIX
 6 |     INDEX FAST FULL SCAN        | CUSTOMERS_PK
*7 |   HASH JOIN                     |
*8 |    TABLE ACCESS FULL            | PRODUCTS
 9 |    PARTITION RANGE ALL          |
10 |     TABLE ACCESS FULL           | SALES

Predicate information (identified by operation Id:)
1 - access ("T2"."CUST_ID"="T3"."CUST_ID")
2 - filter ("T3"."CUST_GENDER"='MALE')
3 - access (ROWID=ROWID)
5 - access ("T3"."CUST_GENDER"='MALE')
7 - access ("T1"."PROD_ID"="T2"."PROD_ID")
8 - filter ("T1"."PROD_LIST_PRICE">1000)

--通过SQL Hint禁用视图合并
> select /*+ no_merge(view_1) */t1.prod_id, t1.prod_name
from products t1, view_1
where t1.prod_id = view_1.prod_id
and t1.prod_list_price > 1000;

执行计划
 0 | SELECT STATEMENT                  | 
*1 |  HASH JOIN                        |
*2 |   TABLE ACCESS FULL               | PRODUCTS
 3 |   VIEW                            | VIEW_1     --//未对view_1作视图合并
*4 |    HASH JOIN                      |
*5 |     VIEW                          | index$_join$_004   --//临时视图
*6 |      HASH JOIN                    | 
 7 |       BITMAP CONVERSION TO ROWIDS |
*8 |        BITMAP INDEX SINGLE VALUE  | CUSTOMERS_GENDER_BIX
 9 |       INDEX FAST FULL SCAN        | CUSTOMERS_PK
10 |     PARTITION RANGE ALL           | 
11 |      TABLE ACCESS FULL            | SALES

Predicate information (identified by operation Id:)
1 - access ("T1"."PROD_ID"="VIEW_1"."PROD_ID")
2 - filter ("T1"."PROD_LIST_PRICE">1000)
4 - access ("T2"."CUST_ID"="T3"."CUST_ID")
5 - filter ("T3"."CUST_GENDER"='MALE')
6 - access (ROWID=ROWID)
8 - access ("T3"."CUST_GENDER"='MALE')
```

可以看到，在使用简单视图合并时，优化器对目标SQL采用了表连接顺序5；禁用了视图合并时，采用的是表连接顺序4，并且此时执行计划中出现了视图`VIEW_1`。一般来说，如果Oracle没有选择对带视图的目标SQL执行视图合并的话，那么在执行计划中就会见到VIEW关键字，且该关键字对应的Name列就是视图的名称。

简单视图合并后的等价改写SQL只是让优化器有了更多的执行路径可以选择，原来的未作视图合并的执行路径还是保留了的，所以做了做了简单视图合并后的等价改写SQL的成本值一定小于或等于未作简单视图合并的原SQL的成本。

:boat:Oracle对包含视图的目标SQL做简单视图合并的一个前提条件是，该SQL所包含的视图定义SQL语句中一定不能出现以下内容（包括但不限于）：
- 集合运算符，如UNION、UNION ALL、INTERSECT、MINUS等；
- CONNECT BY子句；
- ROWNUM关键字；
- DISTINCT关键字；
- GROUP BY等聚合函数。

## 外连接视图合并
外连接视图合并（Outer Join View Merging）是指针对那些使用了**外连接**、以及所带的视图定义语句中不含`distinct`、`group by`等聚合函数的目标SQL的视图合并。这里的使用了外连接，是指外部查询的表和视图之间使用了外连接，或者视图定义语句内部使用了外连接。

外连接视图合并有一个很常用的限制，就是当目标视图和外部查询的表做外连接时，目标视图可以做外连接视图合并的前提条件是：要么该视图被作为**外连接的驱动表**，要么该视图虽然被作为外连接的被驱动表、但是它的**视图定义语句中只包含一个表**。

```sql
--创建测试视图1
> create or replace view view_1_modify as
select t2.prod_id as prod_id
from sales t2, customers t3
where t2.cust_id = t3.cust_id;

--测试一：视图作为外连接的驱动表
> select t1.prod_id,t1.prod_name
from products t1, view_1_modify
where t1.prod_id (+) =  view_1_modify.prod_id;
--//可以做外连接视图合并

--测试二：视图作为外连接的被驱动表
> select t1.prod_id,t1.prod_name
from products t1, view_1_modify
where t1.prod_id =  view_1_modify.prod_id (+);
执行计划
...
 3 | VIEW        | VIEW_1_MODIFY  |   --//未作视图合并

--创建测试视图2：视图定义语句中只包含一个表
> create or replace view view_2 as
select t2.prod_id as prod_id
from sales t2
where t2.amount_sold > 700;

--测试三：视图作为外连接的被驱动表
> select t1.prod_id,t1.prod_name
from products t1, view_2
where t1.prod_id =  view_2.prod_id (+);
--//可以做外连接视图合并
```

## 复杂视图合并
复杂视图合并（Complex View Merging）是指那些所带视图定义语句中含有`distinct`或`group by`等聚合函数的目标SQL的视图合并。和简单视图合并、外连接视图合并一样，复杂视图合并也需要把视图定义语句中的基表拿出来与外部查询中的表做连接。这种情况下，通常会先做表连接，再做group by或distinct操作。也就是说，视图定义语句中的group by和distinct操作一般会被推迟执行。

复杂视图合并的这种推迟group by和distinct操作的做法并不一定能带来执行效率的提升。如果group by和distinct操作能过滤掉绝大部分的数据、且表连接并不能有效过滤数据的话，那么先在视图内部做group by和distinct操作，然后才和外部查询中的表做表连接的执行效率就会更高一些；反之，如果group by和distinct操作并不能有效过滤数据，那么先做表连接的效率就会高一些。

在Oracle 10g及其以后的版本中，对于复杂视图合并，**只有当经过复杂视图合并后的等价改写SQL的成本值小于原SQL时**，Oracle才会对目标SQL执行复杂视图合并。

```sql
--创建测试视图
> create or replace view view_3 as
select cust_id,prod_id,sum(quantity_sold) total
from sales group by cust_id,prod_id;

--测试一：使用复杂视图合并
> select t1.cust_id,t1.cust_last_name
from customers t1, products t2, view_3 t3
where t1.cust_id = t3.cust_id
and t2.prod_id = t3.prod_id
and t3.total > 700
and t2.prod_category = 'Hardware'
and t1.cust_year_of_birth = 1977
and t1.cust_marital_status = 'married';
执行计划
Id | Operation        | Names | ... | Cost (%CPU) |
 0 | SELECT STATEMENT |       | ... | 2889 (1)    |
*1 |  FILTER          |       | ... |             |
 2 |   HASH GROUP BY  |       | ... | 2889 (1)    |
*3 |    HASH JOIN     |       | ... | 2887 (1)    |
...
```

上面的执行计划中没有出现视图`VIEW_3`，并且`HASH GROUP BY`在表连接`HASH JOIN`之后执行，因此判断使用了复杂视图合并。走复杂视图合并的执行计划的成本值为2889。

```sql
--测试二：不使用复杂视图合并
> select /*+ no_merge(t3) */t1.cust_id,t1.cust_last_name
from customers t1, products t2, view_3 t3
where t1.cust_id = t3.cust_id
and t2.prod_id = t3.prod_id
and t3.total > 700
and t2.prod_category = 'Hardware'
and t1.cust_year_of_birth = 1977
and t1.cust_marital_status = 'married';
执行计划
Id | Operation           | Names  | ... | Cost (%CPU) |
 0 | SELECT STATEMENT    |        | ... | 10407 (2)   |
...
 4 |     VIEW            | VIEW_3 | ... | 10405 (2)   |
*5 |      FILTER         |        | ... |             |
 6 |       HASH GOURP BY |        | ... | 10405 (2)   |
...
```

上面的执行计划中出现了VIEW关键字和对应的视图`VIEW_3`，并且`HASH GROUP BY`在视图定义内部执行，因此判断没有使用复杂视图合并。没有走复杂视图合并的执行计划的成本值为10407。

实际上，在做复杂视图合并时，Oracle内部对上面的目标SQL进行了如下的等价改下：
```sql
--原SQL
> select t1.cust_id,t1.cust_last_name
from customers t1, products t2, 
(select cust_id,prod_id,sum(quantity_sold) total
from sales 
group by cust_id,prod_id) view_3 
where t1.cust_id = view_3.cust_id
and t2.prod_id = view_3.prod_id
and view_3.total > 700
and t2.prod_category = 'Hardware'
and t1.cust_year_of_birth = 1977
and t1.cust_marital_status = 'married';

--等价改写
> select t1.cust_id,t1.cust_last_name
from customers t1, products t2, sales t3
where t1.cust_id = t3.cust_id
and t2.prod_id = t3.prod_id
and t2.prod_category = 'Hardware'
and t1.cust_year_of_birth = 1977
and t1.cust_marital_status = 'married'
group by t1.cust_id,t1.cust_last_name,t3.cust_id,t3.prod_id,t1.rowid,t2.rowid
having sum(t3.quantity_sold) > 700;
```

>:carrot:在Oracle数据库中，通过计算查询转换前后的成本来决定是否做查询转换，是受隐含参数`_OPTIMIZER_COST_BASED_TRANSFORMATION`控制的。该隐含参数可以在session级别和系统级别动态修改，其默认值是**LINEAR**，表示默认情况下开启该特性。
>
>:rabbit:在Oracle数据库中，是否允许做复杂视图合并是受隐含参数`_COMPLEX_VIEW_MERGING`控制的。其默认值为**TRUE**，表示默认情况下允许复杂视图合并。

# 星型转换
星型转换（Star Transformation）是优化器处理表连接方法为星型连接的目标SQL时的一种优化手段，它的核心是将原星型连接中针对各个**维度表**的限制条件，通过等价改写的方式以额外的**子查询**施加到**事实表**上，然后再通过对事实表上各**连接列**上已存在的位图索引间的**位图操作**（如按位与、按位或），来达到有效减少事实表上待访问的数据量，避免对事实表做全表扫描的目的，从而提高目标SQL的执行效率。

参数`STAR_TRANSFORMATION_ENABLED`用于控制是否启用星型转换，可以在session级别和系统级别动态修改。默认值为**FALSE**。我们可以把它的值修改为`TRUE`或者`TEMP_DISABLE`，两者都表示启用星型转换。它们的唯一区别是，在执行星型转换时`TEMP_DISABLE`**不**会考虑创建临时表，而`TRUE`会。




