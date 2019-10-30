---
title: Mac版OBS设置详解
date: 2016-03-17 15:27:00
tags: Others
---

### OBS是什么？

OBS是目前为止，最好用的直播软件，它支持Windows 7/8/10, Linux并且还支持OS X(Mac电脑的系统)，老外的软件，无广告，全免费，适用于32和64位的各种电脑，所以成为`斗鱼`，`哔哩哔哩`等各种直播网站主播的必备品。

### 怎样使用OBS？

1. 下载安装

进入OBS[官方网站](https://obsproject.com/)，然后点击绿色的OSX 10.8+(**或者是其它的版本**)，下载安装，然后你会看到如下界面![OBS主页面](http://upload-images.jianshu.io/upload_images/1513759-820e18d263ca116f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 点击设置，进入如下界面

(http://upload-images.jianshu.io/upload_images/1513759-56330d89c8171b75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通用中可以设置OBS的语言，点击串流会看到如下界面:
![通用设置](http://upload-images.jianshu.io/upload_images/1513759-8a432ea667b0c50a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)，串流类型选择**自定义流媒体服务器**，下面的*URL*和*流密钥*需要根据直播间中的*直播信息*进行填写，**登陆斗鱼账号，点击用户名--->个人中心--->主播相关--->直播设置--->进入直播房间**

然后可以看到下图：
![我的直播房间](http://upload-images.jianshu.io/upload_images/1513759-a6203bd85cd9618f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 这个时候点击获取推流码，即可看到*rtmp地址*和*直播码*，将其填入OBS中串流的设置中。

4. 这个时候OBS的主页面还是黑色的，没有任何的输入，原因是没有给他添加输入源，这是点击*场景*下面的**+**添加一个场景，点击来源下面的**+**添加一个来源，一般我们会选择*视频捕获*或者只*窗口捕获* 视频捕获是直播电脑摄像头录取的视频，窗口捕获是直播电脑上打开的窗口，这里以*窗口捕获*为例，添加完窗口捕获之后，点击窗口捕获下面的小齿轮:<br>![设置窗口](http://upload-images.jianshu.io/upload_images/1513759-4ba24c915b50123d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后选择需要直播的窗口，这个时候可能里面没有我们要直播的窗口，比如Xcode，这时，可能是我们没有打开Xcode,打开Xcode之后，我们再，次打开OBS，让其重新识别一次窗口，这时就可以选择Xcode了，选择完之后，点击确定。

5. 这个时候点击开始串流，并且打开直播房间的开始直播就可以了。

### 其它：

Mac电脑使用OBS的时候往往会遇到这样一个问题，外界的声音可以录取，但是电脑自身发出的声音，比如音乐，某些网站的声音，是不能直播，这时候我们需要下载一个软件，[soundflower](http://en.softonic.com/s/soundflower:mac)，下载之后，进入系统偏好设置->声音-->输出，然后选中Multi-Output Device<br>![系统设置](http://upload-images.jianshu.io/upload_images/1513759-62bcf9ab8763b2dd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240),然后点击OB中的视频，将音频中的**桌面音频设备设置为Soundflower(2ch)**<br>![OBS设置Sondflower](http://upload-images.jianshu.io/upload_images/1513759-99598b6e3d7992c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)<br>如果噪音太大，这个时候可以点击麦克风下面的齿轮，设置噪音阈值，来对噪音进行过滤。
