@[TOC](Redis数据库与数据类型)

# 数据库的概念
与关系型数据库不同，Redis中不存在类似的数据库和表的概念。Redis中的数据库实例，更像是一个存储键值对的数据“字典”。

默认情况下，redis会自动创建**16个**数据库实例，并且给这些数据库实例进行编号，从0开始，一直到15，使用时通过编号来使用数据库。可以通过配置文件，指定redis自动创建的数据库个数。只需修改`redis.conf`中下面的行即可：
```typescript
databases 16
```

默认情况下，redis客户端连接的是编号是**0**的数据库实例；可以使用`select index`切换数据库实例。也可以直接在连接串`redis://user:password@host:port/dbnum`中指定。

```typescript
redis-cli -u redis://LJenkins:p%40ssw0rd@redis-16379.hosted.com:16379/0
```

# KEY与过期时间
Redis中的键（**Key**）是字符串，最大可以到**512M**。键名不建议太长也不要太短。一个比较合适的例子比如`user:1000:followers`。

KEY存在过期时间（*TTL, time to live*）的概念，超过过期时间，就会被销毁。使用`EXPIRE`命令来设定KEY的过期时间。

```c
> set key some-value
OK
// 将key的过期时间设置为5秒
> expire key 5
(integer) 1
//set key some-value ex 5
> get key (immediately)
"some-value"
> get key (after some time)
(nil)
```

也可以使用`SET`命令的参数`EX`指定过期时间。`TTL`命令可以查看KEY在被销毁前剩余的存活时间。
```c
> set key 100 ex 10
OK
> ttl key
(integer) 9
```

`PERSIST`命令可以取消KEY的过期时间，使其变为持久化存储对象。

# 数据类型
Redis中包含五种数据类型：字符串、列表、集合、有序集合、哈希。每种数据类型支持不同的操作命令。

Redis也有一些不限制数据类型的通用命令。比如`EXISTS`用于查询指定的KEY在当前数据库中是否存在，返回1表示存在，0表示不存在。`DEL`命令可以删除KEY和对应的VALUE，而不管VALUE存储的是什么数据类型。`TYPE`命令可以查询指定KEY对应的VALUE的数据类型。
```c
> set mykey hello
OK
> exists mykey
(integer) 1
> type mykey
string
> del mykey
(integer) 1
> exists mykey
(integer) 0
> type mykey
none
```

**注**：新版本的Redis中新增了Bit arrays、HyperLogLogs、Streams三种数据类型。

## 字符串 String
Redis中的字符串类型可以是任意类型的字符串，甚至可以是二进制的图片。字符串类型的Value，其大小不能超过512M。

使用`SET`命令来存储Value为字符串类型的键值对，然后使用`GET`命令来获取字符串的值。**如果`Key-Value`已经存在，`SET`命令会更新Value的值，即使原来的Value是非字符串类型的数据**。
```c
> set mykey somevalue
OK
> get mykey
"somevalue"
```

也可以通过参数`NX`或`XX`指定仅在KEY不存在或者已存在时才给VALUE赋值。
```c
//只有在mykey不存在时才将value设置为newval
> set mykey newval nx
(nil)
//只有在mykey已存在时才将value设置为newval
> set mykey newval xx
OK
```

还可以使用`INCR`、`INCRBY`、`DECR`、`DECRBY`命令将字符串转化为整型，从而实现递增、递减操作。
```c
> set counter 100
OK
> incr counter
(integer) 101
> incr counter
(integer) 102
> incrby counter 50
(integer) 152
```

## 列表 Lists
Redis列表底层是由链表结构（*Linked List*）实现的，而不是像Python中一样跟接近数组（*Arrays*）。链表的优点是添加新元素非常快，但是访问元素会比较慢。我们无需手动创建或者删除列表。当我们向列表中添加元素时，Redis会替我们创建列表；当列表中的元素全部被移除后，Redis会自动销毁列表。

使用`RPUSH`命令向列表的右边（尾部）添加新元素，使用`LPUSH`命令向列表的左边（头部）添加新元素。使用`LRANGE`命令查看列表中元素的范围，其第一个参数表示起始位置，第二个参数表示结束位置，`-1`表示列表中最后一个元素。

```c
> rpush mylist A
(integer) 1
> rpush mylist B
(integer) 2
> lpush mylist first
(integer) 3
> lrange mylist 0 -1
1) "first"
2) "A"
3) "B"

> rpush mylist 1 2 3 4 5 "foo bar"
(integer) 9
> lrange mylist 0 -1
1) "first"
2) "A"
3) "B"
4) "1"
5) "2"
6) "3"
7) "4"
8) "5"
9) "foo bar"
```

使用`RPOP`或`LPOP`命令从列表中获取并**移除**元素。
```c
> rpush mylist a b c
(integer) 3
> rpop mylist
"c"
> rpop mylist
"b"
> rpop mylist
"a"
> rpop mylist
(nil)
```

使用`LTRIM`命令来截取列表中的元素，在截取范围以外的元素都会被移除。跟`LRANGE`命令一样同样接受两个分别表示起始位置和结束位置的参数。
```c
> rpush mylist 1 2 3 4 5
(integer) 5
> ltrim mylist 0 2
OK
> lrange mylist 0 -1
1) "1"
2) "2"
3) "3"
```

使用`LLEN`命令可以查询列表的长度。

## 集合 Sets
集合Set是String类型的**无序无重复**集合。

通过`SADD`命令向集合中添加元素。`SMEMBERS`命令可以获取集合中所有的元素。
```c
> sadd myset 1 2 3
(integer) 3
> smembers myset
1. 3
2. 1
3. 2
```

使用`SISMEMBER`命令来判断指定元素是否在集合中，1表示存在，0表示不存在该元素。
```c
> sismember myset 3
(integer) 1
> sismember myset 30
(integer) 0
```

集合也支持交集`SINTER`、并集`SUNION`、差集`SDIFF`操作。
```c
> sinter tag:1:news tag:2:news tag:10:news tag:27:news
... results here ...
```

`SUNIONSTORE`命令将一个或多个集合合并后的结果集存储到给定目标集合中。
```c
> sadd deck C1 C2 C3 C4 C5 C6 C7 C8 C9 C10 CJ CQ CK
  D1 D2 D3 D4 D5 D6 D7 D8 D9 D10 DJ DQ DK H1 H2 H3
  H4 H5 H6 H7 H8 H9 H10 HJ HQ HK S1 S2 S3 S4 S5 S6
  S7 S8 S9 S10 SJ SQ SK
(integer) 52
// 将deck对应集合中的元素拷贝到game:1:deck对应的集合中
> sunionstore game:1:deck deck
(integer) 52
```

`SPOP`命令支持从集合中随机获取并移除一个或多个元素。
```c
> spop game:1:deck
"C6"
> spop game:1:deck
"CQ"
> spop game:1:deck
"D1"
> spop game:1:deck
"CJ"
> spop game:1:deck
"SJ"
```

`SCARD`命令可以获取集合中元素的基数（cardinality）。
```c
> scard game:1:deck
(integer) 47
```

## 有序集合 Sorted Sets
有序集合（Sorted Sets, **zset**）和Set一样也是String类型元素的集合，且不允许重复的成员。不同的是zset的每个元素都会关联一个分数**score**（分数可以重复），Redis通过分数来为集合中的成员进行从小到大的排序。

通过`ZADD`命令来向有序集合中添加元素，需要指定元素关联的分数。下面的例子中以年份作为元素的分数。
```c
> zadd hackers 1940 "Alan Kay"
(integer) 1
> zadd hackers 1957 "Sophie Wilson"
(integer) 1
> zadd hackers 1953 "Richard Stallman"
(integer) 1
> zadd hackers 1949 "Anita Borg"
(integer) 1
> zadd hackers 1965 "Yukihiro Matsumoto" 1914 "Hedy Lamarr"
(integer) 2
> zadd hackers 1916 "Claude Shannon" 1969 "Linus Torvalds" 1912 "Alan Turing"
(integer) 3
```

使用`ZRANGE`命令来查看指定范围的元素。可以发现返回的元素已按照分数进行排序。
```c
> zrange hackers 0 -1
1) "Alan Turing"
2) "Hedy Lamarr"
3) "Claude Shannon"
4) "Alan Kay"
5) "Anita Borg"
6) "Richard Stallman"
7) "Sophie Wilson"
8) "Yukihiro Matsumoto"
9) "Linus Torvalds"
```

如果要逆序输出，执行` zrevrange hackers 0 -1`。如果要连同分数一起输出，执行`zrange hackers 0 -1 withscores`。

获取分数不超过1950的所有元素：
```c
> zrangebyscore hackers -inf 1950
1) "Alan Turing"
2) "Hedy Lamarr"
3) "Claude Shannon"
4) "Alan Kay"
5) "Anita Borg"
```

移除指定分数范围内的集合元素：
```c
> zremrangebyscore hackers 1940 1960
(integer) 4
```

查询某个元素在排序中的位置：
```c
> zrank hackers "Anita Borg"
(integer) 4
```

如果有序集合中元素的分数相同，则会按照**字母顺序**进行排序。
```c
> zadd hackers 0 "Alan Kay" 0 "Sophie Wilson" 0 "Richard Stallman" 0
  "Anita Borg" 0 "Yukihiro Matsumoto" 0 "Hedy Lamarr" 0 "Claude Shannon"
  0 "Linus Torvalds" 0 "Alan Turing"

> zrange hackers 0 -1
1) "Alan Kay"
2) "Alan Turing"
3) "Anita Borg"
4) "Claude Shannon"
5) "Hedy Lamarr"
6) "Linus Torvalds"
7) "Richard Stallman"
8) "Sophie Wilson"
9) "Yukihiro Matsumoto"
```

获取指定字母顺序范围内的元素：
```c
> zrangebylex hackers [B [P
1) "Claude Shannon"
2) "Hedy Lamarr"
3) "Linus Torvalds"
```

## 哈希 Hashes
Redis Hash是一个String类型的`field`和`value`的映射表，Hash特别适合用于存储对象。

使用`HMSET`命令来设定多个`field-value`映射。使用`HGET`命令来获取单个`field`映射的值。
```c
> hmset user:1000 username antirez birthyear 1977 verified 1
OK

> hget user:1000 username
"antirez"
> hget user:1000 birthyear
"1977"
```

使用`HMGET`命令来获取多个`field`映射的值。使用`HGETALL`命令来获取指定KEY所有的`field`映射。
```c
> hmget user:1000 username birthyear no-such-field
1) "antirez"
2) "1977"
3) (nil)

> hgetall user:1000
1) "username"
2) "antirez"
3) "birthyear"
4) "1977"
5) "verified"
6) "1"
```

支持通过`HINCRBY`命令对单个field的value进行递增操作。
```c
> hincrby user:1000 birthyear 10
(integer) 1987
> hincrby user:1000 birthyear 10
(integer) 1997
```

**References**
【1】https://blog.csdn.net/weixin_46718393/article/details/107597846
【2】https://redis.io/docs/manual/data-types/data-types-tutorial/
【3】https://redis.io/commands/
