# 第四章 Working with data

## 浏览数据库、集合

```
use library//连接到library数据库
show dbs//显示所有已创建的数据库
db//显示当前运行的数据库
show collections//显示当前运行的数据库中的所有集合
```

## 向集合中插入数据

以BSON格式向集合中插入数据。

插入数据的两种方式：

1.先定义文档，再用`insertOne()`插入集合

```
> document = ({"Type": "Book", "Title" : "Definitive Guide to MongoDB 3rd ed., The",  "ISBN" : "978-1-4842-1183-0", "Publisher" : "Apress", "Author" : ["Hows, David", "Plugge, Eelco", "Membrey, Peter", "Hawkins, Tim"] } )
> db.media.insertOne(document) 
```

2.不预先定义文档，直接插入数据

```
> db.media.insertOne( { "Type" : "CD", "Artist" : "Nirvana", "Title" : "Nevermind", 
"Tracklist" : [  { "Track" : "1",  "Title" : "Smells Like Teen Spirit",  "Length" : "5:02" },
{  "Track" : "2", "Title" : "In Bloom", "Length" : "4:15"  }  ] }  )
```

+ 插入数据时的规则：

  1. ‘$’不能是键名的第一个字符

  2. 键名中不能包含‘.’
  3. _id保留作主键id，可以存储任何唯一的字符串或整数..作为值

+ 建立集合时的一些规则
  1. 集合名称所占用的空间（包括数据库名称和“.”分隔符）不能超过120个字符
  2. 空字符串（“ ”）不能用作集合名称
  3. 集合名称必须以一个字母或下划线开头
  4. 集合名称不能包含“\0”空字符

## 查询数据（querying for data）

使用`find()`函数，从一个集合中的多个文档中检索数据

1. 查询media集合中的所有文档

```
> db.media.find()
```

2. 查询media集合中的某些文档，添加参数

```
> db.media.find({Artist:'Nirvana'})
```

3. 查询media集合中的某个文档中的部分信息，添加参数`{key:1}`。

   _ id信息默认返回，添加 {_ id:0}不返回_id

```
> db.media.find ( {Artist : "Nirvana"}, {Title: 1} ) 
```

### 使用点表示法（DOT NOTATION）

当处理的数据很复杂，如，包含数组的文档、包含嵌入式文档的文档

1. 当查询嵌入式文档中的某字段，使用点表示法

```
> db.media.find( { "Tracklist.Title" : "In Bloom" } ) 
```

==`"Tracklist.Title"`必须加引号==

2. 当查询数组中元素时

```
> db.media.find( { "Author" : "Membrey, Peter" } )
```

### 使用sort,limit,skip功能

+ 使用`sort()`进行排序

  升序：1

  降序：-1

  例如，查找文档并按照Title升序显示

```
db.media.find().sort( { Title: 1 })
```

+ 使用`limit()`指定返回的最大结果数。当指定数目为0时，将返回所有结果

  例如，返回media集合中的10个文档

```
 db.media.find().limit( 10 )
```

+ 使用`skip()`跳过集合中的前n个文档

  例如，跳过集合中前20个文档返回结果

```
 db.media.find().skip( 20 )
```

*这三个功能可以结合在一起使用，且顺序可以互换而不影响返回的结果*

例如，

```
 db.media.find().sort ( { Title : -1 } ).limit ( 10 ).skip ( 20 )
```

*可以使用快捷方式限制和跳过结果，`find({},{},10,20)`表示将结果限制为10个，跳过前20个文档*

### 固定集合（Capped Collections）, 自然排序方式（Natural Order）, and $natural

1. natural order：集合中数据库本身排序方式

*当在不指定排序顺序的情况下查询集合中的文档时，默认情况下会以自然顺序返回文档。*

看起来顺序与插入数据时的顺序相同，但这个顺序不是预先定义的，可以会根据文档增长方式、索引、存储引擎有所不同。

2. 固定集合（capped collections）：集合中natural order被确保为文档插入顺序，即集合中的文档顺序为由旧到新

固定集合的大小是固定的，当固定集合的数据满足上限时，将清除最旧的数据，并在最后添加最新的数据。**所以插入文档时，应注意不要超过固定集合的大小**

使用`createCollection()`创建固定集合。需提供参数，指定集合的大小（bytes类型）。

例如，创建audit固定集合，大小为20480字节

```
db.createCollection("audit", {capped:true, size:20480})
```

用`max:`创建限制集合中项目数量为100的固定集合

```
db.createCollection("audit100", { capped:true, size:20480, max: 100}) 
```

查看该集合的大小

```
db.audit100.stats()
```

对于固定集合，查询时不再需要指定其他参数或命令。

3. `$natural`：当想要查询固定集合时，逆向返回结果

例如，查找固定集合中最新的十个数据

```
db.audit.find().sort({$natural:-1}).limit(10)
```

*固定集合中的文档可以修改，但是不能其大小不能发生变化。*

*固定集合中的文档不能删除，如果删除必须删除整个集合，然后重新建立新的固定集合。*

### 查询单个文档

使用`findOne()`

### 聚合命令（aggregation commands）

1. 使用`count()`返回指定集合中的文档数量

```
db.media.count()
```

```
db.media.find({Publisher:'Apress',Type:'Book'}).count()
```

*默认情况下`count()`忽视`skip()`、`limit()`两个函数，确保不会跳过这两个函数需要写入`count(true)`*

```
db.media.find( { Publisher: "Apress", Type: "Book" }).skip ( 2 ) .count (true) 
```

2. 使用`distinct()`查找唯一值

当集合中不同文档中同一字段下，有相同的value，理论上讲这个东西只有一个，可以使用`distinct()`返回不重复的结果

例如，media集合中'Title'对应的值一样时，则只返回一个结果

```
db.media.distinct( "Title") 
```

[ "Definitive Guide to MongoDB 3rd ed., The", "Nevermind" ]

3. MongoDB中使用`aggregate()`和`$group`将结果分组，返回一个数组

表达式`{ $group: { _id: <expression>, <field1>: { <accumulator1> : <expression1> }, ... } }`

例如，按照Title字段进行分组

```
 db.media.aggregate([{$group:{
 _id:'$Title',
 count:{$sum:1}
 }}])
```

### 条件运算符（conditional operators）

1. 大于、小于比较

`$gt`（大于）,`$lt`（小于）,`$gte`（大于等于）,`$lte`（小于等于）

示例，

```
db.media.find({Released:{$gte:1999}})
```

2. 查找输出除指定元素以外的其余元素

`$ne`

示例，

```
db.media.find( { Type : "Book", Author: {$ne : "Plugge, Eelco"}})
```

3. 查找与数组中数据匹配的文档

`$in` 数组中的数据只要有一个与文档中某数据匹配即可返回结果

示例，查找发行年份在1999或2008或2009的文档

```
db.media.find( {Released : {$in : [1999,2008,2009] } } ) 
```

`$all` 数组中的数据必须都与某些文档中的数据对应起来，才可以返回结果，只要有一个不对应就不输出结果

`$nin` 查找与数组中数据不匹配的文档

4. 使用`$or` 在一个查询中查询多个表达式，但仅需要匹配一个条件即可返回结果

`$in`只能指定值，`$or`可以指定要匹配的键和值。

```
db.media.find({ $or : [ { "Title" : "Toy Story 3" }, { "ISBN" : "978-1-4842-1183-0" } ] } )
```

5. 使用`$slice`查询与数组的某子集匹配的文档，可用于分页

例如，查询‘cast’字段的值中前3个元素

```
db.media.find({"Title" : "Matrix, The"}, {"Cast" : {$slice: 3}}) 
```

查询‘cast’字段的值中后3个元素

```
db.media.find({"Title" : "Matrix, The"}, {"Cast" : {$slice: -3}}) 
```

跳过前2个元素来检索，并输出跳过之后的前3个元素

```
db.media.find({"Title" : "Matrix, The"}, {"Cast" : {$slice: [2,3] }})
```

跳到最后5个元素，在这5个元素中检索，输出前4个元素

```
db.media.find({"Title" : "Matrix, The"}, {"Cast" : {$slice: [-5,4] }}) 
```

6. 查询奇偶数

`$mod` 只用于查询Integer整数型数值

示例，

查询发行年份为偶数的文档。模数为2，余数为0

```
db.media.find ( { Released : { $mod: [2,0] } }, {"Cast" : 0 } ) 
```

查询发行年份为奇数的文档。模数为2，余数为1

```
db.media.find ( { Released : { $mod: [2,1] } }, { "Cast" : 0 } )
```

7. 使用`$size`过滤结果

例如，查找‘tracklist’数组中有两个元素的文档

```
db.media.find ( { Tracklist : {$size : 2} } )
```

8. 查找键值是否存在并返回结果

   `$exists`

```
db.media.find ( { Author : {$exists : true } } )
```

9. 基于BSON类型匹配结果

`$type`

| -1   | MinKey                     |
| ---- | -------------------------- |
| 1    | Double                     |
| 2    | Character string(UTF8)     |
| 3    | Embedded object            |
| 4    | Embedded array             |
| 5    | Binary data                |
| 7    | Object ID                  |
| 8    | Boolean type               |
| 9    | Date type                  |
| 10   | Null type                  |
| 11   | Regular expression         |
| 13   | JavaScript code            |
| 14   | Symbol                     |
| 15   | JavaScript code with scope |
| 16   | 32-bit integer             |
| 17   | Timestamp                  |
| 18   | 64-bit integer             |
| 127  | MaxKey                     |
| 255  | MinKey                     |

例如，查找包含嵌入式对象类型的项目

```
db.media.find ( {  Tracklist: { $type : 3 } } ) 
```

10. 匹配整个数组

`$elemMatch`

































































































































































































































































































































































































