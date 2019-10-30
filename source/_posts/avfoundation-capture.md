---
title: AVFoundation--媒体捕获
date: 2017-08-24 10:27:12
tags: Objective-C
---

为了管理设备(比如摄像头或者麦克风)捕获的信息，你需要组合一些对象来代表输入和输出，并且使用`AVCaptureSession`对象来调节它们之间的数据。你最少需要如下步骤：

* `AVCaptureDevice`对象来代表输入设备，比如：摄像头或者麦克风
* 具体的`AVCaptureInput`子类实例来配置输入设备的`port`
* 具体`AVCaptureOutput`子类实例来输入一个视频文件或者静态图片
* 一个`AVCaptureSession`实例来协调输入到输入的数据流

为了给用户预览正在录制的视频，你可以使用`AVCaptureVideoPreviewLayer`实例来实现。你可以通过一个单一的session来配置各种输入输出：

![session work](http://upload-images.jianshu.io/upload_images/1513759-c534307e74f3784f.png)

对于很多应用来说，这就已经够用了。然而对于某些操作来说，比如，你要检测音频频道的power level，你需要考虑某个输入设备的各种端口，以及这些端口是怎样和输出相连接的。
捕获输入和捕获输入之间的关联可以用`AVCaptureConnection`对象来表示。捕获输入（`AVCaptureInput`实例）有一到多个输入端口（`AVCaptureInputPort`实例）。捕获输出（`AVCaptureOutPut`实例）可以接受一到多个数据源（比如，一个`AVCaptureMovieFileOutput`对象可以接收视频或音频数据）。

当你往一个session中加入一个输入或者一个输出的时候，这个session可以形成针对所有捕获输入端口和捕获输出的连接。一个捕获输入和捕获输出之间的连接可以被一个`AVCaptureConnection`对象来表示。

![Capture Connection](http://upload-images.jianshu.io/upload_images/1513759-5fc693a0e896e075.png)

对于指定的输入或者输出，你可以使用一个capture connection来使能或者关闭其数据流。你也可以使用connection来检测一个音频频道的平均及最高的power level。

*注：媒体捕获不支持同时捕获IOS设备的前置和后置摄像头。*

## 使用捕获Session来调节数据流

一个`AVCaptureSessin`对象是你用来管理数据捕获的关键调节对象。你可以使用一个实例来调节视频音频输入到输出的数据。你可以给这个session添加捕获设备，然后通过调用session的`startRuning`方法来开始这个数据流，并且通过调用`stopRunning`方法来结束数据流。

```objc
AVCaptureSession *session = [[AVCaptureSession alloc] init];
// Add inputs and outputs.
[session startRunning];
```

### 配置Session

使用`preset`来对session配置你所喜欢的图片质量和分辨率。一个preset是一个常量，它表明了很多种可选配置方案中的一种；在某些情况下实际的配置和具体的设备是相关的。
![prest](http://upload-images.jianshu.io/upload_images/1513759-7fb0bd8bbe4ae251.png)

如果你要对针对某个屏幕大小做配置，你需要在设置前检测其是否支持：
   
	if ([session canSetSessionPreset:AVCaptureSessionPreset1280x720]) {
	    session.sessionPreset = AVCaptureSessionPreset1280x720;
	} else { 
	    // Handle the failure.
	}
	    
如果你要使用preset，以便在一个更加细粒度得调整session的参数，或者你要对一个执行中的session做某些改动，那么你需要你需要将你的方法添加在`beginConfiguration`和`commitConfiguration`方法中间。`beginConfiguration`和`commitConfiguration`可以确保设备的改变作为一组，最小的能见度或者状态的不兼容行。在调用`beginConfiguration`之后，你可以添加或者移除输出，改变`sessionPreset`属性，或者单独配置捕捉输入属性。只有在触发了`commitConfiguration`之后所有的改变才会生效，并且这些改变将会同时生效。

```objc
session beginConfiguration];
// Remove an existing capture device.
// Add a new capture device.
// Reset the preset.
[session commitConfiguration];
```    

### 检测Capture Session的状态

Capture session会在它开始，停止运行或者被打断的时候发出通知，你可以观察这些通知一遍被告知。你可以通过注册`AVCaptureSessionRuntimeErrorNotification`来观察一个运行时错误。你也可以通过查询session的`running`属性来判断其是否正在运行，以及其`interrupted`属性来判断其是否被打断了。除此之外，`running`属性和`interrupted`属性都支持KVO，这个会在主线程中被触发。

## AVCaptureDevice对象代表一个输入设备

一个`AVCaptureDevice`对象时对一个物理捕获设备的抽象，该对象可以给`AVCaptureSession`对象提供输入数据（比如，音频或者视频）。每个对象代表了一种输入设备，比如两个视频输入（前置后置摄像头），一个音频输入（麦克风）。
你可以通过使用`AVCaptureDevice`类方法`devices`和`devicesWithMediaType`来找出当前可用的捕获设备。并且，如果有必要的话你可以找出某个iPhone，iPad或者iPod所提供的特性。尽管如此，可用设备的列表是可以变化的。当前的输入设备可能变得不可用（如果它们被另外的应用所使用），新的输入设备变得可用（如果它们被另外的设备所放弃）。你可以注册`AVCaptureDeviceWasConnectedNotification`和`AVCaptureDeviceWasDisconnectedNotification`通知，以便在可用设备变化时候被通知到。你可以使用一个capture input来讲一个输入设备添加到`capture session`中。

## 设备属性

你可以查询某个设备的不同属性。使用`hasMediaType:`和`supportsAVCaptureSessionPreset:`方法，你可以检测某个设备是否提供某种媒体类型或者支持一种给定的capture session preset。为了给用户提供有用的信息，你可以找出捕获设备的位置(它在检测单元的前面还是后面)，以及它本地化后的名字。如果你要给用户展示一个捕获列表来让其选择，那么这种方式是很有用的。

下图展示了后置（AVCaptureDevicePositionBack）和前置（AVCaptureDevicePositionFront）摄像头。
![front and back facing camera position](http://upload-images.jianshu.io/upload_images/1513759-b1a87fc70f2f3de3.png)

下面的例子遍历了所有的可用设备并且打印出它们的名字（对于视频设备来说，还有它们的位置）。

```objc
NSArray *devices = [AVCaptureDevice devices];
  for (AVCaptureDevice *device in devices) {
      NSLog(@"Device name: %@", [device localizedName]);
      if ([device hasMediaType:AVMediaTypeVideo]) {
          if ([device position] == AVCaptureDevicePositionBack) {
              NSLog(@"Device position : back");
  
} else { 
NSLog(@"Device position : front");
          }
} } 
```  

除此之外，你可以找出设备的模型ID以及其唯一标示。

### 设备捕获设置

不同的设备有不同的功能，比如，某些设备支持聚焦和flash模式，某些可以支持聚焦到某个兴趣点。
下面的代码段展说明了怎样找到有手电筒模式的视频输入设备，并且它支持一个给定的capture session preset：

```objc
NSArray *devices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];
NSMutableArray *torchDevices = [[NSMutableArray alloc] init];
for (AVCaptureDevice *device in devices) {
    [if ([device hasTorch] &&
         [device supportsAVCaptureSessionPreset:AVCaptureSessionPreset640x480]) {
        [torchDevices addObject:device];
} } 
```    

如果你发现很多设备满足你的标准，那么你可以让用户从中选择他们喜欢的来使用。为了给用户描述这个设备，你可以使用它的`localizedName`属性。
你可以用相似的方式来使用不同的特性。指明一种特定的模式方法是固定的，并且你可以查看设备是否支持某种特定模式。在某些情况下，你可以观察一个属性以便属性改变时候得到通知。任何情况下，在某种特定的特性下更改模式前要锁住设备，这在下文的`配置设备`小结中会有提及。

*注：聚焦兴趣点和曝光兴趣点是彼此独立的，它们分别是focus mode和exposure mode。*

#### 聚焦模式

有三种聚焦模式：

* `AVCaptureFocusModeLocked`：聚焦位置是固定的。当你想在锁住聚焦的情况下，让用户来组成场景那么这是很有用的。
* `AVCaptureFocusModeAutoFocus`：摄像头做一次扫描聚焦，然后返回到被锁住的状态。如果你想聚焦选中某个特定的物体，并且对那个物体保持聚焦（尽管那个物体可能不在拍摄场景中心），那么这种模式是很合适的。
* `AVCaptureFocusModeContinuousAutoFocus:`在这种模式下摄像头在需要的情况下持续地进行自动对焦。

在利用`focusMode`属性来设置聚焦模式前，你可以使用`isFocusModeSupported:`方法来决定某个设备是否支持给定的聚焦模式。
除此之外，某种设备可能会支持对某个兴趣点的聚焦。你可以使用`focusPointOfInterestSupported`属性来做判断。如果支持的话，你可以使用`focusPointOfInterest`属性来设置聚焦点。你可以传递一个CGPoint值来指定聚焦的点，其中{0,0}代表左上角，{1,1}在水平模式home键在右侧的情况下。如果设备在竖直模式下，那么这种坐标关系也是适用的。

使用`adjustingFocus`属性来决定是否某个设备当前正在聚焦。使用KVO的形式，你可以再设备开始和停止聚焦的时候收到通知。

如果你改变了聚焦模式的设置，你可以像下面一样，将它们返回到默认设置：

```objc
if ([currentDevice isFocusModeSupported:AVCaptureFocusModeContinuousAutoFocus]) {
    CGPoint autofocusPoint = CGPointMake(0.5f, 0.5f);
    [currentDevice setFocusPointOfInterest:autofocusPoint];
    [currentDevice setFocusMode:AVCaptureFocusModeContinuousAutoFocus];
} 
```  

#### 曝光模式（Exposure Modes）

曝光模式有以下两种：

* `AVCaptureExposureModeContinuousAutoExposure：`在需要的情况下设备自动调整曝光水平。
* `AVCaptureExposureModeLocked：`在当前水平曝光水平是固定的。

使用`isExposureModeSupported:`方法来决定是否某个设备支持某种给定的曝光模式，然后使用`exposureMode`属性来进行设定。

除此之外，某个设备可能会支持兴趣点的曝光。你可以使用`exposurePointOfInterestSupported`来特使其是否支持。如果支持的话，你可以使用`exposurePointOfInterest`属性来进行设置。你可以传递一个CGPoint值来指定聚焦的点，其中{0,0}代表左上角，{1,1}在水平模式home键在右侧的情况下。如果设备在竖直模式下，那么这种坐标关系也是适用的。
使用`adjustingExposure`属性来决定是否某个设备当前正在改变其曝光设置。使用KVO的形式，你可以再设备开始和停止曝光设置的时候收到通知。

如果你改变了曝光设置，那么你可以返回默认设置：

```objc
if ([currentDevice
isExposureModeSupported:AVCaptureExposureModeContinuousAutoExposure]) {
    CGPoint exposurePoint = CGPointMake(0.5f, 0.5f);
    [currentDevice setExposurePointOfInterest:exposurePoint];
    [currentDevice setExposureMode:AVCaptureExposureModeContinuousAutoExposure];
} 
```   

#### 闪光灯模式

闪光灯模式又以下三种：

* `AVCaptureFlashModeOff`：闪光灯不会开启
* `AVCaptureFlashModeOn`：闪光灯总是开启
* `AVCaptureFlashModeAuto`：设备会根据周围的光线环境来决定是否开启闪光灯

使用`hasFlash`方法可以确定某个设备是否有闪光灯。如果返回是`YES`，然后你可以使用`isFlashModeSupported：`方法来判断其是否支持某种给定的闪光灯模式，最后使用`flashMode`属性来对其进行配置。

#### 手电筒模式

在手电筒模式下，为了照亮视频捕捉，闪光灯是在低电耗的情况下持续打开的。一共有三种手电筒模式：

* `AVCaptureTorchModeOff`：手电筒关闭
* `AVCaptureTorchModeOn`：手电筒总是打开
* `AVCaptureTorchModeAuto`：根据需要手电筒选择打开或者关闭

你可以使用`hasTorch`属性来检测一个设备是否有手电筒。然后你可以使用`isTorchModeSupported:`方法来检测一个设备是否有给定的手电筒模式，然后通过`torchMode`属性来设置手电筒模式。
对于有手电筒的设备，如果设备和一个执行中的capture session相连接，那么这个手电筒就会被打开。

#### 视频防抖

依赖于具体的硬件设备，对于操作视频的连接，视频防抖是用的。尽管如此，并非所有的源格式和视频分辨率都支持视频防抖。

使能视频防抖可能会给视频捕捉Pipeline引入多余的延迟。可以使用`videoStabilizationEnabled`属性来检测视频防抖是否正在使用。`enablesVideoStabilizationWhenAvailable`属性可以在摄像头支持的情况下让应用自动的使能视频防抖。由于以上的限制，默认情况下视频防抖是被关闭的。

#### 白色平衡

有两种白色平衡模式：

* `AVCaptureWhiteBalanceModeLocked`：白平衡模式是固定的。
* `AVCaptureWhiteBalanceModeContinuousAutoWhiteBalance`：在需要的情况下，摄像头持续调整白平衡。

你可以使用`isWhiteBalanceModeSupported:`方法来确定某个设备是否支持一种给定的白平衡，然后使用`whiteBalanceMode`属性来设置白屏模式。

你可以使用`adjustingWhiteBalance`属性来决定某个设备是否正在改变其白光模式的设定。你可以通过KVO来观察一个设备开始或者终止其白光设定。

#### 设置设备方向

对于一个`AVCaptureConnection`，你可以给它设定指定的方向来指明你在`AVCaptureOutput`（AVCaptureMovieFileOutput，AVCaptureStillImageOutput 以及AVCaptureVideoDataOutput）中图片的朝向。

使用`AVCaptureConnectionsupportsVideoOrientation`属性来确定设备是否支持改变视频的方向，也可以使用`videoOrientation`属性来指明你想在输出端口中的指向。下面的代码指明了如何将一个`AVCaptureConnection`属性设定为`AVCaptureVideoOrientationLandscapeLeft:`

```objc
AVCaptureConnection *captureConnection = <#A capture connection#>;
if ([captureConnection isVideoOrientationSupported])
{
AVCaptureVideoOrientation orientation = AVCaptureVideoOrientationLandscapeLeft; [captureConnection setVideoOrientation:orientation]; 
} 
```   

#### 设置设备

为了给设备设置一个捕获属性，你必须使用`lockForConfiguration：`属性来获取该设备的锁。这会避免其它应用对该属性做出不能兼容的改变。下面的代码片段说明了怎样在一个设备上改变其聚焦模式（首先确定是否该模式是被支持的），然后尝试锁住该设备来重新配置。如果这个锁是可获取的，那么聚焦模式就被改变，随后这个锁会被立即释放掉。

```objc
if ([device isFocusModeSupported:AVCaptureFocusModeLocked]) {
    NSError *error = nil;
    if ([device lockForConfiguration:&error]) {
        device.focusMode = AVCaptureFocusModeLocked;
        [device unlockForConfiguration];
    }
    else {
        // Respond to the failure as appropriate.
}
```    

只要你需要这个可设置的设备属性保持不变，那么你就应该持有这把设备锁。在非必要的情况下持有设备锁可能会降低其它应用的捕获质量。

#### 转换设备

有时你可能想让用户在不同的输入设备间进行切换，比如，从使用前置摄像头到使用后置摄像头。为了避免暂停或者卡顿，你可以在运行状态下配置一个session，然而你需要将你的配置改变放在`beginConfiguration`和`commitConfiguration`之间。

```objc
AVCaptureSession *session = <#A capture session#>;
[session beginConfiguration];
[session removeInput:frontFacingCameraDeviceInput];
[session addInput:backFacingCameraDeviceInput];
[session commitConfiguration];
```  

当最外面的`commitConfiguration`被触发时候，所有的改变会同时被执行。这就确保了平滑的转变。

### 使用Capture Inputs来给一个Session添加捕获设备

你可以使用一个`AVCaptureDeviceInput`（一个抽象`AVCaptureInput`类的子类）实例来给一个capture session添加一个捕获设备。

```objc
NSError *error;
AVCaptureDeviceInput *input =
        [AVCaptureDeviceInput deviceInputWithDevice:device error:&error];
if (!input) {
    // Handle the error appropriately.
}
```  

你可以使用`addInput:`来个一个session添加输入。如果合适的话，你可以使用`canAddInput`来确定某个捕获输入是否和现存的session是兼容的。

```objc
AVCaptureSession *captureSession = <#Get a capture session#>;
  AVCaptureDeviceInput *captureDeviceInput = <#Get a capture device input#>;
  if ([captureSession canAddInput:captureDeviceInput]) {
      [captureSession addInput:captureDeviceInput];
  }
  else {
      // Handle the failure.
} 
```   

如果想知道如何重新配置一个运行中的session，你可以查看上文中的**Configuring a session**中的细节。

一个`AVCaptureInput`可以给媒体数据提供多个流。比如，一个输入设备可以提供视频或者音频数据。每个输入提供的媒体流都是用AVCaptureInputPort来表示的。一个捕获session使用一个`AVCaptureConnection`对象来决定一组`AVCaptureInputPort`对象和一个`AVCaptureOutput`之间的映射。


### 使用捕获输出来获取一个Session的输出

为了从一个捕获session中获取输出，你需要添加一个或者多个output。一个output是一个`AVCaptureOutput`子类的实例。你使用：

* `AVCaptureMovieFileOutput`来输出一个视频文件
* 如果你想处理正在被捕获的视频的frame，比如，你要创建你自己的ViewLayer，那么你可以使用`AVCaptureVideoDataOutput`
* 如果你要处理正在被捕获的音频你可以使用`AVCaptureAudioDataOutput`
* 如果你要捕获静态图片以及元数据你可以使用`AVCaptureStillImageOutput`

使用`addOutput：`你可以给某个捕获session添加outputs。通过使用`canAddOutput`，你可以检测是否某个捕获输出和现存session相兼容。在某个session运行的过程中，你可以根据需要添加或者移除输出。

```objc
AVCaptureSession *captureSession = <#Get a capture session#>;
AVCaptureMovieFileOutput *movieOutput = <#Create and configure a movie output#>;
if ([captureSession canAddOutput:movieOutput]) {
    [captureSession addOutput:movieOutput];
}
else {
    // Handle the failure.
} 
```   

#### 保存视频文件

你可以使用`AVCaptureMovieFileOutput`对象来将某个视频数据保存到文件中。（`AVCaptureMovieFileOutput `是`AVCaptureFileOutput`的子类，它定义了很多基本操作）。你可以对一个视频文件输出的很多方面做以配置，比如录制的最大时长，文件大小。如果可用的磁盘空间小于指定的值你可以禁止视频的录制。

```objc
AVCaptureMovieFileOutput *aMovieFileOutput = [[AVCaptureMovieFileOutput alloc]
init];
CMTime maxDuration = <#Create a CMTime to represent the maximum duration#>;
aMovieFileOutput.maxRecordedDuration = maxDuration;
aMovieFileOutput.minFreeDiskSpaceLimit = <#An appropriate minimum given the quality of the movie format and the duration#>; 
```  

视频输出的分辨率和比特率取决于一个捕获session的`sessionPreset`值。视频编码通常是H.264格式，音频编码通常是AAC格式。实际的数值可能根据设备的不同而改变。

#### 开始录制

使用`startRecordingToOutputFileURL:recordingDelegate:`你可以开始录制一个QuickTime视频。你需要提供一个基于文件的URL以及一个代理。这个URL一定不能和已存在文件相同，因为视频文件输出不会覆盖已存在的资源。同时，你也需要获得往某个指定的路径下写文件的权限。代理必须遵守`AVCaptureFileOutputRecordingDelegate`协议，并且必须要实现`captureOutput:didFinishRecordingToOutputFileAtURL:fromConnections:error:`方法。

```objc
AVCaptureMovieFileOutput *aMovieFileOutput = <#Get a movie file output#>;
NSURL *fileURL = <#A file URL that identifies the output location#>;
[aMovieFileOutput startRecordingToOutputFileURL:fileURL recordingDelegate:<#The
delegate#>];
```    

在实现`captureOutput:didFinishRecordingToOutputFileAtURL:fromConnections:error:`代理方法的时候，代理可以往相册中写入录制的视频。这也可以检测可能出现的错误。

#### 确保文件被成功写入

在实现`captureOutput:didFinishRecordingToOutputFileAtURL:fromConnections:error:`方法的时候，为了确保文件是否被成功的写入了，你不仅仅需要检查error，还需要检查error的user info字典中`AVErrorRecordingSuccessfullyFinishedKey`的值：

```objc
- (void)captureOutput:(AVCaptureFileOutput *)captureOutput
        didFinishRecordingToOutputFileAtURL:(NSURL *)outputFileURL
        fromConnections:(NSArray *)connections
        error:(NSError *)error {
    BOOL recordedSuccessfully = YES;
    if ([error code] != noErr) {
        // A problem occurred: Find out if the recording was successful.
        id value = [[error userInfo]
objectForKey:AVErrorRecordingSuccessfullyFinishedKey];
        if (value) {
            recordedSuccessfully = [value boolValue];
} } 
    // Continue as appropriate...
```  

你需要检测user info字典中的key`AVErrorRecordingSuccessfullyFinishedKeykey`的值，因为尽管有错误，文件还是可以被存储成功。这个错误可能标志着你达到了某个视频录制的限制，比如，`AVErrorMaximumDurationReached`或者`AVErrorMaximumFileSizeReached`。其它可能导致录制停止的原因有：

* 磁盘占满了：AVErrorDiskFull
* 录制设备断开连接了：AVErrorDeviceWasDisconnected
* session被中止了（比如，收到了一个电话）：AVErrorSessionWasInterrupted 

#### 给文件添加元数据

任何时候你都可以给一个视频文件设置元数据，甚至是在录制的时候。在某些场景下这时很有用的，比如当视频录制开始的时候信息不可以拿到，这可能因为位置信息造成的。某个输出文件的元数据是由一组`AVMetaDataItem`对象来表示的；你使用它可变子类的一个实例，`AVMutableMetadataItem`来创建一个你自己的元数据。

```objc
VCaptureMovieFileOutput *aMovieFileOutput = <#Get a movie file output#>;
NSArray *existingMetadataArray = aMovieFileOutput.metadata;
NSMutableArray *newMetadataArray = nil;
if (existingMetadataArray) {
    newMetadataArray = [existingMetadataArray mutableCopy];
}
else {
    newMetadataArray = [[NSMutableArray alloc] init];
} 
AVMutableMetadataItem *item = [[AVMutableMetadataItem alloc] init];
item.keySpace = AVMetadataKeySpaceCommon;
item.key = AVMetadataCommonKeyLocation;
CLLocation *location - <#The location to set#>;
item.value = [NSString stringWithFormat:@"%+08.4lf%+09.4lf/"
    location.coordinate.latitude, location.coordinate.longitude];
[newMetadataArray addObject:item];
aMovieFileOutput.metadata = newMetadataArray;
```  

#### 处理视频帧

一个`AVCaptureVideoDataOutput`对象使用代理来处理视频帧。你可以通过`setSampleBufferDelegate:queue:`方法来设置代理。除了设置代理，你还可以指定该代理方法被调用的队列。你必须使用一个串行队列来确保传递给代理的帧是按照合适顺序进行的。你可以使用队列来改变既定的调度和处理视频帧的优先权。参考`SquareCam`来作为一个实现的例子。

在代理方法里，帧作为一个`CMSampleBufferRef`类型的实例来被表示`captureOutput:didOutputSampleBuffer:fromConnection:`。默认情况下，缓冲是以视频最有效的格式被发发出的。你可以使用`videoSettings`属性来指明一种定制的输出格式。视频的设定属性是一个字典；目前为止，唯一支持的key是`kCVPixelBufferPixelFormatTypeKey`。

推荐的像素格式是通过`availableVideoCVPixelFormatTypes`属性来返回的，同时`availableVideoCodecTypes`属性返回支持的数值。Core Graphics和OpenGL都和BGRA格式配合的很好。

```objc
AVCaptureVideoDataOutput *videoDataOutput = [AVCaptureVideoDataOutput new];
NSDictionary *newSettings =
                @{ (NSString *)kCVPixelBufferPixelFormatTypeKey :
@(kCVPixelFormatType_32BGRA) };
videoDataOutput.videoSettings = newSettings;
 // discard if the data output queue is blocked (as we process the still image
[videoDataOutput setAlwaysDiscardsLateVideoFrames:YES];)
// create a serial dispatch queue used for the sample buffer delegate as well as
when a still image is captured
// a serial dispatch queue must be used to guarantee that video frames will be
delivered in order
// see the header doc for setSampleBufferDelegate:queue: for more information
videoDataOutputQueue = dispatch_queue_create("VideoDataOutputQueue",
DISPATCH_QUEUE_SERIAL);
[videoDataOutput setSampleBufferDelegate:self queue:videoDataOutputQueue];
AVCaptureSession *captureSession = <#The Capture Session#>;
if ( [captureSession canAddOutput:videoDataOutput] )
     [captureSession addOutput:videoDataOutput];
```     

#### 处理视频时考虑的性能因素

对于你的应用来说，你应该给这个session设置最低的可用分辨率。将输出设定到一个比需求更高的分辨率会浪费处理循环以及消耗更多的电量。
你必须确保你实现了`captureOutput:didOutputSampleBuffer:fromConnection`方法，以便在创建一帧的时候可以处理一个采样缓冲。如果这耗费过长的时间，并且你保持了这个视频帧率，那么AV Foundation对象就会停止传送帧，不尽停止给你的代理传递帧，还会停止向其它的输出传递帧，比如preview layer。
你可以使用捕获视频数据输出的`minFrameDuration`属性来确定你在没有被停止的情况下消耗更少的帧率，以便你有足够的时间来处理这一帧。你也许也要确保`alwaysDiscardsLateVideoFrames`属性也被设置成`YES`（默认情况下也是YES）。这个属性确保了任何延迟了的帧都会被丢弃而不是移交给你进行处理。换句话说，如果你正在录制视频，并且如果输出帧有些延迟不会造成影响，并且你想获取所有的帧，那么你可以将这个属性的值设定为NO。这并不意味这不会掉帧（也就是说，帧仍然会掉），但是它不会掉的那么早或者由于效率原因掉帧。

### 捕获静态图片

如果你想捕获静态图片以及元数据，那么你可以使用`AVCaptureStillImageOutput`来输出。这个图片的分辨率取决于这个session的preset以及这个设备。

#### 像素和编码格式

不同的设备支持不同的图片格式。分别使用`availableImageDataCVPixelFormatTypes`和`availableImageDataCodecTypes`你可以找出某个设备支持的pixel和codec类型。每个方法会返回指定设备所支持数据的数组。你可以设置`outputSettings`字典来设置你所想要的图片格式，比如：

```objc
AVCaptureStillImageOutput *stillImageOutput = [[AVCaptureStillImageOutput alloc]
init];
NSDictionary *outputSettings = @{ AVVideoCodecKey : AVVideoCodecJPEG};
[stillImageOutput setOutputSettings:outputSettings];
```  

如果你要获取一张JPEG格式的图片，那么你通常不需要指定你自己的压缩格式。相反，因为它的压缩格式是硬件驱动的，所以你应该让静态图片输出来给你压缩。尽管你改变了图片的元数据，如果你要得到一张图片的数据表示，那么你可以使用`jpegStillImageNSDataRepresentation:`来得到没有被再次压缩数据的`NSData`对象。

#### 捕获一张图片

当你想捕获一张图片的时候，你可以调用输出的captureStillImageAsynchronouslyFromConnection:completionHandler:方法。第一个参数是你为了捕获要用的连接。你需要找到这个连接，它的输入端是一个，它的输入端口正在搜集视频：

	 AVCaptureConnection *videoConnection = nil;
	  for (AVCaptureConnection *connection in stillImageOutput.connections) {
	       for (AVCaptureInputPort *port in [connection inputPorts]) {
	          if ([[port mediaType] isEqual:AVMediaTypeVideo] ) {
	              videoConnection = connection;
	break; } 
	} 
	      if (videoConnection) { break; }
	     }


`captureStillImageAsynchronouslyFromConnection:completionHandler:`方法的第二个参数是一个具有两个参数的block：一个包含了图片数据的`CMSampleBuffer`类和一个error。采样缓冲自身包含了比如EXIF字典等元数据作为其附属。如果你想你可以改变这个附属，但是要注意在**像素和编码格式**中提到的对JPEG图片的优化。

```objc
[stillImageOutput captureStillImageAsynchronouslyFromConnection:videoConnection
completionHandler:
    ^(CMSampleBufferRef imageSampleBuffer, NSError *error) {
        CFDictionaryRef exifAttachments =
            CMGetAttachment(imageSampleBuffer, kCGImagePropertyExifDictionary,
NULL);
        if (exifAttachments) {
            // Do something with the attachments.
        }
        // Continue as appropriate.
    }];
```  

### 展示录制的内容

你可以给用户提供视频录制的预览（利用preview layer）,或者音频录制的预览（通过监听音频channel）。

#### 视频预览

使用`AVCaptureVideoPreviewLayer`对象，你可以给用户提供一个录制内容的预览。`AVCaptureVideoPreviewLayer`是`CALayer`的子类。为了展示预览，你不需要输出任何东西。

在视频展示给用户之前，使用`AVCaptureVideoDataOutput`类可以让客户端程序具有获取视频像素的能力。

和捕获输出不一样，一个视频预览layer保持一个对其关联session的一个强引用。这就确保了在该layer试图播放视频的时候session不会被释放。下面的代码展示了初始化一个预览layer的方式。

```objc
AVCaptureSession *captureSession = <#Get a capture session#>;
CALayer *viewLayer = <#Get a layer from the view in which you want to present the
 preview#>;
AVCaptureVideoPreviewLayer *captureVideoPreviewLayer = [[AVCaptureVideoPreviewLayer alloc] initWithSession:captureSession]; 
[viewLayer addSublayer:captureVideoPreviewLayer];
```  

一般来说，这个preview layer在渲染树中和其它的CALayer对象是一样的。你可以像处理其它layer一样对图片进行伸缩，旋转等。其中有一个不同点是：为了指明来自摄像头的图片怎样旋转，你需要设置layer的`orientation`属性。除此之外，你可以通过查询`supportsVideoMirroring`属性来决定某个设备是否支持视频镜像。如果有需要你可以设置`videoMirrored`属性，尽管当`automaticallyAdjustsVideoMirroring`属性默认情况下是YES，这个镜像值基于session的配置而自动被设定的。

#### 视频重力模式

这个preview layer支持三种重力模式，你可以通过`videoGravity`属性来设置：

* `AVLayerVideoGravityResizeAspect`：这会按照固定的比例保存视频，当视频不能铺满整个可用屏幕大小的时候会留下黑条。
* `AVLayerVideoGravityResizeAspectFill`：这会保质视频的比例，但是会铺满整个屏幕，并且在需要的时候剪裁视频。
* `AVLayerVideoGravityResize`：这会拉伸视频来铺满整个可用的屏幕空间，尽管这样可能让图片扭曲。

#### 给预览添加`点击捕获`功能

在视频连接的时候，使用preview layer来实现点击捕获，你需要小心。你必须要考虑预览方向和layer的重力，以及这个预览可能被镜像的可能性。参考示例代码中IOS的`AVCam`项目来实现这个功能。

### 展示音频等级（Audio Level）

为了在一个捕获连接中的音频通道中检测平均power和峰值power峰值，你可以使用`AVCaptureAudioChannel`对象。因为音频等级不是KVO的，所以你必须按照你需要的频率来访问其数值来更新UI（比如，每秒钟10次）。

```objc
AVCaptureAudioDataOutput *audioDataOutput = <#Get the audio data output#>;
NSArray *connections = audioDataOutput.connections;
if ([connections count] > 0) {
    // There should be only one connection to an AVCaptureAudioDataOutput.
    AVCaptureConnection *connection = [connections objectAtIndex:0];
    NSArray *audioChannels = connection.audioChannels;
    for (AVCaptureAudioChannel *channel in audioChannels) {
        float avg = channel.averagePowerLevel;
        float peak = channel.peakHoldLevel;
        // Update the level meter user interface.
} } 
```  

## 综合：像UIImage对象一样捕获视频帧

下面简明的代码实例向你展示了怎样捕获一个视频并且将你得到的视频帧转换成UIImage对象。它有以下的功能：

* 创建一个`AVCaptureSession`对象来协调一个AV输入的数据流到一个输出
* 对你想要的输入类型找到`AVCaptureDevice`对象
* 给该设备创建一个`AVCaptureDeviceInput`对象
* 创建一个`AVCaptureVideoDataOutput`对象来产生视频帧
* 对`AVCaptureVideoDataOutput`对象实现一个代理对象来处理视频帧
* 实现一个方法来将该代理收到的`CMSampleBuffer`转换成一个UIImage对象

### 创建并且配置一个捕获Session

你可以使用`AVCaptureSession`对象来把来自一个AV输入设备的数据流来转换为一个输出。创建一个session，并且配置它来生成中等分辨率的视频帧：

```objc
AVCaptureSession *session = [[AVCaptureSession alloc] init];
session.sessionPreset = AVCaptureSessionPresetMedium;
```   

### 创建配置设备及设备输入

捕获设备是由`AVCaptureDevice`对象表示的，该类提供了获取你想要的输入类型的方法。一个设备有一个或者多个端口，这些端口使用`AVCaptureInput`对象来配置。通常情况下，在它的默认配置上，你使用捕获输入。

找到一个视频捕获设备，然后利用该设备创建一个设备输入并且将其添加到session里面。如果一个合适的设备不能够被加载，那么`deviceInputWithDevice:error:`方法将会返回一个引用的错误。

```objc
AVCaptureDevice *device =
        [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
NSError *error = nil;
AVCaptureDeviceInput *input =
        [AVCaptureDeviceInput deviceInputWithDevice:device error:&error];
if (!input) {
    // Handle the error appropriately.
}
[session addInput:input];
```   

### 创建和配置视频输出数据

你使用一个`AVCaptureVideoDataOutput`对象来处理正在被捕获而未被压缩的帧。通常情况下，你可以配置一个输出的很多方面。比如，对于视频来说，你可以通过`videoSettings`属性来表明像素格式以及通过设置`minFrameDuration`属性来设置帧率的峰值。

创建和配置一个视频数据的输出并将其添加到session中，通过将`minFrameDuration`属性值设置为1/15来将帧率峰值设置为15fps：

```objc
AVCaptureVideoDataOutput *output = [[AVCaptureVideoDataOutput alloc] init];
[session addOutput:output];
output.videoSettings =
                @{ (NSString *)kCVPixelBufferPixelFormatTypeKey :
@(kCVPixelFormatType_32BGRA) };
output.minFrameDuration = CMTimeMake(1, 15);=
```   

数据输出对象使用代理来提视频帧。这个代理必须遵守`AVCaptureVideoDataOutputSampleBufferDelegate`协议。当你设置数据输出代理的时候，你也必须提供一个回调将要被触发的队列。

```objc
dispatch_queue_t queue = dispatch_queue_create("MyQueue", NULL);
[output setSampleBufferDelegate:self queue:queue];
dispatch_release(queue);
```  
 
你使用该队列改变给定的优先权来传递和处理视频帧。

### 实现帧缓存的代理方法

在该代理类里，实现`captureOutput:didOutputSampleBuffer:fromConnection:`方法，这个方法会在一个采样缓存被写入的时候被调用。视频数据的输出对象以`CMSampleBuffer`类型的形式来传递帧，因此，你需要将一个`CMSampleBuffer`对象转换为一个UIImage对象。这个转换的方法会在`Converting CMSampleBuffer to UIImage Object`中说明。

```objc
 - (void)captureOutput:(AVCaptureOutput *)captureOutput
           didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
           fromConnection:(AVCaptureConnection *)connection {

       UIImage *image = imageFromSampleBuffer(sampleBuffer);
      // Add your code here that uses the image.
  }
```  

这个代理方法被触发的队列就是你在`setSampleBufferDelegate:queue:`中指明的队列；如果你要更新UI，那么你必须在主线程中执行相关的代码。

### 开始和终止录制

在配置完capture session之后，你应该确保根据用户的偏好设置里你有录制视频的权限。

```objc
NSString *mediaType = AVMediaTypeVideo
[AVCaptureDevice requestAccessForMediaType:mediaType completionHandler:^(BOOL
granted) {
    if (granted)
    {
        //Granted access to mediaType
        [self setDeviceAuthorized:YES];
    }
else { 
        //Not granted access to mediaType
        dispatch_async(dispatch_get_main_queue(), ^{
        [[[UIAlertView alloc] initWithTitle:@"AVCam!"
message:@"AVCam doesn't have permission to use Camera, please change privacy settings" 
}); } 
}]; 
                   delegate:self
          cancelButtonTitle:@"OK"
          otherButtonTitles:nil] show];
[self setDeviceAuthorized:NO];
```  

如果摄像头的session被配置完成并且用户隐私设定允许使用摄像头，你可以调用`startRunning`方法来执行session在串行队列上的启动，以便主队列不会被阻塞（这可以让UI更加快速的被响应）。看iOS的AVCam作为一个实现的例子。

```objc
[session startRunning];
```  

通过调用`stopRunning`方法，你可以停止录制视频。

### 高帧率的视频捕获

在选中的硬件上，iOS7.0引入了高帧率的视频捕获（也称作"SloMo"视频）。所有的AV Foundation框架都支持高帧率的内容。
你可以使用`AVCaptureDeviceFormat`类来确定某个设备的捕获能力。该类也有返回诸如：所支持的媒体类型，帧率，视图的field，最大放大倍数以及视频防抖是否被支持等参数。

* 捕获支持在60fps下全720p分辨率，其中包括视频防抖，可掉P帧（H264编码视频的特性，这种特性可以让视频播放很流畅，及时在老设备上也可以这样。）
* 视频播放可以提高音频对慢速和快速播放的支持，这就让音频的捕获时间（time pitch）可以在更低或者更高的速度下被保存。
* 在可变合成的中，可以支持缩放编辑。
* 在支持60fps视频的时候，输出提供了两种选择。可变的帧率，慢速或快速的移动，都会被保存或者视频帧率将会变得更小，比如30fps。

#### 播放

一个`AVPlayer`实例可以通过`setRate：`方法来自动设置大多数的播放速度。该数值会作为播放速度的一个放大倍数。1.0数值表示正常的播放，0.5数值表示以一半的速度播放，5.0数值表示以正常速度5倍的速度来播放。

`AVPlayerItem`对象支持`audioTimePitchAlgorithm`属性。使用`Time Pitch Algorithm Settings`常量，该属性让你可以指明音频在不同帧率下的播放方式。
下表展示了所支持的time pitch算法，数量，该算法是否会造成音频在某个指定帧率下停止，以及每个算法支持的帧率范围。

![Time Pitch Algorithm](http://upload-images.jianshu.io/upload_images/1513759-f751221f7dba579d.png)

#### 编辑

编辑的时候，你可以使用`AVMutableComposition`类来建立临时的编辑。

* 使用`composition`类方法来创建一个新的`AVMutableComposition`实例
* 使用`insertTimeRange:ofAsset:atTime:error`方法来插入你的视频asset
* 使用`scaleTimeRange:toDuration`来设定某个合成中某个部分的时间比例

#### 输出

使用`AVAssetExportSession`类来导出60fps的视频输出一个asset。该内容可以用以下两种方式输出：

* 使用`AVAssetExportPresetPassthrough`preset来避免视频的重编码。通过对标志为60fps的媒体片段进行时间重置，这些媒体片段就会减速或者加速。
* 为了最大播放兼容，使用恒定帧率输出。设定视频合成的`frameDuration`属性为30fps。你也可以通过设定导出session的`audioTimePitchAlgorithm`属性来指定捕获时间(time pitch)。


#### 录制

使用`AVCaptureMovieFileOutput`类，你可以捕获高帧率的视频，它自动支持高帧率捕获。它会自动选择正确的H264捕获水平。

为了使用定制的录制，你必须使用`AVAssetWriter`类，这需要额外的设置。

```objc
assetWriterInput.expectsMediaDataInRealTime=YES;
```  

该设置可以确保捕获可以和输入的数据保持同步。

