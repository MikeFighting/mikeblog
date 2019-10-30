---
title: Swift进阶
date: 2017-08-07 22:29:53
tags: WWDC
---
本文是对[WWDC2014--AdvancedSwift的总结](https://developer.apple.com/videos/play/wwdc2014/404/)

## 改变参数名

比如我们要改变Thing对象的参数名：

```Swift 
class Thing {
init(location: Thing?, name:String,
     longDescription: String){ ... }
}
```  
 
如果我们不想用默认的`Thing.init(location:Beijing, name:"wall",longDescription:"An amazing city")`这种方式进行初始化，我们可以在参数前面添加`label`的形式：

```Swift
class Thing {
  init(newLocation location: Thing?, newName name: String,
newLongDescription longDescription: String) { ... }
}
```  

这样我们就可以使用`Thing.init(newLocation:Beijing, newName:"wall",newLongDescription:"An amazing city")`这种参数来进行初始化。

## 匿名参数

比如下面的例子，我们不需要字典中的value，那么我们就只需要遍历其`key`即可。

```Swift
for (key,_) in dictionary {
 print(\(key))
}
```   

在这个例子中，我们使用下划线`_`来进行你匿名操作，略过了我们不关心的value值，而只输出了`key`的值。

在上面`Thing`的例子中，如果我们要移除其参数名称，那么我们可以将`label`变为下划线，这样我们在调用的时候就不必写参数名了。

```Swift 
class Thing {
  init(_ location: Thing?, _ name: String,

   _ longDescription: String) { ... }
}
```  

这样我们就可以使用`Thing.init(Beijing, "wall","An amazing city")`来初始化了。

## Protocol

如果我们要给上面的`Thing`对象添加一个方法`performPull`来执行其是否可以被拉动的方法。

```Swift
		// The parser will call this. func performPull(object: Thing) 
{ if /* object is   pullable */ {      /* pull it */ }else{      /* complain */   } } 
```     

这时我们可以添加一个Protocol：

```Swift
protocol Pullable {
  func pull()
} 
class Boards: Thing, Pullable {
func pull() {
....
}
}
```  

这样我们的`Boards`类就遵守了`Pullable`协议，当我们来检查某个对象是否遵守了某个协议时，我们可以这样做：

```Swift
func performPull(object: Thing) { if let pullableObject = object as Pullable { pullableObject.pull()    }else{ 
     print("You are not sure how to print a \(object.name).")
   } 
} 
```   

### 对象转String

如果我们要打印某个对象，并且需要打印出其中的有效信息，那么我们要像OC中实现`description`方法一样，来遵守`CustomStringConvertible`协议并且实现其中的`description`方法。
还有很多类似的方法


Protocol     | 作用
------------ | -------------
ExpressibleByStringLiteral | "abc"
ExpressibleByArrayLiteral | [ a, b, c ]
ExpressibleByDictionaryLiteral | [a: x, b: y]
Sequence | for x in sequence
CustomStringConvertible | "\(convertible)"
 

同样，如果我们要对某个对象使用下表，那么我们要`subscript`。怎样对某个类使用下表呢？


## 泛型
 
 比如以下三个方法，其参数的形式是一样的，但是其参数类型不同，我们想把它变成一个函数，这时候我们最常想到的做法就是使用`Any`来表示任何类型的参数和任何类型的返回值。
 
```Swift
// 之前的三个函数
func peek(interestingValue: String) {
  println("[peek] \(interestingValue)")
}
func peek(interestingValue: Int) {
  println("[peek] \(interestingValue)")
}
func peek(interestingValue: Float) {
  println("[peek] \(interestingValue)")
} 
// 变为一个函数
func peek(interestingValue: Any) {
  println("[peek] \(interestingValue)")
} 
```  

但是这有个问题，在我们要调用返回值的某个方法时候编译器会报错，因为我们的返回值`Any`并没有所调用的方法，也就是说我们需要一次强转，强转在代码层面是很难看的，也很麻烦，这时候我们就可以使用泛型来解决这个问题，如果我们使用了泛型，那么编译器会将输入的参数和输出的参数给我们推断出来，这样就不必强转了。并且当编译器可以推断出来是那种type的时候它会给我们做各种优化。

泛型在Swift中很常见，比如：
Array<T>以及Dictionary<K,V>：是范型的结构体
Optional<T>：范型枚举
我们也可以创建自己的泛型类

### Type间的关系

比如我们要转换两个变量的值，我们可以调用下面的方法：

```Swift
 // Exchange the values of x and y
func swap<T>(inout x: T, inout y: T) { 
let tmp = x     x = y     y = tmp 
} 
var studentCount = 42
var teacherCount = 7
swap(&studentCount, &teacherCount) // OK
var schoolName = “Homestead High School"
swap(&studentCount, &schoolName) // error: 'Int' is not identical to 'String'
```   

有了这样的编译器提示，这样我们就可以保证了输入的两种类型是相同的，这样就会更加**安全**，代码也会少很多bug。

### 对泛型进行Protocol限制

我们可以给泛型加上限制，让它遵守某些协议，比如：

```Swift
func indexOf<T>(sought: T, inArray array: T[]) -> Int? {
  for i in 0..array.count {
    if array[i] == sought { // error: could not find an overload for '==' that accepts the supplied arguments 
      return i
} } 
return nil } 
```    

这是编译不通过的，因为编译器不知道我们的`T`是否遵守了`Equatable`协议，如果我们将其变为`func indexOf<T:Equatable>(Sought: T, inArray array: T[]) -> Int?`就可以编译通过了。

### 实现Equatable协议

Enum，Class以及Struct都可以实现协议，比如我们有如下的Struct，实现Equatable协议如下：
```Swift
struct Temperature : Equatable { 
  let value: Int = 0
}

func == (lhs: Temperature, rhs: Temperature) -> Bool {
  return lhs.value == rhs.value
} 
```    

当我们实现了`Equatable`，那么对于`!=`这种操作，Swift在底层就会自动帮我们实现。

## 斐波那契数列的例子

什么是菲波那切数列？数列的前两个数相加等于后面的一个数。 `0, 1, 1, 2, 3, 5, 8, 13, 21, ...`
我们创建一个菲波那切数列的函数：

```Swift
func fibonacci(n: Int) -> Double {
  return n < 2 ? Double(n) : fibonacci(n - 1) + fibonacci(n - 2)
} 
```     

这个函数的返回值仍然是一个函数，它是一个递归调用的函数，这个函数的效率极低。当你在Playground中实验的时候就会卡的不行。如果调用`fibonacci(44)`估计要十几秒的时间。原因就是它需要一个调用一个树状结构，我们用`fibonacci(5)`做个图示：
![fibonacci(n:5)](http://upload-images.jianshu.io/upload_images/1513759-1d9abd75b75e42ea.png)

在图中我们发现fib(1)，fib(2)等函数被持续的调用，我们如果可以把这个已经计算过的数值存下来，那么以后不是就就可以直接计算了呢？这样不就可以极大的提高计算的速度。这样我们就只计算下面带星号的函数即可：
![valuable func](http://upload-images.jianshu.io/upload_images/1513759-12e6e8ab9ae865d9.png)

这时最先想到的就是用一个全局的字典手动来存储这个数值，实现方法如下：

```Swift
var fibonacciMemo = Dictionary<Int, Double>() // implementation detail  // Return the nth fibonacci number: 0, 1, 1, 2, 3, 5, 8, 13, 21, ... func fibonacci(n: Int) -> Double { 
  if let result = fibonacciMemo[n] {
    return result
  }
  let result = n < 2 ? Double(n) : fibonacci(n - 1) + fibonacci(n - 2)
  fibonacciMemo[n] = result
  return result
} 

// 1.61803399...
let phi = fibonacci(45) / fibonacci(44) //0.1 seconds = 100x speedup 
```        

这样一来，我们之前计算`fibonacci(44)`中十几秒的计算时间一下子缩短到了0.1秒，缩减了100倍。
如果这样做的话，我们以后每次用到这个方法就都需要写一个字典，每次把这个计算过程过一遍，并且更重要的是它不具备通用型，如果我要放进去字符串，那就不起作用了。下面我们写一个通用的函数来解决这种需要保留中间数值，并且不需要额外的全局变量`fibonacciMemo`就可以解决问题的方法。先看一个不递归时候的函数：

```Swift
func memoize<T: Hashable, U>(work: @escaping (T)->U) -> (T)->U {

    var memo = Dictionary<T, U>()
    return { x in
        if let q = memo[x] { return q }
        let r = work(x)
        memo[x] = r
        return r
    }
}
```   

然后再来一个可以递归的方法：

```Swift
func memoize<T: Hashable, U>(work: @escaping ((T)->U, T) -> U) -> (T)->U {
    var memo = Dictionary<T, U>()
    func wrap(x: T)->U {
        if let q = memo[x] { return q }
        let r = work(wrap, x)
        memo[x] = r
        return r
    }
    return wrap
}
```      

这两个函数包含了swift中的很多高级语法：

* 强大的编译器推断匹配,比如你调用：

```Swift
     let fibonacci = memoize {
    (n: Int) in
    String(n)
}
```        

那么上面泛型中的`T`就会被推断成`Int`，而`U`怎会被推断成`String`。

* 尾随闭包，比如调用`let fibonacci = memoize {...}`的时候。
* 更通用，更安全，性能更高的泛型函数。

关于这个函数，有人专门写了一篇博客来说明:https://medium.com/@mvxlr/swift-memoize-walk-through-c5224a558194

## 泛型结构体的一个例子

```Swift
struct StringStack {
  mutating func push(x: String) {
items += x } 
  mutating func pop() -> String {
    return items.removeLast()
} 
  var items: String[]
}
```   

这里我们创建了一个字符串的栈，如果我们要变为泛型，则需要：

```Swift
struct Stack<T> {
  mutating func push(x: T) {
items += x } 
  mutating func pop() -> T {
    return items.removeLast()
} 
  var items: T[]
}
```   

这样我们就可以往里面放任何数据类型了：
 
```Swift
var intStack = Stack<Int>()
intStack.push(42)
```   

但是当我们需要对这个Stack做`for in`操作时候，却得到了下面的错误提示：
![for_in_error](http://upload-images.jianshu.io/upload_images/1513759-9d39a23bddecaefd.png)
这时候我们需要理解下，什么是`Sequence`，当Swift给我们做`for in`操作的时候在底层给我们做了什么？

``` Swift
// 我们代码
 for x in someSequence {
  ...
 }
// Swift翻译后的代码
var __g = someSequence.generate()
while let x = __g.next(){
...
}
```  

从上面可以看到，首先我们的`someSequence`需要关联一个`generate`，并且这个`generate`需要实现一个`next`方法，这样我们就可以对这个结构体做`for in`操作了。
我们先看这个`generate()`，它是一个结构体，其遵守`Generator`（`protocol`)。

```Swift 
protocol Generator {
 typealias Element
 mutating func next() -> Element ?
}
```  
   
然后我们创建一个遵守`Generator`的结构体：

```Swift
struct StackGenerator<T> : Generator {
  typealias Element = T
  mutating func next() -> T? {
    if items.isEmpty { return nil }
    let ret = items[0]
    items = items[1..items.count]
    return ret
} 
  var items: Slice<T>
} 
```   

那么什么是`Sequence`呢？它也是一个`protocol`

```Swift
 protocol Sequence {
  typealias GeneratorType : Generator
  func generate() -> GeneratorType
} 
```  

它关联了一个`Generator`，然后我们让新建的`Stack`遵守这个`Sequence`协议：

```Swift
extension Stack : Sequence {
  func generate() -> StackGenerator<T> {
    return StackGenerator( items[0..itemCount] )
  }
} 
```      

这样就可以对`Stack`做`for in`操作了。

```Swift
func peekStack(s: Stack<T>) {
  for x in s { println(x) }
} 
```   

关于`Sequence`和`Generator`的详细说明，请看我的另外一篇博客[Swift中Collection的Protocol](https://mikefighting.github.io/2017/07/28/note-advance-swift-collection-protocols/)

### 执行效率

Swift是静态编译，很小的runtime。写好的代码传到设备上之后不需要再重新编译，只需要等着运行即可。Swift中的编译器，相比于C，C++，Objective-C的Clang，做了一步优化：
![SwiftComplier](http://upload-images.jianshu.io/upload_images/1513759-3cb2500863892632.png)
除此之外，Swift会在编译时候做以下几点

* 全局分析App
* 使用Struct不会对Runtime的性能造成影响
* Int，Float等很多标准库都是Struct的

Swift还有去虚拟化的特点，它可以让再运行期确定的事情放到了编译期，这样就会更快了，详细内容可以参考我的另外一篇博客[为什么Swift比OC快](https://mikefighting.github.io/2017/08/01/Why-Swift-IS-swift/)


