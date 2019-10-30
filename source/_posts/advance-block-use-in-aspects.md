---
title: Aspects框架中Block的使用
date: 2018-07-04 17:41:04
tags: Objective-C
---

## 前言

Block算是OC语言中比较经典的语法，对其讲解的文章也不少，但大多讲的是其实现原理，而没有将其原理应用于实际中。本文将从实践出发，从Aspects框架中分析Block的本质，以及如何从Block中获取方法签名和参数列表。在分析之前我们先来看下Aspects为什么要使用Block来实现。

## Aspects为什么要使用Block？

我们知道在objc中使用MethodSwizzle来实现AOP是很简单的，具体做法可以参考这两篇：[method-swizzling 详解和使用](https://blog.csdn.net/erice_e/article/details/73293905)，[Objective-C的hook方案](https://blog.csdn.net/yiyaaixuexi/article/details/9374411)文中讲解了怎样在方法中注入相应的代码，但是这样做有以下两个问题：

1. 需要为每个被Hook的类新建一个分类。
2. 对于每一个需要Hook的方法，我们都需要重写一个和它参数列表一样的方法。

这些问题就会让AOP变得复杂，并且Hook方法不能统一添加。比如有这样一个需求：一个数组，数组里面配置了每个类需要Hook的N个方法，这些方法中需要注入相同的代码。这时总不能给每个类都添加一个方法，然后在里面新加一段统一的代码吧？怎样做才更优雅呢？更好的方法是将需要注入或新加的方法实现写在Block中，使用这个Block作为上文中提到的被插入的方法，这样就避免创建一个分类文件，同时再对原方法写一个相同参数的方法。但是使用Block又有以下两个问题：

1. 因为MethodSwizzle最终切换的是方法，而我们写的是Block，Block怎样和方法之间产生关联？
2. 如何获取Bolck的参数列表以便给它传递原方法相应的形参？

对于第一问题，Aspects框架是这样做的：

1. 获取Block的方法签名。
2. 调用`class_replaceMethod`替换掉需要被Hook的方法。
3. 调用`class_replaceMethod`替换掉系统

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

4. 在自己的`forwardInvocation:`方法中，利用1中获取的方法签名来生成新的NSInvocation，利用NSInvocation调用Block。

这样问题的关键就在于获取Block的方法签名了。

## 获取Block的Signature

怎样从block中获取其方法签名呢？我们从Block的语法中是得不到任何信息的。只有看看编译器对我们的Block语法做了哪些“手脚”才可以看到，Block编译后的代码在[苹果开源的LLVM镜像中有提及](https://github.com/llvm-mirror/compiler-rt/blob/master/lib/BlocksRuntime/Block_private.h)：

```c
enum {
    BLOCK_REFCOUNT_MASK =     (0xffff),
    BLOCK_NEEDS_FREE =        (1 << 24),
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25),
    BLOCK_HAS_CTOR =          (1 << 26), /* Helpers have C++ code. */
    BLOCK_IS_GC =             (1 << 27),
    BLOCK_IS_GLOBAL =         (1 << 28),
    BLOCK_HAS_DESCRIPTOR =    (1 << 29)
};

/* Revised new layout. */
struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};

struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};
```

从中我们可以看出`Block`其实是一个对象，我们来分析下这个对象：

### void *isa:指向该Block所属类的指针

在Block中，这个指针可能的值为：

```objc
1. _NSConcreteStackBlock
2. _NSConcreteGlobalBlock
3. _NSConcreteMallocBlock
4. _NSConcreteAutoBlock
5. _NSConcreteFinalizingBlock
```

这五种类型的Block都继承自`_NSAbstractBlock`。其具体的类型是根据Block创建的位置决定的，如果Block被定义在一个方法中做为一个局部变量，则其创建出来为`_NSConcreteStackBlock`（如果我们调用`copy`方法将其复制到上时，其类型将变为`_NSConcreteMallocBlock`），如果定义为全局变量，那么它就是`_NSConcreteGlobalBlock`。至于什么场景会产生什么Block对象不在本文的讨论范围，可以从本文的参考资料中获取。

### int flags: Block属性的标识符

我们可以通过该属性来对Block是否含有某种信息做以判断，我们来详细说明这些标识符：

* `BLOCK_REFCOUNT_MASK，BLOCK_NEEDS_FREE，BLOCK_IS_GC`：这三个与引用计数和GC相关的标识符是在运行时，当Block被拷贝时设定的。
* `BLOCK_IS_GLOBAL`：对于全局存储的Block，这个属性是在编译期被设定的，对这种类型的Block执行Copy和Releas是不起作用的，因为它存储在应用程序的数据区。
* `BLOCK_HAS_DESCRIPTOR`：这个标示总是会被设置，因为在Mac OS的Snow Leopard版本前后Block的实现是不同的，为了区分之后的实现，所以总是要被设置。
* `BLOCK_HAS_COPY_DISPOSE`：如果Block捕获了某个变量，那么就需要将实现copy和dispose方法，这时就会设置该位。
* `BLOCK_HAS_CTOR`：如果Block中含有C++的构造解释器，那么就会设置该位。

### void (*invoke)(void *, ...): 函数指针

这个指针就是我们写Block时候的实现，编译器会给我们创建一个C语言的函数，这个函数的指针被赋值给invoke指针。这时我们就可以通过函数调用来调用Block中函数的实现，请看下文的使用函数指针调用Block。

### struct Block_descriptor *descriptor：Block的描述信息

这个结构体里面保存着Block的大小：`size`，Block捕获`__block`类型变量被Copy时调用的`void (*copy)(void *dst, const void *src);`以及被销毁时候调用的`void (*dispose)(void *);`方法。

这些Block的基本知识了解之后，我们看本小节的主旨：如何通过Block获取方法的签名？其实在上面官方提供的文档中看，早期的Block实现中没有提供方法签名，后来在`flags`的枚举中又新加了一个`BLOCK_HAS_SIGNATURE = (1 << 30)`这个枚举项，通过对这个枚举项的判断我们可以确定一个Block中是否含有方法的签名。方法签名的标识有了，那么这个方法签名存放在哪里呢？它存放在`Block_descriptor`结构体中`dispose`下面，我们来看Aspects框架是怎样将Block转换为方法签名的：

```objc
// Block internals.
typedef NS_OPTIONS(int, AspectBlockFlags) {
    AspectBlockFlagsHasCopyDisposeHelpers = (1 << 25),
    AspectBlockFlagsHasSignature          = (1 << 30)
};
typedef struct _AspectBlock {
    __unused Class isa;
    AspectBlockFlags flags;
    __unused int reserved;
void (__unused *invoke)(struct _AspectBlock *block, ...);
struct {
    unsigned long int reserved;
    unsigned long int size;
    // requires AspectBlockFlagsHasCopyDisposeHelpers
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
    // requires AspectBlockFlagsHasSignature
    const char *signature;
    const char *layout;
} *descriptor;
// imported variables
} *AspectBlockRef;

static NSMethodSignature *aspect_blockMethodSignature(id block, NSError **error) {
    AspectBlockRef layout = (__bridge void *)block;
if (!(layout->flags & AspectBlockFlagsHasSignature)) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't contain a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
    void *desc = layout->descriptor;
    desc += 2 * sizeof(unsigned long int);
    if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
    desc += 2 * sizeof(void *);
    }
if (!desc) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't has a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
const char *signature = (*(const char **)desc);
return [NSMethodSignature signatureWithObjCTypes:signature];
}
```

从中我们可以看出首先判断是否含有方法签名，如果没有则直接返回，如果有则通过指针的移动来指向方法签名所在的位置，并调用`[NSMethodSignature signatureWithObjCTypes:signature];`来生成方法签名。

## 获取Block的参数列表

有了方法签名之后获取参数列表是相对简单的，其关键代码如下所示：

```objc
static BOOL aspect_isCompatibleBlockSignature(NSMethodSignature *blockSignature, id object, SEL selector, NSError **error) {
    //...
    BOOL signaturesMatch = YES;
    NSMethodSignature *methodSignature = [[object class] instanceMethodSignatureForSelector:selector];
    if (blockSignature.numberOfArguments > methodSignature.numberOfArguments) {
        signaturesMatch = NO;
    }else {
        if (blockSignature.numberOfArguments > 1) {
            const char *blockType = [blockSignature getArgumentTypeAtIndex:1];
            if (blockType[0] != '@') {
                signaturesMatch = NO;
            }
        }
        // Argument 0 is self/block, argument 1 is SEL or id<AspectInfo>. We start comparing at argument 2.
        // The block can have less arguments than the method, that's ok.
        // 这里获取Block中的参数列表以便和原方法的参数列表进行比较，看其是否匹配
        if (signaturesMatch) {
            for (NSUInteger idx = 2; idx < blockSignature.numberOfArguments; idx++) {
                const char *methodType = [methodSignature getArgumentTypeAtIndex:idx];
                const char *blockType = [blockSignature getArgumentTypeAtIndex:idx];
                // Only compare parameter, not the optional type data.
                if (!methodType || !blockType || methodType[0] != blockType[0]) {
                    signaturesMatch = NO; 
                    break;
                }
            }
        }
    }

    if (!signaturesMatch) {
        // ...
        return NO;
    }
    return YES;
}
```

这里主要做了原始方法和新方法之间参数的比较，如果参数不对应则注入方法之后再重新调用原始方法时会出错，所以这里获取了Block的参数列表。主要用到的是NSMethodSignature的两个方法:

```objc
@property (readonly) NSUInteger numberOfArguments;
- (const char *)getArgumentTypeAtIndex:(NSUInteger)idx NS_RETURNS_INNER_POINTER;
```

## 使用函数指针调用Block

我们可以通过定义自己的Block结构体，然后将Block变量强制转化，之后调用上文中提到的`invoke`函数指针来实现对Block结构体的调用，运行下面的代码：

```objc
struct MyBlockDescriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};

struct MyBlockLayout {

    void *isa;
    int flags;
    int reserved;
    void (* invoke)(void *, ...);
    struct MyBlockDescriptor *descripter;
};

- (void)callBlockByFuncPointer {

    //简单Block的调用
    void (^block)() = ^{
        NSLog(@"Block called");
    };
    struct MyBlockLayout *block1 = (struct MyBlockLayout *)(__bridge void *)block;
    block1->invoke(block1);

    //含有参数的Block调用
    int (^returnBlock)(int, int ) = ^int(int a, int b) {
        return a + b;
    };

    struct MyBlockLayout *block2 = (struct MyBlockLayout *)(__bridge void *)returnBlock;
    //这是将void * 指针转换为 (int (*)(void *, int a, ...))类型的指针
    int result = (  (int (*)(void *, int a, ...)) (block2->invoke)  ) (block2, 3, 4);
    NSLog(@"result == %d",result);
}
```

这里我们将Block强制转换为我们自定义的`MyBlockLayout`，然后将`void *`类型的指针转换为Block相应实现的指针，这时候就可以实现以函数指针的形式调用Block了。

## 小结

本文小结了Block的本质，以及如何利用自定义的Block结构体，通过桥接将系统的Block转化成我们的Block结构体，从而获取其函数指针和方法签名。同时说明了标示block特性的flags，这个标示在使用Block结构体的时候往往很有用。其实FaceBook开源的框架：FBRetainCycleDetector中用到了相似的手段，感兴趣的可以对比看下。

## 参考资料：

Objective-C高级编程
https://stackoverflow.com/questions/13006685/is-there-a-way-to-wrap-an-objectivec-block-into-function-pointer?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa</br>
http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/</br>
Advanced Mac OS X Programming: The Big Nerd Ranch Guide</br>
http://www.galloway.me.uk/2012/10/a-look-inside-blocks-episode-1/ </br>
http://blog.devtang.com/2013/07/28/a-look-inside-blocks/</br>
https://github.com/llvm-mirror/compiler-rt/blob/master/lib/BlocksRuntime/Block_private.h