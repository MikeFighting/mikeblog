---
title: RunTime应用实例--关于埋点的思考
date: 2016-06-07 11:35:00
tags: Objective-C
---

埋点是现在很多App中都需要用到的，这个问题可能每个人都能处理，但是怎样来减少埋点所带来的侵入性，怎样用更加简洁的方式来处理埋点问题，怎样减少误埋，如果上线了发现少埋了怎么办？下面是本文讨论的重点（[本文Demo已上传GitHub，可以下载讨论](https://github.com/MikeFighting/LogByRunTime)）:

* 什么是埋点？埋点的作用是什么？
* 常规的处理方式是怎样的？
* 我们可以怎样优化？
* 怎样使用RunTime对其进行优化？
* 在实践中遇到了什么问题以及解决方案？
* 最理想的埋点是什么样的？
* 其中可能存在的问题是什么？

接下来将对其一一做以说明:

## 什么是埋点？埋点的作用是什么？

其实埋点也叫日志上报，其实就是根据需求上报一系列关于用户行为的数据，比如：用户点击了哪个按钮，用户浏览了哪个网站，用户在某个页面停留了多久等数据。这些数据对于运营来说很有用，他们可以用来分析某个功能开发的是不是合理，是不是因为某个地方的不合理而到导致了转化率的下降，从而对我们的App进行相应的改进，我们来看下某个第三方平台提供的埋点实例。
![埋点统计字段定义](http://upload-images.jianshu.io/upload_images/1513759-01e20264d3596c3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中说明了，某个时间对应的事件ID,以及针对这个事件需要关联的字段。下面是后台系统对某个埋点所做的数据统计:
![后台系统对埋点的数据分析](http://upload-images.jianshu.io/upload_images/1513759-4630c1435e321dbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样我们就可以详细的分析出用户对于App的反馈，从而及时的修改我们的产品。

## 常规的埋点的做法是怎样的？

其实很简单，我们就在相应的事件里面加入相关的代码，给服务器上报数据不就得了。如下所示:

```objc
// 这个一个按钮的响应事件
- (void)someButtonAction:(UIButton *)someButton{
// 该按钮需要处理的业务
[self upDateSomthing]
// 开始埋点
// eid:事件id，sa:用户id, cI:当前时间
NSDictionary *upLoadDic = @{@"eid":@"311",@"sa":@"706976487532177",@"cI":@"2016-6-4 12:11:34"};
[ZHUpLoadManager upLoadWithDic:upLoadDic];
}
```

这样一个埋点问题就解决了，单同时却隐藏着很多问题:1.这样每点击一个一下按钮就请求一次网络会不会出现性能问题？2.如果这样频繁的数据上报会不会消耗更多的用户流量？3.这样的代码能经受住需求的变更吗？比如字段变了，或者你把`cI`看错了，应该是`cl`。4.这样的代码会不会造成难以测试？5.这样的频繁上报会不会增加服务器端的压力？6.代码整洁吗？......(**程序员的一个好习惯是:这个代码能否经受住需求的变更。**)

## 我们可以怎样优化？

1. 首先我们可以用一个类，来专门处理这些需要上报的埋点的字段，将这些字段作为常量,例如:

```objc
// LogManager.h
extern NSString * const kLogEventKey;   //事件id
extern NSString * const kLogUserIdKey;  //用户id
extern NSString * const kLogOperationInterval;  //操作时间
// LogManager.m
NSString * const kLogEventKey   = @"co"; //事件id
NSString * const kLogUserIdKey  = @"sa"; //用户id
NSString * const kLogOperationInterval  = @"cq"; //操作时间
```

2. 对于用户id，当前时间，用户手机型号，手机品牌，等等与用户所在页面无关的内容，可以用统一的一个类进行处理，将其作为这个类的一个属性，使用`getter`方法将其相应的数值返回即可(对于恒定不变的可以使用懒加载)。
3. 这样的数据传输策略是有问题的，每次点击都上报，可能一个面需要上报的地方很多，这就会造成很大的性能问题，我们可以**先将需要上传的数据缓存起来，然后缓存够50条数据上报一次，或者每隔5分钟上报一次**;
4. 为了节省流量我们可以，1）将数据压缩之后再上报,[可以参考我的另一篇文章](http://www.jianshu.com/p/7016ffdbe97d)；2）和服务端商量，用尽可能短的字段，如:`cityName = @"北京";`变为`cn = @"北京";`3)尽量不要上传的频率过高，如第三点。
5. 如何解决代码的整洁，易于测试的问题？请看下面。


## 怎样使用RunTime来进行优化？

我么能不能利用RunTime来给每一个Button的响应事件中添加一段代码，利用这段代码来进行埋点上报呢？或者进一步来说我们能不能给所有继承自UIControl的对象都添加这样一段代码呢？这样我们不是可以捕获所有的用户事件了吗？(其实答案是否定的，看第五条);这时我们可以利用Mehod Swizzle,或者叫`方法注入`,或者叫`hook`住了某个方法，听着挺玄乎，其实就是RunTime的一个API,这个API能够交换两个方法的实现。通过这个API,我们可以这样实现方法注入。如下图所示:
![方法注入的实现过程](http://upload-images.jianshu.io/upload_images/1513759-4e30c9b337c4c891.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
那么我们点击按钮系统会不会给每个按钮都执行一个统一的方法？然后我们往这个方法中嵌入响应的代码片段就可以了。答案是肯定的。我们可以往

```objc
- (void)sendAction:(SEL)action to:(nullable id)target forEvent:(nullable UIEvent *)event;
```

这个方法里面嵌入相应的代码片段。我们可以这样:1.将互换方法实现的的这个方法放到一个工具类中，因为我们可能不止一处要用到这种方法。2.我们给UIControl添加一个Category,然后在里面调用这个工具类然后实现所插入的代码片段。这里我们既然可以得到`target`还有`action`,那么很多情况下我们就可以唯一确定这个埋点了，那么我们怎样从这么多的埋点中选出这个这个埋点呢？**我们其实可以用字典和数组结合的方式将这些方法的target和方法的参数一一存起来，然后在嵌入的方法内部获取其对应的方法，以及其相应的，这个事先配置好的字典和数组的结合放在哪里比较合适呢？plist。**下面就以最简单的形式展示这种思路:

```objc
// 工具类
@interface ZHSwizzleTool : NSObject
+ (void)zhSwizzleWithClass:(Class)processedClass originalSelector:(SEL)originSelector swizzleSelector:(SEL)swizzlSelector;
@end

@implementation ZHSwizzleTool
 +(void)zhSwizzleWithClass:(Class)processedClass originalSelector:(SEL)originSelector swizzleSelector:(SEL)swizzlSelector{

    Method originMethod = class_getInstanceMethod(processedClass, originSelector);
    Method swizzleMethod = class_getInstanceMethod(processedClass, swizzlSelector);
    BOOL didAddMethod = class_addMethod(processedClass, originSelector, method_getImplementation(swizzleMethod), method_getTypeEncoding(swizzleMethod));
    if (didAddMethod) {
    class_replaceMethod(processedClass, swizzlSelector, method_getImplementation(originMethod), method_getTypeEncoding(originMethod));
    }else{
        method_exchangeImplementations(originMethod, swizzleMethod);
    }
}
@end

// 分类
@implementation UIControl (ZHSwizzle)
+(void)load{

static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{

SEL originSEL = @selector(sendAction:to:forEvent:);
SEL swizzleSEL = @selector(sendSwizzleAction:to:forEvent:);

[ZHSwizzleTool zhSwizzleWithClass:[self class]originalSelector:originSEL swizzleSelector:swizzleSEL];

});
}
 - (void)sendSwizzleAction:(SEL)action to:(id)target   forEvent:(UIEvent *)event{

// 注意这里调用的是原来的系统方法
[self sendSwizzleAction:action to:target forEvent:event];

NSString *selectorName = NSStringFromSelector(action);

// 这个plist中存储的数据格式是这样的:@{@"someViewController":@"selector0":@[para0,para1,para2],@"selector1":@[para0,para1]]};

NSString *pathString = [[NSBundle mainBundle]pathForResource:@"ZHLogInfo" ofType:@"plist"];
NSDictionary *plistDic = [NSDictionary dictionaryWithContentsOfFile:pathString];

//1. 获取Target的名字
NSDictionary *controllerDic = plistDic[NSStringFromClass([target class])];

//2. 获取这个方法对应的参数列表
NSArray *parameterArray = controllerDic[selectorName];
//3. 实例化数据中心
ZHLogDataCenter *logCenter = [[ZHLogDataCenter alloc]init];
NSMutableDictionary *logInfoDic = [NSMutableDictionary dictionary];

for (NSString *parameter in parameterArray) {

NSString *getSelector = [NSString stringWithFormat:@"%@",parameter];
SEL getSeletor = NSSelectorFromString(getSelector);
//4. 从数据中心中获取相应的数据
id value =  [logCenter performSelector:getSeletor withObject:nil];
//5.获取成功则将其存入需要上传的字典
if (value)
[logInfoDic setObject:value forKey:parameter];

}
   //6.将这个字典存入埋点管理类，其会将其存入缓存并等待上传
[ZHLogCenter zhLogWithInforDictionary:logInfoDic];

}
@end
```

 下面是这个代码中用到的Plist中的配置:
![埋点相关字段的plist配置](http://upload-images.jianshu.io/upload_images/1513759-3c28e14477b58263.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 在实践中遇到了什么问题以及解决方案？

1. 并不是所有的事件都是有继承自UIControl的控件来发出的，比如：手势，点击Cell。
2. 并不是所有的按钮点击了之后就立马需要埋点上传？可能在按钮的响应方法中经过了层层的`if(){  } else{  }`最后才需要埋点。
3. 和事件所在类无关的埋点数据可以同意从`ZHLogDataCenter`这个类中中取，那么如果这个数据是和所在类有关呢？
4. 对于代理方法该怎样处理？
5. 如果很多个按钮对应着一个事件该怎样处理？
6. 项目中事件的处理方法不尽相同，方法的参数个数不一样，并且方法的返回值也不一样，如何对他们进行统一的处理?

下面我们来一一解决这些问题。

**问题1**：对于不是来自UIControl的子类发出的事件，我们一样是可以进行hooK，只不过方法有所不同。我们在UIControl的分类中写了一段嵌入的代码，确实hook住了系统UIButton的点击事件，是**因为UIButton自身会调用UIControl的这个方法。但是对于点击事件，这个是我们自己写的一个方法，它的父类`UIViewController`中是没有的，所以在执行我们自己点击事件的方法时UIViewController分类中要嵌入的方法是不会被调用的，这时候怎么办，我们可以动态的给我们自己要hook的ViewController动态的添加一个方法，然后就可以hook了**（这一点不太好理解）。具体的添加方法，可以参考本文的实例代码。

**问题2**：对于是否上传和具体的业务逻辑相关的情况，我们可以用方法所在类的一个属性值进行标记，这个属性写在`.m`文件中即可(KVC可以获取.m文件中的属性值。)，我们先执行要hook那个类的方法，然后根据`plist`中配置的相关标记进行相应的处理（这里的属性值其实也是不必要的，我么可以根据类名和方法名字符串的哈希生成唯一的key，然后利用runtime自动关联到这个类的`mf_condition`属性上，这个属性是一个字典其key就是刚才生成的，value就是运行完这个方法之后得到的值，然后这个值再跟plist中的配置做以比较）。

**问题3**：对于和事件所在类有紧密关联的埋点数据，比如某个页面对应的产品ID,比如某个页面点击了cell，之后这个cell对应的model的ID。这个时候我们可以参考方法2，添加一个属性，用一个属性值来存储这些这些需要上传的具体数据。

**问题4**：代理方法和手势的处理也是一样的，既然一个类实现了某个代理方法，那么其`[someInstance respondsToSelector:someSelector]`所返回的BOOL值应该是YES的，然后其它的就和手势的处理是一样的了。

**问题5**：对于很多按钮对应一个响应事件的情况，我们可以利用RunTime动态的给按钮添加一个属性，比如:buttonIdentifier,这样我们就可以在plist中进行相应的配置，以进行相应的埋点处理。

**问题6**：这个问题其实就是hook住所有的方法，然后给他们添加同一个代码段的问题，这时候我们可以使用`Aspects`这个第三方框架：

```objc
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
  withOptions:(AspectOptions)options
   usingBlock:(id)block
error:(NSError **)error {
return aspect_add((id)self, selector, options, block, error);
 }
```

调用这个接口，因为在`UIViewController`的分类中调用这个接口的对象不一样，并且我们根据`plist`中的配置hook的selector不一样，然而最后执行的block却是一样的，这就很好的解决了问题。

## 最理想的埋点是什么样的？

最理想的埋点是动态的，就是PM给我们说需要哪些埋点，然后服务器给我们发一个类似与上文中提到的`plist`一样的文件，或者一个`json`,我们存到本地，如果这些埋点没有更新，我们就从本地中读取相应的文件，做相应的埋点，如果有更新，我们重新从服务器获取最新的需要埋的点，然后进行相应埋点。这样就解决了少埋，或者埋点不恰当，需要添加埋点的问题。

## 其中可能存在的问题是什么？

当然这里面也有其难以处理的问题，比如我们使用了一个第三方控件，这个第三方控件的事件回调不是用delegate实现的，而是用block实现的，并且这个埋点和具体的业务逻辑有关系，那么这种方法就难以处理了。 如果很多事件的逻辑处理放到了block中进行，那么也将造难以处理。