---
title: 从SDWebImage中学到的（二）
date: 2017-08-23 09:52:09
tags: FrameWork
---

本文是根据SDWebImage，Tag2.0到Tag2.6所做的总结。

## 注意判空

为了安全起见，方法中每个参数的值的异常我们都需考虑到，我们要有一个guard，然后直接返回，否则在某些特殊情况下，比如这个参数是服务端返回的，某次服务端没有返回，就会崩溃，加了guard之后就不会了，在swift中加入了`guard`关键字来做：

```swift
 guard x > 0 else {
        // 变量不符合条件判断,则返回
        return
    }
```     

因为在OC中没有这样的判断，所以我们需要添加如下的代码：
```objc 
  if (!url)
    {
        self.image = nil;
        return;
    }
```   

## 针对某个机型做接口适配

如果某个机型的手机需要做一些特殊处理，我们可以这么做：

```objc
 #ifdef __IPHONE_4_0
 UIDevice *device = [UIDevice currentDevice];
        if ([device respondsToSelector:@selector(isMultitaskingSupported)] && device.multitaskingSupported)
        {
            // When in background, clean memory in order to have less chance to be killed
            [[NSNotificationCenter defaultCenter] addObserver:self
                                                     selector:@selector(clearMemory)
                                                         name:UIApplicationDidEnterBackgroundNotification
                                                       object:nil];
        }
#endif
```   

这样一来就只有iPhone4会执行里面的代码（__IPHONE_4_0是系统自带的宏）。

## 多线程中如何使用`NSFileManager`

如果某个方法是在后台线程中执行的，并且这个线程中使用了`NSFileManager`，为了避免资源竞争，需要使用`[[NSFileManager alloc] init]`的方式进行初始化：

```objc
  // Can't use defaultManager another thread
    NSFileManager *fileManager = [[NSFileManager alloc] init];
```   

## 存储图片时候不改变其图片类型

如果我们在存储图片的时候要存储成某种格式的图片类型，那么可以使用：

```objc
 [fileManager createFileAtPath:[self cachePathForKey:key] contents:UIImageJPEGRepresentation(image, (CGFloat)1.0) attributes:nil];
```   

如果我们不想改变存储时候的数据类型，那么可以使用：

```objc
   [fileManager createFileAtPath:[self cachePathForKey:key] contents:data attributes:nil];
```   

来完成。

## 在NSOperationQueue中添加立即执行的任务

如果要在NSOperationQueue中添加要立即执行的任务，那么可以这样做：

```objc
[cacheOutQueue addOperation:[[[NSInvocationOperation alloc] initWithTarget:self selector:@selector(queryDiskCacheOperation:) object:arguments]
```   


## 头文件引入过多

如果我们的头文件引入过多，那么我们需要将这些头文件都放到一个新的文件中去，然后引入那个单独的文件即可，比如SDWebImageCompat文件的存在就是为了解决这个问题：

```objc
#import <TargetConditionals.h>
#if !TARGET_OS_IPHONE
#import <AppKit/AppKit.h>
#ifndef UIImage
#define UIImage NSImage
#endif
#ifndef UIImageView
#define UIImageView NSImageView
#endif
#else
#import <UIKit/UIKit.h>
#endif
```

## 从数组中找到指向同一块内存空间的指针

如果要从数组中找到数值相同的指针，我们可以使用：

```objc
    NSUInteger idx = [cacheDelegates indexOfObjectIdenticalTo:delegate];
    if (idx == NSNotFound)
    {
        // Request has since been canceled
        return;
    }
```     

这种形式，但是要注意没有找到时候的处理。

## UIImage的解压缩

PNG和JPEG等只是一种图片压缩格式，PNG是无损压缩，含有alpha通道，JPEG是有损压缩，没有alpha通道。我们每次展示图片的时候都需要先将UIImage进行解压缩。解压缩默认是在主线程中进行的，如果图片较大则会造成很大的性能消耗，如果在IO线程中进行则会提高性能，框架中使用的方式是：

```objc
CGImageRef imageRef = image.CGImage;
CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
CGContextRef context = CGBitmapContextCreate(NULL,
                                                 CGImageGetWidth(imageRef),
                                                 CGImageGetHeight(imageRef),
                                                 8,
                                                 // Just always return width * 4 will be enough
                                                 CGImageGetWidth(imageRef) * 4,
                                                 // System only supports RGB, set explicitly
                                                 colorSpace,
                                                 // Makes system don't need to do extra conversion when displayed.
                                                 kCGImageAlphaPremultipliedFirst | kCGBitmapByteOrder32Little); 
    CGColorSpaceRelease(colorSpace);
    if (!context) return nil;

    CGRect rect = (CGRect){CGPointZero, CGImageGetWidth(imageRef), CGImageGetHeight(imageRef)};
    CGContextDrawImage(context, rect, imageRef);
    CGImageRef decompressedImageRef = CGBitmapContextCreateImage(context);
    CGContextRelease(context);

    UIImage *decompressedImage = [[UIImage alloc] initWithCGImage:decompressedImageRef];
    CGImageRelease(decompressedImageRef);
    return [decompressedImage autorelease];
```     

关于UIImage的解压缩，可以参考相关文章：
[怎样避免UIImage解压缩造成的性能消耗？](https://www.cocoanetics.com/2011/10/avoiding-image-decompression-sickness/)，[IOS中的图片解压缩](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/)，[UIImage解压缩的两种形式](https://stackoverflow.com/questions/19682804/why-does-this-code-decompress-a-uiimage-so-much-better-than-the-naive-approach)，[谈谈IOS中图片的解压缩](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/)

## 怎样获取某个目录下的文件大小？

```objc
-(int)getSize
{
    int size = 0;
    NSDirectoryEnumerator *fileEnumerator = [[NSFileManager defaultManager] enumeratorAtPath:diskCachePath];
    for (NSString *fileName in fileEnumerator)
    {
        NSString *filePath = [diskCachePath stringByAppendingPathComponent:fileName];
        NSDictionary *attrs = [[NSFileManager defaultManager] attributesOfItemAtPath:filePath error:nil];
        size += [attrs fileSize];
    }
    return size;
} 
```  

## Struct初始化

在某些情况下，Struct可以使用如下的方法进行初始化：

```objc
CGRect rect = (CGRect){CGPointZero, { CGImageGetWidth(imageRef), CGImageGetHeight(imageRef) }};
```

## 调用方法是需要注意

在某种特殊情况下，调用方法前要判断是否该对象具有某个方法，以免引起崩溃：

```objc
-    if ([((NSHTTPURLResponse *)response) statusCode] >= 400)
+    if ([response respondsToSelector:@selector(statusCode)] && [((NSHTTPURLResponse *)response) statusCode] >= 400)
```  
因为在上述代码中`response`可能不是HTTP的response。

## ImageIO实现图片边下边展示效果

实现渐下渐放效果是在`SDWebImageDownloader.m`文件的`- (void)connection:(NSURLConnection *)aConnection didReceiveData:(NSData *)data{}`方法中。其中的关键代码为：

```objc
 /// Create the image
            CGImageRef partialImageRef = CGImageSourceCreateImageAtIndex(imageSource, 0, NULL);

#ifdef TARGET_OS_IPHONE
            // Workaround for iOS anamorphic image
            if (partialImageRef)
            {
                const size_t partialHeight = CGImageGetHeight(partialImageRef);
                CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
                CGContextRef bmContext = CGBitmapContextCreate(NULL, width, height, 8, width * 4, colorSpace, kCGBitmapByteOrderDefault | kCGImageAlphaPremultipliedFirst);
                CGColorSpaceRelease(colorSpace);
                if (bmContext)
                {
                    CGContextDrawImage(bmContext, (CGRect){.origin.x = 0.0f, .origin.y = 0.0f, .size.width = width, .size.height = partialHeight}, partialImageRef);
                    CGImageRelease(partialImageRef);
                    partialImageRef = CGBitmapContextCreateImage(bmContext);
                    CGContextRelease(bmContext);
                }
                else
                {
                    CGImageRelease(partialImageRef);
                    partialImageRef = nil;
                }
            }
```   
然后根据`CGImageRef`生成对应的`UIImage`并且将结果返回给delegate：

```objc
  UIImage *image = [[UIImage alloc] initWithCGImage:partialImageRef];
  [delegate imageDownloader:self didUpdatePartialImage:image];
```   


