# 第十章 优化（optimization）

## 优化服务器硬件以提高性能

如果数据库服务器的内存太小，或者驱动器太慢，会影响数据库性能。

在生产应用程序过程中，必须谨慎计算正确的硬件配置来实现最佳性能。

## MongoDB存储引擎

两个存储引擎：WiredTiger、MMAPv1

+ 使用`-storageEngine`命令行选项或`storage.engine` YAML选项来指定想要的存储引擎。

当启动MongoDB实例时，存储引擎就是固定的了。通过转储系统中的数据、删除数据库中的内容、再次重新导入数据，才能更改存储引擎。

大多数情况下，MongoDB选择默认的WiredTiger引擎来工作。

WiredTiger引擎相比MMAPv1，有了许多改进。WiredTiger具有内部MVCC（多版本并发控制）模型，完全支持文档级锁定（document-level locking），而MMAPv1只支持集合级锁定。

## MMAPv1下MongoDB内存使用

MongoDB的MMAPv1存储引擎使用内存映射的文件I / O来访问其基础数据存储。

### MMAPv1中工作集的大小

工作集的大小表示在“常规使用过程中”访问的MongoDB实例中存储的数据量。

知道要进行常规使用的部分数据可以使我们正确调整硬件大小从而提高性能，

## WiredTiger下MongoDB内存使用

WiredTiger使用一种内存模型，该模型可以在给定时间内，将尽可能多的相关数据保留在内存中。

将文档储存在内存中的该存储空间成为缓存。

默认情况下，MongoDB保留大约一半的可用系统内存来存储这些文档。使用`wiredTigerCacheSizeGB`命令行选项或`storage.wiredTiger.engineConfig.cacheSizeGB`YAML配置选项来调整剩余可用内存值。

### WiredTiger压缩

WiredTiger支持在硬盘中压缩数据。

+ 三种压缩算法：

1. 默认算法snappy压缩算法，可以提供良好的压缩效果，在CPU使用率方面的开销非常低。
2. zlib压缩算法，提供很高的压缩率，但是在CPU和时间上花费很多。

+ MongoDB中可以进行压缩的内容：

1. 集合中的数据
2. 索引数据
3. 日志数据

*创建的压缩算法会对新集合起作用， 例如，如果使用默认的snappy压缩算法创建一个集合，之后决定希望对集合使用zlib压缩算法，则现有集合将不会切换到snappy，但是任何新集合都将使用zlib进行压缩*

+ 使用压缩算法：

1. 对于日志压缩，使用`storage.wiredTiger.engineConfig.journalCompressor`YAML 配置选项，它采用 想要使用的压缩库的名称的值
2. 对于索引压缩，使用`storage.wiredTiger.indexConfig.prefixCompresstion`YAML 配置选项，它采用true、false值表示是否打开或关闭索引前缀压缩。默认情况下打开。
3. 对于集合压缩，使用`storage.wiredTiger.collectionConfig.blockCompressor`YAML 配置选项，它采用 想要使用的压缩库的名称的值。

### 选择合适的数据库服务器硬件

## 评估查询的性能

MongoDB有两个用于优化的查询性能的主要工具：

1. `explain()`：用来调查单个查询，确定该查询的执行情况

2. MongoDB 分析器（profiler）r：查找性能不佳的查询，并选择候选查询来进行下一步检查

### MongoDB分析器（profiler）

分析器为 每个符合当前分析标准的查询 记录统计信息和代码执行细节。

使用`--profile`或`--slowns`选项启动MongoDB进程。当profiler启动后，MongoDB会将一个（包含每个查询的性能信息和执行代码细节）文档插入到名为`system.profile`的特殊固定集合中。

`system.profile`集合的最大数据限制为1024KB，以便分析器不会将日志信息填充到磁盘中。

1. 启动关闭MongoDB分析器

   ```
   >use inventory
   >db.setProfilingLevel(1)//启动一级分析
   >db.setProfilingLevel(0)//关闭
   >db.setProfilingLevel(1,500)//启动一级分析，并只分析查询执行时间*大于*500毫秒（单位是毫秒）的查询
   >db.setProfilingLevel(2)//启动二级分析，记录一切查询语句
   ```

2. 慢查询 

   查询集合`system.profile`，可以看到刚才查询所执行的计划

   ```
   >db.system.profile.find()
   ```

   •`op`：显示操作类型； 可以是查询，插入，更新，命令或删除。 •`query`：正在运行的查询。 •`ns`：运行此查询的数据库以及集合名称。 •`ntoreturn`：要返回的文档数目。 •`nscanned`：已扫描以返回此文档的索引条目数。 •`ntoskip`：跳过的文档数目。 •`keyUpdates`：此查询更新的索引键的数量。 •`numYields`：此查询产生对另一个查询的锁定的次数。 •`lockStats`：获取或在此数据库的读取和写入锁定中花费的微秒数。 •`nreturned`：返回的文档数。 •`responseLength`：响应的长度（以字节为单位）。 •`millis`：执行查询所花费的毫秒数。 •`ts`：以UTC显示时间戳，指示执行查询的时间。 •`client`：运行此查询的客户端的连接的详细信息。 •`user`：运行此操作的用户。

   ```
   . &gt;db.system.profile.find({millis:{$gt:10}}).sort({millis:-1})//查找执行时间大于10毫秒的查询，并按执行时间逆序排序返回结果
   ```

3. 增加profile集合的大小

   ```
   >db.setProfilingLevel(0)//停止MongoDB分析器
   >db.system.profile.drop()//删除现有的system.profile集合
   >db.createCollection("system.profile",{capped:true,size:50*1024*1024})//重新创建一个固定分析器集合，并指定其大小（bytes形式）。此时创建的集合大小为50MB。
   >db.setProfilingLevel(2)//重新开启MongoDB分析器
   ```

### 使用`explain()`指定某个查询进行分析

使用`explain()`，MongoDB会返回一个 描述这个查询是如何进行的 文档。

```
>use inventory
>db.items.find().explain(true)
```

`queryPlanner`有关如何执行查询的详细信息，包括计划的详细信息。` queryPlanner.indexFilterSet`指示是否使用索引过滤器来满足此查询。` queryPlanner.parsedQuery`正在运行的查询。 这是查询的修改形式，显示了如何在内部对其进行评估。 `queryPlanner.winningPlan`已选择执行查询的计划。 `executionStats.keysExamined`表示扫描以查找查询中所有对象的索引条目的数量。 `executionStats.docsExamined`表示已扫描的实际对象数，而不仅仅是它们的索引条目。 `executionStats.nReturned`cursor上的项目数（即，要返回的项目数）。` executionStages`提供有关如何执行计划的详细信息。` serverInfo`执行此查询的服务器。

### 使用分析器和`explain()`优化查询





























