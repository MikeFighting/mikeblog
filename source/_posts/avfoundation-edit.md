---
title: AVFoundation--视频编辑
date: 2017-08-24 10:26:09
tags: Objective-C
---

AV Foundation框架提供了很多类来方便对视频和音频的资源处理。AV Foundation框架的核心API是合成。一个合成仅仅是对一种或多种不同媒体资源track的组合。`AVMutableComposition`类提供了一个插入，移除以及管理track临时顺序的接口。你可以使用一个mutable composition来将一个新的asset和已经存在的asset集合组合在一起。如果你仅仅需要将很多的asset按序组装到一起，这些就够了。如果你想在合成的过程中执行任何定制的视频，音频处理，你需要分别加入音频混合（audio mix）和视频合成。

![Mutable Composition](http://upload-images.jianshu.io/upload_images/1513759-78492783e7c8d858.png)

使用`AVMutableAudioMix`类，你可以在合成的过程中定制音频。同时，你可以指定一个音轨（audio track）的最大音量，或者设定一个音量坡度（volume ramp）。

![AudioMix](http://upload-images.jianshu.io/upload_images/1513759-c1cb33719c5023b8.png)

对于视频合成中的video track如果要编辑的话，你可以使用`AVMutableVideoComposition`类来进行处理。对于单一的视频合成来说，你可以对输出的视频文件指明渲染的大小，比例，以及时长。通过一个视频的合成指令（`AVMutableVideoCompositionInstruction`类提供），你可以改变视频的背景色并且使用layer指令。在合成过程中，你可以使用这些layer指令(`AVMutableVideoCompositionLayerInstruction`类)来对video track实现旋转，旋转坡度（transform ramps），透明度，透明度坡度（opacity ramp）。视频合成类也可以让你使用核心动画来产生响应的效果（使用`animationTool`属性）。

![Video Composition](http://upload-images.jianshu.io/upload_images/1513759-615e5cde04c0c133.png)

使用`AVAssetExportSession`，你可以将音频混合（audio mix）和视频合成同时组合到你的合成中去。用你的composition初始化这个export session，然后将音频混合和视频混合分别赋值给`audioMix`和`videoComposition`属性即可。

![Audio Mix And Video Composition](http://upload-images.jianshu.io/upload_images/1513759-eea273fda859f11e.png)

## 创建一个Composition

使用`AVMutableComposition`类来创建你自己的composition。为了添加媒体数据到你的composition，你必须通过`AVMutableCompositionTrack`类来添加一个或者多个composition tracks。最简单的情况就是用video track和audio track来创建一个mutable composition。

```objc 
AVMutableComposition *mutableComposition = [AVMutableComposition composition];
// Create the video composition track.
AVMutableCompositionTrack *mutableCompositionVideoTrack = [mutableComposition
addMutableTrackWithMediaType:AVMediaTypeVideo
preferredTrackID:kCMPersistentTrackID_Invalid];
// Create the audio composition track.
AVMutableCompositionTrack *mutableCompositionAudioTrack = [mutableComposition
addMutableTrackWithMediaType:AVMediaTypeAudio
preferredTrackID:kCMPersistentTrackID_Invalid];
```   

### 初始化一个Composition Track时的选项

当你要往一个composition中添加一个新的track的时候，你必须提供一个media type和一个track ID。尽管视频和音频是最常用的media type，你还可以设置其它的media type，比如`AVMediaTypeSubtitle`（字幕）`AVMediaTypeText`（文本）。

每一个和试听数据相关的track都有一个唯一标示叫做 track ID。如果你将这个track ID赋值为`kCMPersistentTrackID_Invalid`，那么在关联的时候系统会自动给你创建一个唯一标识。

## 给一个Composition添加音视频数据

一旦你使用一个或者多个track创建一个composition，那么你就可以开始将你的媒体数据media数据添加到合适的tracks上。为了将媒体数据添加到一个composition track，你需要使用AVAsset对象（媒体数据存储在这个对象内部）。你可以在某个track上使用mutable composition track接口来将多个具有相同隐含媒体类型的`track`放到一起。接下来的例子说明了怎样将两种video asset track按序放到某种composition track中：

```objc

// You can retrieve AVAssets from a number of places, like the camera roll for
example.
AVAsset *videoAsset = <#AVAsset with at least one video track#>;
AVAsset *anotherVideoAsset = <#another AVAsset with at least one video track#>;

// Get the first video track from each asset.
AVAssetTrack *videoAssetTrack = [[videoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0]; 
AVAssetTrack *anotherVideoAssetTrack = [[anotherVideoAsset
tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];

// Add them both to the composition.
[mutableCompositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,videoAssetTrack.timeRange.duration)
ofTrack:videoAssetTrack atTime:kCMTimeZero error:nil];

[mutableCompositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,anotherVideoAssetTrack.timeRange.duration) ofTrack:anotherVideoAssetTrack atTime:videoAssetTrack.timeRange.duration error:nil]; 

```   

### 检索兼容的Composition Track

如果可能的话，你应该对每一种媒体类型仅有一个composition track。可兼容asset track的统一可以将资源的利用降低到最少。当序列地展示媒体数据的时候，你应该将任何相同类型的媒体数据都放到同一个compsition track中。你可以查询一个mutable composition来找出和你需要的asset track相兼容的任何composition track：

```objc
AVMutableCompositionTrack *compatibleCompositionTrack = [mutableComposition
mutableTrackCompatibleWithTrack:<#the AVAssetTrack you want to insert#>];
if (compatibleCompositionTrack) {
    // Implementation continues.
} 
```   

*注：将多个视频片段放到同一个composition track，在过个媒体片段转换时可能会隐性的造成掉帧现象，这种情况在嵌入式设备中尤其明显。对你的视频片段选择合适的合成track完全取决于你App的设计以及其将要展示的平台*

###  生成音量坡度（Volume Ramp）

在你的composition中，一个简单的`AVMutableAudioMix`对象可以单独对所有的音频track进行常用的音频处理。在你合成的过程中，使用`audioMix`类方法来创建一个audio mix，然后使用`AVMutableAudioMixInputParameters`实例来将这个audio mix和特定的track相关联。接下来的例子将会说明说明如何在一个指定的`audio track`上设置一个音量坡度来让音量在合成的过程中缓慢的消失。

```objc
AVMutableAudioMix *mutableAudioMix = [AVMutableAudioMix audioMix];
// Create the audio mix input parameters object.
AVMutableAudioMixInputParameters *mixParameters = [AVMutableAudioMixInputParameters audioMixInputParametersWithTrack:mutableCompositionAudioTrack]; 
// Set the volume ramp to slowly fade the audio out over the duration of the
composition.
[mixParameters setVolumeRampFromStartVolume:1.f toEndVolume:0.f
timeRange:CMTimeRangeMake(kCMTimeZero, mutableComposition.duration)];
// Attach the input parameters to the audio mix.
mutableAudioMix.inputParameters = @[mixParameters];
```    

### 运用透明度坡度（Opacity Ramp）

视频合成指令也可以用于视频和成Layer指令。一个`AVMutableVideoCompsitionLayerInstruction`对象可以对某个composition内部的某个video track使用旋转，旋转坡度，透明度，透明度坡度。某个视频合成指令的`layerInstructions`数组中layer指令的顺序决定了合成指令的过程中，来自source track的视频帧怎样被分层堆放，怎样被合成。在接下来的代码片段说明了怎样设置透明度坡度来在第一个视频结束切换第二个视频的时候产生缓慢消失的效果。

```objc
AVAsset *firstVideoAssetTrack = <#AVAssetTrack representing the first video segment played in the composition#>; 
AVAsset *secondVideoAssetTrack = <#AVAssetTrack representing the second video
  segment played in the composition#>;
  
// Create the first video composition instruction.
AVMutableVideoCompositionInstruction *firstVideoCompositionInstruction =
[AVMutableVideoCompositionInstruction videoCompositionInstruction];

// Set its time range to span the duration of the first video track.
  firstVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero,
  firstVideoAssetTrack.timeRange.duration);
  
// Create the layer instruction and associate it with the composition video track. 
  AVMutableVideoCompositionLayerInstruction *firstVideoLayerInstruction =
  [AVMutableVideoCompositionLayerInstruction
  videoCompositionLayerInstructionWithAssetTrack:mutableCompositionVideoTrack];
  
// Create the opacity ramp to fade out the first video track over its entire duration.
 [firstVideoLayerInstruction setOpacityRampFromStartOpacity:1.f toEndOpacity:0.f
  timeRange:CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration)];
  
// Create the second video composition instruction so that the second video track isn't transparent.
 AVMutableVideoCompositionInstruction *secondVideoCompositionInstruction =
 [AVMutableVideoCompositionInstruction videoCompositionInstruction];
  
// Set its time range to span the duration of the second video track.
secondVideoCompositionInstruction.timeRange =
  CMTimeRangeMake(firstVideoAssetTrack.timeRange.duration,
  CMTimeAdd(firstVideoAssetTrack.timeRange.duration,
  secondVideoAssetTrack.timeRange.duration));
  
// Create the second layer instruction and associate it with the composition video track.
 AVMutableVideoCompositionLayerInstruction *secondVideoLayerInstruction =
  [AVMutableVideoCompositionLayerInstruction
  videoCompositionLayerInstructionWithAssetTrack:mutableCompositionVideoTrack];
  
// Attach the first layer instruction to the first video composition instruction.
firstVideoCompositionInstruction.layerInstructions = @[firstVideoLayerInstruction]; 

// Attach the second layer instruction to the second video composition instruction. 
secondVideoCompositionInstruction.layerInstructions = @[secondVideoLayerInstruction]; 

// Attach both of the video composition instructions to the video composition.
  mutableVideoComposition.instructions = @[firstVideoCompositionInstruction,
  secondVideoCompositionInstruction];
```   

### 加入核心动画效果

通过使用`animationTool`属性，你可以在合成视频的时候添加核心动画效果。通过这个animation tool，你可以完成给视频添加水印，添加标题或者动画覆盖。在视频合成过程中，核心动画有两种不同的使用方式：你可以在它自己composition track的视频帧中添加一个Core Animation layer,或者你可以直接使用（Core Animation layer）来渲染核心动画效果。接下来的代码展示了怎样使用第二种方式在视频的中间来添加一个水印。

```objc 
CALayer *watermarkLayer = <#CALayer representing your desired watermark image#>;
CALayer *parentLayer = [CALayer layer];
CALayer *videoLayer = [CALayer layer];
parentLayer.frame = CGRectMake(0, 0, mutableVideoComposition.renderSize.width,
mutableVideoComposition.renderSize.height);
videoLayer.frame = CGRectMake(0, 0, mutableVideoComposition.renderSize.width,
mutableVideoComposition.renderSize.height);
[parentLayer addSublayer:videoLayer];
watermarkLayer.position = CGPointMake(mutableVideoComposition.renderSize.width/2,
 mutableVideoComposition.renderSize.height/4);
[parentLayer addSublayer:watermarkLayer];
mutableVideoComposition.animationTool = [AVVideoCompositionCoreAnimationTool
videoCompositionCoreAnimationToolWithPostProcessingAsVideoLayer:videoLayer
inLayer:parentLayer];
```   

## 综合：综合各种Assets并且保存到Camera Roll中

接下来的代码例子说明了如何将两个视频asset track和一个音频asset track综合来创建一个简单的video file。它的主要功能有

* 创建一个`AVMutableComposition`对象并且添加很多的`AVMutableCompositionTrack`对象
* 给可兼容的composition track通过`AVAssetTrack`对象添加时间范围
* 检查一个video asset的`preferredTransform`属性来确定视频的方向
* 在一个合成过程中使用`AVMutableVideoCompositionLayerInstruction`对象来对一个视频track实现旋转效果
* 给一个video composition对象设置合适的`renderSize`和`frameDuration`属性值。
* 在导出视频文件的时候使用合成结合另一个视频合成
* 将视频文件导出到`Camera Roll`文件中

*注：为了展示最关键的代码，该实例略去了一个完整的应用所需要的功能点，比如：内存管理，注销观察者（对KVO的观察或者对某个通知的监听）。为了能够很好的使用AV Foundation，你需要对Cocoa有丰富的经验，以便处理可能遗漏的功能点。*

### 创建Composition 

为了将几个不同的assets创建放到一起，你可以使用一个`AVMutableComposition`对象。创建这个composition并且将一个音频和视频track添加进去。

```objc
AVMutableComposition *mutableComposition = [AVMutableComposition composition];
AVMutableCompositionTrack *videoCompositionTrack = [mutableComposition
addMutableTrackWithMediaType:AVMediaTypeVideo
preferredTrackID:kCMPersistentTrackID_Invalid];
AVMutableCompositionTrack *audioCompositionTrack = [mutableComposition
addMutableTrackWithMediaType:AVMediaTypeAudio
preferredTrackID:kCMPersistentTrackID_Invalid];
```  

### 添加Assets

一个空的composition没有什么作用。添加两个video asset track和audio asset track到这个composition中。

```objc
AVAssetTrack *firstVideoAssetTrack = [[firstVideoAsset
tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVAssetTrack *secondVideoAssetTrack = [[secondVideoAsset
tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
[videoCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,
firstVideoAssetTrack.timeRange.duration) ofTrack:firstVideoAssetTrack
atTime:kCMTimeZero error:nil];
[videoCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,
secondVideoAssetTrack.timeRange.duration) ofTrack:secondVideoAssetTrack
atTime:firstVideoAssetTrack.timeRange.duration error:nil];
[audioCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration)) ofTrack:[[audioAsset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0] atTime:kCMTimeZero error:nil]; 
```  

*注：假设你有两个assets，它们每一个至少包含了一个video track，第三个asset中至少包含了一个audio track。这个视频可以从Camera Roll中获取，audio track可以从音乐库或者视频自己的audio tack中获取。*

### 检测视频方向

一旦你将音频和视频track添加到这个composition中，你需要确保两个video track的方向正确的。默认情况下，所有的视频track假定都是水平模式的。如果你的video track在竖直模式拍摄，那么当它们输出的时候是不能被正确输出的。同样的，如果你试图将一个竖直模式的视频和一个水平模式的视频合成一个视频，那么这个输出的session就会失败。

```objc
BOOL isFirstVideoPortrait = NO;
CGAffineTransform firstTransform = firstVideoAssetTrack.preferredTransform;

// Check the first video track's preferred transform to determine if it was recorded in portrait mode. 
if (firstTransform.a == 0 && firstTransform.d == 0 && (firstTransform.b == 1.0 ||
 firstTransform.b == -1.0) && (firstTransform.c == 1.0 || firstTransform.c ==
-1.0)) {
    isFirstVideoPortrait = YES;
} 
BOOL isSecondVideoPortrait = NO;
CGAffineTransform secondTransform = secondVideoAssetTrack.preferredTransform;

// Check the second video track's preferred transform to determine if it was
recorded in portrait mode.
if (secondTransform.a == 0 && secondTransform.d == 0 && (secondTransform.b == 1.0
 || secondTransform.b == -1.0) && (secondTransform.c == 1.0 || secondTransform.c
== -1.0)) {
    isSecondVideoPortrait = YES;
} 
if ((isFirstVideoAssetPortrait && !isSecondVideoAssetPortrait) ||
(!isFirstVideoAssetPortrait && isSecondVideoAssetPortrait)) {
    UIAlertView *incompatibleVideoOrientationAlert = [[UIAlertView alloc]
initWithTitle:@"Error!" message:@"Cannot combine a video shot in portrait mode
with a video shot in landscape mode." delegate:self cancelButtonTitle:@"Dismiss"
otherButtonTitles:nil];
    [incompatibleVideoOrientationAlert show];
return; 
} 
```   

### 使用视频合成层指令

一旦你知道视频片段有可兼容的视频方向，那么你就可以对每个视频段使用必要的图层指令来，并且将这些图层指令添加到视频合成中。

```objc    
BOOL isFirstVideoPortrait = NO;
CGAffineTransform firstTransform = firstVideoAssetTrack.preferredTransform;
// Check the first video track's preferred transform to determine if it was recorded in portrait mode. 
if (firstTransform.a == 0 && firstTransform.d == 0 && (firstTransform.b == 1.0 ||
 firstTransform.b == -1.0) && (firstTransform.c == 1.0 || firstTransform.c ==
-1.0)) {
    isFirstVideoPortrait = YES;
} 
BOOL isSecondVideoPortrait = NO;
CGAffineTransform secondTransform = secondVideoAssetTrack.preferredTransform;
// Check the second video track's preferred transform to determine if it was
recorded in portrait mode.
if (secondTransform.a == 0 && secondTransform.d == 0 && (secondTransform.b == 1.0
 || secondTransform.b == -1.0) && (secondTransform.c == 1.0 || secondTransform.c
== -1.0)) {
    isSecondVideoPortrait = YES;
} 
if ((isFirstVideoAssetPortrait && !isSecondVideoAssetPortrait) ||
(!isFirstVideoAssetPortrait && isSecondVideoAssetPortrait)) {
    UIAlertView *incompatibleVideoOrientationAlert = [[UIAlertView alloc]
initWithTitle:@"Error!" message:@"Cannot combine a video shot in portrait mode
with a video shot in landscape mode." delegate:self cancelButtonTitle:@"Dismiss"
otherButtonTitles:nil];
    [incompatibleVideoOrientationAlert show];
return; 
}  
```

所有的`AVAssetTrack`对象都有一个`preferredTransform`属性，该属性中包含了这个asset track对象的方向信息。只要这个asset track在屏幕上展示，那么这个`transform`都将会被使用。在上面的代码中，这个`layer instruction transform`被设置成了这个`asset track`的transform，以便于这个视频在新的合成中一旦调整渲染尺寸，仍然可以正常展示。

### 设置渲染尺寸和帧时长

为了将视频的方向固定，你必须响应地调整`renderSize`属性。你也应该对`frameDuration`属性挑选一个合适的值。比如每秒钟30次（或者30帧每秒）。默认情况下，renderScale属性被设置为1.0，这在本次合成过程中是合适的。

```objc
  CGSize naturalSizeFirst, naturalSizeSecond;
  // If the first video asset was shot in portrait mode, then so was the second one
   if we made it here.
  if (isFirstVideoAssetPortrait) {
  // Invert the width and height for the video tracks to ensure that they display
  properly.
      naturalSizeFirst = CGSizeMake(firstVideoAssetTrack.naturalSize.height,
  firstVideoAssetTrack.naturalSize.width);
      naturalSizeSecond = CGSizeMake(secondVideoAssetTrack.naturalSize.height,
  secondVideoAssetTrack.naturalSize.width);
} else { 
  // If the videos weren't shot in portrait mode, we can just use their natural
  sizes.
      naturalSizeFirst = firstVideoAssetTrack.naturalSize;
      naturalSizeSecond = secondVideoAssetTrack.naturalSize;
  }
  float renderWidth, renderHeight;
  // Set the renderWidth and renderHeight to the max of the two videos widths and
  heights.
  if (naturalSizeFirst.width > naturalSizeSecond.width) {
      renderWidth = naturalSizeFirst.width;
} else { 
      renderWidth = naturalSizeSecond.width;
  }
  if (naturalSizeFirst.height > naturalSizeSecond.height) {
      renderHeight = naturalSizeFirst.height;
} else { 
      renderHeight = naturalSizeSecond.height;
  }

 mutableVideoComposition.renderSize = CGSizeMake(renderWidth, renderHeight);
  // Set the frame duration to an appropriate value (i.e. 30 frames per second for
  video).
 mutableVideoComposition.frameDuration = CMTimeMake(1,30);
```   


### 导出合成，并且将它保存到Camera Roll中

在该过程的最后一个步骤设计到将整个合成保存到一个单独的视频文件，并且将视频存储到Camera Roll中。你可以使用`AVAssetExportSession`对象来创建这个新的video文件，然后你将它传递到输出文件期望的`URL`中。然后使用`ALAssetLibrary`对象来保存这个结果视频文件到Camera Roll中。

```objc
// Create a static date formatter so we only have to initialize it once.
  static NSDateFormatter *kDateFormatter;
  if (!kDateFormatter) {
      kDateFormatter = [[NSDateFormatter alloc] init];
      kDateFormatter.dateStyle = NSDateFormatterMediumStyle;
      kDateFormatter.timeStyle = NSDateFormatterShortStyle;
} 

// Create the export session with the composition and set the preset to the highest quality. 
  AVAssetExportSession *exporter = [[AVAssetExportSession alloc]
  initWithAsset:mutableComposition presetName:AVAssetExportPresetHighestQuality];
  
// Set the desired output URL for the file created by the export process.
exporter.outputURL = [[[[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil 
create:@YES error:nil] URLByAppendingPathComponent:[kDateFormatter stringFromDate:[NSDate date]]] URLByAppendingPathExtension:CFBridgingRelease(UTTypeCopyPreferredTagWithClass((CFStringRef)AVFileTypeQuickTimeMovie, 
   kUTTagClassFilenameExtension))];
   
// Set the output file type to be a QuickTime movie.
  exporter.outputFileType = AVFileTypeQuickTimeMovie;
  exporter.shouldOptimizeForNetworkUse = YES;
  exporter.videoComposition = mutableVideoComposition;
  
// Asynchronously export the composition to a video file and save this file to the camera roll once export completes. 
 [exporter exportAsynchronouslyWithCompletionHandler:^{
      dispatch_async(dispatch_get_main_queue(), ^{
          if (exporter.status == AVAssetExportSessionStatusCompleted) {
 
 ALAssetsLibrary *assetsLibrary = [[ALAssetsLibrary alloc] init];
              if ([assetsLibrary
  videoAtPathIsCompatibleWithSavedPhotosAlbum:exporter.outputURL]) {
  
[assetsLibrary writeVideoAtPathToSavedPhotosAlbum:exporter.outputURL completionBlock:NULL]; 
         } 
      } 
   }); 
}]; 
``` 

