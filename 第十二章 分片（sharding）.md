# 第十二章 分片（sharding）

分片是指，可以在多台计算机上分布数据。

## 分片的必要性

一台数据库服务器每秒只能处理有限的查询，网络接口和磁盘驱动器每秒也只能从web服务器传输有限兆字节的数据。

复制集可以解决一些扩展性的问题，允许在多个服务器上创建多个同样的数据库复制集，可以将服务器负载分散到更多计算机上。

## 分割水平和垂直数据

数据分区是分割多个独立的数据库中的数据的机制，这些数据可以共存（在同一个系统中），或远程（在单独的系统中）存储。

共存数据分区的动力是减小每个索引的大小，减少为了更新数据所需的输入/输出（I/O）数据量。

远程数据分区的动力是通过拥有更多的用于存储数据的RAM、通过避免硬盘访问、通过拥有更多的网络接口、通过拥有更多的硬盘输入/输出（I/O）通道，以增加访问数据的带宽。

### 垂直分割数据

垂直分区包括分解列数据和将此部分数据存储在单独的表或集合中。

在关系型数据库中，使用具有一对一关系的联接表就是垂直分区的一种形式。

但是MongoDB中不支持这种形式，因为数据在MongoDB中不是以简单的行列形式存储的。

### 水平分割数据

水平分区是MongoDB中的唯一分区方式，分片（sharding）是常用的一种方法。

分片允许在不同服务器之间分割集合，以提升包含大量文档的集合的性能

例如，当将一组用户记录分片在一组服务器上时，所有姓氏字母以A-G开头的人记录在一个服务器上，H-M的记录在另一个服务器上，以此类推。

分割数据时使用的规则叫做分片键。

+ MongoDB有一种独特的分片方法，即<mongos路由进程>管理 数据的分割和请求到所需分片服务器的路线。如果查询需要从多个分片获取数据，mongos会管理把从每一个分片中获得的数据合并回单个游标这个过程。

*这项功能使MongoDB成为了面向云或者面向Web的数据库*

## 分析一个简单的分片场景

使用分片键时，可能会造成分片不均匀，有一部分存储的数据多，一部分存储的数据少这种情况。

+ 要求1：分片系统最重要的特征就在于，必须确保数据在服务器之间是均匀分布的。这样可以防止限制水平缩放性能的hotspots。

+ 要求2：当将数据集分散到各个服务器时，会减少“一个服务器故障导致所有数据不可用”的机会，但是增加了服务器故障的机会和数量。所以，MongoDB需要拥有以容错方式存储分片数据的能力。
+ 要求3：MongoDB需要能够在系统运行时添加或移除分片。

## 在MongoDB中运行分片

MongoDB使用代理机制（proxy mechanism）来支持分片。提供的mongos守护（daemon）进程充当多个基于mongod的分片服务器的控制器。应用程序附加到mongos进程中，就好像它是一个单独的MongoDB数据库服务器。之后，应用程序将所有命令（更新、查询、删除）都发送到该mongos进程中。

mongos进程负责管理从应用程序发送命令的MongoDB服务器；守护进程重新发出从多个分片到多个服务器的查询，并将结果汇总在一起。

**MongoDB可以实现集合级别的分片，不是数据库级别，在系统中可能只有一两个集合增长到需要分片的程度**

+ 分片原理：

  分片系统使用分片键将数据映射为块（chunks）(即文档键的逻辑连续范围)。 每个块标识一些具有特定连续分片键值范围的文档；这些值使 mongos 控制器能够快速找到包含需要处理的文档的块。然后， Mongodb 的分片系统将这个块存储在一个可用的分片存储中； 配置服务器（config server）跟踪哪个分片存储在哪个分片服务器上。

   这是进行分片的一个重要特性，因为它允许您从集群中添加和删除分片，而无需备份和恢复数据。 

+ 配置服务器（config server）：存储分片的配置文件，存储在集合中每个分片服务器的信息。还充当目录（directories），用于确定每个块的位置。

  ***至少使用三个配置服务器***

### 建立分片配置

1. 建立配置服务器

打开命令行窗口，为成员提供一个名为“config”的复制集，并保持打开

```
mkdir -p db\config\data
mongod --port 27022 --dbpath db\config\data --configsvr --replSet config
```

因为创建的是一个config复制集，所以需要初始化，打开一个新的命令行窗口

```
mongo --port 27022
>rs.initiate()
```

2. 创建分片控制器（mongos）

打开新的命令行窗口，创建分片控制器，监听端口27021。chunksize设为最小1MB（实际中chunksize不能小于文档的最大size（16MB），默认情况下chunksize为64MB。

```
mongos --configdb config/<hostname>:27022 --port 27021 
```

更改chunksize

```
>use config
>db.settings.save({_id:"chunksize",value:<sizeInMB>})
```

3. 创建两个分片服务器

新建两个命令行窗口

```
mkdir -p db\shard0\data
mongod --port 27023 --dbpath db\shard0\data//创建分片server
```

```
mkdir -p db\shard1\data
mongod --port 27024 --dbpath db\shard1\data//创建分片server
```

4. 告诉分片系统分片服务器的位置

```
mongo <hostname>:27021//连接mongos实例
>sh.addShard("<replica set>/<hostname>:27023")//添加分片？
>sh.addShard("<replica set>/<hostname>:27024")//添加分片
```

检查分片系统状态

```
>db.printShardingStatus()
```

即成功建立了分片工作环境

5. 输入分片数据

创建testdb数据库，testcollection集合，将该集合分片

```
>sh.enableSharding("testdb")
>sh.shardCollection("testdb.testcollection"，{testkey:1})
```

现在就已经建立了拥有两个分片存储服务器的分片集。

在php中连接到分片控制器（mongos），并使用随机的testkeys和一些testtext插入100000条数据，

```
<?php 
// Open a database connection to the mongos daemon 
$mongo = new MongoClient("localhost:27021");
// Select the test database 
$db = $mongo->selectDB('testdb'); 
// Select the TestIndex collection
$collection = $db->testcollection;
 
for($i=0; $i < 100000 ; $i++){    
$data=array();       
$data['testkey'] = rand(1,100000);  
$data['testtext'] = "Because of the nature of MongoDB, many of the more "                            . "traditional functions that a DB Administrator "          
. "would perform are not required.  Creating new databases, "       
. " collections and new fields on the server are no longer necessary, "     
. " as MongoDB will create these elements on-the-fly as you access them."     
. " Therefore, for the vast majority of cases managing databases and "     
. "schemas is not required.";     
$collection->insert($data); }
```

6. 运行程序

```
$php testshard.php
```

在命令行窗口连接mongos实例，验证数据是否已经存储

```
mongo localhost:27021 
>use testdb 
>db.testcollection.count()
```

连接第一个分片，查询其中有多少记录

```
mongo localhost:27023
>use testdb
>db.testcollection.count()
```

连接第二个分片，查询其中有多少记录

```
mongo localhost:27024
>use testdb
>db.testcollection.count()
```

*根据查询分片的时间不同，记录的数量也会不一样。因为，随时间推移，mongos会重新平衡在两个分片中的数据量，以达到最平均的分配。*

### 添加一个新的分片

```
mkdir -p db\shard2\data
mongod --port 27025 --dbpath db\shard2\data
```

```
mongo localhost:27021
>sh.addShard("replica set/<hostname>:27025")//添加分片
>db.printShardingStatus()//查看状态
```

登录分片服务器

```
mongo localhost:27025
>use testdb
>show collections
>db.testcollection.count()
```

可以看到，shard2分片服务器中的数据量在逐渐增加，表示分片系统正在从shard0 和shard1 服务器中分配数据到shard2中.

### 删除分片

```
mongo localhost:27021
>use admin
>db.runCommand({removeShard:"<hostname>:27025"})
```

运行程序后，mongos开始删除分片，需要一定时间，state为ongoing。最终完成删除分片，state为completed。

检查分片是否已经删除，列出所有分片服务器：

```
>db.runCommand({listshards:1})
```

### 决定连接服务器的方式

应用程序可以连接到标准数据库服务器（mongod），也可以连接到分片控制器（mongos）。在mongod和mongos运行程序的方式是类似的。

MongoDB提供`isdbgrid`命令，用于查询连接的数据库系统

```
mongo
>use testdb
>db.runCommand({isdbgrid:1})
```

或使用`ismaster`

 ### 查看分片集的状态

```
mongo localhost:27021
>sh.status()
```

### 使用复制集进行分片

在向 sharded 集群中添加分片时，可以提供复制集的名称和该复制集的成员的地址，该分片将在每个复制集成员上进行实例化。

mongos会跟踪哪个实例是复制集的当前主服务器，确保将所有分片写在正确的实例上。

*将复制集 和分片结合在一起可以创造高性能、高可靠性的集，并有一定的容错率*

## BALANCER

mongos进程中包含一个平衡器元素，该元素在集中移动逻辑数据块，以确保块被均匀的分布的在各个分片中。

平衡器是自动进行工作的

```
//连接到mongos
>sh.stopBalancer()//停止平衡器
>sh.startBalancer()//启动平衡器
```

+ 查看平衡器的状态

```
>use config
>db.settings.find({_id:"balancer"})
```

即使balancer的状态是停止的，不意味着没有正在进行的“数据迁移”

+ 检查数据是否正在迁移

```
>use config
>db.locks.find({_id:"balancer"})
```

+ 设定启动balancer的时间

  将balancer的启动时间设为8pm开始，6am停止。

```
>use config
>db.settings.update({_id:"balancer"},{$set:{activeWindow:{start:"20:00",stop:"6:00"}}})
```

## 哈希分片（hashed shard keys）

哈希分片会使用MD5算法，为给定字段上的每一个值都创建一个散列，使用这些散列执行组块和分片操作

*当为浮点数创建哈希分片时，可能会造成一些限制，比如2.3/2.4/2.9都会变成同样的散列*

创建哈希分片：

```
>sh.shardCollection("testdb.testhashed",{_id:"hashed"})
```

之后将为_id进行哈希分片，以更加随机的方式分配数据。

## 标签分片（tag sharding）

设置标签，将给定分片键的值，定向到集群中的特定分片上。

1. 添加tags

   例如，基于地理位置分配数据，一个位置是us一个位置是欧盟

```
>sh.addShardTag("shard0000","US")
>sh.addShardTag("shard0001","EU")
```

2. 分割一个新的集合，存放分片键代表的特定区域，并作为第一个分片键(这样能够优先使用tags分割块)

```
>sh.shardCollection("testdb.testtagssharding",{region:1,testkey:1})
```

3. 添加规则

   给定一个值的范围，即分配到某个分片的范围

```
> sh.addTagRange("testdb.testtagsharding", {region:"EU"}, {region:"EV"}, "EU") 
> sh.addTagRange("testdb.testtagsharding", {region:"US"}, {region:"UT"}, "US")
```

之后每个满足条件的文档都会被分配到应该在的分片上。

4. 移除添加的tags规则

```
>use config
>db.tags.remove({ns:"testdb.testtagssharding"})
```

重新添加tags规则，将minkey和maxkey加入范围中，以覆盖所有标记范围

```
> sh.addTagRange("testdb.testtagsharding", {region:MinKey}, {region:"US"}, "EU")
> sh.addTagRange("testdb.testtagsharding", {region:"US"}, {region:MaxKey}, "US")
```

## 添加配置服务器

在生产环境中，需要有不止一个配置服务器

有额外的配置服务器意味着，当一个配置实例崩溃或损坏时，可以有额外的冗余。

```
mkdir -p db\config2\data 
$ mongod --port 27026 --dbpath db\config2\data --configsvr --replSet config 
mkdir -p db\config3\data
$ mongod --port 27027 --dbpath db\config3\data --configsvr --replSet config
```

```
mongo
>rs.add("<hostname>:27026")
>rs.add("<hostname>:27027")
```

































