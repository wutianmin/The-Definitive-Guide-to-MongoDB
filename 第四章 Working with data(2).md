## 更新数据

### 使用`update()`更新数据

`update()`中有三个参数：

+ {criteria}:查询指定要更改的字段

+ {objNew}:指定更新后数据

+ {options}:指定参数“upsert”（为true则指定该更新是 ‘更新存在的数据且创建新的不存在的数据’）和“multi”（指定更新多个文档还是查找到的第一个文档（默认更改一个），或直接使用`updateOne(),updateMany()`）

`update()`中objNew内容会覆盖以前文档中的全部内容 ，基本相当于重写文档

示例：

```
db.media.updateOne( { "Title" : "Matrix, The"}, {"Type" : "DVD", "Title" : "Matrix, The", "Released" : 1999, "Genre" : "Action"}, { upsert: true} )
```

### 使用`save()`实现Upsert

`save()`中只有一个参数即objNew，且必须包含_id值，否则会相当于insert()插入新的文档。

例如，

```
db.media.updateOne( { "Title" : "Matrix, The"}, {"Type" : "DVD", "Title" : "Matrix, The", "Released" : "1999", "Genre" : "Action"}, { upsert: true} )
//等同于
db.media.save( {"_id" : ObjectId("5e951f6bace80293b88df254"),"Type" : "DVD", "Title" : "Matrix, The", "Released" : "1999", "Genre" : "Action"})
```

### 使用操作符自动更新数据

1. 使用`$inc`增加数值

例如，将{        "Type" : "Manga",        "Title" : "One Piece",        "Volumes" : "612",        "Read" : "520" }文档中“read”的值add 4

```
db.media.updateOne({Title:"One Piece"},{$inc:{Read:4}})
```

2. 改变字段的值

`$set`

例如，只将查找到的文档中“Genre”字段值改为“Sci-Fi”

```
db.media.update ( { "Title" : "Matrix, The" }, {$set : { Genre : "Sci-Fi" } } )
```

3. 删除指定字段

例如，从文档中删除“Genre”字段和其值

```
db.media.updateOne ( {"Title": "Matrix, The"}, {$unset : { "Genre" : 1 } } )
```

4. 为数组字段添加值

+ `$push`为数组字段添加一个值

如果该字段是已存在的数组，就会添加到数组中；如果该字段不存在，则该字段将被设置为此值。

例如，在“Arthor”数组中添加"Griffin,  Stewie"

```
db.media.updateOne ( {"ISBN" : "978-1-4842-1183-0"}, {$push: { Author : "Griffin,  Stewie"} } )
```

+ `$push`加`$each`操作符可以向数组中添加多个元素

例如，

```
db.media.updateOne( { "ISBN" : "978-1-4842-1183-0" }, { $push: { Author : { $each: ["Griffin, Peter", "Griffin, Brian"] } } } ) 
```

+ 使用`$push`+`$each`时使用`$slice`限制添加元素的数量

`$slice`运算符取负数或零。

取负数确保仅保留数组的最后n个元素，取零则清空数组。

*`$slice`运算符必须是`$push`的第一个操作符才能起作用*

```
db.media.updateOne( { "ISBN" : "978-1-4842-1183-0" }, { $push: { Author : { $each: ["Griffin, Meg", "Griffin, Louis"], $slice: -2 } } } ) 
```

*当数组是固定大小数组时，`$slice`可以控制其大小*

5. 使用`$addToSet`向数组中添加元素

`$addToSet`和`$push`区别，其**只向数组中添加不存在的元素**。

与`$push`类似，也可以使用`$each`添加多个元素。

### 从数组中删除元素

使用`$pop`, `$pull`, `$pullAll`

1. 使用`$pop`移除数组中的单个元素

   **传递任何负数移除第一个元素，传递任何正数或0移除最后一个元素**

   例如，移除'Author'中的最后一个元素

```
db.media.updateOne( { "ISBN" : "978-1-4842-1183-0" }, {$pop : {Author : 1 } })
```

2. 使用`$pull`删除数组中指定的一或多个元素

   *如果有多个相同值，`$pull`会将其全部删除*

```
db.media.updateOne ( {"ISBN" : "978-1-4842-1183-0"}, {$pull : { Author : "Griffin,  Stewie" } } )
```

3. 使用`$pullAll`移除数组中多个元素

```
db.media.updateOne( { "ISBN" : "978-1-4842-1183-0"}, {$pullAll : { Author : ["Griffin, Louis","Griffin, Peter","Griffin, Brian"] } } )
```

### 指定匹配数组的位置

位置运算符`$`，可更新数组中的元素，无需显示指定该元素在数组中的位置。

前面查找匹配的字段需精确到嵌入文档

例如，

```
db.media.updateOne( { "Tracklist.Title" : "Been a Son"}, {$inc:{"Tracklist.$.Track" : 1} } )
```

### ATOMIC OPERATIONS(原子操作)

原子操作是一组操作，该操作对于系统而言只是进行了一次操作。

满足以下2个条件称为原子操作：

1. 在完成整组操作前，没有对数据进行更改
2. 如果整组操作其中的一个失败，则整个原子操作失败

*原子操作使用update-if-current方法*

下面两个线程的执行过程复现了ABA问题： 
 \- P1读取到共享变量的值为A 
 \- P1的执行权限被抢占，然后P2开始占用CPU 
 \- 在P1恢复执行之前，P2将共享变量的值先从A修改为B，然后再将B修改为之前的A 
 \- P1重新获得时间片，然后发现共享变量的值未发生改变，因此继续执行后面的操作 
虽然，P1重新获得了时间片，能够继续执行后面的操作，但是因为P2在P1被抢占之后，因为P2执行的某些操作可能会对系统的状态有影响，因此如果P1继续执行后面的代码，可能导致系统状态的不一致，或者引发异常。

ABA问题发生的一个常见场景就是实现无锁数据结构。假设在一个无锁的List中，一个元素被删除，紧接着一个新的元素被创建并添加进来，因为系统的优化，新的元素在内存中的位置可能和被删除的元素在内存中的位置相同，也就是说，新元素和被删除的元素的指针是指向同一个内存地址，这个时候就发生了ABA问题。

### 以原子操作方式修改并返回文档（`findAndModify()`）

`findAndModify()`命令包含3个操作符：

1. <query>：查询指定要进行操作的文档
2. <sort>：匹配多个文档时进行排序
3. <operations>：修改操作

例如，查找与"ISBN" : "978-1-4842-1183-0"匹配的文档，并降序排序，将其"Title" :改为" Different Title"

```
db.media.findAndModify( { query: { "ISBN" : "978-1-4842-1183-0" }, sort: {"Title":-1}, update: {$set: {"Title" : " Different Title"} } } )
//update中可以使用任何修改操作符$
```

此时更改了文档中"Title"数据，并直接返回了未更改前的结果

```
//如果要直接返回更改后的结果，添加参数"new:true"
db.media.findAndModify( { query: { "ISBN" : "978-1-4842-1183-0" }, sort: {"Title":-1}, update: {$set: {"Title" : " Different Title"} }, new:true } )
```

## 批量处理数据（processing data in bulk）

可以在单个集合中批量执行写入操作。 即先定义数据集，然后再一次编写所有数据集。可用于插入，更新或删除数据。

+ *在进行批量写入之前，需要定义操作顺序是有序还是无序的*

*若定义有序操作，如果其中一个写入操作失败，则不往下进行*

*若定义无序操作，其中一个操作的失败不会影响其余操作*

例如，首先定义有序批量操作函数

```
var bulk=db.media.initializeOrderedBulkOp();
```

然后定义插入操作

```
> bulk.insert({ "Type" : "Movie", "Title" : "Deadpool", "Released" : 2016}); 
> bulk.insert({ "Type" : "CD", "Artist" : "Iron Maiden", "Title" : "Book of Souls, The" }); 
> bulk.insert({ "Type" : "Book", "Title" : "Paper Towns", "Author" : "Green, John" });
//最多可以包含1000个操作。 当列表超过此限制时，MongoDB将自动将其拆分并处理为1000个或更少的单独操作组
```

然后，执行将数据写入集合操作

```
bulk.execute()；
```

+ bulk()可以进行的操作

  `bulk.insert()`,`bulk.find()`,`bulk.find.removeOne()`,`bulk.find.remove()`,

  `bulk.find.replaceOne()`,`bulk.find.updateOne()`,`bulk.find.update()`,`bulk.find.upsert()`

+ 使用`bulk.getOperations()`评估执行的批量写入操作

  结果中"batchType"显示的是更改的操作类型

  1：insert

  2：update

  3：remove

  *当处理各种类型的批量操作时，建议使用无序列表。MongoDB会按类型将操作分组在一起以提高性能*

+ 使用[`db.collection.bulkWrite()`](https://docs.mongodb.com/manual/reference/method/db.collection.bulkWrite/#db.collection.bulkWrite)也可以执行批量写入操作

## 重命名集合

```
db.media.renameCollection("newname")
```

## 删除文档、集合、数据库

`db.<collection>.deleteOne()`用于删除一个满足条件的文档

```
db.newname.deleteOne( { "Title" : "Different Title" } )
```

`db.<collection>.deleteMany()`用于删除多个文档

`db.<collection>.deleteMany({})`删除集合中的所有文档

`db.<collection>.drop()`删除集合和其中的所有文档

`db.dropDatabase()`删除整个数据库

## 引用数据库（database referencing(DBRef)）

### 1 手动引用数据

例如，新建一个集合，插入出版者信息。

*此时的_id值需要自己设定，并且与要主文档中的某字段值相同，以便引用*

```
db.publisherscollection.insertOne({ "_id" : "Apress", "Type" : "Technical Publisher", "Category" : ["IT", "Software","Programming"] } )
```

插入主文档信息，注意字段"Publisher" 的值 "Apress"与引用文档的_id值相同

```
db.media.insertOne({ "Type" : "Book", "Title" : "Definitive Guide to MongoDB 3rd ed., The",  "ISBN" : "978-1-4842-1183-0", "Publisher" : "Apress","Author" : ["Hows, David","Plugge,  Eelco","Membrey,Peter","Hawkins, Tim"] } )
```

然后，从主文档中寻找附属信息

```
book=db.media.find({'Type':"Book"})//将含有publisher字段的文档查找到并设为一个变量，以便引用
db.publisherscollection.findOne({_id:book.publisher})//利用_id值来引用信息
```

*如果引用的集合始终相同，那么手动引用数据（如上所述）就可以了*

### 2 使用DBRef引用数据*

表达式：

`{ $ref : <collectionname>, $id : <id value>,[$db : <database name>] }`

<collectionname>是所引用的文档（如Publishercollection），$db是放置在其他数据库中的文档

示例，

```
>apress = ( { "Type" : "Technical Publisher", "Category":["IT","Software","Programming"] } ) 
>db.publisherscollection.save(apress)//插入出版者信息(被引用部分)
>book = { "Type" : "Book", "Title" : "Definitive Guide to MongoDB 3rd ed., The", "ISBN" : "978-1-4842-1183-0", "Author": ["Hows, David","Membrey, Peter","Plugge,Eelco","Hawkins, Tim"], Publisher : [ new DBRef ('publisherscollection',apress._id) ] }
>db.media.save(book)//引用出版者信息并插入文档
```

## 索引相关的功能函数

1. 建立索引
+ 为文档中的字段创建索引。其中至少有一个参数，即文档中的某字段

   // Ensure ascending index 

   `db.media.createIndex( { Title :1 } )`

   // Ensure descending index 

   `db.media.createIndex( { Title :-1 } )`

*建立索引使查询信息更加快捷*

+ 为数组中嵌入式文档中的指定键创建索引，例如

```
db.media.createIndex( { "Tracklist.Title" : 1 } )
```

+ 为数组中的每个嵌入式文档创建索引

```
db.media.createIndex( { "Tracklist" : 1 } )
```

+ 在多个键上创建复合索引

```
db.media.createIndex({"Tracklist.Title": 1, "Tracklist.Length": -1})
```

2. 强制指定索引查询数据

*可以增加查询时的性能*

```
db.media.ensureIndex({ISBN: 1}, {background: true}); 
db.media.find( { ISBN: "978-1-4842-1183-0"} ) . hint ( { ISBN: 1 } )
```

在后面加`explain()`函数返回"queryplaner"等信息

```
db.media.find( { ISBN: "978-1-4842-1183-0"} ) . hint ( { ISBN: 1 } ).explain()
```

3. 限制查询匹配（多用于复合键中）

使用`min()`和`max()`函数可以将查询匹配限制为仅 在指定的最小键和最大键之间 

```
//插入数据
db.media.insertOne( { "Type" : "DVD", "Title" : "Matrix, The", "Released" : 1999} ) db.media.insertOne( { "Type" : "DVD", "Title" : "Blade Runner", "Released" : 1982}) db.media.insertOne( { "Type" : "DVD", "Title" : "Toy Story 3", "Released" : 2010} ) 
//为发行年份创建索引
db.media.ensureIndex( { "Released": 1 } )
//查找发行年份在1995年（包括1995）到2005年（不包括2005）之间的文档
db.media.find() . min ( { Released: 1995 } ) . max ( { Released : 2005 } ) 
```





























































