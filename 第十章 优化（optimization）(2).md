## 管理索引

索引用于查询（find、findOne）和排序（sort）。

最好为排序创建索引，或者使用`.limit()`。将索引添加到集合时，MongoDB必须维护该索引，并在每次执行任何写操作（更新，插入或删除）时对其进行更新。如果集合中的索引过多，可能会对写入性能产生负面影响。

### 列出索引列表

使用`getIndexes()`列出一个集合中的所有索引，结果返回一个JSON数组。

```
>use inventory
>db.items.getIndexes()
```

返回的结果中可以看到集合自动创建了一个_id索引。id索引在集合被创建时就会自动生成，在集合被删除时也会自动移除。

当在元素上定义索引时，MongoDB将构造一个btree索引，用于有效地定位文档。

### 创建一个索引

使用`createIndex()`为集合添加新的索引。已添加过的索引不会再被创建。

```
>use inventory
>db.items.createIndex({Type:1})//创建顺序btree索引
>db.items.createIndex({Type:-1})//创建逆序btree索引
>db.items.createIndex({"comments.count":1})//使用点表示法为嵌入式文档创建索引
```

如果创建的文档字段是一个数组形式，则索引会包含数组中的所有元素作为单独的索引项。这称为 **多键索引**，每一个文档都与索引中的多个值连接。

### 创建复合索引

为每一个字段都创建一个单独的索引会对数据库中插入以及删除数据产生很大的影响，这使得索引需要每一次都更新。因此添加很多单独索引不助于查询。

复合索引将多个字段组合到一个索引中，减少了集合中的索引数量。

*当术语列表和排序方向与索引结构完全匹配时，sort索引使用复合索引才会有作用*

+ 复合索引的两种类型：

1. 子文档索引：当使用子文档作为索引键时，用来 建立多键索引的元素顺序 与 它们在子文档的内部BSON表示形式中出现的顺序 一致。这使我们在创建索引的过程中，对其没有足够的控制。

   所以需要确保 使用相同的子文档结构 来创建 形成查询时使用的索引

   ```
   db.articles.find({author:{name:"joe",email:'joe@blogger.com'}})
   ```

2. 手动定义复合索引

   ```
   db.posts.createIndex({"author.name":1,"author.email":1})//列出想要创建索引的全部字段并指定顺序
   ```

## 三步创建复合索引

### 建立数据

```
>use comments
>db.comment.insertMany([{ timestamp: 1, anonymous: false, rating: 3 }, { timestamp: 2, anonymous: false, rating: 5 } ,{ timestamp: 3, anonymous:  true, rating: 1 } ,{ timestamp: 4, anonymous: false, rating: 2 }])//插入评论数据
```

### 条件查询

1. 直接查询并查看详细信息

```
>db.comment.find({timestamp:{$gte:2,$lte:4}})//查找时间戳大于等于2，小于等于4的文档
>db.comment.find({timestamp:{$gte:2,$lte:4}}).explain('executionStats')//显示MongoDB查询的详细信息
//此时的条件查询需要对整个集合进行扫描，当集合内数据过大时，会影响性能
```
*explain()中可添加三个参数`"queryPlanner"`（概要模式）（默认）, `"executionStats"`（执行状态模式）, 和`"allPlansExecution"`（所有信息模式）.*选择`"executionStats"`或者`true`参数。

2. 创建索引后进行查询并查看详细信息

```
>db.comment.createIndex({timestamp:1})//创建索引
>db.comment.find({timestamp:{$gte:2,$lte:4}}).explain('executionStats')//再次查看查询的详细信息
```

两种方式输出结果 `"executionStats"`中有细微差别。

`"totalKeysExamined"`从4变成了3，因为在创建索引之后，MongoDB使用索引直接定位到所需要的文档中，跳过了`timestamp`超出范围的文档。

`"totalKeysExamined"`是Mongo扫描范围内，索引键的数量。 `"totalDocsExamined"`是Mongo获取最终结果所需查看的文档数量。

*`"totalKeysExamined"`>= `"totalDocsExamined"`>=`nReturned`。三者相等时，说明创建了理想的索引*

### 相等性测试 plus 范围查询

+ 当Mongo要使用 指向 与查询条件不匹配的文档的索引键时，`"totalKeysExamined"`>`nReturned`。

```
>db.comment.createIndex({timestamp:1})//创建索引
>db.comment.find({timestamp:{$gte:2,$lte:4},anonymous:false}).explain('executionStats')//查看查询的详细信息
```

返回结果为：` "nReturned" : 2.0`.`"totalKeysExamined" : 3.0`.` "totalDocsExamined" : 3.0`

查询返回的结果为两个文档，但是索引依然检查了3个文档。

因为查询时，依旧先检查在2到4之间的时间戳索引，这些满足时间戳条件的文档包括署名的和匿名的，这个信息在文档对本身进行查找之前无法过滤掉。

+ 在这样的情况下，创建一个复合索引可以使`"totalKeysExamined"`= `"totalDocsExamined"`=`nReturned`

```MongoDB
>db.comment.createIndex({timestamp:1,anonymous:1})//对两个查询条件都创建索引
>db.comment.find({timestamp:{$gte:2,$lte:4},anonymous:false}).explain('executionStats')//查看查询的详细信息
```

但是，返回结果为：` "nReturned" : 2.0`.`"totalKeysExamined" : 3.0`.` "totalDocsExamined" : 2.0`

在这个索引查询中，Mongo按照（timestamp：2，anonymous：false）、（timestamp：3，anonymous：true）、（timestamp：4，anonymous：false）顺序检查索引，检查到（timestamp：3，anonymous：true）时，因为anonymous是true，不满足查询条件，所以不再进行文档本身查找，因此` "totalDocsExamined" 为 2.0`

+ 现在，更改创建索引时两个字段的顺序，可以使`"totalKeysExamined"`= `"totalDocsExamined"`=`nReturned`

```
>db.comment.createIndex({anonymous:1,timestamp:1})
//创建新索引之前使用db.getCollection("<collection name>").dropIndexes()删除之前建立的索引
>db.comment.find({timestamp:{$gte:2,$lte:4},anonymous:false}).explain('executionStats')
```

返回结果为：` "nReturned" : 2.0`.`"totalKeysExamined" : 2.0`.` "totalDocsExamined" : 2.0`

***复合索引中创建索引的字段顺序非常重要，决定了执行查找时的顺序***

### MongoDB如何选择一个索引

如果有多个索引，Mongo会选择`"totalKeysExamined" `最小的那个索引（最优）

### 相等性，范围查询，排序

最后一步是给查询得到的结果排序

```
>db.comment.find( { timestamp: { $gte: 2, $lte: 4 }, anonymous: false } ) .sort({rating:-1}).explain(true)
```

直接使用`sort()`函数会对应用程序服务器的RAM增加负担，在复合索引中添加要排序的字段为索引可以解决这个问题

```
>db.comment.createIndex({anonymous:1,rating:1})
>db.comment.find({timestamp:{$gte:2,$lte:4},anonymous:false}).sort({rating:-1}).hint({anonymous:1,rating:1}).explain(true)
```

`hint()`函数中的参数和`createIndex()`中的一致，但造成了`"totalKeysExamined" `数量增加。

对三个字段都创建索引，可以使`"totalDocsExamined"`数量减少

```
>db.comment.createIndex({anonymous:1,rating:1,timestamp:1})
>db.comments.find( { timestamp: { $gte: 2, $lte: 4 }, anonymous: false } ).sort( { rating: -1 }).hint( { anonymous: 1, rating: 1, timestamp: 1 } ).explain(true) 
```

### -

创建复合索引的步骤：

1. 相等性测试——将所有经过相等性测试的字段以任意顺序添加到复合索引中
2. 排序字段（当有多个排序字段时才有用 ）——按和查询的顺序相同的顺序和方向将排序字段插入索引中
3. 范围过滤器（range filters）——首先为基数最低的字段添加范围过滤器，然后为下一个最低基数字段添加范围过滤器，以此类推直到最高基数的字段。

可以省略一些非选择性的相等性测试字段或范围过滤器字段，来减少索引的大小。*如果该字段不会过滤出至少90%的文档，那么添加此字段为索引是不必要的*

## 指定索引选项

可以使用`ensureIndex()`函数，在创建索引时添加选项，如创建唯一索引或允许后台索引。

```
>db.posts.ensureIndex({author:1},{option1:true,option2:true...})
```

### 使用`{background:true}`在后台创建索引

默认情况下，初始索引在前台被创建。在创建索引时，所有在数据库上进行的操作都被禁止了，直到创建完成。

当使用`{background:true}`使创建索引在后台完成时，所有的查询操作不会进行，但服务器会允许写入和读取操作继续进行。

> 当认为现在的创建索引的过程已挂起或花费时间太长，可以选择取消它
>
> ```
> >db.currentOp()//查看当前的operation id
> >db.killOp(<operation id>)//取消进程
> ```
>
> *当使用`killOp(<operation id>)`函数时，不完整的索引也会被移除，以防止在数据库中建立不相关或损坏的数据

### 使用`{unique:true}`创建索引

为保证唯一性，指定`{unique:true}`选项。这意味着，如尝试插入 索引键与现有文档键匹配 的文档，MongoDB会返回错误。

例如，创建唯一索引，使得重复的email无法被添加到数据库中。

```
>db.comment.createIndex({email:1},{unique:true})
```

*如果要向集合中添加唯一索引，必须保证已经将重复数据删除。*

*{unique：true}只能在添加在单个或复合索引中，不能添加在多键索引中*

### 使用`{sparse:true}`创建稀疏索引

稀疏索引就是只包含有索引字段的文档的条目，即使被索引键的值是null，也会被索引。即稀疏索引会跳过那些索引键不存在的文档（不索引所有文档）。

*非稀疏索引（普通索引）会索引每一篇文档，如果文档中不包含索引的字段，则为它存储一个null值*

例如，创建稀疏索引，不包含email字段的文档将不会被索引。

```
>db.comment.createIndex({email:1},{sparse:true})
```

通常唯一索引和稀疏索引结合起来使用。一个既包含稀疏又包含唯一的索引可以避免集合上存在有重复值的文档。

### 使用`partialFilterExpression(<条件>)`创建部分索引（partial indexes）

创建有条件的索引，即在创建索引时添加一个类似于query函数中的条件。

例如，在有name、SKU、price of food字段的集合中创建部分索引，使得通过name仅索引cost大于10美元的文档。

```
>db.restaurants.createIndex({name:1},{partialFilterExpression:{cost:{$gt:10}}})
```

### 使用`expireAfterSeconds:<seconds>`创建TTL(time to live)索引

TTL索引是一种特殊索引，通过这种索引MongoDB会过一段时间后自动移除集合中的文档。

TTL索引通过指定一个使其无效的时间点，给某些数据或请求一个时间限制。这对于某些类型的信息来说是一个很理想的特性，例如机器生成的事件数据，日志，会话信息等，这些数据都只需要在数据库中保存有限时间。

创建TTL索引，在下一次执行TTL删除任务时，将删除 索引字段的值大于TTL选项设置的值的任何文档。每次执行任务的间隔时间为60秒。需要等待一会才会看到文档被删除。

例如，要删除任何早于28天的评论，28天等于2419200秒。

```
>db.comments.find(); 
{     "_id" : ObjectId("519859b32fee8059a95eeace"),     "author" : "david",     "body" : "foo",     "ts" : ISODate("2013-05-19T04:48:51.540Z"),     "tags" : [ ] }
```

```
>db.comment.createIndex({ts:1},{expireAfterSeconds:2419200})//当此文档比系统时间早28天时，此文档将会被删除
>date = new Date(new Date().getTime()-2419200000);
>db.comments.insert({ "author" : test", "body" : "foo", "ts" : date, "tags" : [ ] });//插入一个早于28天的文档
>db.comments.find()//过一段时间查找文档看到此文档被删除
```

注意：

*被索引的字段必须是BSON日期类型*

*只能在单个索引上创建TTL索引，不支持复合索引。*

*_id字段不支持TTL索引*

*不能在固定集合中创建TTL索引*

### 文本查询索引

MongoDB支持全文本搜索（见第八章）

```
>db.posts.createIndex({body:"text"})//为body字段创建文本索引
>db.posts.find({$text:{$search:"MongoDB"}})//查询所有含有"MongoDB"（默认不区分大小写）的文档
>db.posts.find({$text:{$search:"MongoDB",$caseSensitive:true}})//查询所有含有"MongoDB"（区分大小写）的文档
```

### 删除索引

```
>db.posts.dropIndexes()//删除全部索引
>db.posts.dropIndex({"author.name":1,"author.email":1})//删除指定索引。其中参数与createIndex()中的参数一致。
```

### 重建索引

使用`reIndex()`使MongoDB删除旧索引并重建这些旧索引。

```
>db.posts.reIndex()
```

## 使用`hint()`强制使用一个特定索引

MongoDB的查询优化器会选择最优的索引。当优化器选择的最优索引并不合适时，使用`hint()`函数向查询优化器提供提示，从而做出最优选择。

```
>db.posts.createIndex({author.name:1, author.email:1})
>db.posts.find({author:{name:'joe', email: 'joe@mongodb.com'}}).hint({author.name:1,  author.email:1})//强迫使用上面创建的索引

>db.posts.find({author:{name: 'joe', email: 'joe@mongodb.com'}}).hint({$natural:1})
//强迫查询函数不使用索引(即想使用扫描集合中全部文档的方法来筛选文档)
```

## 使用索引过滤器

有时作为用户可能会对给定查询使用哪个索引更加了解。使用`hint()`函数可以告诉系统要使用哪个索引。

但是在某些情况下，不适合从客户端修改数据，特别是当每个新索引都意味着要更改hint时。

所以MongoDB引入了索引过滤器，提供一种方式，使MongoDB对于特定类型的查询使用特定的索引。

```
> db.stuff.find({letter : {$gt : "B"}, shape : "circle"}).sort({number:1})
> db.adminCommand({setParameter:1, logLevel:1})//从日志文件中得输出信息
> db.runCommand(   {      
planCacheSetFilter: "stuff",     
query: {letter : {$gt : "B"}, shape : "circle"},   
sort: {number:1},     
projection: { },  //用来只返回部分指定字段    
indexes: [ { letter:1, shape:1} ]   } ) //创建索引过滤器
> db.stuff.find({letter : {$gt : "B"}, shape : "circle"}).sort({number:1})//再次执行查询
```

```
> db.runCommand( { planCacheListFilters:"stuff"}) //列出集合中现有的索引过滤器
```

```
> db.runCommand(   {     
planCacheClearFilters : "stuff",   
query: {letter : {$gt : "B"}, shape : "circle"},      
sort: {number:1},     
projection: { },     
indexes: [ { letter:1, shape:1} ]   } )//移除任何索引过滤器
```

## 优化小对象的存储空间

索引是加速查询的关键，另一个影响查询性能的因素是其访问的数据的大小。

第一可以使用批量写入功能，减少通过数据库接口API的往返次数。第二可以减小字段名称所占的空间。第三在写入数据时，避免在集合上添加不必要的索引。第四将事件流分成多个集合，小集合中索引和分析所花费的时间更少。

























































































































