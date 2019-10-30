---
title: AVFoundation--使用Asset
date: 2017-08-04 23:38:03
tags: Objective-C
---

Asset可能来自于文件，或者用户的library或者Photo。创建完Asset对象之后，你所需要的所有信息不回立马就可用。一旦你有一个movie asset，你可以从它里面抽取静态图片，转码成其它格式，或者将其内容剪切。

## 新建一个Asset对象

使用一个URL，就可创建出一个Asset，使用AVURLAsset。最简单的就是从一个文件中创建一个Asset：

```objc  
NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>;
AVURLAsset *anAsset = [[AVURLAsset alloc] initWithURL:url options:nil];
```

最后的`options`参数是一个字典。唯一的key是`AVURLAssetPreferPreciseDurationAndTimingKey`它相应的Value是一个布尔值(包含在NSValue对象里)，它表明了是否这个Asset应该用来指明一个精确的持续时间提供一个精确地随机获取(provide precise random access)。
但是获取精确的持续时间需要提前处理很多东西。使用粗略的持续时间通常是更快捷的操作，并且这对回播来说已经足够了。因此

* 如果你只需要播放这个Asset，那么要么传nil，要么传一个子字典，其key为`AVURLAssetPreferPreciseDurationAndTimingKey`，它相应的值为NO（NSValue对象）。
* 如果你要将该Asset进行视频合成(`AVMutableComposition`)，那么通常你需要一个精确的`random access`。将这个字典的Value值传YES(用NSValue对象)。

创建方式如下所示：

```objc  
   NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>; 
   NSDictionary *options = @{ AVURLAssetPreferPreciseDurationAndTimingKey :
      @YES };
   AVURLAsset *anAssetToUseInAComposition = [[AVURLAsset alloc]
     initWithURL:url options:options];
```

## 获取用户的Asset

为了获取iPod library或者相册，你首先需要得到这个Asset的URL。

* 为了获取iPod Library，你需要创建一个MPMediaQuery对象来找到你需要的item，然后用`MPMediaItemPropertyAssetURL`来获取其URL。
* 为了获取相册的Asset，你需要使用`ALAssetsLibrary`。

下面这个例子说明了怎样获取展示在相册中的第一个视频：

```objc
   ALAssetsLibrary *library = [[ALAssetsLibrary alloc]init];
    // Enumerate just the photos and videos group by using ALAssetsGroupSavedPhotos.
    [library enumerateGroupsWithTypes:ALAssetsGroupSavedPhotos usingBlock:^(ALAssetsGroup *group, BOOL *stop) {
        
        // Within the group enumeration block, filter to enumerate just videos.
        [group setAssetsFilter:[ALAssetsFilter allVideos]];
        [group enumerateAssetsAtIndexes:[NSIndexSet indexSetWithIndex:0] options:0 usingBlock:^(ALAsset *result, NSUInteger index, BOOL *stop) {
            
            if (result) {
                
                ALAssetRepresentation *represention = [result defaultRepresentation];
                NSURL *url = [represention url];
                // Do something interesting with the AV asset.
                AVAsset *avAsset = [AVURLAsset URLAssetWithURL:url options:nil];
                
                
            }
            
        }];
        
    } failureBlock:^(NSError *error) {
        
        // Typically you should handle an error more gracefully than this.
        NSLog(@"No groups");
        
    }];
```

这里需要引入`#import <AssetsLibrary/AssetsLibrary.h>`。

## 准备Asset来用

初始化一个Asset(或者Track)并不意味着你立即就可以使用所有的信息。它还需要时间来计算这个Asset的时长(比如：一个MP3文件可能会包含一些总结信息)。你可以遵守`AVAsynchronousKeyValueLoading`协议来异步获取其结果，这样可以不用阻塞主线程（AVSeet和AVAssetTrack都遵守`AVAsynchronousKeyValueLoading`协议）。可以使用`statusOfValueForKey:error:`来测试是否某个某个属性的值被正常加载了。首次加载的时候，它所有属性的值是`AVKeyValueStatusUnknown`。使用`loadValuesAsynchronouslyForKeys:completionHandler:`，可以加载一个或者多个属性的值。由于网络原因或者加载被取消，所以需要一直准备加载。

```objc
             ALAssetRepresentation *represention = [result defaultRepresentation];
                NSURL *url = [represention url];
                // Do something interesting with the AV asset.
                AVAsset *avAsset = [AVURLAsset URLAssetWithURL:url options:nil];
                NSArray *keys = @[@"duration"];
                [avAsset loadValuesAsynchronouslyForKeys:keys completionHandler:^{
                   
                    NSError *error = nil;
                    AVKeyValueStatus tracksStatus = [avAsset statusOfValueForKey:@"duration" error:&error];
                    
                    switch (tracksStatus) {
                        case AVKeyValueStatusLoaded:
                            
                            [self updateUserInterfaceForDuration];
                            break;
                        case AVKeyValueStatusFailed:
                            
                            [self reportError:error forAsset:asset];
                            break;
                        case AVKeyValueStatusCancelled:
                            // Do whatever is appropriate for cancelation.
                            break;
                    }
                    
                }];
```

如果想准备一个asset来回播，那么还需要加载它的`tracks`属性。

## 从视频中获取静态图片

如果想要从asset中得到了一静态的图片，例如缩略图，你需要使用`AVAssetImageGenerator`对象。使用asset初始化一个generator。尽管在初始化的时候asset不会处理不可视的track，但是初始化还是可能会失败的，这就需要在必要的时候检查是否asset有track中包含有可视化的，使用`tracksWithMediaCharacteristic`方法：

```objc
AVAsset anAsset = <#Get an asset#>;
if ([[anAsset tracksWithMediaType:AVMediaTypeVideo] count] > 0) {
    AVAssetImageGenerator *imageGenerator =
        [AVAssetImageGenerator assetImageGeneratorWithAsset:anAsset];
    // Implementation continues...
}
```

可以对图片生成器的不同方面做不同的配置，例如可以使用maximumSize来指明其最大的尺度，使用`apertureMode`来指明孔径的模式。然后你就可以在给定的时间点来产生一张或者多张图片。`必须要强引用这个图片生成器否则，其会被释放掉。`

## 产生单独的一张图片

使用`copyCGImageAtTime:actualTime:error:`来产生在特定时间点的图片。AVFoundation框架可能不会在你请求的时候立即产生出一张图片，因此你可以给第二个参数传递一个指针，指向一个CMTime，该指针包含了图片实际产生的时间。

```objc
AVAsset *myAsset = <#An asset#>];
AVAssetImageGenerator *imageGenerator = [[AVAssetImageGenerator alloc]
initWithAsset:myAsset];
Float64 durationSeconds = CMTimeGetSeconds([myAsset duration]);
CMTime midpoint = CMTimeMakeWithSeconds(durationSeconds/2.0, 600);
NSError *error;
CMTime actualTime;
CGImageRef halfWayImage = [imageGenerator copyCGImageAtTime:midpoint
actualTime:&actualTime error:&error];
if (halfWayImage != NULL) {
NSString *actualTimeString = (NSString *)CMTimeCopyDescription(NULL, actualTime); 
    NSString *requestedTimeString = (NSString *)CMTimeCopyDescription(NULL,
midpoint);
    NSLog(@"Got halfWayImage: Asked for %@, got %@", requestedTimeString,
actualTimeString);
    // Do something interesting with the image.
    CGImageRelease(halfWayImage);
}
```

### 产生一系列的图片

为了产生一系列的图片，你应该调用图片生成器的`generateCGImagesAsynchronouslyForTimes:completionHandler:`方法，第一个参数是一个包含NSValue对象的数组，NSValue对象是由`CMTime`生成的，它指明了你需要生成的图片所在的时间点。第二个参数是每个图片生成之后的回调。其中包含的字段包括

* 图片
* 请求图片的时间和图片生成的真实时间
* 失败时候错误信息的error对象

在实现block的过程中，需要检测结果来决定图片是否生成了。除此之外：**在生成图片之前要一直强引用这个图片生成器**。
    
```objc   
AVAsset *myAsset = nil;
self.imageGenerator = [AVAssetImageGenerator assetImageGeneratorWithAsset:myAsset]; 
Float64 durationSeconds = CMTimeGetSeconds([myAsset duration]);
CMTime firstThird = CMTimeMakeWithSeconds(durationSeconds/3.0, 600);
CMTime secondThird = CMTimeMakeWithSeconds(durationSeconds*2.0/3.0, 600);
CMTime end = CMTimeMakeWithSeconds(durationSeconds, 600);
NSArray *times = @[ NSValue valueWithCMTime:kCMTimeZero],
                    [NSValue valueWithCMTime:firstThird], 
                    [NSValue valueWithCMTime:secondThird],
                    [NSValue valueWithCMTime:end]];
[imageGenerator generateCGImagesAsynchronouslyForTimes:times completionHandler:^(CMTime requestedTime, CGImageRef image, CMTime actualTime,
AVAssetImageGeneratorResult result, NSError *error) {
NSString *requestedTimeString = (NSString *) CFBridgingRelease(CMTimeCopyDescription(NULL, requestedTime));
NSString *actualTimeString = (NSString *)
                      CFBridgingRelease(CMTimeCopyDescription(NULL, actualTime));
 NSLog(@"Requested: %@; actual %@", requestedTimeString, actualTimeString);
  
if (result == AVAssetImageGeneratorSucceeded) {
    // Do something interesting with the image.
} 
if (result == AVAssetImageGeneratorFailed) { NSLog(@"Failed with error: %@", [error localizedDescription]); 
}
if (result == AVAssetImageGeneratorCancelled) {
    NSLog(@"Canceled");
}
}];
``` 

如果想要在中途终止图片序列的生成，可以调用`cancelAllCGImageGeneration`方法。

### 视频的裁剪和转码

使用`AVAssetExportSession`对象，可以将视频从一种格式到另一种格式进行转码，并且可以裁剪视频。一个export session是一个控制器对象，它管理着一个asset的异步输出。你可以将想要输出的asset以及输出的用来表明输出选项的`preset`来初始化一个`session`。然后你可以配置这个输出的session来指明输入的URL，以及文件格式，以及其它设置的一些可选配置，如metadata，以及是否输出需要被优化。

![视频转码](http://upload-images.jianshu.io/upload_images/1513759-7454ffe396927bff.png)
可以使用`exportPresetsCompatibleWithAsset:`来检测你是否可以输出一种给定的`asset`，如下所示

```objc           
AVAsset *anAsset = <#Get an asset#>;
NSArray *compatiblePresets = [AVAssetExportSession
exportPresetsCompatibleWithAsset:anAsset];
if ([compatiblePresets containsObject:AVAssetExportPresetLowQuality]) {
    AVAssetExportSession *exportSession = [[AVAssetExportSession alloc]
        initWithAsset:anAsset presetName:AVAssetExportPresetLowQuality];
    // Implementation continues.
}
```       

你可以通过提供输出的URL(该URL必须是一个文件的URL)来配置这个session，`AVAssetExportSession`可以根据输出的`URL`的路径扩展，来推断需要输出的文件类型；然而也可以通过使用`outputFileType`直接进行配置。你也可以指明一些其它的属性，比如时间范围，输出长度的限制，是否输出的文件需要被优化来作为网络使用，以及一个视频的合成。下面的例子说明了如何使用`timeRange`属性来裁剪视频：

```objc         
exportSession.outputURL = <#A file URL#>;
exportSession.outputFileType = AVFileTypeQuickTimeMovie;
CMTime start = CMTimeMakeWithSeconds(1.0, 600);
CMTime duration = CMTimeMakeWithSeconds(3.0, 600);
CMTimeRange range = CMTimeRangeMake(start, duration);
exportSession.timeRange = range;
```     

调用`exportAsynchronouslyWithCompletionHandler:`方法可以用来创建一个新的文件。当输出操作结束的时候这个文成的block将会被回调。如果你要处理这个回调，那么你需要检测这个seesion的`status`值来确定是否这次输出是成功的，失败的，或者是被取消的。

```objc         
[exportSession exportAsynchronouslyWithCompletionHandler:^{
          switch ([exportSession status]) {
              case AVAssetExportSessionStatusFailed:
                  NSLog(@"Export failed: %@", [[exportSession error]
  localizedDescription]);
                  break;
              case AVAssetExportSessionStatusCancelled:
                  NSLog(@"Export canceled");
                  break;
              default:
break; } 
}];
```   

通过调用session的`cancelExport`方法，可以取消某次输出。
如果你试图往一个已经存在的文件中重复写入，或者往应用的沙盒之外的其它地方写文件，那么将会造成输出文件失败。下面的情况也会造成输出失败：

* 电话呼入
* 你的应用推到了后台，同时另一个应用也开始执行回播

这种情况下，你需要提醒用户输出失败，以便其可以重新输出。




