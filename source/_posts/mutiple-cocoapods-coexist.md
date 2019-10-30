---
title: Cocoapods多版本共存并自由切换
date: 2017-04-25 07:43:00
tags: Others
---

Cocoapods是iOS中的第三方框架管理工具，一台电脑为什么要安装多个版本的Cocoapods呢？在公司里可能存在不同的IOS开发团队分别对不同的业务线进行开发，各个团队之间所用的Cocoapod版本不同，这时你被外派到另外一个团队做开发。

**Rubyems**：简称gems是一个用于对rails组建近些年个打包的ruby打包系统，它提供了一个分发ruby程序喝库的标准格式，还提供了一个管理程序包的工具。Rubyems的功能类似于linux下的apt-get，是个包管理器，可以从远程下载所需的包。
**gem**：你可以这样理解，gem是一系列文件和包的总称，是一些rails项目依赖的软件或者环境，或者是依赖的关系库，当你的项目中缺少的时候，你可以用gem install 来进行安装，这种安装是通过RubyGems这个包管理工具来安装的，当然你也可以通过bundleer来安装。
**RVM**：Ruby Version Manager,ruby版本管理工具，利用它可以很方便的安装多个版本的Ruby。

### 实现的原理

通过RVM来安装多个版本的ruby,再根据不同版本的ruby来安装相应版本的cocoapods，最后使用`rvm use`命令切换不同的ruby环境来使用不同版本的cocoapods.

### 常用的几个指令

*`ruby -v`查看rugy的版本()
*`rvm -v` (查看rvm的版本)
*`gem sources -l`(查看gem shources)
*`rvm list`(查看已安装的所有版本:ruby)
*`rvm use rubyVersion`(使用某个版本的ruby),例如:`rvm use ruby-2.3.3`
*`rvm install rubyVersion`(安装某个版本的ruby),例如:`rvm install 2.3.3`
*`rvm use rubyVersion --default`(将某个版本的ruby设置为默认版本),例如`rvm use 1.9.3 --default`
*`rvm remove rubyVersion`(删除某个版本的ruby),例如:`rvm remove 1.9.3`
*`rvm list known`查看所有可用的ruby版本
*`sudo gem install cocoapods -v <Version> -n /usr/local/bin`安装cocoapod
*`gem list`查看当前gem下的所有安装包

### 实现步骤

步骤一、 执行`rvm -v`，如果发现没有`rvm`则执行`curl -L get.rvm.io | bash -s stable && source ~/.rvm/scripts/rvm`安装rvm.
步骤二、 执行`rvm list known`查看所有可用的ruby版本，然后执行`rvm install someVersion`来执相应版本的ruby； 或者从[ruby官网](https://rvm.io/binaries)上下载不同版本Ruby时，一定要下载osx操作系统的，否则在执行`rvm mount ~/Downloads/ruby-2.3.3.tar.bz2`时，将会出现`Libraries missing for ruby-2.3.3: xcrun. Refer to your system manual for installing libraries`，下载完之后到响应的目录下执行`rvm mount ruby-2.2.3.tar.bz2`就可以安装对应的ruby。
步骤三、重复执行步骤二，安装不同版本的ruby，

### 各种错误及处理方式

#### cannot execute binary file这种错误

执行完:`rvm use ruby-2.3.3`和`sudo gem install cocoapods`之后出现:`/Users/a58/.rvm/rubies/ruby-2.3.3/bin/ruby: /Users/a58/.rvm/rubies/ruby-2.3.3/bin/ruby: cannot execute binary file`这种错误。

#### Gemset''does not exist

执行完`rvm use ruby-2.2.3`,出现:

```bash
Gemset''does not exist,'rvm ruby-2.2.3 do rvm gemset create ' first, or append '--create'.
```

这种错误是由于没有设置`default`，在执行`rvm list`的时候会出现如下`

```bash
# Default ruby not set. Try 'rvm alias create default <ruby>'.
```

这样的提示。使用`rvm --create ruby-2.1.9` 之后这种提示消失。逐个将其他版本的ruby也使用`rvm --create rubyVersion`这个指令，然后就可以切换至不同版本的ruby了。

#### ERROR:Could not find a valid gem

在执行`sudo gem install cocoapods`来安装cocoapods 的时候,

```bash
ERROR:Could not find a valid gem 'cocoapods' (>= 0), here is why:
Unable to download data from https://ruby.taobao.org/ - SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed (https://ruby.taobao.org/specs.4.8.gz)
```

这是因为淘宝镜像最近出问题了，使用`gem sources -a http://rubygems-china.oss.aliyuncs.com`再安装一个镜像，然后可以执行`sudo gem install cocoapods`了，但是在执行`pod -v`,`pod search AFNetworking`,`pod setup`时却发现:

```bash
/Users/a58/.rvm/rubies/ruby-2.2.3/lib/ruby/2.2.0/rubygems/dependency.rb:315:in 'to_specs': Could not find 'cocoapods' (>= 0) among 6 total gem(s) (Gem::LoadError)`。
```

#### xcrun: error: active developer path...

在执行`rvm install 2.1.0`时报错:

```bash
`xcrun: error: active developer path ("/Applications/Xcode.app/Contents/Developer") does not exist, use 'xcode-select --switch path/to/Xcode.app' to specify the Xcode that you wish to use for command line developer tools (or see 'man xcode-select')
```

这是因为rvm寻找的路径是`/Applications/Xcode.app/Contents/Developer`，而我的Xcode被我修改成了`Xcode8.0`,找不到路径了，所以把Xcode的名字改过来就好了。

#### Error running...

需要更新Homebrew

```bash
Error running 'requirements_osx_brew_update_system ruby-2.1.0',
showing last 15 lines of /Users/a58/.rvm/log/1487732911_ruby-2.1.0/update_system.log`
```

这个时候需要更新`Homebrew`,执行`brew update`来更新Homebrew，这时却发现`Error: /usr/local must be writable!`，然后点击`Command + Shift + G`,然后输入`/usr`这个时候就看到**usr**目录，找到下面的**local**文件夹，右击"Get Info"，将最下面的权限中的everyone改为可读写的，这时就可以执行`brew update`指令了。执行完之后再执行`rvm install ruby 2.2.2`，就可看到如下图所示，就说明ruby安装成功了: 执行`sudo gem install cocoapods`这个指令就可以成安装cocoapods了，接着执行`pod --version`，就可以查看当前的pod版本号了:

```bash
/Users/a58/.rvm/gems/ruby-2.2.2@global/gems/cocoapods-1.2.0/lib/cocoapods/executable.rb:89: warning: Insecure world writable dir /usr/local in PATH, mode 040777
1.2.0
```

在执行完`brew update`之后再执行有关`pod`的指令还是会报错

```bash
/Users/a58/.rvm/rubies/ruby-2.2.3/lib/ruby/2.2.0/rubygems/dependency.rb:315:in 'to_specs': Could not find 'cocoapods' (>= 0) among 6 total gem(s) (Gem::LoadError)
```

这样的错误。然后把现有的cocoapod卸载，执行:

```bash
sudo gem uninstall cocoapods
```

卸载完之后执行

```bash
sudo gem install cocoapods -v 1.2.0 -n /usr/local/bin
```

这时依然会出现这个错误

#### 在使用2.0.0版本的ruby安装pod的时候出现如下错误

```bash
ERROR:SSL verification error at depth 1: unable to get local issuer certificate (20)
ERROR:You must add /O=Digital Signature Trust Co./CN=DST Root CA X3 to your local trusted store
ERROR:SSL verification error at depth 2: self signed certificate in certificate chain (19)
ERROR:Root certificate is not trusted (/C=US/O=GeoTrust Inc./CN=GeoTrust Global CA)
ERROR:While executing gem ... (Errno::EPERM)
Operation not permitted - /usr/bin/pod
```

#### Operation not permitted -

在osx是10.11.6的时候,gem update --system会出错

```bash
ERROR:While executing gem ... (Errno::EPERM) Operation not permitted - /usr/bin/update_rubygems
```

这时候需要到[rubyGem的官网](https://rubygems.org/pages/download#formats)现在最新的zip文件，解压进入到rubygems-2.6.10文件中，然后执行`ruby setup.rb`就可以安装gem了。

####  missing bin/ruby

删除某个版本的`ruby`的时候，出现：

```bash
ruby-2.2.3 [ missing bin/ruby ]
```

这时前往`/Users/用户名/.rvm/rubies/ruby-2.2.3`,然后删除对应的`ruby-2.2.3`即可。

#### rvm instlall 2.2.3 报错

```bash
`Empty path passed to certificates update, functions stack: requirements_osx_update_openssl_cert_run rvm_requiremnts_fail_or_run_action __rvm_osx_ssl_certs_ensure_for_ruby __rvm_osx_ssl_certs_ensure_for_ruby_except_jruby external_import_setup external_import main`，
```

这时执行`rvm reinstall 2.2.3 --disable-binary`这个时候又出现错误:

```bash
dyld: lazy symbol binding failed: Symbol not found: _clock_gettime
dyld: Symbol not found: _clock_gettime
```

其原因在于没有安装Xcode的CommandLineTools工具，执行下面的代码：`xcode-select --install`即可。

#### ERROR:While executing gem...(TypeError)

安装pod时候出现：

```bash
`ERROR:While executing gem ... (TypeError)
no implicit conversion of nil into String
```

执行：gem update --system
