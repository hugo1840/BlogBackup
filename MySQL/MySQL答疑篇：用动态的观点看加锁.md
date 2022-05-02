@[TOC](MySQL答疑篇：用动态的观点看加锁)

为了方便你理解，我们再一起复习一下加锁规则。这个规则中，包含了**两个“原则”、两个“优化”和一个“bug”**：
- 原则 1：加锁的基本单位是 next-key lock。希望你还记得，next-key lock 是**前开后闭**区间。
- 原则 2：查找过程中**访问到的**对象才会加锁。
- 优化 1：索引上的**等值查询**，给**唯一索引**加锁的时候，next-key lock 退化为**行锁**。
- 优化 2：索引上的**等值查询**，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为**间隙锁**。
- 一个 bug：**唯一索引**上的**范围查询**会访问到不满足条件的第一个值为止。

考虑下面这个表t：

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

# 不等号条件里的等值查询
我们一起来看下这个例子，分析一下这条查询语句的加锁范围：

```sql
begin;
select * from t where id>9 and id<12 order by id desc for update;
```


# 等值查询的过程


# 怎么看死锁


# 怎么看锁等待


# update的例子
