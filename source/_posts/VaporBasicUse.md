---
title: Vapor的安装和部署
date: 2017-07-26 20:26:22
tags: Server-Swift
---
![Vapor](http://upload-images.jianshu.io/upload_images/1513759-fcef541624cb576e.png)
在Swift后端框架中，Vapor是比较常用的，它发展迅速，语法简洁，社区活跃，现将其在Mac上的简单的使用流程做以介绍。

## 安装

### 一、安装最新版的Xcode

Xcode是免费的可以直接在App Store中直接下载。下载完之后需要打开Xcode来完成安装，这可能需要等一段时间。
![安装Xcode](http://upload-images.jianshu.io/upload_images/1513759-63e932cd79cf7564.png)

### 二、验证Swif是否安装

通过执行`eval "$(curl -sL check.vapor.sh)"`来第二次确定安装是否成功。

### 三、安装Vapor

确定Swift成功安装之后我们来安装Vapor toolbox，这其中包含了Vapor的所有依赖以及创建项目时好用的CLI。

#### 安装Homebrew

Homebrew在安装OpenSSL，MySQL，Postgres，Redis，SQLite等依赖的时候极其有用，没有的时候执行如下命令安装：

	/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"


#### 安装Homebrew Tap

Vapor的Homebrew Tap可以让你安装Vapor的所有包。执行如下命令安装

```bash
brew tap vapor/homebrew-tap
brew update
```  

#### 安装Vapor

执行如下命令安装

```bash
brew install vapor
```  

安装完成之后会出现如下图案：

## 新建项目

### 新建

接下来我们用api模板（toolbox提供了web,api等各种模板）来创建一个Hello的项目。执行如下指令

```bash
vapor new Hello --template=api
``` 

然后我们执行`tree`指令就可以看到如下的目录结构：

![Vapor 目录结构](http://upload-images.jianshu.io/upload_images/1513759-bb8633f9c5a20c04.png)
如果出现`Command not found`不能执行请安装tree软件

```bash
brew install tree
``` 

我们打开如下的目录找到`Routes.swift`文件。在Build方法中我们看到：

```Swift
get("plaintext") { req in

return "Hello, world!"
}
```   

上述表示执行get方法，闭包返回请求结果。

### 编译

确保你所在的是项目的根目录执行如下指令来编译：

```bash
vapor build
```  

执行完毕之后将会看到

``` 
Building Project [Done]
```  

### 发布状态编译

在发布状态编译会提升其性能，执行如下指令：

```bash
vapor build --release
```  

## 启动本地服务

执行如下指令来启动服务

``` bash
vapor run serve
``` 

然后你会看到如下信息：

```bash
Server starting....
``` 

然后就可以在浏览器中执行`localhost:8080/plaintext`来看到刚才的`Hello, World`了。

### 生成Xcode项目

我们刚才做的启动等操作都是通过终端来的，我们也可以使用Xcode来完成，要使用Xcode来执行，我们需要首先创建一个*.xcodeproj文件，执行如下指令来创建：

```bash
vapor xcode
``` 

## 部署

完成上述操作之后你就可以在本地访问自己的服务了，但怎样才能部署到远程，生成自己的链接来访问呢？我们使用Heroku，来完成，Heroku的免费版完全可以满足我们日常练习的需求，并且其简单快捷的Git操作指令一定会让你爱不释手。

### 创建Heroku账户

请在[Heroku官网](https://devcenter.heroku.com/)创建自己的账号。

注意要记住自己的邮箱和密码，因为一会儿需要在终端进行登录。

### 安装Heroku CLI
Heroku CLI用来创建、管理Heroku上apps的命令行工具。执行如下指令来完成安装：

```bash
brew install heroku
``` 

安装完成之后执行如下指令来登录，输入邮箱和密码来登录：

	heroku login
	Enter your Heroku credentials.
	Email: adam@example.com
	Password (typing will be hidden):
	Authentication successful.
 
### 创建程序

进入你要部署的App，比如上文的Hello项目，然后执行：

```bash
$cd Hello
$git init
$git add.
$git commit -m "add hello app"
``` 

然后在Heroku中创建一个app，以便其接收你的代码，执行`heroku apps:creat [NAME]`指令其中名字必须以字母开头，字呢个包含小写字母，数字和连字符，并且其在heroku的所有程序中必须是唯一的。如果出现：

```bash
Creating app... done, ⬢ young-island-91962
         https://young-island-91962.herokuapp.com/ | https://git.heroku.com/young-island-91962.git
```  

这说明执行成功了。
执行`git remote -v`，就会出现远程git的URL了

        heroku	https://git.heroku.com/young-island-91962.git (fetch)
        heroku	https://git.heroku.com/young-island-91962.git (push)
然后执行

```bash
heroku create
Creating falling-wind-1624... done, stack is cedar-14
http://falling-wind-1624.herokuapp.com/ | https://git.heroku.com/falling-wind-1624.git
Git remote heroku added
```  

### 注意坑

然后执行`git push heroku master`
这时会报错：

```bash
No default language could be detected for this app.
          remote: HINT: This occurs when Heroku cannot detect the buildpack to use for this application automatically.
``` 

这说明heroku官方没有swift的`buildpack`，所以我们要自己添加

     heroku create --buildpack https://github.com/kylef/heroku-buildpack-swift.git
     heroku buildpacks:set https://github.com/kylef/heroku-buildpack-swift.git
     
最后在执行push的时候还会报错：

`error at=error code=H10 desc="App crashed" method=GET path="/plaintext"`，这时需要修改`Procfile`文件，`Procfile`需要放到项目的根目录里，内容如下：

```bash
web: Run --env=production --workdir=./ --config:servers.default.port=$PORT
``` 

这样就可以执行Git的push指令进行部署了：

```bash
git push heroku master
``` 

随后的操作就变得像平时提交项目一样简单。

```bash
git commit -m "change something" -a
git push heroku master
``` 
这样一个项目就部署成功了。


