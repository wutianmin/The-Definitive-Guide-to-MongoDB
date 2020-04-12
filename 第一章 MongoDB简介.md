# 第一章 MongoDB简介

## MongoDB数据库的设计理念、建立原则

### 选择正确的工具完成正确的工作

一直以来，人们使用传统的关系型数据库（MongoDB是文档型数据库）来存储各种各样的数据，不考虑数据是否适合其相关的模型。

+ MongoDB是在“处理文档而不是行、更加快捷、大规模扩展、简单易操作”的概念上建立起来的，这就意味着需要抛弃一些传统数据库所具有的特点。

但是，缺乏事务（transaction）和其他传统数据库功能并不意味着MongoDB不稳定或不能用于管理重要数据。

MongoDB非常适合解决某些问题如，分析（为网站提供实时的Google Analytics）、复杂化数据结构（博客文章、评论）。

+ MongoDB的另一个关键概念是，数据库始终要有一个以上的副本。

当一个数据库故障时，则可以很容易地从其他服务器还原该数据库。

### 缺乏对事务的固有支持（lacking innate support  for transactions）

MongoDB数据库不包括事务性语义（为数据一致性和存储提供保证的元素）。

### JSON和MongoDB

JSON（JavaScript对象表示法）不只是一种交换数据的好方法，也可以很好地存储数据。

+ MongoDB将所有内容存储在一个文档中，这种情况下MongoDB就像JSON，并且这个模型提供了丰富且富有表现力的数据存储方式。

*JSON有效地描述了给定文档中的所有内容，无需事先指定文档的结构。JSON是无模式的，文档可以单独更新或独立于其他任何文档进行更改。JSON还可以将所有相关数据存储在一个位置。*

+ MongoDB使用**BSON**（binary JSON）来存储数据。

​    BSON添加了一些在JSON中不可行的特点，例如，数字型数据的扩展类型（int32，int64），并支持处理二进制数据。

*JSON与CSV不同，可以存储结构化内容；与XML不同，JSON使内容更易于理解和使用。*

下面就是一个JSON格式存储数据：

```
{    
"firstname": "Peter",
"lastname": "Membrey",
"numbers": [
   {
      "phone": "+852 1234 5678"
   },
   {
      "fax": "+44 1234 565 555"
   }
  ] 
}
```

+ 当在文档中添加复杂内容如列表（数组）时，就会创建嵌入式文档。

### 采用非关系方法（nonrelational approach）

+ MongoDB在BSON文档中存储数据，所以这些数据是独立的。即使相同的文档存储在一起，每个文档之间不构成相关关系。

+ 由于MongoDB中的查询是在文档中查找特定的键和值，因此信息可以轻松的分布到尽可能多的服务器上。每一个服务器都可以查看其包含的内容并返回结果。这几乎有效地实现了线性可扩展性和性能。

+ MongoDB不提供主复制，即两个单独服务器都可以接受写入请求。它具有分片功能，即允许在多台计算机之间划分数据，每台计算机负责更新数据集的不同部分。

### 选择性能与功能（performance & features）

memcached——提供高速数据缓存

## FITTING EVERYTHING TOGETHER

### Generating or creating a key

+ document是MongoDB中的存储单位，在RDBMS中，称为row。

document可以存储复杂数据如：lists,dictionaries,lists of dictionaries。

文档可以由任意数量的key和values组成，key不仅仅是一个标签，它大致相当于RDBMS中的列，我们用key来引用文档中的数据。

MongoDB中每个文档都具有唯一标识符`_id`。

*如果不指定_id值，MongoDB会自动生成，默认 _id格式是ObjectId，是一个12字节的唯一标识符，可以在分布式环境中独立生成。*

### 键和值（keys and values）

documents是由键和值组成的。

键和值总是成对出现。

MongoDB不要求每个文档都有相同的字段或每个有相同名字的字段都得有相同类型的值。

如果键值对不在文档的中，那就相当于不存在，不需要像RDBMS一样得输入“null”。

### Implementing collections

collections一定程度上类似于tables，但是没有tables严格。

在RDBMS中，tables是严格定义的，其中只可以放入指定元素；在MongoDB中，集合中可以放入任何类型的元素，具有很强的灵活性，但是否应将某些文档存储在同一集合中取决于应用程序。

### Database

database就是一系列集合的构成。

## FEATURE LIST

### WiredTiger

MongoDB的默认存储引擎，使MongoDB可以更好地优化驻留在内存和磁盘中的数据，而不会出现以前的一些混乱情况。

### 面向文档的存储方式（BSON）

在许多情况下，BSON占用的空间比其JSON等价的空间更多，但是BSON可以使MongoDB更快处理数据，所以存储的少量开销是可以接受的。

+ BSON更容易遍历和索引数据。

+ 可以轻松快捷地将BSON转换成编程语言的本机数据格式。如果数据以纯JSON存储，则需要进行相对较高级别的转换。
+ BSON提供了JSON的一些扩展。所以BSON可以存储任何JSON文档，但有效的BSON文档可能在JSON中无效。

### 支持动态查询（dynamic queries）

RDBMS：静态数据、动态查询。数据库预先知道数据的结构，所以可以进行假设和优化实现快速动态查询。

CouchDB：动态数据，静态查询。查询之前先要定义。

MongoDB：只需要提供要匹配的文档部分，其余部分由MongoDB自己完成。

### 索引 文档（indexing documents）

+ MongoDB中的所有文档自动用**_id键**建立索引，且不可删除。这个索引确保了每个值都是唯一的。

+ 可以自己建立索引，并确定其是否是唯一的。默认情况下，在重复键上创建唯一索引将返回错误。

+ MongoDB还可以在嵌入式文档上建立索引。例如使用composite indexes（复合索引)。

### 地理空间索引（geospatial indexes)

此功能为基于**位置**的数据建立索引，可以查询例如给定坐标集，一定距离内有多少项等问题。

### 分析查询（profiling queries）

查询分析器（query profiler）（MongoDB的`explain()`）可以提供有价值的信息。  

### 就地更新信息（仅内存映射的数据库）

MongoDB可以随时更新数据，不需要分配另外的空间，索引也无需变化。

此方法的另一个好处是，MongoDB执行延迟写入。 写入和读取内存的速度非常快，但是写入磁盘的速度却慢了数千倍。一个缺点是数据可能不能安全的存入磁盘中。

### 存储二进制数据（binary data）

——GridFS

BSON支持在文档中保存多达16MB的二进制数据。

GridFS通过在文件集合中存储有关文件的信息（元数据）来工作，数据本身被分成许多“块”，这些“块”
存储在“块集合”中。

### 复制数据（replicating data）

起初，MongoDB利用**主从复制（master-slave replication）**，即在任何时候都只有一个数据库来进行写入，来保证数据的安全性。

后来，**副本集（replica）**替代了这个方法。

+ 副本集有一个优先服务器（~master），处理客户的各种写入请求，记录在primary's oplog中。

+ oplog被辅助服务器（可以有很多个）复制，还被用来和当前服务器保持更新。

  *如果主服务器崩溃，副本集中会有一个辅助服务器成为主服务器，并来负责处理客户写入请求。*

严格来说，大多数正常副本集节点必须能够相互连接。例如，一个三节点副本集需要两个正常节点来维持优先级。

### 分片（sharding）

自动分片：MongoDB会处理所有数据，将其拆分和重组，确保将数据发送到正确的服务器，并确保查询能够以最有效的方式运行和合并。

### 映射（map）和归约（reduce）功能

MapReduce 一种编程模型，用于大规模数据集（>1TB）的并行运算。

映射和归约功能以JavaScript形式写入并运行于服务器上。

+ map的功能是寻找满足条件的全部文档，然后reduce功能处理数据。
+ reduce功能一般不会返回一个包含文档的集合，而是一个新的包含所提取数据的文档。

### AGGREGATION FRAMEWORK（聚合框架）

聚合框架是基于pipeline的，可以将查询的各个部分串在一起并得到结果。

















