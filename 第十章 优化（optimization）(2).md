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

`db.posts.ensureIndex({author:1},{option1:true,option2:true...})`

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
> 













































































































































