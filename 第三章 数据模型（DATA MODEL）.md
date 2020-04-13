# 第三章 数据模型（DATA MODEL）

## 设计数据库

MongoDB有很强的灵活性，没有预定义的结构，可以在一个集合中放入成千上百的不同结构的文档而不违反MongoDB的规则。

### 集合（drilling down on collections）

MongoDB数据库模型：

![MongoDB数据库模式](C:\Users\wutianmin98\Desktop\11.png)

关系型数据库模型：

![2](C:\Users\wutianmin98\Desktop\22.png)

+ MongoDB中有好几种类型的集合。

  默认集合类型是大小可扩展的，即向其中添加的数据越多，集合就会变得越大。也可以定义有上限的集合。

+ 使用createCollection()创建集合时，集合名称应以字母或下划线开头。

  不能使用空字符串，“null”，最好控制在9个字符之内。

+ 每个集合至少占用两个namespaces，一个用于集合本身，另一个用于集合中创建第一个索引。

### 文档（using documents）

文档由键值对（key-value pairs）构成。

键是字符串（strings）类型，但其值的类型可以有很大不同，例如数组或二进制数据。

> 文档中可以添加的数据类型：
>
> String（字符型） 此常用数据类型包含文本字符串（或其他种类的字符），用于存储文本值。{"country":"Japan"}
>
> Integer(32-bit,64-bit)（整数型） 用于存储数值型数据
>
> > {"rank":1}
>
> Boolean（布尔型） 值为'TRUE'或'FALSE'
>
> Double（浮点型）
>
> Min/Max keys 用于将值分别与最低和最高的BSON元素进行比较
>
> Arrays（数组） 存储数组
>
> > ["Membrey, Peter","Plugge, Eelco","Hows, David"]

> Timestamp 用于存储时间戳，特别是在修改文档时记录时间
>
> Object 用于嵌入式文档
>
> Null
>
> Symbol 与字符串类型类似，通常保留给使用特定符号的类型
>
> Date 用来以Unix时间格式来存储当前日期时间
>
> ObjectID 存储_id
>
> Binary data 
>
> Regular expression 用于正则表达式，所有选项按字母顺序提供特定的字符表示
>
> JavaScript code

*使用`$type`操作符来定义数据类型*

#### 嵌入式文档（embedding）&从另外文档中引用信息（referencing）

+ 嵌入式文档指将某种类型的数据（如包含更多数据的数组）放入文档本身。

+ 引用信息是创建了一个包含指定数据的另一个文档的引用。

*通常在关系型数据库中引用信息。在MongoDB中，使用嵌入式文档更加方便快捷，保证了所有相关的信息都在一个文档中。*

### 创建_id字段

_id是创建新文档时所建立的第一个属性。

+ 自动创建的_id值是一个12个字节的值，包括一个4字节的时间戳（timestamp）（自1970年1月1日以来的秒数），一个3字节的machine ID，一个2字节的process ID和一个3字节的计数器（counter）。

  时间戳和计数器以Big Endian形式存储

+ 手动创建_id值 

  `ObjectId()`

  

## 索引（building indexes）

索引是一个数据结构，用来收集集合中文档的指定字段值的信息。索引可以确保在文档中快速查找。

创建索引的最大好处是，查找常用文档将更加快捷，因为此时查询不需要遍历整个数据库来收集信息。

### 索引对性能的影响

索引会占用空间，所以我们常常需要删除索引、重建索引…这样可以清除一些不合规定的内容。

每个集合最多可以有64个索引。

+ 创建索引可以提升查找速度，但是降低了插入或删除数据的速度。
+ 最好仅为读取次数大于写入次数的集合添加索引。

`listIndexes()`快速浏览目前为止已存储的索引

`db.collection.getIndexes()`查看为某特定集合创建的索引

## 地理空间索引（geospatial indexing）

为一个文档添加地理空间信息，必须包含一个子对象或一个数组。数组的第一个元素指定对象类型，后面是对象的经度、纬度。

例如，

```
> db.restaurants.insert({
name: "Kimono", 
loc: { type: "Point",  coordinates: [ 52.370451, 5.217497]}
})
```

loc中的type用来指定文档的GeoJSON对象类型，可以是 **a Point, a MultiPoint, a LineString, a MultiLineString, a Polygon, a MultiPolygon, or a GeometryCollection**。

> **Point**类型指定对象所在的精确地点位置，需要一个经纬度信息。

> **LineString**类型指定对象沿一条特定线（如街道）延伸，需要起点、终点的经纬度信息。
>
> 如`> db.streets.insert( {name: "Westblaak", loc: { type: "LineString",  coordinates: [ [52.36881,4.890286],[52.368762,4.890021] ] } }  )`

> **Polygon**类型用于指定一个区域，要确保第一个经纬度与最后一个经纬度相同。
>
> 点坐标以**数组中的数组** 写入
>
> 如`> db.stores.insert( {name: "SuperMall", loc: { type: "Polygon",  coordinates: [ [ [52.146917,5.374337], [52.146966,5.375471], [52.146722,5.375085], [52.146744,5.37437], [52.146917,5.374337] ] ] } } )`

> **Multi-**类型是一个所选数据类型的数组
>
> `db.restaurants.insert({name: "Shabu Shabu", loc: { type: "MultiPoint",  coordinates: [52.1487441, 5.3873406], [52.3569665,4.890517] }})`

在文档中添加地理空间信息之后，就可以使用`ensureIndex()`创建索引了。

```
db.restaurants.ensureIndex({loc:'2dsphere'})
```

+ ‘2dsphere’是索引类似地球的球体上的坐标或其他二维信息，默认范围是-180到180。在后面添加min、max参数 可以改变这个范围。

```
db.restaurants.ensureIndex({loc:'2dsphere'},{min:-500,max:500})
```

+ 当查询多个值（位置、类别（升序）），使用复合键扩展地理空间索引

```
> db.restaurants.ensureIndex( { loc: "2dsphere", category: 1 } )
```

### 查找地理位置信息

当插入地理位置信息，创建地理位置索引之后，就可以进行查找了。

+ 如果查找的经纬度太宽泛，将找不到具体数据，使用`$near`操作符

例，

```
> db.restaurants.find( { loc : { $near : { $geometry : { type : "Point",  coordinates: [52.338433,5.513629] } } } } )
```

*当不带任何其他运算符的情况下使用时，$ near将返回前100个数据，并根据它们与给定坐标的距离对它们进行排序。*

使用`$maxDistance`和`$minDistance`（单位为米）操作符来限制查找的结果数。

例如，查找离初始点40千米的餐厅

```
> db.restaurants.find( { loc : { $near : { $geometry : { type : "Point",  coordinates: [52.338433,5.513629] }, $maxDistance : 40000 } } } ) 
```

+ 使用`$geoWithin`操作符查找特定范围内的数据

用于找到位于`$box`（矩形），`$polygon`（选择的特定形状），`$center`（圆形）和`$centerSphere`（球体上的圆形）形状中的数据

例如，

```
> db.restaurants.find( { loc :
{ $geoWithin :
{ $geometry :
{ type : "Polygon" , 
coordinates : [ [          [52.368739,4.890203],  [52.368872,4.890477],  [52.368726,4.890793],          [52.368608,4.89049], [52.368739,4.890203] 
] ]      }    }  } )
```

```
> db.restaurants.find( { loc: { $geoWithin : { $center : [ [52.370524, 5.217682], 10] } } } )
```

+ MongoDB 4.2提供`$geoNear`功能类似`find()`，还显示了结果中每个项目距指定点的距离。

  例如，

```
db.restaurants.aggregate([
   {
     $geoNear: {
        near: { type: "Point", coordinates: [ 52.338433,5.513629 ] },
        distanceField: "dis",
        spherical: true
     }
   },
   { $limit: 3 }
])
```

## Pluggable storage engines(插拔式存储引擎)

不同的存储引擎各有优缺点。

MongoDB的两个存储引擎：the legacy MMAPv1, and the new WiredTiger storage engine. 

WiredTiger存储引擎提供了更精细的并发控制以及本机压缩功能，这样可以更好地利用硬件，降低存储成本，并提高性能。

















