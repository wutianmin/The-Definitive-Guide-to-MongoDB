# 第五章 GridFS

## GridFS简介

+ GridFS 用于存储和恢复那些超过16M（BSON文件限制）的文件(如：图片、音频、视频等)
+ GridFS也是文件存储的一个方式，但是存储在MongoDB集合中

+ GridFS库由两个集合组成：

1. fs.files集合：保存文件名和相关信息（元数据metadata（如大小，内容类型...））
2. fs.chunks集合：保存文件数据本身。GridFS 会将大文件对象分割成多个小的chunk(文件片段)，一般为255k/个，每个chunk将作为MongoDB的一个文档被存储在chunks集合中。

## GridFS命令

***在MongoDB安装目录下的bin目录中，在命令窗口使用GridFS***

例如，

```
mongofiles put <filename>//存入文件,shift+右键复制路径粘贴到<filename>处
mongofiles list//列出文件
mongofiles get <filename>//下载文件
mongofiles delete <filename>//删除文件
```

+ 当在mongofiles中再次执行相同的put命令，将放入相同的文档（除了_id键不同）

  但MongoDB不会认为有相同文件名（甚至有相同大小的文件）就是同一文件

### 文件长度

+ 在编写应用程序时，知道文件的大小很重要

例如，通过web（HTTP）发送文件时，需要指定文件的大小

+ 文件被分割成很多个文件块，每个文件块占255K，但这个大小可以改变。

要想知道一个文件需要分割成多少个块，一需要知道文件大小，二需要知道每个块的大小。

### 块的大小

默认的块的大小可以在逐个文件的基础上被更改。

根据文件大小决定块的大小，使用GridFS可以在块级别上查找数据。

例如，如果chunk的大小为默认255K，则可以开始从任何255K的数据块中检索数据

*不能创建大于16MB的块*

### 记录上传文件的日期

"uploadDate"键存储了文件在MongoDB中上传的日期

*"files"集合和MongoDB的其他集合一样，包含文档，可以任意插入需要的键值对*

### HASHING FILES

MongoDB自带MD5 hashing algorithm（哈希算法）

MD5算法可以保证文件的安全性和完整性

## Looking Under MongoDB’s Hood

查看上传的文档的详细信息

```
db.fs.files.find()
```

{ 
    "_id" : ObjectId("5e993ac6453a8700f7f4e0d8"), 
    "length" : NumberLong(21963), 
    "chunkSize" : 261120.0, 
    "uploadDate" : ISODate("2020-04-17T05:12:38.773+0000"), 
    "filename" : "C:\\Users\\wutianmin98\\Desktop\\附件3.信息学院2020届毕业生就业情况摸排-4.15信息16-2.xlsx", 
    "metadata" : {                                                                                                    }

}

查看chunks集合（必须添加一个投影以排除二进制数据；否则，结果会显示所有原始二进制数据）

```
db.fs.chunks.find({},{'data':0})
```

{ 
    "_id" : ObjectId("5e993ac6453a8700f7f4e0d9"), 
    "files_id" : ObjectId("5e993ac6453a8700f7f4e0d8"), 
    "n" : 0.0
}

### 查找文件

使用search命令查找文件

```
mongofiles search <文件名中的部分内容>
```

### 删除文件

delete命令基于文件名删除文件。

*delete命令会删除拥有相同文件名的所有文件*

```
mongofiles delete <filename>
```

### 从MongoDB提取文件

```
mongofiles get <filename>
```

*mongofiles会将数据写入具有相同名称和路径的文件中，这将覆盖原文件*

## GridFS在Python中的应用

```python
from pymongo import MongoClient
import gridfs//引入相关库
db=MongoClient().test//建立数据库连接，mongofiles默认连接test数据库
fs=gridfs.GridFS(db)//准备好GridFS使用
```

+ 存储文件

使用python的`with open('<filename>','rb') as dictionary`以二进制形式读取文件

使用`fs.put()`将文件存入GridFS

```
>>> with open('C:/Users/wutianmin98/Desktop/helloworld.dic','rb') as dictionary:
	  uid=fs.put(dictionary)
```

```
>>> uid
ObjectId('5e996f913e0b1d0ac5d975bb')
```

*pyMongo基于这个uid引用文件。如果想将此文件连接到指定的用户上，需要使用_id*

put()命令可以添加两个参数：filename，content_type分别设置文件名和内容类型，对于直接从磁盘加载文件非常有用

+ 读取文件

直接使用上面定义的uid：_id查找文件

```
>>> new_dictionary=fs.get(uid)
>>> for word in new_dictionary:
	print(word)
```

+ 删除文件

```
fs.delete(uid)
```

























































