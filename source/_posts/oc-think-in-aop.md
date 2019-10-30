---
title: AOP实践小结
date: 2018-06-05 15:55:07
tags: Objective-C
---

![AOPLogo](https://upload-images.jianshu.io/upload_images/1513759-ba27b7fbc5747446.png)
Objective-C中利用Method Swizzle实现AOP编程是相对简单的，只需要调用runtime框架的`method_exchangeImplementations`方法，将相应方法的实现进行调换即可。然而在实际的使用中可能会遇到一些问题，本文总结了在项目中使用AOP时遇到的问题，并分析了常用框架中AOP的处理方法。

## 为什么要使用AOP？

需求背景：在某版的开发中Listing页面的改版较大，产品将UI分为A，B，C三个版本，需要根据后续的数据分析来对A，B，C三个版本进行相应的取舍。所以就需要我们在之前Listing页的**所有埋点**中都加入一个版本号。因为Listing经过了很多的迭代，总共统计下来有100+的埋点，很难将每个埋点中都加入一个字段。基于此，采用了面向切面的思想，Hook住公共埋点最终要调用的方法，然后在这个方法的参数中添加一个版本的字段，这种方式可谓“一劳永逸”，只需要在一个一个地方加一段代码就可以解决Listing页面的所有埋点问题。

## 遇到的问题

### 问题一、添加和移除

因为主App涉及到很多的业务线，所有的业务线最终都要调用这个方法来进行埋点。如果我Hook住了这个方法，那么其它业务线的埋点最终也会调用我写的埋点方法，这显然是不好的，同时如果我们的页面从Listing页进入Detail页面就就不需要这个版本号，这时也需要这个Hook移除。也就是说要调用两次`method_exchangeImplementations`，第一次用来添加注入代码，第二用来移除注入的代码。那么接口就变成了这样子：

```objc
// YPDataLog+YPAddition.m
- (void)yp_addAspect {
    SEL orignSelector = //;
    SEL newSelector = //;
    BOOL responsed = [self respondsToSelector:orignSelector];
    NSAssert(responsed = YES, @"The log method has been changed");
    if (responsed) {
        [self p_swizzleWithClass:[self class]
                originalSelector:orignSelector
                 swizzleSelector:newSelector];
    }
}
- (void)yp_removeAspect {
     [self yp_addAspect];
}
```

这时问题就出现了，**因为这个方法是有副作用的**：用户在调用`yp_addAspect`和`yp_removeAspect`时候必须要保证是一一对应。也就是说要调用一个`yp_addAspect`，然后调用`yp_removeAspect`，如果连续调用了两次`yp_addAspect`，再接着调用`yp_removeAspect`那么，就会造成错误，这就给用户使用这个方法带来了麻烦。

### 问题二、非线程安全

比如我们注入的方法是这样的：

```objc
- (void)yp_logPage   :   (NSString *)pagetype
         logAction   :   (NSString *)actionType
            params   :   (NSArray  *)paramtes {

      //步骤1. 添加相应的参数
      NSMutableArray *mutableArray = [NSMutableArray  arrayWithArray:paramtes];
      NSString *version // 获取相应的A，B，C版本号;
      [mutableArray addObject:version ?: @""];

      //步骤2. 调用之前的方法
      [self   yp_logPage:pageType
               logAciton:actionType
                  params:mutableArray];
}
```

在线程A调用这个方法，执行到步骤1的时候，线程B调用了`- (void)yp_removeAspect`，这时方法已经被调换回来了。这时线程A仍然会调用：

```objc
[self    yp_logPage:pageType
          logAciton:actionType
             params:paramtes];
```

这时就发生循环调用，因为在这个循环中会调用步骤一并创建相应的`NSMutableArray`，所有会造成栈溢出并最终崩溃。

## 解决方案

为了解决问题一，我们需要给这个hook添加一个标示，用来标注该方法是否已经被hook，如果已经被hook，那么再调用`yp_addAspect`就直接返回，如果没有调用`yp_addAspect`方法而先调用了`yp_removeAspect`方法，我们直接使用断言提醒用户就可以达到相应的目的。</br>

对于问题二，由于调用我们没有办法限定调用`yp_logPage:logAciton:params`是在主线程中还是在子线程中，所以**要保证该方法的调用和`yp_removeAspect`之间的互斥该怎么做到呢？。**</br>

### 方案一：加锁

为了保证互斥加锁不就行了?加锁只能保证`yp_removeAspect`和`yp_logPage:logAction:params:`中的**执行**是互斥的，这同样是非线程安全的。考虑下面的执行路径：

1. 如果线程A调用了`yp_removeAspect`，同时线程B进入了`yp_logPage:logAction:params:`
2. 因为线程A持有锁，所以线程B被阻塞
3. 线程A执行完removeAspect，线程B获取锁被唤醒
4. 线程B调用`yp_logPage:logAction:params:`,进入死循环

所以在这两个方法上加锁解决不了非线程安全问题。

### 方案二：全局标识

利用标识是否可以解决呢？如果在调用`yp_logPage:logAction:params:`中我们发现swizzle切换了，那么就调用该类原来的方法`logPage:logAction:params:`。来看看伪代码：

```objc
- (void)yp_removeAspect {

    SEL orignSelector = //;
    SEL newSelector = //;
    BOOL responsed = [self respondsToSelector:orignSelector];
    NSAssert(responsed = YES, @"The log method has been changed");
    if (responsed) {
        [self p_swizzleWithClass:[self class]
                originalSelector:orignSelector
                 swizzleSelector:newSelector];
        self.isHooked = NO;
    }
}

- (void)yp_logPage   :   (NSString *)pagetype
         logAction   :   (NSString *)actionType
            params   :   (NSArray  *)paramtes {

      //步骤1. 添加相应的参数
      // ...
      //步骤2. 调用之前的方法
      if(self.isHooked) {
      [self   yp_logPage:pageType
               logAciton:actionType
                  params:paramtes];
      }else{
         [self   logPage:pagetType
               logAciton:actionType
                  params:paramtes];
      }
}
```

在多线程之间做标示时，要注意：

>这个标示要被声明成volatile类型的，以确保其在各个线程之间是可见的。

这时仍然是非线程安全的，因为比如线程A执行完了swizzle之后时间片刚好到了，被操作系统换出，然后线程B执行`yp_logPage:logAction:params`的时候，if语句仍然是成立的。

### 方案三：方案一二结合

利用加锁和标示结合的方式可以解决问题，在方案一加锁的同时，在内部再利用标示进行判断。这种方式可以解决问题，但是开销太大，我们知道埋点方法的调用是很频繁的，这种频繁的加锁解锁会造成很大的上下文切换开销，同时绝大部分的埋点方法调用都是在主线程所以没有必要加锁解锁。同时，这种在直接在接口上加锁的方式太简单粗暴了，粒度太大。

### 方案四：GCD和标示

既然百分之九十以上的埋点都是在主线程中调用的，我们可以调用利用让在子线程的方法切换到主线程就行了，同时在内部加标示判断即可，这样既防止了开销，又保证了线程安全。

```objc
- (void)yp_logPage   :   (NSString *)pagetype
         logAction   :   (NSString *)actionType
            params   :   (NSArray  *)paramtes {

dispatch_async(dispatch_get_main_queue(), ^{
      //....
      if(self.isHooked){
      //...
      }else{
      //...
      }
    }
  );
}
```

我在2.7GHz，i5处理器的的Mac Pro上的6s模拟器上模拟了10000条多线程的日志输出，发现用Lock的形式和切换到主线程的形式，用时分别是：4.494608s和3.382740s，可以看出使用GCD进行切换的形式性能更优。这里模拟的每条日志都是在GCD的线程池中抽取的线程中执行的，而项目中的实际埋点大多都是在主线程中调用的，所以性能的提高会更高。这里要注意：

>虽然埋点方法的调用是在主线程中的，但是最终将埋点写入文件（等到了时间阈值统一上传，以减少网络IO和流量损耗）时应该在子线程中，因为磁盘IO造成的性能损耗是很大的。

## 常用框架的处理

下面们说说常见的框架是如何来进行AOP的。

### DZNEmptyDataSet中AOP的实现

DZNEmptyDataSet可以说是做空白页的鼻祖，它hook的是tableView和collectionView的reloadData方法，然后在这个方法内部去判断是否没有数据，如果没有就展示相应的空白页面。它其实没有调用`method_exchangeImplementations`方法，而是先将原来的方法的实现替换掉：

```objc
 // Swizzle by injecting additional implementation
    Method method = class_getInstanceMethod(baseClass, selector);
    IMP dzn_newImplementation = method_setImplementation(method, (IMP)dzn_original_implementation);

    // Store the new implementation in the lookup table
    NSDictionary *swizzledInfo = @{DZNSwizzleInfoOwnerKey: baseClass,
                                   DZNSwizzleInfoSelectorKey: NSStringFromSelector(selector),
                                   DZNSwizzleInfoPointerKey: [NSValue valueWithPointer:dzn_newImplementation]};
    [_impLookupTable setObject:swizzledInfo forKey:key];
```

这里在调用`method_setImplementation`是方法原来的实现（这也就决定了这个方法是不可重入的，因为再次调用，他就将返回之前注入的实现），它会将这个方法原来的实现的指针`dzn_newImplementation`以NSValue的形式存放到`_impLookupTable`这个字典中，在然后在新注入的方法执行完之后，以函数指针的形式调用原来的方法：

```objc
void dzn_original_implementation(id self, SEL _cmd)
{
    // Fetch original implementation from lookup table
    Class baseClass = dzn_baseClassToSwizzleForTarget(self);
    NSString *key = dzn_implementationKey(baseClass, _cmd);
    NSDictionary *swizzleInfo = [_impLookupTable objectForKey:key];
    NSValue *impValue = [swizzleInfo valueForKey:DZNSwizzleInfoPointerKey];

    IMP impPointer = [impValue pointerValue];

    // We then inject the additional implementation for reloading the empty dataset
    // Doing it before calling the original implementation does update the 'isEmptyDataSetVisible' flag on time.
    [self dzn_reloadEmptyDataSet];
    // If found, call original implementation
    if (impPointer) {
        ((void(*)(id,SEL))impPointer)(self,_cmd);
    }
}
```

在if语句中就是利用`((void(*)(id,SEL))impPointer)(self,_cmd);`这个函数指针的形式调用回了原来的函数。从中可以看出，它其实也是利用字典来对**这些不可重入的方法做以限制**。

### Aspects框架的实现

Aspects框架可以说是iOS中实现AOP的经典框架了。因为它要Hook住所有用户想要Hook住除了下面方法之外的所有方法：

```objc
retain
release
autorelease
forwardInvocation:
```

所以它采用了一种非常巧妙的方式：

1. 调用`class_replaceMethod`替换掉原来需要被Hook的方法。
2. 调用`class_replaceMethod`替换掉系统的`- (void)forwardInvocation:(NSInvocation *)anInvocation`方法。

因为我们在第一步替换掉了原来的方法，所以在runtime的时候系统会发现找不到原来的方法，这时系统会自动调用`forwardInvocation`这个消息转发的方法，因为它刚好被替换了，所以，无论你调用任何方法，到最后都会被Hook在`forwardInvocation`方法中，这也就解决了Hook住所有方法的目的，也正是Aspects框架的精巧所在，关于Aspects框架中Block的详细讲解请看在[Aspects框架中Block的使用](https://mikefighting.github.io/2018/07/04/advance-block-use-in-aspects/)中的说明。

### 为什么不能直接利用Aspects框架？

问题在于我们的埋点最后传参的数组是NSArray，而不是NSMutableArray，这也就导致没有办法给它添加参数，为了说明白这一点，我们看一个例子：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    NSArray *someArray = @[@"A",@"B",@"C"];
    [self p_addPamas:someArray];
    NSLog(@"someArray:%@",someArray);
}

- (void)p_addPamas:(NSArray*)params {
    NSMutableArray *resultParams = [NSMutableArray arrayWithArray:params];
    [resultParams addObject:@"E"];
    [resultParams addObject:@"F"];
    params = resultParams;
}
```

可能我们会采用这种方式来添加`E`，`F`来给NSArray添加两个元素，但是结果输出的却是：

```objc
someArray:(
    A,
    B,
    C
)
```

这是为什么呢？因为我们`p_addPamas:`的入参是一个指针，对这个指针形参有如下的性质：

>改变形参的数值本身不会对实参造成影响，然而如果我们想改变实参，那么可以改变形参指针所指的内容，而不是指针本身。

也就是说如果我们传入的pamas的数值是：`0x600000244fb0`，那么改变这个值是不能改变实参`someArray`的，除非我们改变了`0x600000244fb0`所指的内容，而此时这个NSArray又是不可变数组，所以不能往里面添加元素。试想下，如果是`NSMutableArray`，那么问题将会变得简单很多：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    NSMutableArray *someArray = [NSMutableArray arrayWithArray:@[@"A",@"B",@"C"]];
    [self p_addPamas:someArray];
    NSLog(@"someArray:%@",someArray);
}

- (void)p_addPamas:(NSMutableArray*)params {
    [params addObject:@"E"];
    [params addObject:@"F"];
}
```

这时我们只需要改变指针所指的内容就可以了。输出的结果是：

```objc
someArray:(
    A,
    B,
    C,
    E,
    F
)
```

所以直接使用Aspects框架不能给NSArray的形参中添加元素，因此需要自己写方法进行替换。

## 其它

如果在版本迭代中这种AB测的埋点很多，粒度很细怎么办？比如某个组件需要加AB测，可能涉及到两三个埋点。这时可以将将相应的埋点字段放到一个Array中，然后在注入的方法判断Array中是否有该埋点，如果有就添加版本，如果没有就不添加。

## 小结

在Objective-C中使用Method Swizzle实现AOP时一定要注意多线程的问题，同时要保证该方法的执行顺序（因为这个方法本身就是带有副作用的）否则就可能出现难以排查的bug。使用`method_setImplementation`和`class_replaceMethod`同样可以达到方法调换的效果。Aspects框架使用替换原方法和替换`forwardInvocation:`方法巧妙得达到了Hook所有方法的目的。最后，试图改变形参的指针本身是不起作用的，然而可以改变指针所指的内存空间。
