---
title: 理解Swift的性能
date: 2017-07-30 13:26:43
tags: WWDC
---
**草稿：**
本博客主要来源于WWDC2016-416，UnderStanding Swift Performance。
Swift性能指标：
1. Stack VS Heap
2. Reference Count
3. Method Dispatch中Static多还是Dynamic多
栈上的操作更加高效，降低栈指针来创建，增加栈指针来释放栈。
堆上可以存放高级数据结构，但是堆的操作效率是更低的，因为每次开辟堆空间都需要寻找空闲的内存块。在消除的时候需要重新插入一个内存块。除此之外，Class等存在堆上的对象，需要开辟我们看不到的空间，这些空间是需要消耗内存的。
引用计数的不断操作会降低性能。所以要尽可能少的创建Class。
在Struct中包含Class是会降低性能的，我们要尽量减少这种情况的出现，比如在使用纯字符串的时候，我们可以使用Enum，或者其它的值类型来替代。
如果Struct中包含Class对象，那么在CopyStruct的时候也会拷贝其引用计数，包含的Class越多，就需要操作更多的retain和release方法方法。
Swift在调用一个函数的时候，我们需要跳到对应的方法实现。
Static 调度：如果在编译时期就可以确定某个方法的实现的话，就可以直接跳转到相应的方法的。并且是inline方法。既然静态调度这么好，为什么还需要Dynamic Dispatch呢？因为我们需要执行各种业务逻辑，需要根据具体的场景来决定究竟调用那个方法，我们需要使用多态。
对于类，那么编译器会增加一个指针来存储来指向那个类的Type信息，并且存到静态区。在执行函数调用时，编译器会到该类型对应的虚拟方法表（VTable）中找到相应的方法实现。将类标记为`final`时，其方法的调用将变为静态派发。在不需要动态派发的时候，我们应该尽量使用静态派发。

怎样是用`Struct`写多态呢？Protocol类型的变量是怎样被存储，被拷贝，以及方法调度是怎样工作的呢？
Static Dispatches VS Dynamic Dispatch
Static方法：可以在编译时确定需要调用的方法实现，在运行时，可以直接调用并且可以做`inline`优化。
Dynamic方法：需要在运行时在表中找到对应的方法的实现。并且会妨碍inlining及其它的性能优化。当我们利用了多态之后，我们就难以确定这个方法究竟是调用那个方法，也就是说必须在运行时去确定到底要执行那个方法的实现。
Swift中Protocol Type的多态，由于没有继承，所以就没有了`V-Table`方式的调度。Swift使用了一种Protocol Witness Table的技术来调度Protocol Type的方法，这个表格的入口和该Type的实现先链接，因此找到这个表格就找到了方法的实现，那么我们怎样找到这个Table呢？，注意数组中的数值都有相同的offset，因此Swift使用Existential Container来封装protocol type，它的前三个字（两个字节称为一个字）是valueBuffer，小数据，例如两个word的Point可以存储，但是大于三个字时候，swift将会开辟一个堆空间，并且将这个控件的地址存储在Existential Container里面。
Swift让value Type，比如Strut和protocol一起获得了动态调度行为，实现了动态多态。
Copy on write：Swift自己提供了Copy on Write的机制，但是如果我们自己写了结构体，并且结构体比较重，还有其它的引用类型，在Copy结构体的时候，我们同时需要Copy这个对象，所以在Write的时候，我们需要判断这个引用类型的引用计数，然后做相应的改变。
因此Existential Container需要处理不同的数据类型，这是怎样实现的呢？这需要另外一个基于表的机制--Value Witness Table，程序中的每一个type都有一个这样的表，它包含以下几部分：
allocate:
copy:
destruct:
deallocate:
Whole Model优化可以让同一个模块中的不同文件都可以同时优化。
Type不会在运行时改变。

Generics-Small Value
如果使用泛型，那么每次函数的调用就只能产生一个call context，并且call context中的类型是一定的，这里Swift不会使用Existential container，它可以直接传递value witness table和 protocol witness table
1. 没有比必要开辟堆空间，没有在堆上进行操作，所以不需要考虑线程安全，无需对线程进行加锁。
2. 没有引用计数，操作引用计数也需要是线程安全的，因为引用计数操作极其频繁，所以其性能消耗会逐渐增多，并且不是可以忽略的。
3. 通过Protocol Witness Table进行动态调度以实现多态
Generics-Large Value
1. 使用indirect storage 来开辟堆空间
2. 如果包含引用类型的话会有引用计数
3. 通过Protocol Witness Table进行动态调度实现多态

总结：
尽可能少的使用dynamic runtime，选择合适的抽象。这样编译器可以做错误检查，并且可以做优化，提升代码执行速度。
Class类型：identity或者OOP类型的多态。
结构体&枚举类型：值语义。
Protocol types：动态多态。
泛型和值语义相结合可以实现静态多态。
使用indirect Storage来处理大数值。



