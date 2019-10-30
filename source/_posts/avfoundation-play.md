---
title: AVFoundation--视频播放
date: 2017-08-05 12:23:18
tags: Objective-C
---

使用`AVPlayer`对象可以用来控制asset的播放。在播放期间，你可以使用AVPlayerItem实例来asset的presentation state。并且一个`AVPlayerItemTrack`对象可以管理某个独立track的展示状态。展示一个视频，你可以使用`AVPlayerLayer`对象。

## 播放Assets

一个player是一个控制器对象，你可以使用它来管理一个asset的播放，比如，开始和停止播放以及寻找特殊的时间点。使用`AVPlayer`实例来播放一个asset。你可以使用`AVQueuePlayer`对象来按序播放一系列的item（AVQueuePlayer是AVPlayer的一个子类）。

一个player给你提供了播放状态的信息，如果有需要，你可以让你的UI和player的状态相同步。通常情况下，你可以直接指出player的输出到一个特定的Core Animation的layer上(`AVPlayerLayer`或者`AVSynchronizedLayer`)对象。

> 多个player layer:你可以对一个AVPlayer实例创建很多的AVPlayerLayer对象，但是只有最近创建layer才可以在屏幕上展示在视频内容。

你不用给AVPlayer对象直接提供assets，尽管你最终想要播放的是asset。相反，你需要提供一个AVPlayerItem的实例。以个item用来管理其相关联的asset的presentation state。一个item包含一个AVPlayerItemTrack的实例，这个实例和asset中的track相对应。结构如下：
![PlaeyItem关系图](http://upload-images.jianshu.io/upload_images/1513759-f0dabcb6010b69ad.png)
下面的图说明了你可以用不同的player同时播放一个指定的asset，但是每个player都可以用不同的方式进行渲染。例如，使用item track，你可以在播放期间让一个特定的track失效（比如，你可能不想播放一个音频部分）。
![AVPlayerItem](http://upload-images.jianshu.io/upload_images/1513759-2a0e8b3648efb92a.png)

你可以使用一个已经存在的asset来初始化一个player，或者你可以用一个URL来初始化一个player，以便于你可以再一个特定的点来播放这个资源(`AVPlayerItem`将会对这个资源的创建和配置这个asset)。和`AVAsset`一样，仅仅初始化一个player item并不意味着它可以直接用来播放。你可以使用KVO来观察这个item的`status`属性来决定播放的时机及播放的逻辑。

## 处理不同类型的Asset

你可以根据将要播放的不同的Asset类型来决定怎样配置asset。一般说来，有两种不同的类型：文件类型的assets，有几种可以选择，比如：本地文件，相机胶卷，或者媒体库；另外就是基于流的assets（HTTP直播流形式）。

**基于文件的视频加载**，为了播放基于文件的视频，有以下步骤：
* 创建一个`AVURLAsset`对象。
* 使用asset创建一个`AVPlayerItem`对象
* 将一个`AVPlayer`和这个item对象相关联
* 等待，一直到这个item的`status`属性指明可以播放了（利用KVO）


**基于HTTP直播视频流来播放**，利用该URL创建一个AVPlayerItem。（你不可以直接创建一个`AVAsset`对象来代表`HTTP Live Stream`的媒体）

```objc
	NSURL *url = [NSURL URLWithString:@"<#Live stream URL#>];
	// You may find a test stream at
	<http://devimages.apple.com/iphone/samples/bipbop/bipbopall.m3u8>.
	self.playerItem = [AVPlayerItem playerItemWithURL:url];
	[playerItem addObserver:self forKeyPath:@"status" options:0
	context:&ItemStatusContext];
	self.player = [AVPlayer playerWithPlayerItem:playerItem];
```   

当你将这个player item和以个player相结合的时候，它就变为待播放状态了。当它准备好播放的时候，这个player item创建`AVAsset`以及`AVAssetTrack`实例，你可以利用它来检测直播流的内容。想要得到这个item的播放时长，你可以观察其`duration`属性。当这个item状态变为可以播放时，这个属性就会更新到这个视频流的准确数值。
*注：当这个状态变为AVPlayerItemStatusReadyToPlay的时候，这个播放时长可以使用下面的代码来获取时长*

```objc 
    [[[[[playerItem tracks] objectAtIndex:0] assetTrack] asset] duration];
```  

如果你仅仅想要播放一个直播流，那么你可以走个捷径，直接使用这个`URL`创建一个player：

```objc
	self.player = [AVPlayer playerWithURL:<#Live stream URL#>];
	[player addObserver:self forKeyPath:@"status" options:0
	context:&PlayerStatusContext];
```  

和asset和item一样，初始化完player之后并不意味着你可以立即使用播放了。你需要观察其`status`属性，该属性变为`AVPlayerStatusReadyToPlay`的时候就表明它可以播放了。你也可以观察`currentItem`来获取已经创建的item。

**如果你不知道你要播放的URL的类型**，那么你需要这样做：

1. 尝试使用这个URL来初始化一个`AVURLAsset`对象，然后加载其`tracks`key。如果tracks加载成功，那么你可以为这个asset创建一个player item。
2. 如果1失败了，那么利用这个URL直接创建一个`AVPlayerItem`对象。观察这个player的`status`属性，来决定是否可以播放了。

只要其中一个成功，你就可以得到一个player item，然后将其和一个player对象相关联。

## 播放一个Item

为了开始播放，你需要调用player的`play`方法即可：

```objc
	- (IBAction)play:sender {
	      [player play];
	      
	   }
```   

除了紧紧播放以外，你可以管理播放过程中各种方面，比如，playhead的位置和速度。你也可以观察player的stata。比如，你如果想将UI和asset的presention state相同步，你就要这样做。

### 改变播放速度

你可以通过设定player的`rate`属性来改变其播放的速度。

```objc
	aPlayer.rate = 0.5;
	aPlayer.rate = 2.0;
```  

1.0的数值表示利用当前`item`的正常速度播放，0.0速度和暂停是一样的效果。
支持回播的player，可以使用一个负值来设置这这个播放速度。你可以使用`canPlayReverse`(是否支持数值-1.0的播放速度)属性来检测其是否可以支持回播，使用`canPlaySlowReverse`(支持0.0到1.0的播放速度)，以及`canPlayFastReverse`(支持小于-1.0的播放速度)

### 寻找-重置Playhead

为了将playhead移动到一个特定的时间点，你通常需要使用`seekToTime`：

```objc
	CMTime fiveSecondsIn = CMTimeMake(5,1);
	[player seekToTime: fiveSecondsIn];
```    

然而，这个`seekToTime:`方法不是很精确，尽管其性能较高。如果你要精确得移动这个`playhead`，你可以使用下面的`seekToTime:toleranceBefore:toleranceAfter:`方法。

```objc
	CMTime fiveSecondsIn = CMTimeMake(5,1);
	[player seekToTime: fiveSecondsIn toleranceBefore: kCMTimeZero toleranceAfter: kCMTimeZero];
```  

上面的例子中将`tolerance`设置为零需要框架解码大量的数据。因此，仅仅在必要的时候再使用零，比如：你需要写一个精确的媒体编辑应用，它需要精确的控制。

在视频播放之后，player的head被设定在了item的尾部，因此接下来调用`play`操作是不起作用的。为了将playhead放到item的起始位置，你需要注册一个item的`AVPlayerItemDidPlayToEndTimeNotification`通知，在该通知的回调方法中，你调用`seekToTime:`方法，并且传入`kCMTimeZero`参数。

```objc
	    // Register with the notification center after creating the player item.
	    [[NSNotificationCenter defaultCenter]
	        addObserver:self
	        selector:@selector(playerItemDidReachEnd:)
	        name:AVPlayerItemDidPlayToEndTimeNotification
	        object:<#The player item#>];
	        
	    - (void)playerItemDidReachEnd:(NSNotification *)notification {
	    [player seekToTime:kCMTimeZero];
	 } 
```  

## 很多Item的播放

你可以使用`AVQueuePlayer`对象来播放一系列的`item`。这个`AVQueuePlayer`类是`AVPlayer`类的子类。通过一个`item`的数组，你可以初始化一个`queue player`。

```objc
	NSArray *items = <#An array of player items#>;
	AVQueuePlayer *queuePlayer = [[AVQueuePlayer alloc] initWithItems:items];
```   

然后调用其`play`方法即可。这个player会按序播放这些`item`。如果不想播放某个`item`可以调用它的`advanceToNextItem`方法。
可以使用`insertTtem:afterItem:`，`removeItem:`，以及`removeAllItems`方法，如果要插入一个item，你需要首先调用`canInsertItem：afterItem:`方法来确定它是否可以插入到这个`queue`中。你可以传给第二个参数`nil`，来检测是否新的`item`可以被加到`queue`的后面。

```objc
	AVPlayerItem *anItem = <#Get a player item#>;
	if ([queuePlayer canInsertItem:anItem afterItem:nil]) {
	    [queuePlayer insertItem:anItem afterItem:nil];
	}
```   

## 监控视频播放

你可以监控正在player的显示状态以及其正在播放的item的各个方面。这对你所不能控制的状态改变来说是极其有益的，比如：

* 比如如果用户使用多任务操作来切换应用，那么一个player的`rate`属性就会掉到0.0;
* 如果你正在播放一个远程的媒体，一个player item的`loadedTimeRangs`和`seekableTimeRanges`属性就会在更多的数据变得可用的时候改变。这些属性告诉你这些player item的那些部分是可用的。
* 在HTTP直播流被创建的时候，player item的`tracks`属性就会改变。如果这个视频流对内容提供了不同的编码格式，那么这就会发生；在player切换不同的编码的时候，这个`tracks`就改变了。
* 如果一个视频播放失败，那么这个player或者player item的`status`属性可能会改变。

你可以使用KVO来监控这些属性值的改变。

>你应该将注册KVO以及取消注册KVO都放到主线程中。这样，如果另外一个线程发生了改变，这将会避免收到部分通知的可能。尽管这些属性的变化会在其它线程上，但是AV Foundation触发`observeValueForKeyPath:ofObject:change:context:`是在主线程上。

### 响应某个状态的改变

当一个player或者一个player item的状态改变的时候，它发出一个KVO的变化通知。如果一个对象由于某种原因不能够被播放（比如，媒体服务被重置），其状态将会变为`AVPlayerStatusFailed`或者`AVPlayerItemStatusFailed`。在这种情况下，这个对象的`error`属性将会变为一个error 对象，这个对象中包含了其不能够播放的原因描述。

AVFoundation不会指明这个通知发送的线程。如果你想改变UI，你必须要确保任何相关的代码会在主线程中执行。比如，你可以使用`dispatch_async`来在主线程中执行代码。

```objc
	- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
	                          change:(NSDictionary *)change context:(void *)context {
	      if (context == <#Player status context#>) {
	
	          AVPlayer *thePlayer = (AVPlayer *)object;
	          if ([thePlayer status] == AVPlayerStatusFailed) {
	              NSError *error = [<#The AVPlayer object#> error];
	              // Respond to error: for example, display an alert sheet.
	              return;
	} 
	          // Deal with other status change if appropriate.
	      }
	      // Deal with other change notifications if appropriate.
	      [super observeValueForKeyPath:keyPath ofObject:object
	             change:change context:context];
	      return;
	} 
```   

### 追踪可视化播放的准备状态

你可以观察`AVPlayerLayer`对象的`readyForDisplay`属性来获取当`layer`的用户可视内容改变时发出的通知。尤其重要的是，只有在用户可以看到一些东西的时候，你才可以将一个player layer插入到layer tree中，然后执行转化操作。

### 追踪时间

为了追踪一个`AVPlayer`对象中`playhead`的位置，你可以使用addPeriodicTimeObserverForInterval: queue: usingBlock:或者addBoundaryTimeObserverForTimes: queue: usingBlock:比如，你可以用过去的时间以及剩余的时间来更新用于界面，或者执行其它的UI同步操作。

* 如果时间超过了你所指定的周期点，以及视频播放开始或者停止，这个时候`addPeriodicTimeObserverForInterval:queue:usingBlock:`将会被触发。
* 你也可以指定在某些时间触发`addBoundaryTimeObserverForTimes:queue:usingBlock:`，这里你需要传递一个包含被`NSValue`封装的`CMTime`数组。

如果你要想这个基于时间的`observation block`被触发，那么你必须要对这两个方法返回的对象做强引用。同时你必须在每次触发这些方法时调`removeTimeObserver:`。使用这些方法，AV Foundation不会保证在每次的时间间隔或者时间范围达到的时候都触发这些操作。如果之前的block没有执行完毕，那么AV Foundation不会执行接下来的block。因此，你必须保证在block中执行的任务不能消耗太多的CPU资源。

```objc
	/ Assume a property: @property (strong) id playerObserver;
	
	Float64 durationSeconds = CMTimeGetSeconds([<#An asset#> duration]);
	CMTime firstThird = CMTimeMakeWithSeconds(durationSeconds/3.0, 1);
	CMTime secondThird = CMTimeMakeWithSeconds(durationSeconds*2.0/3.0, 1);
	NSArray *times = @[[NSValue valueWithCMTime:firstThird], [NSValue
	valueWithCMTime:secondThird]];
	
	self.playerObserver = [<#A player#> addBoundaryTimeObserverForTimes:times queue:NULL usingBlock:^{ 
	NSString *timeDescription = (NSString *) CFBridgingRelease(CMTimeCopyDescription(NULL, [self.player currentTime])); 
	    NSLog(@"Passed a boundary at %@", timeDescription);
	}];
```  

### 某个Item结束了

当一个Item播放结束的时候，你可以收到一个`AVPlayerItemDidPlayToEndTimeNotification`通知。你可以注册这个通知。

```objc
	[[NSNotificationCenter defaultCenter] addObserver:<#The observer, typically self#> selector:@selector(<#The selector name#>) 
	name: AVPlayerItemDidPlayToEndTimeNotification  object:<#A player item#>];
```  

## 综合：使用AVPlayerLayer播放一个视频文件

接下来会用简单的代码实例来演示怎样使用`AVPlyer`对象来播放一个视频文件。它包含以下几部分内容：

* 使用`AVPlayerLayer`来配置View
* 创建一个`AVPlayer`对象
* 基于视频文件创建一个`AVPlayerItem`，并且使用KVO来观察其状态
* 通过使能按钮来让该Item准备播放
* 播放该`Item`并且将这个播放完的Item的head重置到开始

*注：为了展示最关键的代码，该实例略去了一个完整的应用所需要的功能点，比如：内存管理，注销观察者（对KVO的观察或者对某个通知的监听）。为了能够很好的使用AV Foundation，你需要对Cocoa有丰富的经验，以便处理可能遗漏的功能点。*

### Player View

为了播放一个Asset的可视部分，你需要一个包含了AVPlayerLayer的View以便这个AVPlayer对象的输出可以被获取。创建一个UIView的子类就可以完成这些内容：

```objc   
#import <UIKit/UIKit.h>
#import <AVFoundation/AVFoundation.h>
@interface PlayerView : UIView
@property (nonatomic) AVPlayer *player;
@end
@implementation PlayerView
+ (Class)layerClass {
    return [AVPlayerLayer class];
}
- (AVPlayer*)player {
    return [(AVPlayerLayer *)[self layer] player];
}
- (void)setPlayer:(AVPlayer *)player {
    [(AVPlayerLayer *)[self layer] setPlayer:player];
}
@end 
```    

### 一个简单的ViewController

假如你有一个类似下面的ViewController：

```objc
@class PlayerView;
@interface PlayerViewController : UIViewController

@property (nonatomic) AVPlayer *player;
@property (nonatomic) AVPlayerItem *playerItem;
@property (nonatomic, weak) IBOutlet PlayerView *playerView;
@property (nonatomic, weak) IBOutlet UIButton *playButton;
- (IBAction)loadAssetFromFile:sender;
- (IBAction)play:sender;
- (void)syncUI;
@end
```
 
这个`syncUI`的方法可以将button和player的状态相同步。

```objc
 - (void)syncUI {
    if ((self.player.currentItem != nil) &&
        ([self.player.currentItem status] == AVPlayerItemStatusReadyToPlay)) {
        self.playButton.enabled = YES;
    }
    else {
        self.playButton.enabled = NO;
} } 
```   

在ViewController的`viewDidLoad`方法中你可以触发这个`syncUI`来确保View在首次展示时候的用户界面是统一的。

```objc 
- (void)viewDidLoad {
    [super viewDidLoad];
    [self syncUI];
```    

其余的属性和方法将会在接下来的部分中给以描述。

### 创建Asset

使用`AVURLAsset`来将一个URL创建为一个Asset（接下来的例子假如你的项目包含了一个可用的Video资源）。


```objc
- (IBAction)loadAssetFromFile:sender {
    NSURL *fileURL = [[NSBundle mainBundle]
        URLForResource:<#@"VideoFileName"#> withExtension:<#@"extension"#>];
    AVURLAsset *asset = [AVURLAsset URLAssetWithURL:fileURL options:nil];
    NSString *tracksKey = @"tracks";
    [asset loadValuesAsynchronouslyForKeys:@[tracksKey] completionHandler:
     ^{
         // The completion block goes here.
     }];
} 
``` 

在完成的Block回调中，你可以给这个asset创建一个`AVPlayerItem`对象，并且将其设置为`player view`的`player`。和创建`asset`一样，仅仅创建一个`player item`并不意味着就可以立即使用了。为了确定什么时候可以播放，你需要观察`item`的`status`属性。你需要将该player item对象和player关联之前来配置这种KVO的监听。

当你将player item和player关联的时候，你需要触发player item的准备。

```objc  
// Define this constant for the key-value observation context.
  static const NSString *ItemStatusContext;
  // Completion handler block.
           dispatch_async(dispatch_get_main_queue(),
              ^{
                  NSError *error;
  
error:&error];
AVKeyValueStatus status = [asset statusOfValueForKey:tracksKey
if (status == AVKeyValueStatusLoaded) {
                      self.playerItem = [AVPlayerItem playerItemWithAsset:asset];
                      
 // ensure that this is done before the playerItem is associated with the player
 [self.playerItem addObserver:self forKeyPath:@"status"
             options:NSKeyValueObservingOptionInitial
  context:&ItemStatusContext];
                      [[NSNotificationCenter defaultCenter] addObserver:self
  selector:@selector(playerItemDidReachEnd:)
  name:AVPlayerItemDidPlayToEndTimeNotification
  object:self.playerItem];
self.player = [AVPlayer playerWithPlayerItem:self.playerItem]; [self.playerView setPlayer:self.player]; 
} else { 
  
  // You should deal with the error appropriately.
  NSLog(@"The asset's tracks were not loaded:\n%@", [error
  localizedDescription]);
  } 
}); 
```    

### 响应Player Item的状态变更
  
当一个player item的状态变更的时候，这个View Controller收到一个KVO的变更通知。AV Foundation不会去指明这个通知发送到哪个线程上。如果你需要变更UI,那么你必须确保相关代码要在主线程中执行。改代码使用`diapatch_async`来将将同步UI的操作放到主线程中去。

```objc   
 - (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
                          change:(NSDictionary *)change context:(void *)context {
      if (context == &ItemStatusContext) {
          dispatch_async(dispatch_get_main_queue(),
          ^{ 
                 [self syncUI];
           });
     
           return; 
      } 
      [super observeValueForKeyPath:keyPath ofObject:object
             change:change context:context];
       
       return; 
} 
```    

### 播放该Item

播放视频需要给player对象发送一个`play`的消息：

```objc
- (IBAction)play:sender {
    [player play];
} 
```   

这个Item被播放了一次。在播放完成之后，playhead被放置在item的尾部，所以如果进一步触发其`play`方法是不起作用的。为了playhead放置在item的起始位置，你可以对该item注册并收到一个`AVPlayerItemDidPlayToEndTimeNotification`的通知。然后在通知的会调中触发`seekToTime:`方法，传入`kCMTimeZero`参数即可：

```objc
// Register with the notification center after creating the player item.
    [[NSNotificationCenter defaultCenter]
        addObserver:self
        selector:@selector(playerItemDidReachEnd:)
        name:AVPlayerItemDidPlayToEndTimeNotification
        object:[self.player currentItem]];
- (void)playerItemDidReachEnd:(NSNotification *)notification {
    [self.player seekToTime:kCMTimeZero];
} 
```

