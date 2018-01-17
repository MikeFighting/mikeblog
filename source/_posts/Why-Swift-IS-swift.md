---
title: 为什么Swift比OC快？
date: 2017-08-01 19:34:23
tags: Swift
---

Swift相比OC以及其它语言，有很多的优化点，这篇文章将从方法调度的角度去说明为什么Swift要比OC更快。OC是一门动态的语言，很多实际执行需要在运行时才可以确定，Swift不一样，**Swift将很多在运行时才可以确定的信息，在编译期就决定了**。这就让Swift更加快速。
方法调度就是程序在触发方法时选择需要执行指令的过程，它在每次方法执行时都会发生。如果这种调度发生在编译期，我们称它为静态调度（Static Dispatch），如果调度发生在运行时，那么我们称它为动态调度（Dynamic Dispatch）。静态调度往往要比动态调度要快。那么问题来了，为什么我们需要动态调度呢？全部用静态调度不就得了？
问题就在于我们很多时候我们需要用到多态，看看下面这段非常简单的代码
```Swift
class Animal {

    func eat() {
        print("animal eat");
    }
    func sleep() {
        print("animal sleep")
    }
}

class Dog: Animal {

    override func sleep() {
        print("dog sleep")
    }
}

class Rabbit: Animal {
    
    override func eat() {
        print("rabbit sleep");
    }
   override func sleep() {
        print("rabbit sleep")
    }
    
}
var animal:Animal?
var somThingTrue = false
//执行很多业务逻辑
if somThingTrue {
    animal = Rabbit()
    
}else{

    animal = Dog()
}
animal?.eat()
```
上面的代码中`animal?.eat()`就不能够在编译期确定，因为其中需要很多的业务逻辑(比如根据用户的不同，或者网络请求结果的不同)来确定就究竟创建出来的对象是Rabbit还是Dog，也就无法最终确定要调用那个对象的eat()方法。相似的代码在OC中是怎样执行的呢？在OC中编译器会将这个方法翻译成`objc_msgSend(target,@selector(eat),nil)`这个方法，然后到了运行时，会分为以下几步进行调用：

1. 找到方法target中isa对应的Class（如果是类方法要到其metaClass中找）。
2. 从其中的`struct objc_method_list **methodLists
`找到对应的方法实现。
3. 如果没有找到就到superClass的`methodLists`中找。

如果在Swift中，它是怎样做方法调度的呢？
1. 找到target对应的class
2. 从class的V-Table中的那得到函数的实现
Swift中的类会创建一个V-Table，这个Table是一个数组，其中存放的是函数指针。子类会按照父类V-Table中函数的存放，如果子类没有覆盖某个方法，那么就会拷贝父类方法的地址，如上面的例子会得到下面的V-Table。

    Animal
    -----
    Index0 eat 0x0001
    Index1 sleep 0x0004
    
    Dog
    -----
    Index0 eat 0x0001 (copied)
    Index1 sleep 0x0008 (overrideen)
    
    Rabbit
    -----
    Index0 eat 0x0002 (overrideen)
    Index1 sleep 0x0003 (overrideen)
    
 可以注意到Dog因为没有覆盖父类的`eat`方法，所以其copy了父类的`0x0010`指针。因为Swift是Type Safe的，所以在调用它的时候它不会变成`Robot`或者其它的类（如果不能通过编译），所以无论是调用上面结构中的Animal，Dog，还是Rabbit类，它都是调用相同的Index，得到对应的方法实现。**将函数指针和Index所做的映射在编译期就确定了，这就大大减少了运行时的工作量，提高了运行速度。**所以在运行时它没有必要知道是哪个类型的实例调用了这个方法，只需要找到相应的V-Table即可，至于是其中的哪个Index已经在编译期确定了，没必要再去查找Index的值。
 然而Swift的方法调度不仅仅是动态方法调度，还有很多静态方法调度。
 **如果我们将某个方法标记为final或者private，或者我们不用类，而使用结构体，枚举，这时就不需要动态调度，只需要静态调度即可，这样速度会更快。**







