---
title: Swift学习Tip之Closures
date: 2016-05-31 06:39:00
tags: Swift
---

## 闭包简介

什么是`Closures`？    `Closures`也就是我们常说的闭包，闭包是一个具有函数功能的代码块，它类似于OC中的`Block`,它能够在它被定义的上下文中，被常量或者变量所捕捉和存储。也就是是说把常量和变量`Closing`(包)住了，这也是`Closures`的由来。全局函数和嵌套函数(`Global and nested functions`)是闭包的一种特例，闭包有一下三种形式:

* 全局函数是有名称但是不会捕获任何数值得闭包。
* 嵌套函数是有名称并且可以从它们的封闭的函数体内部捕获数值的闭包。
* 闭包表达式是用轻量级的语言所书写，并且能够从它们所在的上下文中获取数值的一种没有名称的closures。

由于Swift本身对闭包做了很多的优化，这使得闭包的使用更加的简单明了，这些优化包括:

* 从上下文中推断出参数类型和返回值类型
* 在单一表达的闭包(`single-expression`)中隐含返回值
* 简化参数名
* 尾随闭包语法

## 闭包的基本语法

```objc
//闭包表达式的类型和函数的类型是一样的，都是参数加上返回值,但是闭包后面有一个in, 这个关键字来告诉编译器，闭包的参数和返回值已经定义完了，下面是闭包执行语句了。
{
    (参数) -> 返回值类型 in
    执行语句
}
```

下面说明闭包的基本用法:

* 闭包的完整写法:

```objc
  // 闭包的完整写法
let sayHiToSomeBody:(String) -> Void = {
        (name: String) -> Void in
        print("Hi \(name)")
    }
    sayHiToSomeBody("Mike")
    // Print: Hi Mike  
```

* 没有返回值得闭包

```objc
        // 由于这个闭包没有返回值，所以Void可以省略
        let sayHiToSomeBodyWithoutReturnValue:(String) -> Void = {
            (name: String) in
            print("Hi \(name)")
        }
        sayHiToSomeBodyWithoutReturnValue("Jimme")
```

* 没有参数没有返回值得闭包

```swift
        // 没有参数没有返回值的写法
        let sayHiToSomeBodyWithoutParaAndReturnValue:() ->Void = {
          print("Hi Wallace")
        }
        sayHiToSomeBodyWithoutParaAndReturnValue() 
        // Print: Hi Wallace
```

* 闭包做为一个参数

```swift
       // 闭包做为一个参数
func functionToRunClosureAsParam(someClosure:() -> ()) ->String{
        print("run before the closure")
        someClosure()
        return "I ran the closure"
    }

print(functionToRunClosureAsParam({print("try it now")}))
    // Print:
    // run before the closure
    // try it now
    // I ran the closure
```

* 闭包做为函数的返回值

```swift

 // 闭包做为函数的返回值
 func makeFunctionWithClosureToReturn(somePara:String) ->() -> (){ 
        let resultClosure = { print("\(somePara)")}
        return resultClosure
    }
  let yellSomeString = makeFunctionWithClosureToReturn("I Love Swift")
       yellSomeString()
     // Print: I Love Swift
```

* 捕获值，（在闭包内部捕获闭包所在的上下文(`surrounding context`)的变量）

```swift
    // 捕获值(这里也是一个以闭包做为返回值的函数)
    func someFuncWithClosureReturn(forIncrement amount: Int) -> () ->Int{

        var runningTotal = 0
        // 这个闭包是没有参数，只有返回值，这个返回值是由someFuncWithClosureReturn()的参数，和内部的变量
        let incrementer:()->Int = {
            runningTotal += amount
            return runningTotal
        }
        //            函数其实也是闭包的一种，所以可以这样写
        //            func incrementer() ->Int {
        //
        //                runningTotal += amount;
        //                return runningTotal
        //            }
        return incrementer
    }
    let incrementByTen = someFuncWithClosureReturn(forIncrement: 10)
    print(incrementByTen()) // Print 10
    print(incrementByTen()) // Print 20
    print(incrementByTen()) // Print 30
```

* 尾随闭包`Trailing Closures`:如果闭包是函数的最后一个参数，并且闭包比较长，这个时候可以在()的后面写一个{}来做为闭包。上文提到的functionToRunClosureAsParam()就是这样一个例子，所以我们可以使用尾随闭包来简化函数的调用。

```swift
    //闭包做为函数的最后一个参数
    func functionToRunClosureAsParam(someClosure:() -> ()) ->String 
    print("run before the closure")
    someClosure()
    return "I ran the closure"
    }
    // 这里将闭包提出了()
    // 而不是上文中的:print(functionToRunClosureAsParam({print("try it now")}))
    print(functionToRunClosureAsParam(){print("Try it now ")})
```

## 闭包的高级用法

* `Noescape Closures`:非逃逸型闭包,如果闭包是函数的一个参数，那么这个闭包只能在函数内部使用；`Escape Closures`:逃逸型闭包,当闭包做为函数的一个参数的时候，有时候在函数已经执行完，并且已经返回，但是这之后才调用闭包，这种闭包就称为逃逸闭包（Escaping Closure),逃逸型闭包可以用在异步操作。

```swift
// 非逃逸型闭包
func someFunctionWithNoescapeClosure(@noescape closure: () -> Void) {
    closure()
}

// 逃逸型闭包，如果强制将其改变为非逃逸型的将会报错，因为非逃逸型只能在函数内部使用，而这个闭包却被存在了函数之外的数组中，所以必须是逃逸型的，如果强制改为非逃逸型的，会报错
var completionHandlers: [() -> Void] = [] // 这是一个存贮闭包的数组
func someFunctionWithEscapingClosure(completionHandler: () -> Void) {
    completionHandlers.append(completionHandler)
}

// 非逃逸型的闭包如果使用了全局变量，那么可以省去self,下面举例说明
class SomeClass {
    var x = 10
    func doSomething() {
        someFunctionWithEscapingClosure { self.x = 100 }
        someFunctionWithNoescapeClosure { x = 200 }
    }
}

let instance = SomeClass()
instance.doSomething()
print(instance.x)
// prints "200"

completionHandlers.first?() // 这个时候取出数组中存放的已经逃逸的闭包，然后执行它，就会让 instanc.x变为100
print(instance.x)
// prints "100”
```

下面在举个例子说明`Noescape Closures`的用法:

```swift
    // 再比如非逃逸型闭包
func doIt(@noescape code: () -> ()) {
    /* 我们可以这样做 */
    // 仅调用它
    code()
    // 做为非逃逸型参数传递给另外一个函数
    doItMore(code)
    // 在另外一个非逃逸的函数中捕获智
doItMore {
        code()
    }
    /* 我们不可以做 */

    /*
    // 将它赋值为一个需要可逃逸闭包参数的函数
dispatch_async(dispatch_get_main_queue(), code)
        // 存储它
        let _code:() -> () = code
        // 在另一个可逃逸的闭包
        let __code = { code() }
    */
    }
func doItMore(@noescape code: () -> ()) {}
```

* `Autoclosures`自动闭包:如果函数中的一个闭包参数被`@autoclosures`修饰，那么这个函数在被调用的时候可以省去花括号,注意如果一个闭包被autoClosures修饰的时候默认是noescape型的，所以在不能用到异步处理中，这是只需要在将`@autoclosure`变为`@autoclosure(escaping)`就可以了

下面举例说明:

```swift
// customersInLine is ["Ewa", "Barry", "Daniella"]
func serveCustomer(@autoclosure customerProvider: () -> String) {
    print("Now serving \(customerProvider())!")
}
serveCustomer(customersInLine.removeAtIndex(0)) // 这个时候回省去{}

// 再比如一个以闭包做为参数的函数
func f(@autoclosure pred: () -> Bool) {
    if pred() {
        print("It's true")
    }
}
f(2 > 1) // 如果不加@autoclosure，我们不得不f({2 > 1})
// It's true
```