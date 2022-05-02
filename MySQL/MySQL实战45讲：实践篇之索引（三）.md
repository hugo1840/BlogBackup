@[TOC](MySQL实战45讲：实践篇之索引（三）)

# 基本概念
# 普通索引和唯一索引怎么选择
# MySQL为什么有时会选错索引
# 怎么给字符串字段加索引
## 实验背景
假设有一个支持邮箱登录的系统，用户表定义如下

```sql
mysql> create table SUser(
ID bigint unsigned primary key,
email varchar(64), 
... 
)engine=innodb; 
```

MySQL 是支持前缀索引的，也就是说，可以定义字符串的一部分作为索引。默认地，如果创建索引的语句不指定前缀长度，那么索引就会包含整个字符串。

```sql
#不指定索引前缀长度
mysql> alter table SUser add index index1(email);
#索引前缀长度为6
mysql> alter table SUser add index index2(email(6));
```

使用前缀索引后，可能会导致查询语句读数据的次数变多。定义好前缀长度，就可以做到既节省空间，又不用额外增加太多的查询成本。那么当要给字符串创建前缀索引时，有什么方法能够确定我应该使用多长的前缀呢？

实际上，我们在建立索引时关注的是区分度，区分度越高越好。因为区分度越高，意味着重复的键值越少。因此，我们可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。

计算 email 列有多少个不同的值：

```sql
mysql> select count(distinct email) as L from SUser;
```

然后，依次选取不同长度的前缀来看这个值，比如我们要看一下 4~7 个字节的前缀索引，可以用这个语句：

```sql
mysql> select 
  count(distinct left(email,4)）as L4,
  count(distinct left(email,5)）as L5,
  count(distinct left(email,6)）as L6,
  count(distinct left(email,7)）as L7,
from SUser;
```

当然，使用前缀索引很可能会损失区分度，所以需要预先设定一个可以接受的损失比例，比如 5%。然后，在返回的 L4~L7 中，找出不小于 `L * 95%` 的值，假设这里 L6、L7 都满足，你就可以选择前缀长度为 6。

## 前缀索引对覆盖索引的影响
考虑下面的 SQL 语句

```sql
select id,email from SUser where email='zhangssxyz@xxx.com';
```

如果使用 index1（即 email 整个字符串的索引结构）的话，可以利用覆盖索引，从 index1 查到结果后直接就返回了，不需要回到 ID 索引再去查一次。而如果使用 index2（即 email(6) 索引结构）的话，就不得不回到 ID 索引再去判断 email 字段的值。

即使将 index2 的定义修改为 email(18) 的前缀索引，这时候虽然 index2 已经包含了所有的信息，但 InnoDB 还是要回到 id 索引再查一下，因为系统并不确定前缀索引的定义是否截断了完整信息。

## 其他方式
遇到前缀的区分度不够好的情况时，我们要怎么办呢？比如，我们国家的身份证号，一共 18 位，其中前 6 位是地址码，所以同一个县的人的身份证号前 6 位一般会是相同的。

按照前面的方法，可能需要创建长度为 12 以上的前缀索引，才能够满足区分度要求。但是，索引选取的越长，占用的磁盘空间就越大，相同的数据页能放下的索引值就越少，搜索的效率也就会越低。

如果我们能够确定业务需求里面只有按照身份证进行等值查询的需求，还有没有别的处理方法，既可以占用更小的空间，也能达到相同的查询效率？

第一种方式是使用**倒序存储**。如果你存储身份证号的时候把它倒过来存，每次查询的时候，可以这么写：

```sql
mysql> select field_list from t where id_card = reverse('input_id_card_string');
```

第二种方式是使用 **hash 字段**。你可以在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引。

```sql
mysql> alter table t add id_card_crc int unsigned, add index(id_card_crc);
```

然后每次插入新记录的时候，都同时用 `crc32()` 这个函数得到校验码填到这个新字段。由于校验码可能存在冲突，也就是说两个不同的身份证号通过 `crc32()` 函数得到的结果可能是相同的，所以你的查询语句 `where` 部分要判断 `id_card` 的值是否精确相同。

```sql
mysql> select field_list from t where id_card_crc=crc32('input_id_card_string') and
id_card='input_id_card_string';
```
这样，索引的长度变成了 4 个字节，比原来小了很多。

使用倒序存储和 hash 字段这两种方法的相同点是，==都不支持范围查询==。倒序存储的字段上创建的索引是按照倒序字符串的方式排序的，已经没有办法利用索引方式查出身份证号码在 [ID_X, ID_Y] 的所有市民了。同样地，hash 字段的方式也只能支持等值查询。

它们的区别，主要体现在以下三个方面：

 1. 从占用的额外空间来看，倒序存储方式在主键索引上，不会消耗额外的存储空间，而 hash 字段方法需要增加一个字段。当然，倒序存储方式使用 4 个字节的前缀长度应该是不够的，如果再长一点，这个消耗跟额外这个 hash 字段也差不多抵消了。
 2. 在 CPU 消耗方面，倒序方式每次写和读的时候，都需要额外调用一次 `reverse` 函数，而 hash 字段的方式需要额外调用一次 `crc32()` 函数。如果只从这两个函数的计算复杂度来看的话，`reverse` 函数额外消耗的 CPU 资源会更小些。
 3. 从查询效率上看，使用 hash 字段方式的查询性能相对更稳定一些。因为 crc32 算出来的值虽然有冲突的概率，但是概率非常小，可以认为==每次查询的平均扫描行数接近 1==。而倒序存储方式毕竟还是用的前缀索引的方式，也就是说还是会增加扫描行数。

## 小结
在为字符串字段创建索引的场景，可以使用的方式有：

 1. 直接创建完整索引，这样可能比较占用空间；
 2. 创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引；
 3. 倒序存储，再创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题；
 4. 创建 hash 字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，都不支持范围扫描。
 
 在实际应用中，需要根据业务字段的特点选择使用哪种方式。



References：
[1\] https://time.geekbang.org/column/article/70848
[2\] https://time.geekbang.org/column/article/71173
[3\] https://time.geekbang.org/column/article/71492
[4\] https://www.cnblogs.com/zjfjava/p/6922494.html
[5\] https://dev.mysql.com/doc/refman/8.0/en/create-table.html
[6\] https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html
[7\] https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html
[8\] https://dev.mysql.com/doc/refman/8.0/en/innodb-system-tablespace.html
[9\] https://dev.mysql.com/doc/refman/8.0/en/innodb-architecture.html
[10\] https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html
