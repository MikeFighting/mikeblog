---
title: 怎样用好Value Type?
date: 2017-08-02 16:37:56
tags: WWDC
---

# Swift--怎样用好Value Type?

![Value Type](http://upload-images.jianshu.io/upload_images/1513759-6b88775e60bbef0f.png)
## 为什么要用Value Type?

首先我们要说明为什么我们需要`Value Type`。因为我们通常使用的`Reference Type`不能满足我们的需求，并且容易出Bug，什么是`Reference Type`?
Reference Type就是我们常说的Class，我们不是用的很好吗？我们举两个例子来看看
例一：但是现在又一种需求：进入用于信息页面编辑页面，如果用户的信息改变了，并且没有点击保存，这时如果点击导航栏的退出，要给用户提醒"您编辑了信息，确定不保存退出吗？"，这时为了保存进入编辑页面的信息，我们一般这样做

1. 增加一个originInfo的property。
2. 在ViewDidLoad的时候赋值：self.originInfo = passedInfo;（这个passedInfo从上个页面传来）
3. 在点击退出的时候对比passedInfo和self.originInfo是否相等，如果不相等则提醒用户，如果相等则返回。
这时你会发现一个bug，这两个对象永远都是相同的，为什么？

例二：我们要从商品列表中进入商品编辑页面，这时，我们点击了编辑，将`goodModel`传到第二个页面，在第二个页面操作完之后，用户没有保存信息，返回了，这时你也可能会发现，你列表中的商品信息改变了。

例三：Cell复用造成的布局混乱。
上述两种情景是`Reference Type`，也就是说他们的都指向了堆上的相同对象，这种类型对对象的公用造成了一系列的bug。并且这种bug很难被发现，并且往往也不是必现的。
怎样解决呢？
这时我们就需要Copy原来的instance，在OC中我们需要遵守NSCopying，或者NSMutableCopying协议，因为这些，然后实现相应的协议，例如：

``` Objective-C
@interface HYLocationModel : NSObject<NSMutableCopying>
// cityDic{@"name":@"",@"id":@""}
@property (nonatomic, strong) NSDictionary *cityDic; //City
@property (nonatomic, strong) NSDictionary *areaDic; //
@property (nonatomic, strong) NSDictionary *districtDic;

@end
@implementation HYLocationModel
- (id)mutableCopyWithZone:(NSZone *)zone {
    
    HYLocationModel *locationModel = [[HYLocationModel alloc]init];
    locationModel.cityDic = self.cityDic;
    locationModel.areaDic = self.areaDic;
    locationModel.districtDic = self.districtDic;
    return locationModel;
}
@end
```  

这样我们就可以Copy对象了，将Copy的对象赋值给self.originInfo，就可以解决上述的bug。
但是这会消耗性能，因为需要在堆里开辟内存空间。NSCopying协议在OC中很常见，比如NSString，NSArray，NSDictionary等都遵守NSCopying协议。其中NSDictionary的Key，默认是实现了NSCopying协议的，因为在给NSDictionary赋值的时候，系统默认是Copy了它的Key，因为如果不Copy它的Key，如果你给字典赋值之后改变了这个Key，那么它将会使整个NSDictionary混乱，出现意想不到的Bug。当然这种Copy也会消耗性能。

## 不可变对象是否可以解决上述问题呢？

在函数式编程中，我们会使用不可变的`Reference Type`来消除其可变所带来的问题，想象下如果你做数学题题目A中的X值被题目B改变了，那么会有怎样的结果？
在Swift中，我们可以使用let来使其不可变，但是这种不可变的数据结构有以下弊端：

1. 可能导致很恶心的接口（见下文）。
2. 不能有效地和机器模型相匹配(因为我们的寄存器，我们的Cache，Memory，Storage都是可变状态的)。

比如下面的代码

```Swift
// With mutability 
home.oven.temperature.fahrenheit += 10.0

//Without mutability  let temp = home.oven.temperature home.oven.temperature = Temperature(fahrenheit: temp.fahrenheit + 10.0) 
```  

在上面的例子中，我们把Temperature类的某个属性改成了`let`，那么如果我们要更改这个数值，我们就需要在堆上开辟内存空间然后创建一个新的Temperature，最后更换掉整个Temperature类。
Cocoa[Touch]中有很多的不可变类比如：NSDate，NSURL，UIImage，NSNumber等
这更加安全了（不需要使用Copy），也不必担心接下来的程序会改变这个数值。

``` Objective-C
NSArray<NSString *> *array = [NSArray arrayWithObject: NSHomeDirectory()];
NSString *component;  while ((component = getNextSubdir()) { 
	 array = [array arrayByAddingObject: component]; } 
	 url = [NSURL fileURLWithPathComponents: array];
``` 

## Value Type将怎样解决这种问题呢？

Swift中的所有基础类型都是Value Type的，像：Int，Double，String ...
Swift中所有的Collection都是Value Type的，像：Array，Set，Dictianry...
Swift中的`Tuples，Struct，Enums`如果只包含Value Types那么他们自身也是Value Type的。
Value Type要是完全可以直接比较的，可以直接使用`==`，**自定义的Value Type并且其必须要遵守`Equable`协议，覆盖`==`方法才可以使用的**。

```Swift
var a: [Int] = [1, 2, 3]
var b: [Int] = [3, 2, 1].sorted(by:<)
assert(a == b) // true
```   

如果是自身的定义的Struct，那么需要遵守`Equable`协议，并且覆盖`==`方法来实现。比如：

```Swift
struct Temperature: Equatable {
  var celsius: Double = 0
  var fahrenheit: Double {
    get { return celsius * 9 / 5 + 32 }
    set { celsius = (newValue - 32) * 5 / 9 }
 }
}
func ==(lhs: Temperature, rhs: Temperature) -> Bool {
  return lhs.celsius == rhs.celsius
} 
```  

使用Value Type不用担心竞争条件。也就是不用担心资源抢夺，加锁等。看下面的代码：

```Swift
var numbers = [1, 2, 3, 4, 5]
scheduler.processNumbersAsynchronously(numbers) //异步处理numbers
for i in 0..<numbers.count { numbers[i] = numbers[i] * i }
scheduler.processNumbersAsynchronously(numbers) //异步处理numbers
``` 

在Reference Type中这个`numbers`将会发生资源抢夺，但在Swift中是Value Type的，在执行`for i in 0..<numbers.count { numbers[i] = numbers[i] * i }`的时候会发生Copy操作，所以不会发生资源抢夺。也就是说每次将Value Type赋值给其它的Value Type的时候会发生拷贝（逻辑拷贝）操作，但是这种Copy消耗的时间很微小，并且系统会将Copy推迟到写操作执行的时候，这就是Copy on Write。*在Swift中可以利用Protocol将Struct等ValueType封装，类似于OOP中的多态*

## Swift中的Value Type和Reference Type混用会怎样呢？
我们来看Structh中含有结构体的情况：

```Swift
   struct ButtonWrapper {
      var button: Button
   }
   ``` 
   
在这种情况下，复制ButtonWrapper的时候将会共享button这个Reference Type。这就违背了我们上文所说的Value Type在重新赋值的时候拷贝（深拷贝，和原来的Struct没有关系）。怎样才可以做到这一点呢？比如下面的

### Value Type 中含有不可变的Reference Type
```Swift
struct Image: Drawable {
  var topLeft:CGPoint
  var image: UIImage
}
var image = Image(topLeft:CGPoint(x:0,y:0),image:UIImage.init(named: "someImage.png"))
var image2 = image
``` 

这时image和image2将会公用一个UIImage：
![ValueType Contains Reference Type](http://upload-images.jianshu.io/upload_images/1513759-fb29f1978b1a0b66.png)
在实现`Equatable`协议的时候，我们这么做：

```Swift
 extension Image: Equatable {   }
 func == (left:Iamge, right:Image) -> Bool {
  return left.topLeft == right.topLeft && left.image === right.image
 }
 ``` 
 
但是由于UIImage是不可变的，所以我们不必担心image2的image对象的改变会影响到image的image对象。
*注：上面`===`表示引用相同，但是不表示其指向`Image`是相同的，如果要表示其相同，需要使用`==`操作。*

### Value Type 中含有可变的Reference Type

下面我们来看一个可变的Reference Type。

```Swift
struct BezierPath: Drawable {
  var path = UIBezierPath()
  var isEmpty: Bool {
    return path.empty
} 
  // **注意这种写法是错误的**
  func addLineToPoint(point: CGPoint) {
    path.addLineToPoint(point)
} } 
```  

其内存结构是这样的：
![Value Contains Reference Type](http://upload-images.jianshu.io/upload_images/1513759-f574f73819356261.png)

这时如果我们如果执行下面的代码

```Swift
var bezierPath1 = bezierPath0
``` 

就会发现意想不到的Bug，因为你对bezierPath1的任何改动都将会显示到bezierPath0上。
怎样解决这样的问题呢？这时我们需要使用**Copy On Write**

>对Value Type中的Reference Type做改动将会破坏Value Type的"完全独立"特性。
>所以我们必须将可变的Reference Type和不可变的操作分开
>不可变操作总是安全的
>可变操作必须首先Copy

怎样做到Copy On Write呢？我们需要给BezierPath中加入如下代码：
```Swift
struct BezierPath: Drawable { 
private var _path = UIBezierPath()  var pathForReading: UIBezierPath { 
return _path 
}  var pathForWriting: UIBezierPath {
    mutating get {      _path = _path.copy() as! UIBezierPath 
    return _path 
} } 
}
```  

这样我们就可以将上述的错误代码改为：

```Swift
extension BezierPath { 
var isEmpty: Bool {
return pathForReading.empty 
} 
 mutating func addLineToPoint(point: CGPoint) {
    pathForWriting.addLineToPoint(point)
  }
}
``` 

这样，我们在执行：
```Swift
var path = BezierPath()
var path2 = path
if path.empty { print("Path is empty") }
var path2 = path
path.addLineToPoint(CGPoint(x: 10, y: 20))
path.addLineToPoint(CGPoint(x: 100, y: 125))
```  

这段代码的时候就会在addLineToPoint的时候执行Copy，这就不会出现改动path而影响path2的现象了。
但是还有一个问题，每次执行addLineToPoint的时候都需要执行Copy操作，有时候如果这个对象只有一个引用那么就不需要这种操作，所以我们可以利用`isUniquelyReferencedNonObjC()`方法来判断时候需要Copy,如果返回true，说明只有一个对象在用，就不必Copy，如果返回false，说明很多对象在用，这个时候就需要执行Copy操作了。
用法如下：
```Swift
struct MyWrapper {
  var _object: SomeSwiftObject
  var objectForWriting: SomeSwiftObject {
    mutating get {
     if !isUniquelyReferencedNonObjC(&_object)) {
        _object = _object.copy()
     }
     return _object
    }
} }
```  

>注：
>1. 需要标示记忆过程的时候，比如实现撤销操作，需要恢复之前数值的时候。（备忘录模式）
>2. 比如需要对新的变化做特殊处理的时候，因为我们已经记忆了之前的过程，只需要对最新的Value Type改变，比如：本博客中的第一张图片，如果衣服颜色改变了，那么就只改变衣服颜色的那几个方格的值即可。

### 参考资料
https://developer.apple.com/videos/play/wwdc2015/414/























