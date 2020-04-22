# 第九章 DATABASE ADMINISTRATION

由于MongoDB是非关系数据库，因此不能执行一些传统功能，例如创建新的数据库、集合或字段，MongoDB会在访问时立即创建这些元素。因此大多情况下，不需要管理数据库和模式架构。

但是，这种不必预先定义内容的方式和任何名称都区分大小写的方式可能会导致生成很多意外元素（无关的集合或文档），导致数据混乱。

## 使用管理（administrative）工具

### `mongo`，MongoDB控制台（console）

mongo是一个基于JavaScript的命令行控制台，与其他主流的关系型数据库提供的许多查询工具类似。

mongo可以使用JavaScript编写的脚本程序，存储在.js文件中，根据需要运行，直接与MongoDB数据库进行交互。

+ 将命令存储在以.js为拓展名的文件中，在mongo shell中通过将文件名添加到命令行或使用load()函数来运行这个文件。shell将执行文件中的内容。这对运行重复命令非常有用

*还可以使用许多第三方管理工具*

## 备份MongoDB服务器（mongodump）

### 创建备份101

在mongo.exe程序下以管理员身份打开Windows powershell

+ 使用mongodump程序来备份数据

```
cd ~
mkdir testmongobackup
cd testmongobackup
mongodump//输出之后，此时数据库已经备份到testmongobackup/dump目录中
cd ~/testmongobackup//将数据库还原到执行备份时的状态
mongorestore --drop//--drop选项告诉mongorestore程序 在数据库重新存储每个集合之前先删除它们
                   //这样备份的数据就代替了当前数据库中的数据，避免了数据重复出现在数据库中
```

+ *默认情况下，mongodump程序使用默认端口连接到本地数据库，并提取与每个数据库和集合关联的所有数据，将它们存储在预定义的文件夹结构中*

mongodump创建的默认文件夹结构采用以下形式：`./dump/[databasename]/[collectionname].bson`

*mongodump将它从数据库服务器检索到的json数据以保存在.bson文件中，这只是MongoDB用于存储文档的内部BSON格式的副本*

+ MongoDB服务器维护创建的索引，记录每个集合定义的索引。这些（元数据）也会重新存储在metas.json文件中，所以在还原数据时可以重建索引

+ 当完成备份之后，可以将文件夹归档并存储在任何在线或离线的媒介上，如CD,USB,TAPE,S3 FORMAT

### 备份一个数据库

当只想单独备份每个数据库，不想整块进行备份时

使用mongodump在命令行中添加-d database_name选项来实现

创建的./dump文件夹仅包含单个的数据库备份文件

### 备份一个集合

添加-c选项指定要备份的集合

```
mkdir ~/backuptemp
cd ~/backuptemp
mongodump -d blog -c posts
mongodump -d blog -c tagcloud
cd ~//返回上一级
rm -rf backuptemp//将转储文件夹~/backuptemp存档为tar文件
```

重要选项：

+ -o [ --out] arg：指定文件夹路径，存放备份文件
+ --authenticationDatabase arg：指定保存用户凭据的数据库，Mongodump将默认使用不带此选项的-db指定的数据库
+ --authenticationMechanism arg

## 恢复单个数据库或集合（mongorestore）

可以使用mongorestore还原转储目录中包含的集合或数据库的备份文件

重要选项：

+ --drop：使mongorestore在还原现有集合之前将其删除。 有助于确保没有重复项。 如果不使用此选项，则将还原的数据附加（插入）到目标集合中。

+ --noobjcheck：使mongorestore在将对象插入目标集合之前，忽略验证对象这个步骤

### 恢复单个数据库

添加-d选项恢复单个数据库

```
cd ~/testmongobackup
mongorestore -d blog --drop
```

### 恢复单个集合

添加-c选项回复单个集合

```
cd ~/testmongobackup
mongorestore -d blog -c posts --drop
```

## 自动备份

### 使用本地数据存储方式

























































