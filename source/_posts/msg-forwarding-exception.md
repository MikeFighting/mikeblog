---
title: iOS消应用实例--异常处理
date: 2016-05-19 10:16:00
tags: Objective-C
---

### 问题来源

最近发现了一个在项目中常用的异常处理工具NullSafe，分析了它的实现原理，不小心发现了一个小Bug，现将其分享出来，[关于这篇文章的Demo已经上传至GitHub](https://github.com/MikeFighting/NSNullHandler)，看完如有收获，欢迎Star，如有疑问欢迎issue，大家一起学习。在IOS开发中我们可能会遇到下面的情景:服务器给我们返回得某个字段是null,比如`someValue:null`，这个时候我们利用第三方工具转化之后会得到`someValue = <null>`,这个时候如果我们判断这个`someValue`的类型，会看到其为:NSNull。那么问题来了，如果这个someValue是要给控件赋值，比如:`someLabel.text = someVlaue`，这个时候相当于`someLabel.text = nil`,显然，是不会有问题的。**但是有时候我们可能会给这个貌似是NSString的对象发送消息(因为我们在Model里定义了NSString * someValue)**，比如:`[someValue length]`。这个时候由于`null`这个对象没有这个方法,也就是是说：`null 这个对象不能处理这个消息`所有就会Crash，让程序闪退。那么我们怎样处理来避免这种Crash呢？我们怎样处理这个消息呢？

### OC的消息转发流程

首先我们来看一下`NSObject.h`中我们不常用到的几个方法,以及它们的含义:

```objc
// 判断是否发现了这个method，如果发了，将其添加给该对象，并且返回YES,如果没有返回NO。
+ (BOOL)resolveClassMethod:(SEL)sel; 
// 同上类似
+ (BOOL)resolveInstanceMethod:(SEL)sel;
// 这个方法来指定未被识别的消息首先要指向得对象
- (id)forwardingTargetForSelector:(SEL)aSelector;  
+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector;
// 遍历一个类的实例方法得到这个消息得NSMethodSignature，这个返回值包含了对这个method相关的描述，如果这个method不能找到，那么返回nil.
//遍历一个类的实例方法或者类方法来得到这个消息的NSMethodSignature
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
// NSObject的子类可以覆盖这个方法来讲消息转发给其它的对象。当一个对象发送一个消息，但是这个对象不能响应这个消息，那么RunTime会给这个对象一个来转发这个消息的机会。
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

这个对象通过创建一个NSInvocation对象作为参数来调用这个forwardInvocation方法，然后改对象会调用这个方法来将消息转发给其它对象。
这个NSInvocation对象其实是有上一个方法methodSignatureForSelector中返回的NSMethodSignature来得到的，所以在重写这个方法之前我们必须重写methodSignatureForSelector方法。

```objc
// 这个类的实例是否具有相应这个selector的能力，也就是说这个类有没有这样一个方法
+ (BOOL)instancesRespondToSelector:(SEL)aSelector;
// 遍历一个类的实例方法或者类方法 得到这个方法的IMP(函数指针，指向这个方法的具体实现)
- (IMP)methodForSelector:(SEL)aSelector;
// 遍历一个类的实例方法列表，得到这个方法IMP
+ (IMP)instanceMethodForSelector:(SEL)aSelector;
// 如果一个对象收到了一个消息，但是它不能处理这个消息，并且这个消息没有被转发，那么系统将会调用这个方法。
- (void)doesNotRecognizeSelector:(SEL)aSelector;
同时这个方法会引发一个NSInvalidArgumentException，并且引发error.
```

那么这几个方法，在系统中是怎样的调用顺序呢？我们来看下图:

![IOS中消息处理得流程](http://upload-images.jianshu.io/upload_images/1513759-2e5449addfd826bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从中可以看出在给一个对象发送消息的时候，如果对象没有对应的IML，那么会调用对象所属类的

```objc
 + (BOOL)resolveInstanceMethod:(SEL)sel
```

方法，然后看对于这个SEL对象是否可以执行，如果不可以执行则会调用

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector
```

来找到一个对象处理这个方法(我们可以返回一个对象，这个对象可以处理这个方法)，如果这个方法返回的是`nil`，那么会调用这个对象的

```objc
 - (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
```

这个方法，如果这方法调用之后，仍然没有找到对应的`NSMethodSignature`，那么会调用:

```objc
 - (void)doesNotRecognizeSelector:(SEL)aSelector
```

这个方法，并且抛出异常，如果这个时候返回了一个有效的`NSMethodSignature`，那么会调用

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation
```

到这里消息的处理结束。

那么对于NSNull这个对象，我们如何让它处理一个其自身不能处理的消息呢？这时候，我们肯定会想到，将这个消息传递给其它可以处理的对象。那么问题来了：我们如何能找到这样一个对象呢？这时候我们想到了**RunTime**,利用`RunTime`的`objc_getClassList`方法，我们可以获取整个项目中注册得所有类（只要在项目中添加了这个类文件，无论这个类是否被使用），**这个时候我们可以过滤掉用不到的父类，以节约循环得次数，因为子类已经继承了父类的方法，所以具有处理这个消息得能力**，之后首先利用上文提到的`instancesRespondToSelector`来判断这个类是否可以响应这个消息，如果可以响应，那么可以利用上文提到的`instanceMethodSignatureForSelector`来得到这个`NSMethodSignature`,并且返回。通过上述分析，系统会调用这个`forwardInvocation`,这个是时候我们调用NSInvocation的`invokeWithTarget`这个方法来将这个消息发送给nil,在OC中向一个nil发送任何消息都不会引起程序Crash,至此一个由于服务器返回数据异常而导致的Crash被解决了。

>这显然增加了系统的容错能力，在项目调试阶段，可能由于数据不完善，所以可以利用这个方法来规避Crash,但是在数据基本完善之后，我们可以去掉这种方法以便我们在程序Crash的时候，及时提醒后台人员来完善数据。

### NullSafe的实现详解

```objc
 @implementation NSNull (NullSafe)

 - (NSMethodSignature *)methodSignatureForSelector:(SEL)selector
{
 @synchronized([self class])
 {
  // 寻找 method signature
  NSMethodSignature *signature = [super methodSignatureForSelector:selector];
  if (!signature)
  {
//改消息不能被NSNull处理，所以我们要寻找其它的可以处理的类  
static NSMutableSet *classList = nil;
static NSMutableDictionary *signatureCache = nil;// 缓存这个找到的 method signature，以便下次寻找
if (signatureCache == nil)
{
 classList = [[NSMutableSet alloc] init];
 signatureCache = [[NSMutableDictionary alloc] init];

 // 获取项目中的所有类，并且去除有子类的类。
 // objc_getClassList：这个方法会将所有的类缓存，以及这些类的数量。我们需要提供一块足够大得缓存来存储它们，所以我们必须调用这个函数两次。第一次来判断buffer的大小，第二次来填充这个buffer。
 int numClasses = objc_getClassList(NULL, 0); 
 Class *classes = (Class *)malloc(sizeof(Class) * (unsigned long)numClasses);
 numClasses = objc_getClassList(classes, numClasses);

 NSMutableSet *excluded = [NSMutableSet set];
 for (int i = 0; i < numClasses; i++)
 {
  
  Class someClass = classes[i];
  
  // 筛选出其中含有子类的类，加入:excluded中
  Class superclass = class_getSuperclass(someClass);
  while (superclass)
  {
      // 如果父类是NSObject,则跳出循环，并且加入classList
if (superclass == [NSObject class]) {
 // 将系统中用到的所有类都加到了ClassList中
 [classList addObject:someClass];
 break;
}

//父类不是NSObject,将其父类添加到excluded
[excluded addObject:superclass];
 superclass = class_getSuperclass(superclass);
  }
 }

 // 删除所有含有子类的类
 for (Class someClass in excluded)
 {
  [classList removeObject:someClass];
 }

 //释放内存
 free(classes);
}

// 首先检测缓存是否有这个实现
NSString *selectorString = NSStringFromSelector(selector);
signature = signatureCache[selectorString];
if (!signature)
{
 //找到方法的实现
 for (Class someClass in classList)
 {
  if ([someClass instancesRespondToSelector:selector])
  {
signature = [someClass instanceMethodSignatureForSelector:selector];
break;
  }
 }
 //缓存以备下次使用
 signatureCache[selectorString] = signature ?: [NSNull null];
}
else if ([signature isKindOfClass:[NSNull class]])
{
 signature = nil;
}
  }
  return signature;
  }
}
- (void)forwardInvocation:(NSInvocation  *)invocation {
 // 让nil来处理这个invocation
 [invocation invokeWithTarget:nil];
 }
@end
```

在原文中作者是这样写的:  [excluded addObject:NSStringFromClass(superclass)];
**这样ClassList中存放的是Class，而excluded中存放的确是String,这样就不能过滤掉不必要的类**。所以，我将其改为了: [excluded addObject:superclass];*不知道作者是不是考虑了其他问题，也可能是由于其大意。*

### 延伸阅读：

1. http://www.jianshu.com/p/8774e192d8db
2. http://www.cocoabuilder.com/archive/cocoa/48930-objc-getclasslist-pointers-and-nsarray.html
3. https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html
