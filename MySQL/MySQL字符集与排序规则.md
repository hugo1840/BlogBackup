
@[TOC](MySQL字符集与排序规则)

字符集（**character set**）可以理解为符号（*symbols*）与编码（*encoding*）的集合，而排序规则（**collation**）则是指用来比较字符集中所有字符的规则。MySQL支持多种字符集，用于存储数据、比较字符串、以及客户端程序与MySQL服务器之间的通信。MySQL支持在以下五个层级设定字符集：

- 服务器（*server*）
- 数据库（*database*）
- 表（*table*）
- 列（*column*）
- 字符串字面常量（*string literal*）

# MySQL支持的字符集
## 字符集与排序规则
执行下面的命令查看MySQL支持的字符集：
```sql
--方法1
> SELECT * FROM INFORMATION_SCHEMA.CHARACTER_SETS;
--方法2
> SHOW CHARCATER SET;
> SHOW CHARACTER SET LIKE 'latin%';
+---------+-----------------------------+-------------------+--------+
| Charset | Description                 | Default collation | Maxlen |
+---------+-----------------------------+-------------------+--------+
| latin1  | cp1252 West European        | latin1_swedish_ci |      1 |
| latin2  | ISO 8859-2 Central European | latin2_general_ci |      1 |
| latin5  | ISO 8859-9 Turkish          | latin5_turkish_ci |      1 |
| latin7  | ISO 8859-13 Baltic          | latin7_general_ci |      1 |
+---------+-----------------------------+-------------------+--------+
```

**每个字符集支持至少一种排序规则**。执行下面的命令来查看某个字符集支持的排序规则：

```sql
--方法1
> SELECT COLLATION_NAME FROM INFORMATION_SCHEMA.COLLATIONS
  WHERE CHARACTER_SET_NAME LIKE 'utf8';
--方法2
> SHOW COLLATION WHERE Charset = 'latin1';
+-------------------+---------+----+---------+----------+---------+
| Collation         | Charset | Id | Default | Compiled | Sortlen |
+-------------------+---------+----+---------+----------+---------+
| latin1_german1_ci | latin1  |  5 |         | Yes      |       1 |
| latin1_swedish_ci | latin1  |  8 | Yes     | Yes      |       1 |
| latin1_danish_ci  | latin1  | 15 |         | Yes      |       1 |
| latin1_german2_ci | latin1  | 31 |         | Yes      |       2 |
| latin1_bin        | latin1  | 47 |         | Yes      |       1 |
| latin1_general_ci | latin1  | 48 |         | Yes      |       1 |
| latin1_general_cs | latin1  | 49 |         | Yes      |       1 |
| latin1_spanish_ci | latin1  | 94 |         | Yes      |       1 |
+-------------------+---------+----+---------+----------+---------+
```

排序规则具有以下特点：

- **一种排序规则最多只能关联一种字符集**，即两个字符集不会有相同的排序规则；
- 每个字符集都有一种**默认**的排序规则；
- 排序规则的名称以它关联的字符集名称开头，再加上一些能表名其排序特性的后缀结束。

## 排序规则的命名
排序规则的名称以它关联的字符集名称开头，加上一个或多个后缀结束。后缀可以指明排序规则支持的语言、以及是否大小写敏感（*case sensitive/insensitive*）、声调/重音敏感（*accent sensitive/insensitive*），等等。例如

- `utf8_general_ci`：utf8字符集的通用排序规则，大小写不敏感；
- `utf8_turkish_ci`：utf8字符集、土耳其语的排序规则，大小写不敏感；
- `latin1_general_cs`：latin1字符集的通用排序规则，大小写敏感。

# 使用字符集与排序规则
## 服务器级别的字符集
MySQL服务器级别的默认字符集和排序规则分别是`latin1`和`latin1_swedish_ci`。

使用mysqld命令初始化服务器时可以指定字符集和排序规则。
```bash
#方法一
$ mysqld --character-set-server=utf8
#方法二
$ mysqld --character-set-server=utf8 --collation-server=utf8_general_ci
```

如果仅指定了字符集，则会采用默认的排序规则。因此上面两次执行的命令是等价的。

查看当前服务器级别的字符集信息：
```sql
> show [global|session] variables like 'character_set_server';
> show [global|session] variables like 'collation_server';
```

如果在使用`CREATE DATABASE`命令创建数据库时，没有指定字符集和排序规则，则会默认使用服务器级别的字符集和排序规则。

## 数据库级别的字符集
在创建数据库时指定字符集和排序规则：
```sql
> CREATE DATABASE db_name
    [[DEFAULT] CHARACTER SET charset_name]
    [[DEFAULT] COLLATE collation_name];
```

修改数据库的字符集和排序规则：
```sql
> ALTER DATABASE db_name
    [[DEFAULT] CHARACTER SET charset_name]
    [[DEFAULT] COLLATE collation_name];
```

在上面的命令中，中括号表示非必选项。

- 如果仅指定了字符集，则会使用默认的排序规则；
- 如果仅指定了排序规则，则会使用其关联的字符集；
- 如果字符集和排序规则都未指定，则会使用**服务器级别**的字符集和排序规则。

查看指定数据库的字符集和排序规则：
```sql
USE db_name;
SELECT @@character_set_database, @@collation_database;
```
或者
```sql
SELECT DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
       FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = 'db_name';
```

数据库级别的字符集和排序规则具有以下影响：

- 在使用`CREATE TABLE`命令创建表时，如果没有指定字符集和排序规则，则会默认使用数据库级别的字符集和排序规则；
- 在使用`LOAD DATA`导入数据时，如果没有指定字符集，则会默认使用数据库级别的字符集；
- 在创建存储过程或者函数时，declare的参数如果没有指定字符集和排序规则，则会默认使用创建该存储过程/函数时、数据库级别的字符集和排序规则。


## 表级别的字符集
在创建表时指定字符集和排序规则：
```sql
> CREATE TABLE tbl_name (column_list)
    [[DEFAULT] CHARACTER SET charset_name]
    [COLLATE collation_name]];
```

修改表的字符集和排序规则：
```sql
> ALTER TABLE tbl_name
    [[DEFAULT] CHARACTER SET charset_name]
    [COLLATE collation_name];
```

在上面的命令中

- 如果仅指定了字符集，则会使用默认的排序规则；
- 如果仅指定了排序规则，则会使用其关联的字符集；
- 如果字符集和排序规则都未指定，则会使用**数据库级别**的字符集和排序规则。

如果表中的列没有定义字符集和排序规则，则会使用表级别的字符集和排序规则作为默认值。

## 列级别的字符集
字符类型的列，比如`char`、`varchar`、`text`等，也可以定义自己的字符集和排序规则。

在创建表时指定列的字符集：
```sql
> CREATE TABLE t1
(
      col1 VARCHAR(5)
      CHARACTER SET latin1
      COLLATE latin1_german1_ci
);

```

修改列的字符集：
```sql
> ALTER TABLE t1 MODIFY
      col1 VARCHAR(5)
      CHARACTER SET latin1
      COLLATE latin1_swedish_ci;
```
**注**：如果修改前后的字符集不兼容，可能会丢失数据。

在上面的命令中

- 如果仅指定了字符集，则会使用默认的排序规则；
- 如果仅指定了排序规则，则会使用其关联的字符集；
- 如果字符集和排序规则都未指定，则会使用**表级别**的字符集和排序规则。

## 字符串级别的字符集
SQL语句`SELECT 'string'`中，字符串string的默认字符集和排序规则分别由系统变量`charcater_set_connection`和`collation_connection`定义。

可以使用**introducer**（`_charset_name`）来指定字符串字面常量的字符集和排序规则：
```sql
SELECT 'abc';
SELECT _latin1'abc';   --指定字符集
SELECT _utf8'abc' COLLATE utf8_danish_ci;   --指定字符集和排序规则
```

同样地

- 如果仅指定了字符集，则会使用默认的排序规则；
- 如果仅指定了排序规则，则会使用其关联的字符集；
- 如果字符集和排序规则都未指定，则会使用`charcater_set_connection`和`collation_connection`定义的字符集和排序规则。

# National字符集
标准SQL中定义了`NCHAR`或者`NATIONAL CHAR`来作为`CHAR`类型的列应该使用的预定义的字符集。MySQL使用`utf8`作为预定义的字符集。

因此，下面的语句是等价的：
```sql
SELECT N'some text';
SELECT n'some text';
SELECT _utf8'some text';
```


**References**
【1】[Character Sets and Collations in MySQL](https://dev.mysql.com/doc/refman/5.7/en/charset-mysql.html)
