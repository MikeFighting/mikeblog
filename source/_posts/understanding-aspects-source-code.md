---
title: IOS中AOP框架Aspects源码分析
date: 2017-04-07 16:38:00
tags: FrameWork
---

## iOS中AOP框架Aspects源码分析

AOP是Aspect Oriented Programming的缩写，意思就是面向切面编程。具体的解释可以到[维基百科](https://en.wikipedia.org/wiki/Aspect-oriented_programming)上或者其它地方查看。在IOS中使用Swizzle技术可以实现面向切面编程，我在[RunTime应用实例--关于埋点的思考](http://www.jianshu.com/p/69859d580354)博文中也提到了Aspects框架，下面就来对该框架做以分析。

## 一般的Swizzle是怎么实现的？

在RunTime应用实例--关于埋点的思考，也讲到了MethodSwizzle的技术，下面来看最常见的实现方案。比如我们要Hook住UIButton的`sendAction:to:forEvent:`，这时候我们一般这样做:

```objc
+(void)load{
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    SEL originSEL = @selector(sendAction:to:forEvent:);
    SEL swizzleSEL = @selector(swizzleSendAction:to:forEvent:);
    Class processedClass = [self class];
    Method originMethod = class_getInstanceMethod(processedClass, originSEL);
    Method swizzleMethod = class_getInstanceMethod(processedClass, swizzleSEL);
    method_exchangeImplementations(originMethod, swizzleMethod);
  });
}
- (void)swizzleSendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event{
// 执行相应的逻辑
// 调用原来的系统方法
[self swizzleSendAction:action to:target forEvent:event];
}
```

具体的解释可以参见**RunTime应用实例--关于埋点的思考**这篇文章。

## 这样做有什么弊端？

* 我们要给每一个要Hook的方法额外新加一个方法，方法的参数个数是一样的。
* 如果每个被Hook的方法内部的实现逻辑都一样，那么就需要在每个新添加的方法中调用这段实现逻辑。
* 我们新加的代码是在被Hook方法之前还是之后调用的呢？
* 我们能放将这个新加的方法转化为一个Block呢？这样代码会更紧凑，逻辑更清晰。
* 如果这个方法转化为Block，那么如何将这个Block替换掉原来的方法实现Swizzle呢？怎样在合适的时候调用这个block呢？

带着这些问题我们来看Aspects是如何实现的。

## 主要的实现思路

1. 使用和原方法相同参数不同方法名的方法，替换被hook的方法，这样系统在找不到这个方法的时候就会走到`forwardInvocation:`。
2. 使用`__ASPECTS_ARE_BEING_CALLED__`替换掉系统的`forwardInvocation:`。 
3. 给类增加`AspectsForwardInvocationSelectorName`方法，它的实现是原来的`forwardInvocation:`的IMP。
4. 当要hook的方法被调用时，系统会调用`forwardInvocation:`方法。由于这个方法也被替换掉了，所以会调用`__ASPECTS_ARE_BEING_CALLED__`。
5. `__ASPECTS_ARE_BEING_CALLED__`内部，先调用被hook方法之前的block,再调用替换被hook方法的block，以及没有替换的实现，最后调用被hook方法之之后的block。
6. 如果hook出错，则再调用原来的`AspectsForwardInvocationSelectorName`的方法。

## `Aspects.m`包含的内部类及分类

* **AspectInfo**:存储被hook方法的信息。
* **AspectIdentifier**:记录每一次Aspect的信息。
* **AspectsContainer**:某个类或者某个对象所有被hook方法的集合。
* **AspectTracker**:对所有hook方法的操作（增加或者减少）。
* **NSInvocation (Aspects)**:获取NSInvocation的参数。
* **NSObject (Aspects)**:框架的主要分类，定义公共接口。

各类之间的关系如图：
![Aspect框架各类之间的结构](http://upload-images.jianshu.io/upload_images/1513759-4268aad7e1e7cd89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/510)

## 执行流程

公共方法主要调用：`aspect_add`方法，该方法内部主要调用三个方法

1. `AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);`获取该方法对应的`AspectsContainer`。
* 将要hook的方法转化为aliasSelector。
* 取得关联的AspectsContainer，如果没有，怎设置关联的AspectsContainer；
2. `identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];`,将调用的参数封装成AspectIdentifier。

* 通过：`aspect_blockMethodSignature`取的block对应的`NSMethodSignature`,
* 创建`AspectIdentifier`并且返回

3. `aspect_prepareClassAndHookSelector(self, selector, error);`，准备工作及hook方法。

* aspect_hookClass，内部调用`aspect_swizzleForwardInvocation`，将系统的`forwardInvocation:` 替换为`__ASPECTS_ARE_BEING_CALLED__`。
* 在改方法内部分别调用被hook方法之前，替换被hook方法，被hook方法之后的方法。 

```objc
   // Before hooks.  
  aspect_invoke(classContainer.beforeAspects, info);
  aspect_invoke(objectContainer.beforeAspects, info);
  // Instead hooks.
  BOOL respondsToAlias = YES;
  if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
  aspect_invoke(classContainer.insteadAspects, info);
  aspect_invoke(objectContainer.insteadAspects, info);
  }else {
  Class klass = object_getClass(invocation.target);
  do {
  if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
    [invocation invoke];
    break;
  }
  }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
  }
  // After hooks.
  aspect_invoke(classContainer.afterAspects, info);
  aspect_invoke(objectContainer.afterAspects, info);
```

从中可以看出Aspect重要使用了NSInvocation来避免了不同参数个数的限制，通过将block转换为NSMethodSignature对象，可以实现对原有方法的替换或者Swizzle，然后调用其`invoke`方法，可以在适当的时机触发这个block，通过标识将block分为，原方法之前，之后，替换原方法等做法。而不必像一般实现方法那样，改变每个新加方法中调用原来方法的位置来实现。

## 其它Tips

1. 如果某个类只可能被其它一个类用到，那么可以将它们写到一个`.m`文件中，这是高内聚的一种表现。
2. 类，包括分类的出现，其实是为了更好的组织代码，让代码的功能更清晰易懂。
3. 将一段代码加锁的方式：可以将代码块作为参数，然后对其执行前后加锁，这样做的好处是：如果以后要换锁的类型，那么只要在这个方法中改变就可以了：例如：

```objc
static void aspect_performLocked(dispatch_block_t block) {
static OSSpinLock aspect_lock = OS_SPINLOCK_INIT;
OSSpinLockLock(&aspect_lock);
block();
OSSpinLockUnlock(&aspect_lock);
}
```

4. 可以通过自定义结构体的方式将block转换成NSMethodSignature对象，然后在适当的时候调用。

```objc
   - (void)invoke;
   - (void)invokeWithTarget:(id)target;
```

 这两个方法来触发这个block。具体做法请见：AspectBlockRef这个结构体和`aspect_blockMethodSignature`这个方法。**这里有一个疑问：**根据[苹果官方提供的block定义](https://llvm.org/svn/llvm-project/compiler-rt/tags/Apple/Libcompiler_rt-10/BlocksRuntime/Block_private.h)，其中是没有signature这个字段的:

```objc
/* Revised new layout. */
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
 };
```

 但是Aspect中的block转化为响应的结构体：`AspectBlockRef   layout = (__bridge void *)block;`之后就会自动有signature字段，这点不是很理解还望大神指教。
5.  给定一个对象，和这个对象的SEL，如果参数非常多，使用`performSelector:withObject:`这种方式不太合适，这时可以借鉴YYKit中的`NSObject+YYAdd`里面的方法，如果这时有NSInvocation对象，那么可以直接调用`objc_msgSend(self, SEL, invocation)`这种方式来实现。