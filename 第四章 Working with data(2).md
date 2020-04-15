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

2. 使用`$pull`数组中多个相同元素

```
db.media.updateOne ( {"ISBN" : "978-1-4842-1183-0"}, {$pull : { Author : "Griffin,  Stewie" } } )
```

3. 使用`$pullAll`移除数组中多个元素（不同）

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

### ATOMIC OPERATIONS(原子操作符)



























































































