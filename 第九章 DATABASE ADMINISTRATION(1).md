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

### 创建备份

`mongodump -h{mongodb主机名}:{端口} -u{账号}-d{数据库名称}-o{存储路径}`

+ 在MongoDB/bin文件夹下以管理员身份打开Windows powershell

+ 使用mongodump程序来备份数据

  例如，

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

### 备份单个数据库

当只想单独备份每个数据库，不想整块进行备份时

使用mongodump在命令行中添加`-d <database_name>`选项来实现

创建的./dump文件夹仅包含单个的数据库备份文件

### 备份单个集合

添加`-c`选项指定要备份的集合

```
mkdir ~/backuptemp
cd ~/backuptemp
mongodump -d blog -c posts
mongodump -d blog -c tagcloud
cd ~//返回上一级
rm -rf backuptemp//将转储文件夹~/backuptemp存档为tar文件
```

重要选项：

+ `-o [ --out] arg`：指定文件夹路径，存放备份文件
+ `--authenticationDatabase arg`：指定保存用户凭据的数据库，Mongodump将默认使用不带此选项的-db指定的数据库
+ `--authenticationMechanism arg`

## 恢复单个数据库或集合（mongorestore）

可以使用mongorestore还原转储目录中包含的集合或数据库的备份文件

重要选项：

+ `--drop`：使mongorestore在还原现有集合之前将其删除。 有助于确保没有重复项。 如果不使用此选项，则将还原的数据附加（插入）到目标集合中。

+ `--noobjcheck`：使mongorestore在将对象插入目标集合之前，忽略验证对象这个步骤

### 恢复单个数据库

添加`-d`选项恢复单个数据库

```
cd ~/testmongobackup
mongorestore -d blog --drop
```

### 恢复单个集合

添加`-c`选项恢复单个集合

```
cd ~/testmongobackup
mongorestore -d blog -c posts --drop
```

## 自动备份

### 使用本地数据存储方式

只需在指定目录中创建档案的简单备份脚本，在脚本顶部编辑变量以匹配本地系统的变量。

本地数据存储备份脚本中使用的变量：

+ `MONGO_DBS`：这个变量设为空，即在本地服务器上备份所有数据库。或在其中选择要备份的数据库名称。

+ `BACKUP_TMP`：将此变量设置为适合保存备份的转储文件的临时目录。例如，如果使用脚本在本地账户中创建备份，使用"~/tmp"；如果将其用作在系统账户下运行的cron job，使用"/tmp"；在Amazon EC2实例上，使用"/mnt/tmp"

+ `BACKUP_DEST`：此变量存放备份的目标文件夹，单个文件夹会在此文件夹下被创建。将该目录放置在与备份脚本存放位置相关的位置。

+ `MONGODUMP_BIN`：使用此变量来指定此二进制文件的完整路径，在终端窗口中键入"which mongodump"确定合适的路径

+ `TAR_BIN`：此变量设置tar 二进制文件的完整路径，在终端窗口中键入"which tar"确定合适的路径

### 使用云端数据存储方式

+ rsync/ftp/tftp/scp
+ S3 storage

## 备份大型数据库

在复制数据库时，需要保持数据库的一致状态，所以备份不包含在不同时间点复制的文件。

一个数据库备份系统的重点是一个可以快速进行的 point-in-time snapshot。snapshot进行得越快，数据库服务器需要被冻结的时间越短。

### 使用隐藏服务器备份数据库

从隐藏的辅助数据库（hidden secondary）进行备份，备份时该备份数据库可以被冻结。备份完成后，此辅助服务器将重新启动以赶上应用程序。

在MongoDB中很容易建立、配置隐藏的辅助数据库（hidden secondary），它可以使用MongoDB的复制机制追踪主服务器

### 使用日志式文件系统创建snapshot

许多volume managers都有能力在任何特定时间点创建驱动器当前状态下的snapshot。

snapshot允许在进行snapshot时读取驱动器。系统的volume或文件系统管理器保证了，任何在snapshot进行后被更改的磁盘中的数据块 都不会被写回驱动器的同一位置上。这样保留了要被读取的磁盘中的所有数据。

+ 进行snapshot的过程：

1. 创建一个snapshot
2. 由volume管理器决定，从snapshot中复制数据还是 将snapshot重新存储到另一个volume中
3. 释放snapshot.这样做可以将所有的保存的不需要的磁盘块释放回驱动器的可用空间链中
4. 当服务器正在运行时，从复制的数据中备份数据

*该方法的好处是，在进行snapshot时，可以不受阻碍地持续读取数据*

*可行的volume管理器：windows server using shadow copies*

这些volume管理器大多有能力在很短的时间内执行snapshot，即使数据量很大。volume管理器在驱动器上插入了书签，以便在进行snapshot时读取当前状态下的驱动器。

+ 为了使这种方法成为创建备份的有效方法，必须在同一设备上存在MongoDB日志式文件，或者让MongoDB将所有未完成的磁盘刷新写入磁盘中，以便进行snapshot

  该强制刷新功能叫`fsync`，阻止进一步写入的功能叫`lock`.MongoDB可以同时进行这两个功能，在执行fsync后，释放lock之前不会进行下一步写入操作。

```
use admin
db.fsynclock()//使MongoDB进入fsync和lock状态
db.currentOp()//查看当前lock的状态
```

`"fsyncLock" : true`表示MongoDB的fsync进程当前被禁止执行写入操作

此时，可以创建任何snapshot所需的命令，在完成snapshot之后

```
db.fsyncUnlock()//使用此命令解锁
```











































