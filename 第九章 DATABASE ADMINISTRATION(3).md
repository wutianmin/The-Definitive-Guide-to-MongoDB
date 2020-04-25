### 添加只读用户

`createUser()`函数可以添加一个参数，允许创建一个仅有只读权限的用户。

```
>use admin
>db.auth("admin","pass")
>use blog
>db.createUser({user:"shadycharacter",pwd:"shadypassword",roles:[{role:"read",db:blog}]})//为用户提供了访问数据库的权限，目的是监视状态和报告问题
```

### 删除用户

使用`removeUser()`从数据库中删除用户。

```
>use admin
>db.auth("admin","pass")//仍然需要先在admin数据库中进行身份验证
>use blog
>db.removeUser("shadycharacter")//删除用户名
```

## 管理服务器

*需要定期调整服务器以实现最佳性能，或者重新配置使其更好匹配当前运行的环境*

+ 三种配置服务器的方法：

  1. 将命令行选项与mongod服务器守护程序结合起来使用

  2. 通过加载配置文件来实现

  3. 可以使用`setParameter`命令来更改大多数的设置

     如，

     ```
     >db.adminCommand({setParameter:1,logLevel:0})//将logLevel重新设置为默认0
     ```
+ 关闭服务器
```
>use admin
>db.shutdownServer()
```

+ 显示服务器状态

```
>db.serverStatus()
```

​      opcounters部分 显示针对数据库服务器执行的每种类型的操作数，如果超出正常比例，表示应用程序有问题

​       asserts部分 显示已引发的服务器和客服端异常或警告的数量。当这个数量急剧增加时，需要仔细浏览服务器的日志文件。这也表示数据库中的数据出现了问题。

+ 显示服务器的版本信息

```
>db.version()
```

## 使用MongoDB日志文件

默认情况下，MongoDB将整个日志文件输出到`stdout`中。可以使用`logpath`选项重新定义日志文件输出的位置。

可以从日志文件的内容中发现问题（如单个计算机过多连接了服务器，其他应用程序逻辑或数据方面存在问题），最重要的是定期对日志文件进行rotate，使其保持合理的大小

```
db.adminCommand({logRotate:1})//之后MongoDB将写入一个新的日志文件，以rotate时的时间戳重命名现有文件，旧文件将会被安全删除
```

## 验证和修复数据

数据被损坏、不完整的几种情况：

•数据库服务器拒绝启动，指出数据文件已损坏

•使用`db.serverStatus()`命令时，在服务器日志文件中，看到assert计数很高。

•从查询中得到奇怪或意外的结果

•集合中的数据数目与期望不符

### 修复服务器

```
mongod --dbpath /data/db --repair
```

修复大型数据库时，磁盘空间可能不足，添加`repairpath`命令行参数，可以指定有足够空间的驱动器

```
mongod -f /etc/mongodb.conf --repair --repairpath /mnt/bigdrive/tempdir
```

### 验证集合

+ 使用`validate()`命令，查看服务器是否被损坏，验证数据库中集合的内容。

```
>use blog
>db.posts.ensureIndex({Author:1})
>db.posts.validate()
```

默认情况下，`validate()`不会检查全部文档。`validate()`选项检查数据文件和索引，并在执行之后给出一些关于集合的统计信息。

+ 如果要检查所有文档，添加参数

```
db.posts.validate(true)
```

+ 当数据库很大，只想验证索引时

  服务器不会扫描数据文件，仅报告存储的集合的信息

```
>use inventory
>db.runCommand({validate:"posts",scandata:false})
```

### 修复验证集合时出现错误

首先查看MongoDB的日志，查看是否有关于错误的信息

+ 修复集合的索引

```
>use inventory
>db.items.reIndex()
```

MongoDB服务器会丢弃当前集合中的全部索引，再重新建立它们。

*如果使用了`repair`服务器选项，也会重建索引*

### 修复集合的数据文件

修复数据库中所有数据文件的最佳（最危险）方法是,在cmd中使用`--repair`或在mongo shell中使用`db.repairDatabase()`。

但是`db.repairDatabase()`会在重建数据文件时，阻止任何进入数据的请求。

```
>use inventory
>db.repairDatabase()
```

修复数据最好还是从数据库备份中恢复。

### 压缩集合的数据文件

对给定集合的数据文件结构进行碎片整理和重组。使用默认的WiredTiger存储引擎恢复磁盘空间。

```
>use inventory
>db.runCommand({compact:"items"})
```

## 升级MongoDB

1. 备份数据，保证数据备份是可行的
2. 停止应用程序或将其转移到另一台服务器
3. 停止MongoDB服务器
4. 升级版本
5. 使用shell对数据执行初始完整性检查
6. 检查数据，重启应用程序

### 执行rolling升级

## 监测MongoDB

`mongostat`工具用来提供 在服务器上发生的情况的简单概述

从概述中可以读出 数据库执行操作的频率，命中索引的速度...等信息

## 使用MongoDB云端管理器













































































































































































































