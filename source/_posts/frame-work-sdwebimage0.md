---
title: SDWebImage学习笔记（一）
date: 2017-04-26 21:17:00
tags: FrameWork
---

### 保证一段代码在主线程中运行，怎么做更好？

可以使用一个宏来替代，这样代码更加整洁，如

    #define dispatch_main_sync_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_sync(dispatch_get_main_queue(), block);\
    }
    #define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }

### Block除了常见的回调，还有什么应用场景？

在具体的处理方式需要客户端传入时。
如：

```objc
typedef NSString *(^SDWebImageCacheKeyFilterBlock)(NSURL *url);
@property (nonatomic, copy) SDWebImageCacheKeyFilterBlock cacheKeyFilter;
```

该处就是利用Block将处理CacheKey的方法开放给了客户端。通过block的返回值获取。其实这里用代理也可以实现，但是代理相对来说代码量会更多，并且代码较为分散。

### 我们在加锁的时候一直用@synchronized (self)合理吗？

不合理，@synchronized (objc)，只要这个objc是同一个对象，那么就会获得同一把锁。如果访问的是两种不同的资源，那么就需要使用两种不同的objc,比如：

```objc
       @synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }
      @synchronized (self.failedURLs) {
        isFailedUrl = [self.failedURLs containsObject:url];
    }
```

这里分别是对`runningOperations`和`failedURLs`同步，那么就需要使用两种不同的objc，当然都用`self`不能算错，但是将不需要同步的代码同步了，就降低了系统的性能。另外使用self,还容易引起死锁，比如下代码：

```objc
     //class A
      @synchronized (self) {
      [_sharedLock lock];
      NSLog(@"code in class A");
      [_sharedLock unlock];
   }
    //class B
     [_sharedLock lock];
    @synchronized (objectA) {
        NSLog(@"code in class B");
    }
    [_sharedLock unlock];
```

### 如果要让数组中的每个对象都调用某个方法怎么做？

```objc
- (void)makeObjectsPerformSelector:(SEL)aSelector
- (void)makeObjectsPerformSelector:(SEL)aSelector withObject:(nullable id)argument
```

以上两个方法可以实现，而不需要分别遍历每个对象，然后分别调用`performSelector:`

### 内存缓存为啥要用`NSCache`？

NSCache和NSDictionary极其相似，他的方法如下：

```objc
- (nullable ObjectType)objectForKey:(KeyType)key;
- (void)setObject:(ObjectType)obj forKey:(KeyType)key; // 0 cost
- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;//该方法不常用，因为精确地计算对象所占的字节是很费力的，并且计算也会影响缓存的效率。
- (void)removeObjectForKey:(KeyType)key;
- (void)removeAllObjects;
@property NSUInteger totalCostLimit;  // limits are imprecise/not strict
@property NSUInteger countLimit;  // limits are imprecise/not strict
@property BOOL evictsObjectsWithDiscardedContent;//内存吃紧时是否删除废弃的对象
```

从中可以看到，他和NSMutableDictioary非常相似，但是为啥在做内存缓存时要用它呢？原因在于：

1. 当系统资源将要耗尽时，它可以自动删减缓存。如果采用字典，那么就要自己编写相关逻辑，在系统发出“低内存”通知时手工删减缓存。
2. NSMutableDictionary是非线程安全的，而NSCache是线程安全的。
3. NSMutableDictionary中的key必须实现NSCopying协议，NSCache中的key不必实现copy因为它是"保留"键的(强引用)，而不是"拷贝"键的。
4. 如果缓存设置超过了设置的最大值，则会清除旧的数据，保留最新缓存的数据。

### FOUNDATION_STATIC_INLINE放在方法名前有何作用？

```objc
FOUNDATION_STATIC_INLINE NSUInteger SDCacheCostForImage(UIImage *image) {
return image.size.height * image.size.width * image.scale * image.scale;
}
```

内联函数的代码会被直接嵌入在被调用的地方，调用几次就嵌入几次，没有使用call指令，这样就减少了在函数调用过程中保存现场（压栈）恢复现场（弹栈）的操作，可以加快执行速度。不过调用次数多的话，会使可执行文件变大，这样会降低速度。相比起宏来说，内核开发者一般更喜欢使用内联函数。因为内联函数没有长度限制，格式限制。编译器还可以检查函数调用方式，以防止其被误用。

### `inline`(内联函数)在什么时候使用？

在`SDWebIamgeCompat`中使用了`inline UIImage *SDScaledImageForKey(NSString *key, UIImage *image)`,这是个内联函数(函数代码被放入符号表中，在使用时进行替换,比调用一般的函数更加高效)，那么我们在什么时候使用内联函数呢？[经过查找相关资料](http://stackoverflow.com/questions/1932311/when-to-use-inline-function-and-when-not-to-use-it)，总结下inline的使用场合：
使用**inline**的场合：

1. 想要使用`inline`替换`#define`时。
2. **短函数**。（如果函数的代码较长，使用内联将消耗过多栈内存）
3. 函数调用很频繁。

不应使用`inline`的场合：

1. 很大的函数。
2. 和I/O相关的函数。
3. 构造函数和析构函数。
4. 在开发框架时候，使用`inline`可能会破坏框架的兼容性。

### 获取某个目录下文件的个数：

```objc
NSDirectoryEnumerator *fileEnumerator = [_fileManager enumeratorAtPath:self.diskCachePath];
count = [[fileEnumerator allObjects] count];
```

### 如何让某个属性只在固定版本的时候才会有？

```objc
     #if TARGET_OS_IPHONE && __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_4_0
     @property (assign, nonatomic) UIBackgroundTaskIdentifier backgroundTaskId;
     #endif
```

上面这段代码就可以让backgroundTaskId在iPhone的版本在4.0以后才会有。

### Notification的方法调用所在的线程是根据`Post`时候所在线程决定的

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    [[NSNotificationCenter defaultCenter]   postNotificationName:SDWebImageDownloadStopNotification object:self];
});  
```

这样注册该通知的对象就可以在主线程中调用响应的方法了。

### 如何保持后台下载图片的线程一直存在？

使用RunLoop可以让线程常驻(具体解释在我的[实例化讲解RunLoop]中有说明(https://mikefighting.github.io/2016/04/25/understanding-run-loop/))，调用`CFRunLoopRun()`和`CFRunLoopStop(CFRunLoopGetCurrent())`分别用来开始和结束一个RunLoop

### 分类中需要填加属性怎么办？

如果分类中的属性只是分类内部使用，那么其实可以直接使用关联，而不必非要显式创建一个属性，这样也可以直接使用`.`语法，这时没有属性，所以`.`语法的无论是在`=`左边，还是在`=`右边最终都会调用这个方法，例如：

```objc
static char imageURLStorageKey;
- (NSMutableDictionary *)imageURLStorage {
NSMutableDictionary *storage = objc_getAssociatedObject(self, &imageURLStorageKey);
if (!storage){
    storage = [NSMutableDictionary dictionary];
    objc_setAssociatedObject(self, &imageURLStorageKey, storage, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
return storage;
}
```

### 如何让一个`id`类型的对象调用某个具体的方法？

让这个对象遵守某项协议就可以调用,如下

```objc
for (id <SDWebImageOperation> operation in operations) {
if (operation) {
    [operation cancel];
}
```

同理，如果想让某个方法的返回值具有某个方法，也可以让这个返回值遵守某协议，如：

```objc
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url
                                        options:(SDWebImageOptions)options
                                        progress:  (SDWebImageDownloaderProgressBlock)progressBlock
                                        completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
```

### 设计接口时，要尽量考虑使用者的习惯，并对常见错误进行处理

在需要传入URL参数的地方，使用者很可能不小心传入了字符串，这个时候要么在方法中抛出异常，要么就在内部判断类型并替使用者做相应的转换，如：

```objc
- (id <SDWebImageOperation>)downloadImageWithURL:(NSURL *)url {
        if ([url isKindOfClass:NSString.class]) {
    url = [NSURL URLWithString:(NSString *)url];
}
}
```

### 某个类添加通知的方式

很多时候某个类要发出通知，我们经常放到宏里，但是如果这两个类是相关的，我们其实可以将通知放到对应的头文件中，然后在`.m`文件中将其赋值。

```objc
///.h
extern NSString *const SDWebImageDownloadStartNotification;
///.m
NSString *const SDWebImageDownloadStartNotification = @"SDWebImageDownloadStartNotification";
```

### 使用NSURLConnection时，怎样控制是否缓存请求到的数据？

```objc
- (NSCachedURLResponse *)connection:(NSURLConnection *)connection willCacheResponse:(NSCachedURLResponse *)cachedResponse {
responseFromCached = NO; // If this method is called, it means the response wasn't read from cache
if (self.request.cachePolicy == NSURLRequestReloadIgnoringLocalCacheData) {
    // Prevents caching of responses
    return nil;
}
else {
    return cachedResponse;
}
}
```

如果这里返回`nil`，那么将不会缓存这个response，如果返回`cachedResponse`表示可以缓存这个response.

IOS5.0之后，如果请求和响应满足以下条件，系统就会在如下目录中生成一个Cache.db这样一个数据库来存储缓响应的数据。
![生成的缓存目录](http://upload-images.jianshu.io/upload_images/1513759-541dae36f7d02f98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在缓存期间如果访问相同的`URL`，那么就会直接从这个数据库中得到相应的数据;同时在系统的内存告紧时，会自动把内存缓存清空。
这个缓存协议被回调的条件是:

* HTTP或者HTTPS请求（如果是自定义的协议，那么协议需要支持缓存）
* 请求必须是成功的(状态码为200-299)
* 响应必须是服务端传回来的，而不是本地缓存传回来的
* 进行该请求的NSURLRequest对象的`cachePolicy`是允许缓存的
* 服务的响应头含有支持缓存的字段
* 响应的内容大小没有超过缓存的大小（例如，提供磁盘缓存时，响应内容不能超过磁盘缓存的5%）

**注意**：如果要自定义NSURLCache,那么在自定义NSURLCache进行数据缓存时，一定要在`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`中进行初始化，