---
title: SDWebImage学习笔记（三）
date: 2018-01-17 10:44:35
tags: FrameWork
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

## 解除Block中的循环引用

在使用Block的时候我们要小心Block和对象之间的相互持有不能释放的问题，比如下面的情况：

```objc
// MKAnnotationView+WebCache.m
- (void)setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options completed:(SDWebImageCompletedBlock)completedBlock
{
    [self cancelCurrentImageLoad];
    self.image = placeholder;
    if (url)
    {
        __weak MKAnnotationView *wself = self;
        id<SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadWithURL:url options:options progress:nil completed:^(UIImage *image, NSError *error, BOOL fromCache, BOOL finished)
        {
            __strong MKAnnotationView *sself = wself;
            if (!sself) return;
            if (image)
            {
                sself.image = image;
            }
            if (completedBlock && finished)
            {
                completedBlock(image, error, fromCache);
            }
        }];
        objc_setAssociatedObject(self, &operationKey, operation, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
}
```

在这里我们创建了一个operation，并且将其关联给了一个MKAnnotationView对象，而这个operation中的Block又持有了这个MKAnnotationView对象，这就造成了循环引用导致两者都不能释放。这时最好的做法就是先在Block外层声明一个`__weak`的引用，然后再Block内部重新将其变为`__strong`，这样就既能保证只在Block执行的范围内强引用了MKAnnotationView对象，随着Block执行完毕，这个强引用就被销毁（它是局部变量，存储在栈上，执行完毕系统自动将其释放），同时又能保证在Block内部使用该对象的过程中，对象不被意外销毁。

下面再看一个例子：

```objc
//SDWebImageDownloaderOperation.m
- (id)initWithRequest:(NSURLRequest *)request queue:(dispatch_queue_t)queue options:(SDWebImageDownloaderOptions)options progress:(void (^)(NSUInteger, long long))progressBlock completed:(void (^)(UIImage *, NSData *, NSError *, BOOL))completedBlock cancelled:(void (^)())cancelBlock
{
    if ((self = [super init]))
    {
        _queue = queue;
        _request = request;
        _options = options;
        _progressBlock = [progressBlock copy];
        _completedBlock = [completedBlock copy];
        _cancelBlock = [cancelBlock copy];
        _executing = NO;
        _finished = NO;
        _expectedSize = 0;
    }
    return self;
}
```

这里在初始化的时候传入了一个`completedBlock`，那么在执行完毕，调用完completedBlock的时候，要注意调用`self.completionBlock = nil;`来消除循环引用：

```objc
- (void)connectionDidFinishLoading:(NSURLConnection *)aConnection
{
    //....
	SDWebImageDownloaderCompletedBlock completionBlock = self.completedBlock;
	if (completionBlock)
	{
		dispatch_async(self.queue, ^
		{
			UIImage *image = [UIImage decodedImageWithImage:SDScaledImageForPath(self.request.URL.absoluteString, self.imageData)];
			dispatch_async(dispatch_get_main_queue(), ^
			{
				completionBlock(image, self.imageData, nil, YES);
				self.completionBlock = nil;
                [self done];
			});
		});
    }
   //...
}
```

这里将completionBlock置为nil是为了防止客户端在这个block内部使用了SDWebImageDownloaderOperation对象，从而造成循环引用，导致对象无法正常销毁。相关内容在《Effective Objective-C 2.0》书中第二十条有较为详细的描述。

## 获取内存和磁盘中图片大小的方式

获取磁盘中文件大小的方式：

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

获取磁盘中文件个数的方式：

```objc
- (int)getDiskCount
{
    int count = 0;
    NSDirectoryEnumerator *fileEnumerator = [[NSFileManager defaultManager] enumeratorAtPath:diskCachePath];
    for (NSString *fileName in fileEnumerator)
    {
        count += 1;
    }
    return count;
}
```

获取内存中图片大小的方式：

```objc
- (int)getMemorySize
{
    int size = 0;
    for(id key in [memCache allKeys])
    {
        UIImage *img = [memCache valueForKey:key];
        size += [UIImageJPEGRepresentation(img, 0) length];
    };
    return size;
}
```

## 参考资料：

 https://www.jianshu.com/p/404e0ea5f6d7
 