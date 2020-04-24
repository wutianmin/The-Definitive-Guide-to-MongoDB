### DISK LAYOUT TO USE WITH VOLUME MANAGERS

一些volume管理器可以为分区上的子目录创建snapshot，但大多数不能。

因此最好将计划存储的MongoDB数据的volume，挂载到文件系统上的适当位置（例如，/ mnt / mongodb）。然后，使用服务器配置选项将数据目录、配置文件以及任何其他与MongoDB相关的文件（例如，journal）仅放置在该安装上。 

## 向MongoDB中导入数据

*在bin目录下打开控制窗口*

当需要将大量数据（ZIP code表，IP地理位置表，parts catalogs等）加载到MongoDB中作为引用数据时，使用`mongoimport`将数据导入到集合中。

`mongoimport`可以从CSV,TSV,JSON三类文件中加载数据。

+ `mongoimport`以以下三种形式导入数据：

  1. 字符串

  2. 具有一组列标题名称的文件（这些名称构成MongoDB中的元素名称）

  3. 用于控制数据如何解释的几个选项
     + `--headerline`：使用文件的第一行作为文档内字段名称（只有CSV,TSV格式文件可行）
     + `--ignoreblanks`：不允许导入空字段。如果字段为空，那么那一行相关联的元素不会在文档中被创建。如果调用此选项，则会创建一个有列名称的空元素（？）
     + `--drop`：删除集合，然后使用 从该集合中导入的数据重新创建这个集合。否则，数据会被附加到集合中，形成重复。
     + `--numInsertionWorkers`：当前执行插入操作的数量（默认为1）
     + `--jsonArray`：允许导入/导出 以一个JSON数组形式表达的多个MongoDB文档的数据

*在使用`mongoimport`时，需要指定数据库名称`-d`和集合名称`-c`，默认是JSON格式*

                 ```
$mongoimport -h <host> -d <database name> -c <collection name> --type csv(文件类型) --headerline(选项) -f D:/csvimportfile.csv(文件路径)
                 ```

## 从MongoDB中导出数据

使用`mongoexport`从MongoDB中存在的集合中，导出数据，创建导出文件。

导出选项：

+ `-q`：指定查询要导出的数据。此查询是与`db.collection.find()`函数一起使用的任何用于过滤条件的JSON字符串（{x:{$gt:1}}）。默认选择全部数据。

+ `-f`：列出要导出的数据库中的元素名称

  ```
  $mongoexport -h <host> -u <username> -- password <password> -d <database name> -c <collection name> -q {} -f _id,Title,Message,Author -o D:/blogposts.csv(文件名)`
  ```

## 通过限制对MongoDB服务器的访问来保护数据

当处理的数据是敏感数据时（社交网络上的用户记录，商务应用程序中的付款明细），必须确保对数据库系统中敏感数据的访问受限。

MongoDB支持一个基于角色的身份验证系统，可以允许控制谁有权限访问每个数据库以及授予的访问级别。

使用`admin`数据库，运行一些更改数据配置的特殊命令。

## 通过身份验证保护服务器

MongoDB支持一个 具有预定义系统角色（predefined system roles）、用户自定义角色（user-defined custom roles）的基于角色的访问控制（role-based access control/RBAC）身份验证模型。

MongoDB支持在每个数据库上，单独访问控制记录，这些记录存储在`system.users`集合中。为了访问两个数据库，必须将用户的凭证和r权限添加到两个数据库中。只在一个数据库上更改内容，不会同步到另一个数据库中。

### 添加一个admin 用户

```
use admin
db.createUser({user : "admin", pwd : "pass", roles: [ 
{ role : "readWrite", db : "admin" },
{ role:  "userAdminAnyDatabase", db : "admin" } ] })
```

*此时只需要添加一个admin user即可，定义该用户后，可以使用此用户将其他admin users添加到admin数据库中，或将其他普通用户添加到任何其他数据库中*

### 启用身份验证

终止服务器并且添加`--auth`到startup参数中。

### 在mongo控制台中进行身份验证

```
>db.auth("admin","pass")
1//结果为1则身份验证成功，0为验证失败
>show collections
system.users
system.version
>db.getUsers()//列出所有admin数据库中system.users集合中的内容
```

*如果使用管理员凭证来访问除admin以外的数据库，则也必须首先对admin数据库进行身份验证。 否则，将无法访问系统中的任何其他数据库*

### MongoDB用户角色

`read` `readWrite` `dbAdmin` `userAdmin` `dbOwner` `clusterManager` `clusterMonitor` `hostManager` `clusterAdmin` `backup` `restore` `readAnyDatabase` `readWriteAnyDatabase` `userAdminAnyDatabase` `	dbAdminAnyDatabase`

### 改变用户的凭证

使用`updateUser()`改变用户的进入许可和密码。

可以使用任何常规的数据操作命令来更改给定集合的system.users中的用户记录，但是只有`updateUser()`可以更改创建密码字段。

```
use admin
db.auth("admin","pass")//身份验证admin数据库
use inventory//连接其他数据库
db.createUser({user : "foo", pwd : "foo" , roles: [ { role : "read", db : "inventory" } ] } )//创建初始foo用户
db.updateUser("foo", { roles: [ { role : "dbAdmin", db : "inventory" } ] }) 
//更改授予foo用户dbAdmin角色而不是read角色
```

























































































































































































