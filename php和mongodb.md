## 安装php和mongodb模块

*注意其版本一定要一致*

命令行窗口中输入`powershell`

`php -v`查看php的版本

`php -m`查看php中的模块

`php -S localhost:8000`连接到8000端口

## 插入数据

在根目录（document root）下，创建demo.php文件，在其中写入代码

```
<?php
/*phpinfo();*/
//连接mongodb数据库
$manager=new MongoDB\Driver\Manager("mongodb://localhost:27017");
//如果有密码,则输入
/*$manager=new MongoDB\Driver\Manager("mongodb://phpadmin:123456@localhost:27017/php");*/
//插入数据
$bulk=new MongoDB\Driver\BulkWrite;
$bulk->insert(['name'=>'liubei','age'=>12]);
$bulk->insert(['name'=>'zhangfei','age'=>22]);
$bulk->insert(['name'=>'guanyu','age'=>32]);
//执行插入操作。将数据插入php数据库的stu集合中。
$manager->executeBulkWrite('php.stu',$bulk);
echo 'ok';
?>
```

在浏览器输入`http://localhost:8000/demo.php`打开这个php代码

出现ok则表示插入数据成功。

打开一个新的命令行窗口

输入`mongo php`

之后就可以按mongo shell中的方式进行查询数据等操作。

/*如果有密码，则输入`db.auth('phpadmin','123456')`后再对数据库进行操作*/

## 查询数据

```
//过滤条件
$filter=['age'=>['$gt'=>20]];
//选项（排序，显示不显示某个字段……）
$options=[
        'projection'=>['_id'=>0],
        'sort'=>['age'=>-1],
];
//创建一个查询对象
$query=new MongoDB\Driver\Query($filter,$options);
$cursor=$manager->executeQuery('php.stu',$query);
//显示结果
foreach($cursor as $document){
       print_r($document);
}
```

刷新demo.php文件即可看到查询结果

## 更新数据

```
$bulk=new MongoDB\Driver\BulkWrite;
$bulk->update(
       ['name'=>'liubei'],
       ['$set'=>['name'=>'wutianmin','email'=>'wutianmin@126.com']]
       ['multi'=>false,'upsert'=>false]
       );
$writeConcern=new MongoDB\Driver\WriteConcern(MongoDB\Driver\WriteConcern::MAJORITY,1000);
$result=$manager->executeBulkWrite('php.stu',$bulk,$writeConcern);
```











