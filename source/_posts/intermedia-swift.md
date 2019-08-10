---
title: Swift中级
date: 2017-08-04 08:28:26
tags: WWDC
---

本文源于对：[WWDC2014--Intermediate Swift](https://developer.apple.com/videos/play/wwdc2014/403/)的总结。

## 为什么需要Optional?

我们队某个对象的操作可能会返回错误的结果：比如我们将某个字符串转为Int型，执行下面的指令：

```Swift 
let age = response.toInt()
```  

这response可能是用户输入的,可能会输入不确定的数值`Do you konw?`那么肯定会得到错误的结果，在OC中遇到这种情况我们怎么处理呢？我们有如下的值可以表示错误：
![Wrong!](http://upload-images.jianshu.io/upload_images/1513759-65ce6dec3ca6cade.png)
但是你必须从不同的接口中去选择相应的错误类型，并且要记住这些错误类型。为了解决这个问题，Swift中引入了Optional的概念，将这种可能是nil的值进行打包。它可以表示上述所有错误的类型，同时，**如果我们使用Optional，就一定要对它进行拆包，使用`!`号，如果不拆包会造成编译器错误**，如下所示：
![Need Unwrap](http://upload-images.jianshu.io/upload_images/1513759-03cea3691052ac9b.png)
**也可以使用Optional Binding将判断是否有值和拆包结合在一起使用：`if let`**。

```Swift 
var neighbors = ["Alex", "Anna", "Madison", "Dave"] let index = findIndexOfString("Anna", neighbors)  if let indexValue = index { 
    println("Hello, \(neighbors[indexValue])")
} else {
    println("Must've moved away")
}
```   

当然我们还可以进一步使用Optional Binding--Optional Chain：

![Optional Chain Binding](http://upload-images.jianshu.io/upload_images/1513759-040ccbafce764e18.png)

在Optional Chain中，只要其中一个Optional的值是nil，那么整个Optional Chain都将是nil，并且不会再执行接下来的取值，如果不是nil则继续执行。这样让我们的代码更加简洁更加安全。
Swift中的Optional其实是个枚举：

```Swift
enum Optional<T> {
   case None
   case Some(T)
}
```  
 
## Swift中的内存管理

在Swift中也用的是ARC，也容易出现循环引用，这时需要使用`weak`属性。需要注意的是
> `weak`引用的类型是Optional的。
> Binding该Optional Type将会产生一个强引用。
> 如果仅仅是判断即用`if`判断，则不会产生强引用。

例如：

```Swift 
if let tenant = apt.tenant {
  tenant.buzzIn()
 } 
```  

但是有些时候我们同时需要weak，又同时需要非Optional的。那么该怎么办？我们需要`unowned`属性，他也是weak的。

```Swift
class Person {
    var card: CreditCard?
} class CreditCard { unowned let holder: Person 
    init(holder: Person) {
        self.holder = holder
  } 
} 
```   

这说明`holder`没持指向Person，但是holder离开了Person它就不存在了。`unowned`很像`unsafe unretain`

## Swift中的初始化

在Swift的初始化中需要谨记：

>所有的变量在使用前必须初始化
>设定完自己所有的变量之后再调用Super的初始化方法

在下面这个初始化的例子中：
![init wrong](http://upload-images.jianshu.io/upload_images/1513759-71bbd45dcf36f0a3.png)
这样在init方法中没有初始化完自己的`hasTurbo`变量就直接调用super方法是会在编译的时候报错的，Swift为什么要这么做呢？因为可能会出现如下的情况：
![why init wrong](http://upload-images.jianshu.io/upload_images/1513759-bd1edb7b7a89a7bc.png)
也就是说在父类的`init`方法中可能会调用`filGasTank()`这个方法，而这个方法被子类所覆盖了，所以这时候就可能发生意向不到的bug。
初始化方法的覆盖也可能会产生问题：
比如我们有这样一个Car的类：

```Swift
class Car {
    var paintColor: Color
    func fillGasTank() {...}
    init(color: Color) {
        paintColor = color
        fillGasTank()
    }
} 
class RaceCar: Car {
    var hasTurbo: Bool
    init(color: Color, turbo: Bool) {
        hasTurbo = turbo
        super.init(color: color)
} 
    convenience init(color: Color) {
        self.init(color: color, turbo: true)
} 
    convenience init() {
        self.init(color: Color(gray: 0.4))
} } 

class FormulaOne: RaceCar {
    let minimumWeight = 642
    
    // inherited from RaceCar
    /*init(color: Color, turbo: Bool) {
        hasTurbo = turbo
        super.init(color: color)
    }
    convenience init(color: Color) {
        self.init(color: color, turbo: true)
    }
    convenience init() {
        self.init(color: Color(gray: 0.4))
    }
    */
} 
```  

上面注释的内容是从父类中继承过来的，如果我们在子类中调用`convenience init(color: Color)`这个方法的时候，想让`turbo`这个参数的默认值为`false`，这时候我们就需要覆盖掉父类的`convenience init`方法了。这时我们需要实现自己的`designed initializer`  

```Swift
class FormulaOne: RaceCar {
    let minimumWeight = 642
    init(color: Color) {
        super.init(color: color, turbo: false)
} 
    // not inherited from RaceCar
    /*init(color: Color, turbo: Bool)
    convenience init()
    */
} 
```  

这样以后被注释的内容就不会再被继承了。就会直接掉用子类的`designed init`方法了。

## 懒加载属性

如果我们的某个属性需要很大的性能消耗，那么我们希望在使用的时候再创建该类，那么我们不必像在OC中那样重写其`get`方法，我们只需要在变量声明的前面加上`lazy`关键字即可。

```Swift
lazy var color:UIColor = UIColor.red
```  

这样就可以声明了一个懒加载的属性了。

## Closures

### 基本用法

Swift中Array的sort方法实现了Closure，我们来看下：

```Swift
var clients = ["Pestov", "Buenaventura", "Sreeram", "Babbage"]

clients.sort({(a: String, b: String) -> Bool in 
return a < b }) 

println(clients)
// [Babbage, Buenaventura, Pestov, Sreeram]
```   

这样就实现了数组中的元素排序。
但是基于`Swift`强大的类型推断功能，我们可以将其简化为：

```Swift
clients.sort({ a, b in
return a < b
})
```  

因为这个`Closure`是有返回值的，所以编译器可以再次推断，所以我们可以这样写

```Swift
clients.sort({ a, b in a < b })
```  

编译器还可以推断出其参数值，所以，我们这里可以写成

```Swift
clients.sort({$0 < $1})
```  

因为我们还有尾随闭包，所以我们可以进一步简化

```Swift
clients.sort{$0 < $1}
```  

### Functional Programming 

我们有很多函数式编程的高阶函数可以供调用：

```Swift
let result = words.filter{ $0.hasSuffix("gry")}.map{$0.uppercaseString}
```  

这样我们就可以找到所有以`gry`结尾的单词，并且将其转化为大写字母。如果这时结果是

```Swift
ANGRY
HUNGRY
```  

我们还可以调用`reduce`方法将其和成一个字符串
```Swift
let reducedResult = result.reduce("HULK"){"\($0) \($1)"}
```   

这时结果如下：
```Swift
HULK ANGRY HUNGRY
```  

### 函数值

比高可以传递一个函数，例如：

```Swift
 numbers.map {
        println($0)
} 
numbers.map(println)    // 可以将一个函数传递过去


var indexes = NSMutableIndexSet()
numbers.map {
    indexes.addIndex($0)
} 

numbers.map (indexes.addIndex) // 可以将一个Method传过去
```  

### 闭包是一个ARC对象

我们可以声明一个Closure属性：

```Swift
var onTempratureChange: (Int) -> Void = {}
func logTemperatureDifferences(initial: Int) {
    var prev = initial
    onTemperatureChange = { next in
        println("Changed \(next - prev)°F")
prev = next 
} 
```  

因为function也是closure，那么我们可以这样写：

```Swift
func logTemperatureDifferences(initial: Int) {
    var prev = initial
    func log(next: Int) {
        println("Changed \(next - prev)°F")
prev = next } 
    onTemperatureChange = log
```  

### 闭包的循环引用问题

和OC中的Block一样，Swift中也会出现循环引用的问题，我们来看看怎样解决：

```Swift
class TemperatureNotifier {
    var onChange: (Int) -> Void = {}
    var currentTemp = 72
    init() {
        onChange = { temp in
currentTemp = temp    } // error: requires explicit 'self' 
  } 
} 
```    

如果出现上面的循环引用问题，编译器会直接报错的，所以我们可以用上文提到的`unowned`来解决。我们可以将init()方法用下面的来取代：

```Swift
init() { 
    unowned let uSelf = self
    onChange = { temp in
      uSelf.currentTemp = temp
    }
```   

但是这样写还会出现一个问题，就是如果别处有一份逻辑一样的代码，某个人不注意拷贝过来了忘记将`self`改成uSelf，或者这个方法很长，写到下面的是忘记了将`self`改成uSelf，那么就会出现内存泄漏的问题。为了解决这个问题Swift中提出了下面的优雅做法：

```Swift
init() { onChange = {[unowned self] temp in 
self.currentTemp = temp 
} } 
```     

## Pattern Matching

`switch`中可以有范围，字符串和数字，并且Enum中可以关联属性，比如：
```Swift
// case中含有范围
func describe(value: Int) { switch value { 
case 0...4: println("a few") 
case 5...12: println("a lot") 
      default:
        println("a ton")
} } 
// case中

enum TrainStatus {
    case OnTime
    case Delayed(Int)
}
```  

使用的时候如下：
```Swift
switch trainStatus {
  case .OnTime:
} 
  println("on time")
case .Delayed(let minutes)
                        :
println("delayed by \(minutes) minutes")
```   

我们可以对这个`delay`做各种各样的匹配：
```Swift
switch trainStatus {
  case .OnTime:
    println("on time")
  case .Delayed(1):
    println("nearly on time")
  case .Delayed(2...10):
    println("almost on time, I swear")
  case .Delayed(_):
    println("it'll get here when it's ready")
``` 

### Pattern Compose

也就是说Pattern可以组合出现，一个Pattern中可以包含其它的Pattern，比如对上文的`TrainStatus`再做以Pattern Compose：

```Swift
enum VacationStatus {
    case Traveling(TrainStatus)
    case Relaxing(daysLeft: Int)
} 

switch vacationStatus {
  case .Traveling(.OnTime):
    tweet("Train's on time! Can't wait to get there!")
  case .Traveling(.Delayed(1...15)):
    tweet("Train is delayed.")
  case .Traveling(.Delayed(_)):
    tweet("OMG when will this train ride end #railfail")
  default:
  print("relaxing")
```   

### Type Pattern

`Pattern`不仅仅可以作用于`Enum`，还可以作用于动态的类型，如：`Class`

```Swift
func tuneUp(car: Car) {
    switch car {
      case let formulaOne as FormulaOne:
        formulaOne.enterPit()
      case let raceCar as RaceCar:
        if raceCar.hasTurbo { raceCar.tuneTurbo() }
        fallthrough
      default:
        car.checkOil()
        car.pumpTires()
} } 
```  

这样在多态中就会变得非常有用了。

### Tuple Patterns
Tuple pattern有其极其强大的功能，其可以对tuple的各个数值做以类型匹配。

```Swift
let color = (1.0, 1.0, 1.0, 1.0)
switch color {
  case (0.0, 0.5...1.0, let blue, _):
    println("Green and \(blue * 100)% blue")
  case let (r, g, b, 1.0) where r == g && g == b:
    println("Opaque grey \(r * 100)%")
```  

我们甚至可以对其中的各个数值做以相应的模式匹配。

### Pattern Matching的应用PList校验
比如我们有下面的方法来校验Plist中的内容是否有效

```Swift
func stateFromPlist(list: Dictionary<String, AnyObject>)
  -> State?
stateFromPlist(["name": "California",
                "population": 38_040_000,
                "abbr": "CA"])
``` 

这时我们要对`population`的值做以限制，如果是字符串返回`nil`，如果是超过某个范围的时候返回`nil`，如果是`abbr`中字母的个数大于2时候我们也返回`nil`，利用tuple pattern matching的强大特性，我们可以这样去做：

```Swift
func stateFromPlist(list: Dictionary<String, AnyObject>)
  -> State? {

switch (list["name"], list["population"], list["abbr"]) { 
case ( 
        .Some(let listName as NSString), 
        .Some(let pop as NSNumber),
        .Some(let abbr as NSString)
      ) where abbr.length == 2:
    return State(name: listName, population: pop, abbr: abbr)
  default:
return nil 
   } 
} 
```  

这就利用了tuple和限制想结合的方式优雅的解决了这个问题。

