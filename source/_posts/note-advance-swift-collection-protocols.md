---
title: Swift中Collection的Protocol
date: 2017-07-28 17:37:51
tags: Notes
---

## Sequences
`Sequence`协议位于系统架构的底层，其定义是
   ```swift
     protocol Sequence {
   
      associatedtype Iterator: IteratorProtocol
      func makeIterator()-> Iterator
   
     }
     ```
     
## Iterators
     ```Swift
     protocol IteratorProtocol{
     associatedtype Element
     mutating func next()-> Element?
     
     }
     ```
    
实现了IteratorProtocol协议就可以遍历实例中的数据了，比如：
	```Swift 
	struct PrefixIterator:IteratorProtocol {
    let string:String
    var offset:String.Index
    init(string:String) {
        self.string = string
        offset = string.startIndex
    }
    typealias Element =  String // 这里可以省略，因为编译器可以从返回值的类型中推断出来
    mutating func next() -> String? {
 
        guard offset < string.endIndex else { return nil }
        offset = string.index(after: offset)
        let offSetString = string[string.startIndex..<offset]
        return offSetString;
    }
    
	}
	
	 var prefixIterator = PrefixIterator.init(string: "Hello")
	 while let content = prefixIterator.next() {
	    print(content)
	}
	
     struct PrefixSequence:Sequence {
     let string: String
     func makeIterator() -> PrefixIterator {
        
        return PrefixIterator(string:string)
        
       }
    }
    
     for prefix in PrefixSequence(string:"Hello"){
     print("for in\(prefix)")
    }
    PrefixSequence(string:"Hello").map {$0.uppercased()}
    ```
如上所示，我么要想对一个Type实现for in 操作，需要两步：

1. 创建一个遍历器：让其遵守`IteratorProtocol`协议
2. 创建一个Type：让其遵守`Sequence`协议，在实现`makeIterator`方法时将第一步中的`Iterator`返回即可
`Iterator`有两种不同的语义，**值语义**(将要遍历的实例拷贝)和**引用语义**（不拷贝所遍历的实例），比如

```Swift
		let sequ = stride(from: 0, to: 10, by: 1)
		var i1 = sequ.makeIterator() // StrideToIterator
		i1.next() //Optional(0)
		i1.next() //Optional(1)
		
		var i2 = i1;
		i1.next() //Optional(2)
		i1.next() //Optional(3)
		
		i2.next() //Optional(2)
		i2.next() //Optional(3)
		```	
这个StrideToIterator是值语义，所以在赋值的时候会执行拷贝操作。
AnyIterator会将基础Iterator用内部box对象封装，这个box对象时引用类型的，所以其是引用语义。比如:
      ```Swift
      var i3 = AnyIterator(i1)
      var i4 = i3   
      i3.next() // Optional(4)
      i4.next() // Optional(5)
      let fibsSequence2 = sequence(state:(0, 1)) { (state:inout (Int, Int)) -> Int? in
   
    let upcomingNumber = state.0
    state = (state.1 , state.0 + state.1)
    return upcomingNumber
    
     }
     Array(fibsSequence2.prefix(10))
     ```
  Sequence的闭包是懒执行的，只有到获取某个数值的时候才执行，比如
  
     Array(fibsSequence2.prefix(10))
  
   这才让构造方法`fibsSequence2.prefix(10)`产生作用，如果`Sequence`提前计算了其数值，因为其实无穷`Sequence`，所以当越界的时候其就会崩溃。
   
  有些Sequence是不稳定的，也就是说每次的遍历，结果可能不一样，比如网络流，磁盘文件，UI事件流，以及其它类型的数据，它们都可以被建模，作为`Sequence`。这也就是为什么，**取出第一个元素的属性只存在于Collection中**，而不是在`Sequence`中，因为改这个`first`方法一定要是`nondestructive`的，也就是说取出第一个元素不应该对其输出结果产生影响，也就是说必须是稳定的。
  
  怎样判断一个`Sequence`是否是稳定的？
  如果一个`Sequence`遵守`Collection`，那么它就是稳定的。
  反过来，就不成立了：如果一个`Sequence`是稳定的，那么它就是遵守`Collection`协议的。这时不成立的。比如：`StrideTo`和`StrideThrough`类型，就是稳定的，然而他们没有遵守`Collection`协议。
  
  
## Sequences和Iterator之间的关系是怎样的？
Sequences和Iterator那么相似，为什么不把他们合并成一个呢？在`destructively consumed sequence`中可以，因为他们可以共用一个Iterator，但是在`stable sequence`中是不行的，因为它们需要Iterator提供的隔离的遍历状态和遍历逻辑（这种遍历状态就是Iterator创建的）。`makeIterator`也就是为了创建这种遍历状态。

## SubSequence

Sequence有另外的一个关联属性SubSequence
```Swift
    public protocol Sequence {
    associatedtype Iterator : IteratorProtocol
    associatedtype SubSequence 
    ...
    }
    ```
    
我们可以对其`SubSequence`做以限制，比如：
```Swift
    extension Sequence where Iterator.Element: Equatable,
    SubSequence: Sequence,
    SubSequence.Iterator.Element == Iterator.Element {

    func headMirrorsTrail(_ n:Int) -> Bool {
    
        let head = self.prefix(n)
        let trail = self.suffix(n).reversed()
        return head.elementsEqual(trail)
        
    }
    }
    let result = [1,2,3,4,2,1].headMirrorsTrail(1)
    ```
      
## Collection Protocol
   
   如上文所示Collection都是稳定的`Sequence`，它可以稳定的被屡次遍历，并且没有破坏。并且其元素可以使用脚标的形式被访问。Colllection的`Index`通常是`integer`，存在数组中，并且是有限的。也就是说不像`Sequence`，`Collection`必须是有限的。这也就是为啥其有`count`属性。
不仅仅标准库中的`Array`,`Set`,`Dictionary`,`CountableRange`以及`UnsafeBufferProinter`等等遵守`Collection`协议，Foundation库中的`Data`和`IndexSet`也遵守`Collection`协议。  

A Queue Implementation中对算法的解释不是很懂。

如果一个Type要遵守`Collection`协议，那么它需要满足以下要求：
1. 提供startIndex属性
2. 提供endIndex属性
3. 提供一个subscript，它至少是readonly的，用来获取Type的Element
4. index函数，来寻找Collection的Index

由于Collection的协议的很多方法都有默认实现，所以我们在没有特殊处理的时候不需要再实现其他的方法，比如我们定义完Collection之后就可以使用`map`,`flatMap`,`filter`,`sorted`,`joined`等方法。
为了让我们定义的Collection使用字面量的形式来创建，我们需要实现`ExpressibleByArrayLiteral`协议，该协议只有一个方法：

```Swift
public protocol ExpressibleByArrayLiteral {

    /// The type of the elements of an array literal.
    associatedtype Element
    /// Creates an instance initialized with the given elements.
    public init(arrayLiteral elements: Self.Element...)
}
```
也就是在这个初始化方法里面调用自己的`Designated`初始化方法。

注：
   只要是遵守`ExpressibleByArrayLiteral`的Type都可以使用如
 ```Swift
 let queue:FIFOQueue = [1,2,3,4]
 ```
 这样的方式进行创建，至于创建的类型需要根据声明的类型以及上下文来推断，也就是说在利用[1,2,3,4]这样的方式进行初始化的时候，系统会自动的调用协议中规定的初始化方法。
 


