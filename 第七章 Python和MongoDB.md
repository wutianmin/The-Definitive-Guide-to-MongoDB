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



































































































































































































































































