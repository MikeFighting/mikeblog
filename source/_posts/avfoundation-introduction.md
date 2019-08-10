---
title: AVFoundation--简介
date: 2017-08-04 23:37:34
tags: Objective-C
---

AVFoundation是很多处理基于时间的音视频文件的框架之一。你可以用它来检查，创建，编辑或者对媒体文件重编码。可以从设备中得到输入流，以及在实时捕捉和播放的时候对视频进行处理。

![AVFoundation](http://upload-images.jianshu.io/upload_images/1513759-d41cc62c5bec457a.png)

 * 如果你仅仅需要播放视频，在IOS上你可以使用Media Player框架中的`MPMoviePlayerController 
`或者`MPMoviePlayerViewController`，如果是基于Web的视频，那么你可以使用`UIWebView`。
 * 为了录制视频，并且几乎不需要关注其格式，那么你可以使用UIKit框架中的`UIImagePickerController`。

然而请注意：在AVFoundation中使用的一些原始数据（包括基于时间的数据结构以及携带及包含媒体信息的封装对象）都是在`Core Media`框架中声明的。

# AVFoundation框架简介

AV Foundation中有两个方面的API(处理视频的和处理音频的)。

* 使用`AVAudioPlayer`来播放音频文件
* 使用`AVAudioRecorder`来录制音频文件

你可以通过`AVAudioSessin`对象来对配置音频的播放行为，相关配置可以参考：`Audio Session Programming Guide`。

AVFoundation中用来代表媒体信息的关键类是AVAsset。该框架的大部分功能都是有AVAsset来表现的。理解AVAsset将有助于理解整个框架是如何工作的。AVAsset是一片或者多片媒体数据的集合。它统一提供这个集合的信息，包括其标题，时长，自然表现大小（natural presentation size）等。一个AVAsset不合具体的数据格式相绑定。AVAsset是其它用URL来创建asset实例的父类。
asset中的每一个独立媒体片是一个统一的类型并且其成为`track`。

## 使用Assets

AVAsset是AVFoundation的基础类。AVFoundation框架的设计很大程度上都是通过这个类来表现的。它提供了视频的名称，时间，正常展示的大小(natural presentation size)。AVAsset没有和具体的数据形式相绑定。AVAsset是从一个`URL`来创建出Asset实例的父类，并且可以创建出新的组合。每个asset中独立的视频片段都是统一的类型叫做`track`。一般情况下：一个track代表音频成分，一个track代表视频成分。很重要的一点是：初始化一个asset并不意味着立即就可以用。它还需要一些时间来计算。但是这种计算不会阻塞主线程，他会异步的执行。


## 播放

`presentation state`是由`player item`对象管理的，某个track的`presentation state`是由`player item track`对象管理的。你可以使用`player`对象来播放，并且直接将其输出到`Core Animation`上。可以使用一个`player queue`有续地来管理一系列的`player items`。


## 读，写以及重新编码`Asset`

AV Foundation让你给一个Asset用不同的方式创建出新的展示。你可以仅仅对一个已经存在的asset进行重编码（ios4.1之后），你可以在一个asset的内容上执行不同的操作，然后将结果用新的asset来存储。将一种表现转化成其他类型的表现，可以使用`asset reader`和`asset writer`来串联两个视频动画。

## 缩略图

使用`AVAssetImageGenerator`对象可以来创建缩略图。编辑，AV Foundation使用`compositions 
`来从现有的媒体片段中新建assets。可以设置相对volume，音频`track ramp`，设置透明度。

## Media捕捉和使用Camera

可以使用`preview player`来展示camera录制的内容。

## AV Foundation和多线程并发
有两点需要注意的：

* UI相关的通知应该在主线程中触发。
* 需要自己明确指定类和方法所在线程的话会在响应的线程中年触发通知

如果要开发多线程的应用，那么你应该使用`isMainThread`或者`[[NSThread currentThread] isEqual:<#A stored thread reference#>]`来提前测试是否该线程是你期望的要执行的线程。如果要跨线程，你可能需要如下方法

```objc
performSelectorOnMainThread:withObject:waitUntilDone: 
performSelector:onThread:withObject:waitUntilDone:modes:
dispatch_async 
```

为了更好的开发AVFoundation，你需要具备如下前提知识：

* 全面理解基础的Cocoa开发工具及技术
* 基本理解Block
* 基本理解KVC和KVO
* 对于视频回播，需要理解合理动画，对于基本的回播，需要理解AV Kit框架参考手册

