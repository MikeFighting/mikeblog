---
title: 影响正交性的常见因素
date: 2018-11-21 09:45:27
tags: Others
---

<img src="http://blog.unidropper.com/blog/pragmatic_orthogonality.png" width="394" heigt="262" style="display:block;margin:0 auto">

## 什么是正交性?

正交性是从几何学中借鉴过来的，比如上图中的X轴和Y轴，它们就是正交的。这里的X轴和Y轴的发展是完全独立的，X轴的伸展不会影响到其投影到Y轴的内容。从软件开发的角度来看，就是一个方法，类，模块的改动不对另一个方法，类，模块造成影响，那么它们就是正交的。比方说你改了数据库的表结构但是不影响到UI，改了UI层的展示方式不能要求数据库schema跟着变更，那么这二者就是正交的。

## 影响正交性的危害有哪些？

如果缺少正交性，就会严重影响到软件的维护性，这种危害将会随着项目的迭代而越来越严重。比方说你移动了一个方法的位置，可能就会造成严重的bug，因为各个方法相互依赖，你就必须理清楚所有的方法，才可以增加一个小功能。再比如说模块A依赖了模块B，那么模块B的改动就需要模块A重新加载模块B，这样它两者之间其实就缺失了模块的概念了。比如下面这个图：

![dependent_cycle](http://blog.unidropper.com/blog/clean_architecture_dependent_cycle.png)

比如上图中的Interactors，Authorizer和Entities就因为出现了依赖环而导致正交性缺失。这就导致这三个模块的发布必须依赖考虑到和其它两个模块之间的兼容性，任何依赖这三者之一的模块都必须同时兼容其余的两个模块(比如要考虑到其它模块是不是最新版本，因为在老版本中出现了Bug)，如果一个模块出了问题就会很难定位，因为它们之间是一个环形的结构，很难确定是Interactors调用的Entities出了问题，还是其自身出了问题（因为Entities调用Authorizer而Authorizer又会调用Interactor）。这种维护的成本会严重影响到软件的开发效率。曾经在YouTube上看到过一篇演讲给出这样一个结论：

> 程序员写代码的平均时间不超过10%，其它90%的时间在看代码。

起初还不认同，但是随着所做项目越来越大，迭代次数越来越多，项目的年代越来越久远，这种感觉就越强烈。90%的时间用于熟悉所有的业务逻辑，熟悉层层嵌套的if else，熟悉各个方法调用之后产生的副作用。而真正需要加的功能更或许也就几行代码而已。既然正交性这么重要，下面就来聊聊影响正交性的常见因素有哪些。

## 影响正交性的因素有哪些？

### 不必要的属性

属性是我们在保存某个数值以便在某个时刻使用的常用方式，它往往可以保持和所属对象相同的生命周期。但是属性的使用同时却带来了弊端：

> 使用属性的方法会产生副作用，而副作用是让方法之间缺少正交性的关键因素。

考虑如下的方法：

```objc
@interface MFPeopertyShowController ()
@property (nonatomic, assign) CGFloat level;
@end

@implementation MFPeopertyShowController

- (void)viewDidLoad {
    [super viewDidLoad];
    ...
    NSDictionary *fullData = @{
                               ...
                               @"level":@(10.0),
                               @"name":@"NoBody",
                               ...
                             };
    ...
    [self p_a:fullData];
    ...
    [self p_b];
    ...
    [self p_c];
}

- (void)p_a:(NSDictionary *)data {
    self.level = [data[@"level"] floatValue];
    //其它业务逻辑
}

- (void)p_b {
    if (self.level < 2) {
        // ...
    }else if (self.level < 4) {
        // ...
    }else if (self.level < 6) {
        // ...
    }else {
        // ...
    }
    ...
}

- （void）p_c {
    //用到level做了其它的并且赋值
    ...
}
```

这个例子中我们为级别作为属性，在`p_a`中对其赋值，之后调用了`p_b`方法，这个方法中用到了level属性。到这里`p_a`和`p_b`方法就缺少了正交性。我们必须先调用`p_a`，然后再调用`p_b`，并且任何对属性level产生的影响都将影响到`p_b`和`p_c`。再加上很多人会对这个`p_a`这个方法命名极其不规范，导致不熟悉该业务的人在解决bug时发现把`p_c`移到最上面好像可以解决，试了之后发现又引入了新的bug，仔细研究才发现是以为level的值设置错了。然后就他需要全局去搜索这个level，看看都有哪些地方用对它设置了值，最后发现搜索到了N个......。这里举得例子可能不太贴切，但是最终的结果是相同的**属性的使用导致了各个方法之间没有了正交性，每个方法之后的重构或者新增功能都必须考虑到这个属性值。**

怎么做才可以尽量减少属性的使用呢？在这个方法中，我们其实可以让`p_a`方法返回level，然后将它做为参数传入`p_b`，最后`p_c`传入一个level，并且返回一个新的level。这样做有以下几个优点：

1. 各个方法之间的关系很清晰，如果改动了本来的顺序，编译器就直接警告你了。
2. 一个方法不依赖其它方法了，它所依赖的只是一个输入的值。
3. 有一天如果这个方法在另外一个项目中能用，这时，只需要做很少的改动就行了。
4. 在多线程中，如果是属性则需要利用加锁等手段来保证线程安全。但是如果用了局部变量，则就不会出现竞态条件了，也就无需加锁。也就是说我们使用了全局变量让方法变得不可重用了。
5. 利用局部变量往往可以提高程序的性能，因为编译器会将某些局部变量(OC对象除外)直接存储在CPU的寄存器堆中，而不是在内存中（参考CSAPP第3章，第5章）。

但是有时候使用属性是不好避免的，比如我们给一个组件传递了一个数据，想在组件被用户点击的时候将数值传递出去，因为在OC中这是基于Target-Action实现的，除了属性，我们没有办法来保存数据以便在Action的方法中传递，如果这里是一个Block能够捕获主局部变量，我们就可以少写个属性了，其实在Android开发中事件的回调用的是匿名内部类，刚好用的就是这种思想。

### 单例模式的滥用

在一个应用程序中如果某个对象应该是唯一的，那么需要用到单例模式，比如UIApplication对象，每个应用对应一个。可能是因为单例模式实现简单的缘故，导致它很容易被滥用。**比如很多人用单例模式来传值。** 单例传值确实很简单，一个单例能够解决需要将参数层层传递到目标对象的繁琐工作。**它却带来了维护的灾难。** 因为单例严重影响了各个类之间的正交性，页面A正在是用着这个值，然后跳转到页面B，页面B改了之后页面A的值就变了。试想下，如果这个是单例是公司级的工具，每个业务线都在用，你根本看不到其它业务线的代码，独立测试顺利通过了，集成之后出现Bug(集成之后代码的测试程度往往会小于独立测试的强度，结果导致线上Bug)。除此之外单例更容易出现线程安全的问题，我就曾经见到过因为单例的非线程安全而造成难以排查的Crash。

> 单例模式不是用来传值的，用单例传值往往会造成维护的灾难。

### 违背最小知道原则

最小知道原则告诉我们：一个类对其它类知道的越少越好。还用一个中比较有趣说法是：**编写害羞的代码**，让一个类暴露的越少越好。这样两个类之间就越正交，一个类的变动对另一个类的影响也就最少。比如下面这个例子：

```objc
- (void)processDate(NSDate aData, MFSelection aSelection) {
    TimeZone tz = aSelection.getRecorder().getLocation().getTimeZone();
    ...
}
```

在这个方法中我们需要的是一个TimeZone的对象，然而我们需要层层寻找，在这个过程中我们不经意间依赖了Recorder和Location这两个本来没有必要依赖的类，忽然有一天发现从Recoder中获取时区的方式会有问题，那么我们就需要查找到整个项目改动所有的方法。应该怎样解决呢？给`MFSelection`添加一个`getLocationTimeZone`的方法，将上文获取时区的方法放到里面即可。这样`processDate`所在的类就只知道了一个`MFSelection`类，而不知道其内部的其它类。再举个例子：

```objc
@interface MFPerson : NSObject
@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@end
```

这是一个Person类，它有一个firstName和lastName，这时服务端返回的数据，可是有一个场景需要展示fullName。大多数人会在调用Person类的地方自己做个字符拼接来得到所需要的fullName。后来这种场景越来越多，你就拼接的地方也会越来越多。再后来用户体验师发现fullName的展示可以优化下，在firstName和lastName中间加上一个特殊符号会更好。这时需要改的地方就会很多。在这里，Person类是无需外界知道其fullName的拼接过程的，所以我们应该给Person添加一个属性：

```objc
@property (nonatomic, copy) NSString *fullName;
```

其实这里还有一点需要注意，如果没有对Person属性进行写的需求，要将其变成readonly，这样它就更好的保证了自身的封装性。

### 滥用继承

继承是很多人用来实现复用的手段，但它会严重影响到程序的正交性。在继承中，子类在开发新功能时要考虑到父类的代码逻辑，父类变动更会影响到很多子类。除此之外因为父类往往会加一些模板方法，而模板方法的逻辑在父类中。这就导致新人在熟悉代码的时候要将父类也熟悉一遍。父类的方法调用依赖子类的实现，子类又天生的依赖了父类，这就导致了环形的依赖，容易产生难以排查的Bug：子类调用了方法A，但是莫名其妙得又触发了方法N，调试了很久就才发现是父类的方法A调用了B，B又调用了C...最后调用到了N。所以能不用继承的时候就尽量不用继承，改用组合。

## 小结

为了让写的代码保持正交性，就要尽力避免和其它方法或者类持有相同的对象，要尽量避免使用继承。同时要满足最小知道原则，减少它暴露的信息。

### 参考资料：

《程序员的修炼之道--从小工到专家》
《深入理解计算机系统》
《Clean Architecture》