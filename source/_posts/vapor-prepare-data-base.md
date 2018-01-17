---
title: Swift后端Vapor安装数据库
date: 2017-08-11 09:52:02
tags: Server-Swift
---

![MySQL](http://upload-images.jianshu.io/upload_images/1513759-b14c0ae08140a1d3.png)

Vapor中使用`Fluent`作为数据库的驱动，它现在可支持的数据库类型有：MySQL，SQL lite，MongoDB，PostgreSQL。因为MySQL用得较多，我们先来学习它。

## 安装过程

### 	安装`MySQL-Provider`

在`Package.swift`文件中加入

    .Package(url: "https://github.com/vapor/mysql-provider.git", majorVersion: 2)
    
然后执行执行`vapor clean`和`rm -rf .build Package.pins`，最后执行`vapor update`和`vapor build`。
安装完MySql之后报错`mysql/mysql.h file not found`以及`Could not build Objective-C module CMySQL`，这时因为MySql数据库需要更新，执行下面的指令
 
    brew update && brew install mysql vapor/tap/cmysql pkg-config

然后再执行`vapor xcode`就可以运行成功了。

## 配置

### 在Droplet中添加驱动

我们要往`Config`对象中添加相应的`Provider`，如下所示

```Swift   
import MySQLProvider
let config = try Config()
try config.addProvider(MySQLProvider.Provider.self)
let drop = try Droplet(config)
```     

### 配置Fluent

在`fluent.json`文件中加入如下的配置
          
	{
	    "//": "The underlying database technology to use.",
	    "//": "memory: SQLite in-memory DB.",
	    "//": "sqlite: Persisted SQLite DB (configure with sqlite.json)",
	    "//": "Other drivers are available through Vapor providers",
	    "//": "https://github.com/search?q=topic:vapor-provider+topic:database",
	    "driver": "mysql",
	}
	      
### 配置MySQL 

在Config文件夹下面新建文件`mysql.json`，并添加如下内容
  
	{
	    "hostname": "localhost",
	    "user": "root",
	    "password": "yourPassword",
	    "database": "yourDatabase"
	    "poort": "3306"
	}      

也可以将证书作为url传入MySQL。
 
	{
	"url": "http://root:password@172.0.0.1/hello"
	}
	
### 多份读取（Read Replicas）

多份读取可以通过配置hostname或者是`readReplicas`接口数组来进行配置。在mysql.josn中加入：

		{
		   {
		    "master": "master.mysql.foo.com",
		    "readReplicas": ["read01.mysql.foo.com", "read02.mysql.foo.com"],
		    "user": "root",
		    "password": "password",
		    "database": "hello"
		   }
		}
 
*Tip:也可以将readReplicas用字符串表示，多个字符串用逗号分隔开。*

### 驱动

你可以在Routes中得到`MySQL Driver`（前提是在你自己的MySQL数据库中创建了`your_table`)表。

```Swift 
 import MySQLProvider
 get("mysql") { req in
            
            let mysqlDriver = try self.mysql()
            let user = try mysqlDriver.raw("SELECT * FROM your_table")
            let reusltJon = try JSON(node: user)
            return reusltJon
            
        }
```  

然后在浏览器中输入`http://localhost:8080/mysql`，如果看到输出了相应的JSON传就证明安装成功了。

### 配置缓存

在`Config/droplet.json`里面可以配置`fluent`缓存，这里`fluent`缓存走的是`mysql`：

```Swift   
{
 "driver": "fluent"
}
```  

下次，当启动Droplet的时候，如果出现：

```bash 
Database prepared
```  

就说明安装成功了。

## 帮助

1. 如果运行出现 

```bash
The current hash key "0000000000000000" is not secure.
Update hash.key in Config/crypto.json before using in production.
Use `openssl rand -base64 <length>` to generate a random string.
The current cipher key "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=" is not secure.
Update cipher.key in Config/crypto.json before using in production.
Use `openssl rand -base64 32` to generate a random string.
```  

这说明我们需要运行`openssl rand -base64 <length>`，以及`openssl rand -base64 32`来产生新的hash key和cipher key，并且将原来的数值替换掉。

2. MySql更改密码：

请参考：
1. https://stackoverflow.com/questions/2101694/mysql-how-to-set-root-password-to-null 
2. https://stackoverflow.com/questions/30692812/mysql-user-db-does-not-have-password-columns-installing-mysql-on-osx
3. https://sraji.wordpress.com/2011/08/10/how-to-reset-mysql-root-password/
4. https://stackoverflow.com/questions/9624774/after-mysql-install-via-brew-i-get-the-error-the-server-quit-without-updating
5. https://stackoverflow.com/questions/9624774/after-mysql-install-via-brew-i-get-the-error-the-server-quit-without-updating/9704993#comment16367803_9704993

需要注意的是：*`sudo mysqld_safe --skip-grant-tables`执行完之后，要重新打开一个终端执行*。

