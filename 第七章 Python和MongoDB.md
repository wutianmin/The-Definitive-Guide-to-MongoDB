# 第七章 Python和MongoDB

## 在Python中处理文档

MongoDB使用BSON类型文档，Python使用字典（dictionary）

## 使用Python modules

每个任务都有一个单独的模块，比如建立连接时，使用数据库，利用集合，操纵游标，使用DBRef模块，转换ObjectId以及运行服务器端JavaScript代码...

## 连接数据库

引入MongoClient模块，建立连接

```python
>>> from pymongo import MongoClient
```

```python
>>> c=MongoClient()//不指定参数时，默认接口是localhost：27017
>>> db=c.inventory//使用inventory数据库
>>> db
Database(MongoClient(host=['localhost:27017'], document_class=dict, tz_aware=False, connect=True), 'inventory')//表示已连接到localhost，并使用inventory数据库
>>> collection=db.items//使用items集合
```

## 插入数据

先定义文档item（一个dictionary）、two_items（是一个包含多个dictionaries的array）

然后使用`collection.insert_one(item)`或`collection.insert_many(two_items)`将文档插入集合中

## 查找数据

### 查找单个文档

`collection.find_one()`查找匹配条件的（第）一个文档

`collection.find({})`根据其中的参数查找文档（如果不指定参数，默认返回集合中的全部文档）

例如，查找"ItemNumber" 为 "3456TFS"的文档

```python
collection.find_one({'ItemNumber':'3456TFS'})
```

### 查找多个文档

使用游标读取多个文档

例如，使用for循环读取集合中的全部文档

```python
for doc in collection.find():
        doc
```

```python
for doc in collection.find({"Location.Owner" : "Walker, Jan"}):
        doc
```

### 使用点表示法查找文档

点表示法用于 以嵌入式对象中的元素为条件查找文档

在嵌入式对象中指定项目的键名来进行查询

例如，

```python
>>> for doc in collection.find({"Location.Department" : "Development"}): 
            doc
```

### 查找之后返回指定字段

在`find()`中包含一个附加参数，指定仅需要返回的某些字段为True，不需要返回的为False

*除_id字段之外，不能在查询中混合使用True和False*

例如，查找'Status' 为 'In use'的文档，并仅显示其'ItemNumber'、'Location.Owner'、_id（默认id都会返回，除非指定为False）

```python
>>> for doc in collection.find({'Status' : 'In use'}, {'ItemNumber' : True, 'Location.Owner' : True}): 
             doc 
```

### 简化查询

+ 使用`sort()`对查询结果进行排序

例如，查找'Status' 为'In use'的文档，仅显示其'ItemNumber'、id值和'Location.Owner'，且以'ItemNumber'为准升序排序返回结果

```python
>>> for doc in collection.find ({'Status' : 'In use'}, {'ItemNumber' : True, 'Location.Owner' : True}).sort('ItemNumber'):
	      doc 
```

+ 使用`limit()`限制查询返回的结果数

例如，查找所有文档，仅显示"ItemNumber"和id值，且限制返回结果为前两个

 ```python
>>> for doc in collection.find({}, {"ItemNumber" : True}).limit(2):                          doc 
 ```

+ 使用`skip()`跳过前n个文档再进行查找

例如，跳过前两个文档，查询之后的所有文档，并只显示"ItemNumber"和id值

```python
>>> for doc in collection.find({},{"ItemNumber":True}).skip(2):
             doc
```

将三个函数结合起来使用（顺序可以互换不影响结果）

```python
>>> for doc in collection.find( {'Status' : 'In use'}, {'ItemNumber' : True, 'Location.Owner' : True}).limit(2).skip(1).sort ('ItemNumber'): 
                doc 
```

### 聚合查询（aggregating queries)

1. 使用`count()`计算数目

例如，查看集合中有多少个文档

```python
collection.find({}).count()
//collection.count()也可以返回结果但不推荐使用
```

```python
collection.count_documents({"Status" : "In use", "Location.Owner" : "Walker, Jan"})
//collection.find().count()也可以返回结果但不推荐使用
```

2. 使用`distinct()`返回唯一项的值

当集合中存在重复项时，使用`distinct()`返回不重复项

例如，仅查找"ItemNumber"互不相同的值并返回其值

```python
collection.distinct("ItemNumber") 
```

返回结果为：[u'1234EXD', u'2345FDX', u'3456TFS']

3. 使用聚合框架将数据分组

聚合框架无需使用MapReduce即可计算聚合值

例如，`aggregate（）`最强大的管道运算符之一`$group`将用于按标签将先前添加的文档分组，并使用`$sum`聚合表达式对其进行计数。

```python
>>> cursor=collection.aggregate([
     {'$unwind' : '$Tags'}, //使用$unwind管道运算符从文档的‘$tags’数组中创建tag文档流
     {'$group' : {'_id' : '$Tags', 'Totals' : {'$sum' : 1}}} 
    //调用$group管道运算符，使用‘$tags’的每个值作为_id，为每个唯一tag创建单独行（row）
    //使用$sum计算每个tag出现的次数
     ])
>>> for c in cursor:
    print(c)
```

得到结果：{  u'ok': 1.0,  u'result': [    {u'_id': u'Laptop', u'Totals': 4},    {u'_id': u'In Use', u'Totals': 3},    {u'_id': u'Development', u'Totals': 3},    {u'_id': u'Storage', u'Totals': 1},    {u'_id': u'Not used', u'Totals': 1}  ] }

+ 添加`$sort`管道运算符按顺序返回结果

```python
>>> from bson.son import SON//引入SON模块
>>> collection.aggregate([
     {'$unwind' : '$Tags'}, 
     {'$group' : {'_id' : '$Tags', 'Totals' : {'$sum' : 1}}}, 
     {'$sort' : SON([('Totals', -1)])} //按‘Totals’值降序返回结果
])
```

除了`$sum`管道运算符之外，`$group`中还支持以下运算符：

•	`$push`: Creates and returns an array of all the values found in its group. 

•	`$addToSet`: Creates and returns an array of all the unique values found in its group.

 •	`$first`: Returns only the first value found in its group. 

•	`$last`: Returns only the last value found in its group.

 •	`$max`: Returns the highest value found in its group.

 •	`$min`: Returns the lowest value found in its group. 

•	`$avg`: Returns the average value found in its group. 

### 使用`hint()`指定索引

在查询数据时，可以使用`hint()`指定使用的索引

可以优化查询速度、性能

```python
>>> from pymongo import ASCENDING //引入模块，使能够升序排序
>>> collection.create_index([("ItemNumber", ASCENDING)])//以"ItemNumber"创建索引
>>> for doc in collection.find({"Location.Owner" : "Walker, Jan"}) .hint([("ItemNumber", ASCENDING)]): 
             doc 
```

### 使用条件操作符优化查询

1.  `$lt`, `$gt`,` $lte`（小于等于）,  `$gte`（大于等于）,`$ne` (不等于)

```python
>>> for doc in collection.find({"Location.Desk" : {"$lt" : 120102} }): 
             doc 
//查找"Location.Desk"小于120102的文档
>>> for doc in collection.find({"Location.Desk" : {"$gt" : 120102} }):
             doc 
//查找"Location.Desk"大于120102的文档
```

2. 使用`$in`,`$nin`,`$all`匹配数组中的值进行查找

   *`$in`只需要满足[ ]内的某个值即可返回相关文档，`$all`必须满足[ ]内的所有值*

例如，查找"tags"数组中含有"Not used"或"Development"的文档

```python
>>> for doc in collection.find({"Tags" : {"$in" : ["Not used","Development"]}} ,{"ItemNumber":"true"}).limit(2): 
             doc 
```

例如，查找"tags"数组中含有"Storage"和"Not used"的文档

```python
>>> for doc in collection.find({"Tags" : {"$all" : ["Storage","Not used"]}},  fields={"ItemNumber":"true"}): 
             doc 
```

3. 使用`$or`匹配多项条件

   例如，只要满足 "Location.Department" : "Storage"或"Location.Owner" : "Anderson, Thomas"其中一个条件即返回相关文档

```python
>>> for doc in collection.find({"$or" : [ { "Location.Department" : "Storage" },      { "Location.Owner" : "Anderson, Thomas"} ] } ): 
             doc 
```

4. 使用`$slice`从数组中读取项目

`$slice`仅对一个文档中的一个数组进行操作。

例如，查找'ItemNumber' 为 '6789SID'的文档，并只显示'PreviousLocation' 数组的前三个元素

```python
collection.find_one({'ItemNumber' : '6789SID'}, {'PreviousLocation' : {'$slice' : 3} })
```

 `{'$slice' : -3}`仅显示数组中最后三个元素

`{'$slice' : [5, 3] } `跳过前五个元素，显示之后的三个元素

*`$slice`是对网页分页很有用的运算符*

### 使用正则表达式查询

例如，

```python
>>> import re//引入正则表达式模块
>>> for doc in collection.find({'ItemNumber':re.compile('4')})：
            doc//查找'ItemNumber'中包含数字4的文档
>>> for doc in collection.find({'ItemNumber' : re.compile('.FS$')}): 
                 doc//查找'ItemNumber'中以FS为结尾的文档
```

如果不想区分大小写（默认区分大小写），添加其他参数

例如，查找'Location.Owner' 中以anderson开头的文档，不区分大小写

```python
>>> for doc in collection.find({'Location.Owner' : re.compile('^anderson. ', re.IGNORECASE)}):
            doc 
```

## 修改数据

### 更新数据

`collection.update_one()`

`collection.update_many()`

### 使用修改操作符

1. 使用`$inc`增加整数值

```python
>>> collection.update_one({'ItemNumber':'6789SID'},{'$inc':{'Location.DeskNumber':20}})//使'Location.DeskNumber'值增加20
```

2. 使用`$set`修改存在值

```python
>>> collection.update_many({"Location.Department" : "Development"},
    {"$set" : {"Location.Building" : "3B"} })//将匹配文档的"Location.Building"全部修改为‘3B’
```

3. 使用`$unset`删除字段和其值

```python
>>> collection.update_one({"Status" : "Not used", "ItemNumber" : "2345FDX"}, 
     {"$unset" : {"Location.Building" : 1 } } )
```

4. 使用`$push`向数组中添加值

```python
>>> collection.update_many({"Location.Owner" : "Anderson,Thomas"},
    {"$push" : {"Tags" : "Anderson"} } )
```

5. 使用`$push`和`$each`向数组中添加多项值

```python
>>> collection.update_one({ "Location.Owner" : re.compile("^Walker,") },
   {  '$push' : { 'Tags' : { '$each' : ['Walker','Warranty'] } } } )
```

6. 使用`$addToSet`向数组中添加值

*`$addToSet`与`$push`的不同处在于`$addToSet`在更新数据之前会检查该数组是否存在*

```python
>>> collection.update_many({"Type" : "Chair"}, {"$addToSet" : {"Tags" : "Warranty"} } )
```

7. 将`$addToSet`和`$each`结合起来使用，向数组中添加多个元素

```python
>>> collection.update_one({"Type" : "Chair", "Location.Owner" : re.compile("^Martin,")},
    {"$addToSet" : { "Tags" : {"$each" : ["Martin","Warranty","Chair","In use"] } } } )//重复值不会被添加进去
```

8. 使用`$pop`从数组中移除（第一个、最后一个）元素

`$pop`只允许从数组中删除第一个(使用负数)或最后一个元素(使用正数)

例如，删除'Tags'数组中的第一个元素

```python
>>> collection.update_one({"Type" : "Chair"}, {"$pop" : {"Tags" : -1}})
```

9. 使用`$pull`移除数组中的指定元素

`$pull`可以删除数组中指定值的每一个出现位置，只要该值相同，就会被删除

```python
>>> collection.update_one({"Type" : "Chair"}, {"$pull" : {"Tags" : "Double"} } )
//“Double”在“Tags”数组中出现了两次，这两个“Double”都会被删除
```

10. 使用`$pullAll`移除数组中的多个元素

```python
>>> collection.update_one({"Type" : "Chair"}, {"$pullAll" : {"Tags" : ["Bacon","Spam"] } } ) 
```

### 使用`replace_one()`修改文档

使用`replace_one()`函数快速查找和替换与过滤条件相匹配的文档，如果找不到匹配的文档，则会创建新文档

```python
>>> collection.replace_one(<old document>,<new document>,upsert=True)
```

### 原子操作修改文档

1. 使用`find_one_and_update()`自动查找并修改文档，只用于更新单个文档

`find_one_and_update()`函数中可以包含的参数：

+ query：指定过滤条件查找匹配的文档。如果不指定条件则查找全部文档，只修改第一个文档 
+ update：修改的文档信息。用于`update()`的所有操作符都可以用于此函数
+ upsert：一般设置为True，如果找不到匹配条件的文档，则生成一个新文档
+ sort：按指定顺序排列匹配的文档
+ return_document：将其设置为`return_document:True`，可以返回更新后的文档而不是旧文档（默认）
+ projection

```python
collection.find_one_and_update({'Type':'Chair'},update={'$set' : {'Status' : 'In repair'} }, return_document=True)
```

2. 使用`find_one_and_delete()`移除文档

参数：

+ query(mandatory)
+ sort(optional)
+ projection(optional)

```python
>>> collection.find_one_and_delete(query={'Type' : 'Desktop'}, 
    sort={'ItemNumber' : -1} )
```

## 批量处理数据（in bulk）

批量处理数据只能在一个集合中进行，可以插入数据、更新数据、移除数据...

*原理跟mongo shell中一样*

```python
>>> bulk=collection.initialize_ordered_bulk_op()//初始化有序列表
>>> bulk.insert({"Type" : "Laptop", "ItemNumber" : "2345EXD", "Status" : "Available" })
>>> bulk.insert( { "Type" : "Laptop", "ItemNumber" : "3456EXD", "Status" : "Available" }) 
>>> bulk.find( { "ItemNumber" : "3456EXD" }).update( { "$set" : { "Status" : "In Use" }})//批量写入
>>> results=bulk.execute()//执行写入操作
>>> print(results)//查看批量执行的操作
```

## 删除数据

删除单个或多个文档：`collection.delete_one()`， `collection.delete_many()`

删除一整个集合：`db.items.drop()`， `drop_collection()`

删除一整个数据库：`c.drop_database('inventory')`

## 在两个文档之间建立联系

通过DBRef模块中的`DBRef()`函数在PyMongo中进行数据库引用

可用于在不同位置的两个文档之间创建连接

1. 手动创建链接：

例如，

```python
>>> jan = {"First Name" : "Jan", "Last Name" : "Walker", "Display Name" : "Walker, Jan", "Department" : "Development", "Building" : "2B", "Floor" : 12, "Desk" : 120103, "E-Mail" : "jw@example.com" }
>>> people=db.people
>>> people.insert(jan)//向people集合插入jan个人信息
>>> laptop = {"Type" : "Laptop", "Status" : "In use", "ItemNumber" : "12345ABC",      "Tags" : ["Warranty","In use","Laptop"], "Owner":jan[ "_id" ] }
//laptop的“owner”是jan，使用jan[ "_id" ]来引用jan个人信息
>>> items=db.items
>>> items.insert_one(laptop)//向items集合中插入laptop信息
```

如果要查找jan个人信息，即查找“owner”字段中给定的ObjectId

2. 使用`DBRef()`函数

*只知道原始数据存在的集合就可以使用此功能*

`DBRef()`函数的三个参数：

+ collection(mandatory)：指定原始数据所在的集合（例如people集合）

+ id(mandatory)：指定被引用的文档的id值

+ database(optional)：指定要引用信息的数据库名称

```
>>> from bson.dbref import DBRef//引入模块
```

与手动引用数据区别不大

将`>>> laptop = {"Type" : "Laptop", "Status" : "In use", "ItemNumber" : "12345ABC",      "Tags" : ["Warranty","In use","Laptop"], "Owner":jan[ "_id" ] }`中的`"Owner":jan[ "_id" ]`

改为`"Owner":DBRef('people',jan["_id"])`

*采用这种方法的好处是，在查询引用信息时，无需查找集合名称*

### 检索信息

使用`dereference()`读取引用的信息

```
>>> person=items.find_one({'ItemNumber':"12345ABC"})//将查找的文档存储在person中
>>> db.dereference(person['Owner'])
```









































































































































































































































