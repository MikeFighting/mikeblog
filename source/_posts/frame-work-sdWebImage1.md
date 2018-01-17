---
title: SDWebImage学习笔记（二）
date: 2018-01-17 10:44:35
tags: Note
---

## 内存缓存大小的限制

在设计缓存框架是的时候，如果内存我们使用的类是NSDictionary，或则NSArray这种通用类的话，如果内存占用率过高，导致系统RAM中少于12M内存（这个数值可能会随着系统版本和手机机型的不同而不同），那么系统的看门狗（watch dog）会将我们的App杀死。这时，我们要限制占用内存的大小。获取系统内存大小的方案如下：

```objc
#import <mach/mach.h>
#import <mach/mach_host.h>
static natural_t minFreeMemLeft = 1024*1024*12; // reserve 12MB RAM
// inspired by http://stackoverflow.com/questions/5012886/knowing-available-ram-on-an-ios-device
static natural_t get_free_memory(void)
{
    mach_port_t host_port;
    mach_msg_type_number_t host_size;
    vm_size_t pagesize;

    host_port = mach_host_self();
    host_size = sizeof(vm_statistics_data_t) / sizeof(integer_t);
    host_page_size(host_port, &pagesize);

    vm_statistics_data_t vm_stat;

    if (host_statistics(host_port, HOST_VM_INFO, (host_info_t)&vm_stat, &host_size) != KERN_SUCCESS)
    {
        NSLog(@"Failed to fetch vm statistics");
        return 0;
    }

    /* Stats in bytes */
    natural_t mem_free = vm_stat.free_count * pagesize;
    return mem_free;
}
```

这里我们定义了最小的内存空间12M，然后在框架中，如果我们使用`get_free_memory`获取可用内存小于`minFreeMemLeft`，那么我们就移除内存缓存：

```objc
- (void)storeImage:(UIImage *)image imageData:(NSData *)data forKey:(NSString *)key toDisk:(BOOL)toDisk
{
    if (!image || !key)
    {
        return;
    }
    if (get_free_memory() < minFreeMemLeft)
    {
        [memCache removeAllObjects];
    }
    [memCache setObject:image forKey:key];
    //.....
}
```

>如果既想避免占用内存过高而被Kill掉，同时也想避免在每个方法中都去判断当前内存可用空间的大小，那么使用`NSCache`替代上文中提到的NSDictionary或者NSArray来做内存缓存的类，因为NSCache在内存吃紧的情况下会自动清除部分缓存。

## 关于图片的Alpha通道

图片的Alpha通道会造成离屏渲染从而带来FPS的下降，所以没有特别必要的情况下应该尽量避免使用Alpha通道，如果从网络上下载下来的图片含有Alpha通道该怎样处理呢？我们可以在强制解码阶段来将Alpha通道去除：

```objc
+ (UIImage *)decodedImageWithImage:(UIImage *)image
{
    CGImageRef imageRef = image.CGImage;
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    // 使用kCGImageAlphaNoneSkipLast，去除了Alpha通道
    CGContextRef context = CGBitmapContextCreate(NULL,
                                                 CGImageGetWidth(imageRef),
                                                 CGImageGetHeight(imageRef),
                                                 8,
                                                 kCGImageAlphaNoneSkipLast | kCGBitmapByteOrder32Little);
    CGColorSpaceRelease(colorSpace);
    if (!context) return nil;

    CGRect rect = (CGRect){CGPointZero,{CGImageGetWidth(imageRef), CGImageGetHeight(imageRef)}};
    CGContextDrawImage(context, rect, imageRef);
    CGImageRef decompressedImageRef = CGBitmapContextCreateImage(context);
    CGContextRelease(context);

    UIImage *decompressedImage = [[UIImage alloc] initWithCGImage:decompressedImageRef scale:image.scale orientation:image.imageOrientation];
    CGImageRelease(decompressedImageRef);
    return SDWIReturnAutoreleased(decompressedImage);
}
```

如果想要保存原图片的Alpha通道，那么可以先获取原图片的Alpha通道信息，然后在调用CGBitmapContextCreate的时候将Alpha信息传递过去：

```objc
    CGImageAlphaInfo alphaInfo = CGImageGetAlphaInfo(imageRef);
    CGContextRef context = CGBitmapContextCreate(NULL,
                                                 CGImageGetWidth(imageRef),
                                                 CGImageGetHeight(imageRef),
                                                 8,
                                                 alphaInfo | kCGBitmapByteOrder32Little);
```
