@[TOC](MongoDB插入数据与等值查询)

# 切换数据库
切换数据库时，无需保证目标数据库已经存在。
```cpp
//显示当前数据库
db
//切换数据库
use testdb
```
如果上面的testdb不存在，在首次向testdb中插入数据或者创建`collection`时，MongoDB会自动创建testdb。

# 插入数据
MongoDB将文档数据存储在`collection`中，`collection`类似于关系型数据库中的`table`。

查看数据库中已有的Collections信息：
```cpp
// 查看已有的Collections详细信息
db.getCollectionInfos() 
// 查看已有的Collections名称
db.getCollectionNames() 
```

如果某个collection不存在，在首次向其中插入数据时，MongoDB会自动创建该collection。

通过`db.collection.insertOne()`方法向名为`inventory`的collection中插入一条记录：
```cpp
db.inventory.insertOne(
   { item: "magazine", qty: 32, status: "B", size: { h: 35, w: 20, uom: "cm" }, tags: [ "black" ] }
);
```

通过`db.collection.insertMany()`方法向名为`inventory`的collection中插入多条记录：
```cpp
db.inventory.insertMany([
   { item: "journal", qty: 25, status: "A", size: { h: 14, w: 21, uom: "cm" }, tags: [ "blank", "red" ] },
   { item: "notebook", qty: 50, status: "A", size: { h: 8.5, w: 11, uom: "in" }, tags: [ "red", "blank" ] },
   { item: "paper", qty: 10, status: "D", size: { h: 8.5, w: 11, uom: "in" }, tags: [ "red", "blank", "plain" ] },
   { item: "planner", qty: 0, status: "D", size: { h: 22.85, w: 30, uom: "cm" }, tags: [ "blank", "red" ] },
   { item: "postcard", qty: 45, status: "A", size: { h: 10, w: 15.25, uom: "cm" }, tags: [ "blue" ] }
]);

// MongoDB adds an _id field with an ObjectId value if the field is not present in the document
```


# 等值查询
使用`db.collection.find()`来查询某个collection中所有或者特定的记录。

查询collection中的所有记录：
```cpp
db.inventory.find({})
//使用pretty方法来美化查询输出
db.inventory.find({}).pretty()
```

查询匹配某个属性的所有记录：
```cpp
//查询status值为D的所有记录
db.inventory.find( { status: "D" } );
//查询qty值为0的所有记录
db.inventory.find( { qty: 0 } );
//查询status值为D、且qty值为0的所有记录
db.inventory.find( { qty: 0, status: "D" } );

//查询sizes属性的uom子属性值为in的所有记录
db.inventory.find( { "size.uom": "in" } )
//查询sizes属性等于给定内容的所有记录
db.inventory.find( { size: { h: 14, w: 21, uom: "cm" } } )
```

当键值为数组时，等值查询同时考虑内容匹配和**顺序匹配**。且以下两种查询方式意义不同：
```cpp
db.inventory.find( { tags: "red" } )
```
表示查询`tags`对应数组中包含`"red"`元素的所有记录。
```cpp
db.inventory.find( { tags: [ "red", "blank" ] } )
```
表示查询`tags`对应数组为`[ "red", "blank" ]`的所有记录。如果某条记录的tags对应数组为`[ "blank", "red" ]`，也**不**会被包含在查询结果集中。

对于查询的结果，可以通过`db.collection.find(<query document>, <projection document>)`方法指定仅输出部分内容，其中1表示输出内容，0表示隐藏内容。
```cpp
db.inventory.find( { }, { item: 1, status: 1 } );
```
表示只输出所有记录的`_id`、`item`、`status`属性（`_id`默认会输出）。

```cpp
db.inventory.find( {}, { _id: 0, item: 1, status: 1 } );
```
表示只输出所有记录的`item`、`status`属性。
