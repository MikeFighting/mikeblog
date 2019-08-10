---
title: Java多线程基础
date: 2017-11-21 14:09:19
tags: Notes
---

## 多个线程安全类的方法在一起一定是线程安全的吗？

  尽管线程安全类的每个方法都是原子的，但是当很多原子操作合并为一个复合操作的时候，需要额外加锁，否则就会出现竞态条件（race condition）造成线程不安全。但是这里额外的加锁可能会导致性能损耗并且可能引起死锁。如：

```java
if (!vector.contains(element))
    vector.add(element);
```

这里`contains`方法和`add`方法都是线程安全的，但是综合起来就是非线程安全的了。

## 方法的局部变量无需加锁

   某个线程进入一个方法就会创建一个栈，这个栈中存储了某个局部变量，这个局部变量是每个线程所独有，不是共享的，他们之间互不影响，没有必要加锁。

## 加锁时所发生的事情

* 操作互斥，很多个线程不能同时对一个代码块进行操作
* 加锁之后可以保证变量的可见性
* 抑制了编译器优化，导致指令不会被重排序
* 使用内存栅栏（Memory Barrier）从而使缓存无效
* 由于锁竞争而导致阻塞时，持有锁的线程在释放锁的时候需要告诉操作系统，这个锁可以用了，进而操作系统会唤醒其它被挂起的线程

## 锁的粒度该怎样控制

代码块加锁的粒度应该越小越好，但是如果代码块中加锁的粒度很小（代码中相互竞争的临界资源没有相互的依赖性，可以将每种资源加一把锁），频繁的加锁和开锁也会造成性能的开销，降低CPU的利用率，所以并不是加锁越多越好。同时加锁也会造成代码的复杂性，这就是简单性和性能之间存在的互相制约。当我们实现某个同步策略时，一定不要盲目的为了性能而牺牲简单性。

## 可见性和原子性

   `volatile`保证了属性的可见性，但是不能保证某个操作的原子性。如下所示：

```java
   volatile long number;
   //...
   public void addMethod() {
    number++;
   }
```

  这里的`number`就是可见的，但是`addMethod`这个方法不是线程安全的，也就是说`number++`这个操作不是原子操作，因为它只是保证了，线程`A`对`number`的操作对线程`B`是可见的，但是不能保证在线程`A`对`number`操作的时候，线程`B`也可以对`number`进行操作。比如线程`A`和`B`同时读取了`number`的数值，发现它是`12`，这时线程A和线程B

>当执行时间较长的计算或者可能无法快速完成的操作时一定不要加锁，比如：网络IO或者控制台IO。

## 如何正确地发布对象

 这里的发布指的是将某个类的属性值公开。这里如果没有正确的公开，那么就会造成线程不安全。比如：

```java
   public class Holder {
      private int n;
      public Holder (int n ) {this.n = n;}
      public void assertSanity() {
       if(n != n)
       throw new AssertionError("This statement is false.");
      }
   }
```

这里如果在线程A中创建对象，这时线程B调用这个对象的`assertSanity`方法，那么这个方法就可能会触发断言，也就是说在对象的创建过程中，它的属性n的值可能还没有确定，`this.n = n;`。这个代码还没有被执行，而在执行`n != n`的过程中执行了。

## 对象的可见性

   对象的引用对另外的线程可见，并不意味着对象的状态对另外的线程可见。
   As we’ve seen, that an object reference becomes visible to another thread does not necessarily mean that the state of that object is visible to the consuming thread。

## 不可变对象的线程安全性

   不可变对象在正确的初始化之后是线程安全的，所以发布的时候就不必使用锁机制。但是如果某个引用是不可见的，并且引用的对象是可变的，那么在这个不可变的引用在发布的时候也需要加锁。

## 哪些操作必须是原子的？

   如果有两个变量，其中一个变量值的更改会影响另外一个变量的，那么如果要同时改变这两个变量，那么它们需要是原子的，否则其中一个变量改变，而另外的一个变量没有变，那么从这个没有变化的变量中取到的值就有可能是过期的值。比如：

* 我们给每个请求都做一个**标记**，如果某个请求和上一个请求的标记相同，那么就从**缓存**去取这个结果。在这里，这个标记和这个结果是一体的，所以对它们两个的操作必须是原子操作，否则就不能保证取出的结果就是正确的。因为有可能**标记变了，但是缓存还没有变。**
* 我们有一个Range这样的对象，它有一个**下界**和一个**上界**，上界要大于下界，所以对上下界的操作就必须保证是原子的，否则如果一个线程改变了下界，这时上界没有跟着变化，就可能会造成下界大于上界的情况。

也就是说：

> 是规则和限制产生了必须要原子操作的需要，这就是线程安全的需要。

## 线程安全是有粒度的

某个类不是线程安全的，但是如果封装它的类做了线程安全的处理，那么使用它的时候也就是线程安全的了：

```java
public class PersonSet {
   @GuardedBy("this")
   private final Set<Person> mySet = new HashSet<Person>();
   public synchronized void addPerson(Person p) {
        mySet.add(p);
    }
    public synchronized boolean containsPerson(Person p) {
        return mySet.contains(p);
    }
}
```

## 如果某个类包含了一个线程安全的类，那么它不一定是线程安全，比如：

两个`AtomicLong`类型的变量，这两个变量有关联，那么就必须让对这两个变量的操作变为原子化之后才可以是线程安全的，否则仍然不是线程安全的。

## 给线程安全的类添加线程安全的方法

如果要给线程安全的类添加线程安全的方法，那么最好不要使用类扩展，因为如果使用类扩展，那么原来类的线程安全策略做了改动，那么被扩展的类就失效了，比如改了线程安全所用的锁，有些时候这些错误还很难被发现。同时需要特别注意多个线程对线程安全类的同时操作，例如：

```java
public static Object getLast(Vector list) {
int lastIndex = list.size() - 1;
return list.get(lastIndex);
}
public static void deleteLast(Vector list) {
int lastIndex = list.size() - 1;
list.remove(lastIndex);
}
```

上面两个方法都是针对于线程安全的`Vector`所做的，那么它们是线程安全的吗？答案是否定的。
比如在`getLast`方法中，如果线程A在执行完`int lastIndex = list.size() - 1;`之后恰巧有线程B也对这个Vector做了`deleteLast`操作，那么就可能引起`list.get`越界的情况。也就是说，所有针对同一个Vector的操作都应该是原子的。所以正确的做法应该是这样子的：

```java
public static Object getLast(Vector list) {
synchronized (list) {
int lastIndex = list.size() - 1;
return list.get(lastIndex); }
}
public static void deleteLast(Vector list) {
synchronized (list) {
int lastIndex = list.size() - 1;
list.remove(lastIndex);
          }
}
```

这样通过给`Vector`加锁就确保了这些方法的操作是线程安全的。下面还有个很类似的问题：

```java
for (int i = 0; i < vector.size(); i++) {
    doSomething(vector.get(i));
}
```

这里也是非线程安全的，因为不能保证在执行`vector.size()`和`vector.get(i)`之间不会有另外一个线程对`vector`做其它的操作。这个问题的解决思路和上面是一样的：

```java
synchronized(vector) {
for (int i = 0; i < vector.size(); i++) {
doSomething(vector.get(i));
     }
}
```

加锁之后，就可以保证`vector.size()`和`vector.get(i)`的操作是原子的，中间不会有其它的线程会对`Vector`做响应的操作。

## 某个方法加了同步锁就一定是线程安全的吗？

答案是否定的。因为**加锁实现的互斥是基于锁的，多个线程必须使用同一把锁才可以实现互斥。**，比如：

```java
@NotThreadSafe   public class ListHelper<E> {
public List<E> list = Collections.synchronizedList(new ArrayList<E>());
...  public synchronized boolean putIfAbsent(E x) {
boolean absent = !list.contains(x); if(absent)
    list.add(x);
    return absent;
  }
}
```

这个`putIfAbsent`中的加锁并不能保证`ListHelper`的线程安全，因为这个锁是对象锁，锁住的是`ListHelper`，而并没有锁住真正需要锁的`list`上。客户端如果有多个线程同时对list做其它的操作，那么就不能保证线程的安全性。这时争取的做法是：

```java
@ThreadSafe 
public class ListHelper<E> {
public List<E> list = Collections.synchronizedList(newArrayList<E>()); 
...public boolean putIfAbsent(E x) {
synchronized (list) {
    boolean absent = !list.contains(x);
       if (absent) {
          list.add(x);
         }
      return absent;
    }
  }
}
```

这样就保证了putIfAbsent的线程安全性。但是这种通过客户端加锁的方法不是很可靠，因为你不能确定客户端做怎样的操作，有时候会造成死锁。

## 最理想的并发是什么样的？

   The best way to implement concurrency is to reduce the interactions and inter-dependencies between your concurrent tasks。实现并发最好的方式就是避免并发任务之间的交互和相互之间的依赖。

## 容易被忽略的线程安全问题

```java
public class HiddenIterator {
@GuardedBy("this")
private final Set<Integer> set = new HashSet<Integer>();
public synchronized void add(Integer i) {
    set.add(i);
}
public synchronized void remove(Integer i) {
    set.remove(i);
}
public void addTenThings() {
    Random r = new Random()
    for (int i = 0; i < 10; i++)
    add(r.nextInt());
    System.out.println("DEBUG: added ten elements to " + set);
} }
```

这里同样是非线程安全的，因为在执行

```java
System.out.println("DEBUG: added ten elements to " + set);
```

的时候，系统会默认调用`StringBuilder.append(Object)`方法，在这个方法里面会再次调用Object的`toString`方法，在这个`toString`方法内部将会调用迭代器方法并且生成相应的字符串（容器的hashCode和equals方法也有相似的问题）。所以这里是非线程安全的，可能会抛出`ConcurrentModificationException`方法。也就是说如果一个状态和保护这个状态的同步代码之间相隔越远，那么开发人员就越容易忘记在访问这个状态时使用正确的同步。这时如果将HashSet用`synchronizedSet`来封装一下，那么就不会忘记了。

>封装对象的状态有助于维持不变形条件；封装对象的同步机制有助于确保实施同步策略。

## ConcurrentHashMap

`ConcurrentHashMap`的出现就是为了解决同步容器性能差的问题。在一些操作中，比如HashMap.get或者List.contains，可能会包含大量的工作，在执行这些大量工作的时间段内，其它的线程都是被阻塞的，这极大的影响了并发的性能。虽然`ConcurrentHashMap`和`HashMap`一样是基于散列的Map，但是它们使用不同的加锁策略来提供更高的并发性和伸缩性，从而使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制成为分段锁（Lock Striping）。

## 什么是工作密取(work stealing)方法，它有什么优点？

在生产者-消费者模型中所有的消费者有一个共享的工作队列。工作密取的每个消费者都含有一个双端队列。如果一个消费者完成了自己工作队列中的所有问题，那么其它就可以从其它的队列**末尾**秘密的获取工作。密取的工作模式比传统的消费者-生产者模式具有更好的可伸缩性，因为工作者线程不会在单个共享的任务队列上发生竞争。在大多数情况下他们都只访问自己的双端队列，从而极大地减少了竞争。当工作者线程要访问另外一个工作者线程的队列时它将从队列的末尾获取工作，因此进一步降低了队列的竞争程度。
