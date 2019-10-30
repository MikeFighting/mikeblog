---
title: iOS开发Tips
date: 2016-05-18 07:41:00
tags: Others
---

在iOS开发中，应用大多以XML或JSON的格式传输数据的，并且XML和JSON通常会比较大，所以客户端需要用下载或者上传的时间会较长，这时我们可以考虑压缩数据，Gzip是一种比zip更优的压缩技术，它可以将数据压缩到60%，因此对客户端和服务器端来说就更加的轻量级了。总结下Gzip的优点:

1. 降低客户端对数据的下载时间和上传时间
2. 节省流量**。那么在IOS中我们如何使用gZip呢？

我们可以使用：`LFCGzipUtility`这个框架来进行，我们可以先讲字符串转换成NSData，然后压缩成NSData:

```objc
NSData * gZipData = [LFCGzipUtility gzipData:[needCompressedString dataUsingEncoding:NSUTF8StringEncoding]];
```

解压缩和这个几乎一样，这里就不再赘述

1.iOS开发中，由于版本经常更新，为适配新的版本我们通常要相应的更新Xcode,通常我们可以在iTunes上直接更新，但是由于网速的问题，一般会非常之慢。这时候我们可能会选择在网上找一个安装包，由于前段时间在网上看到有些Xcode有病毒，所以最好用官方的版本。这里提供了官方的下载地址，要比在iTunes上更新快很多的。打开这个网址:
https://developer.apple.com/downloads/

详细得步骤如下所示:
![官网下载各种版本Xcode详解](http://upload-images.jianshu.io/upload_images/1513759-c25cec0f43eb304e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.当我们的Xcode升级了，这时候很多第三方插件都不可用了，这时候怎么办？

这个时候我们需要给`plist`文件加一个键值对就可以了,找到plist止呕中的** DVTPlugInCompatibilityUUIDs**新加一个item,这个item的value值可以在终端中执行：`defaults read /Applications/Xcode.app/Contents/Info DVTPlugInCompatibilityUUID`来进行获取，如下图所示:
![给plist添加DVTPlugInCompatibilityUUIDs](http://upload-images.jianshu.io/upload_images/1513759-28104bc507eea0f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.要发布新版本了，忽然发现**This certificate has an invalid issuer**，这时候我们一般这么办：

* 下载这个证书,并且双击安装到Keychain。
https://developer.apple.com/certificationauthority/AppleWWDRCA.cer
* 在KeyChain中选中"View"->"Show Expired Certificates"
* 确保"Certificates"是选中的。如下图所示:

![选中Certificates](http://upload-images.jianshu.io/upload_images/1513759-2f3de7433172f4ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 从"login"和''system"选项中，移除和苹果开发者相关的证书即可。[该问题在Stackoverflow中有详细的说明。](http://stackoverflow.com/questions/35390072/this-certificate-has-an-invalid-issuer-apple-push-services)

GitHub上LFCGzipUtility的下载地址:
[https://github.com/levinXiao/LFCGzipUtility](https://github.com/levinXiao/LFCGzipUtility)