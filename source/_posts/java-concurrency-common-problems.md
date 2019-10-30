---
title: Java多线程中的常见问题
date: 2018-02-02 21:27:44
tags: Java
---

## long和double不具有顺序一致性

Java中的long和double无论在32位机器还是64位机器上都是8个字节，因为它需要JVM提供这种平台无关的抽象。那么，我们来看看为啥8个字节的读写操作在32位机器上不具有一致性。CPU和内存的通信是通过总线进行的，总线的宽度是固定的并且只有一条，如果有多个CPU同时请求总线进行读/写事务的话，会由总线仲裁（Bus Arbitration）进行裁决，获胜的CPU才能进行数据传递。并且如果某个CPU占用了总线，那么其它CPU要请求总线仲裁是要被拒的。这种机制就保证了CPU对内存的访问以串行的方式进行。如果有一个32位的处理器，但是要进行64位的写数据操作，这时CPU会将数据分为两个32位的写操作，并且这两个32位数据在请求总线仲裁的时候可能被分配到了不同的总线事务中，所以此时这个64位的写操作就不具有原子性了。这时如果处理器A写入了高32位，在写入低32位期间，处理器B对该数据进行了访问，那么就会产生错误的数据。（Java5之前读写都可以被拆分，Java5之后写操作可以被拆分，但是读操作必须是原子的）。如果要将变量前面加上volatile修饰符，那么对其操作将具有原子性，因为加上volatile之后，对其进行读写操作就像是“加锁”了一样。

## ABA问题

乐观锁在实现的过程中利用到了CAS机制，这个机制是这样的：

>有三个操作数：内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做或者使用新数值再次进行尝试。

这时就会出现一种问题：比如线程T1刚开始判断某个变量X，刚开始的数值是A，将要把它赋值为D。于此同时线程T2进来将X变为B，最后又变为A。那么线程T1在使用CAS判断的时候就会认为该数值没有变化（也就是说这个变化为B值的过程被忽略了），然后线程T1就把X的值赋成了D。[在链表等场景下，这可能会引发问题](https://www.cnblogs.com/549294286/p/3766717.html)。为了解决这个问题，从Java 1.5开始，JDK给我们提供了AtomicStampedReference，它的解决思路是这样的：给该类提供一个变量，每次数值变化的时候就将该变量加一，然后在利用CAS机制进行判断的时候不光判断X的值，还要判断该变量的值，这样就避免了该问题：

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
            newStamp == current.stamp) ||
            casPair(current, Pair.of(newReference, newStamp)));
}
```

## 双重锁定问题

在项目开发过程如果某个类很Expensive，我们希望创建一次，然后一直使用它，早期人们通常这样写：

```java
    private static ExpensiveObj instance;
    public static ExpensiveObj getInstance() {

        if (instance == null) { // 1
            instance = new ExpensiveObj(); // 2
        }
        return instance;
    }
```

显然这种方式非线程安全的，当线程A和线程B同时进入步骤1的时候，会发现instance没有被创建，这时它们同时创建instance。这时有人就提出了一种线程安全的写法：

```java
    private static ExpensiveObj instance;
    public synchronized static ExpensiveObj getInstance() {
        if (instance == null) {
            instance = new ExpensiveObj();
        }
        return instance;
    }
```

这种实现方法是线程安全的，但是因为使用`synchronized`加了锁，所以每次调用`getInstance`方法，不论`instance`已经被创建，都会加锁。这种频繁的加锁，解锁，会造成性能开销，因此有人提出了一种看似完美的解决方案：

```java
    private static ExpensiveObj instance;
    public  static ExpensiveObj getInstance() {
        if (instance == null) {
            synchronized (DoubleCheck2.class) {
                if (instance == null) {
                    instance = new ExpensiveObj();
                }
            }
        }
        return instance;
    }
```

在这个方案里，如果`instance`已经被创建，那么就不需要加锁了，直接返回。如果没有创建，那么再加锁创建对象，因为这个对象只被创建一次，所以这个锁只会使用一次。这个看似完美的解决方案，其实有一个致命的缺陷：**因指令重排而造成未被初始化成功的对象逸出。**我们先用伪代码来看下对象的创建过程：

```java
memory = alloc(); //1: 开品内存空间
ctorInstance(memory); //2: 初始化对象
instance = memory; //3: 将instance指定为刚才分配的内存地址
```

但是在步骤2和步骤3之间可能会发生指令重排，从而造成如下的执行顺序：

```java
memory = alloc(); //1: 开品内存空间
instance = memory; //3: 将instance指定为刚才分配的内存地址
ctorInstance(memory); //2: 初始化对象
```

这时是如果线程A正在创建对象，同时线程B调用了`getInstance`方法，那么这时`instance != null`。所以就会直接使用这个对象，然而此时线程A可能没有完成对象的初始化，所以线程B使用的就是没有被完全初始化的对象，所以要想解决这个双重锁定问题只需要避免指令重排即可，我们将`instance`声明为`volatile`就行了：

```java
private volatile static Instance instance;
```

关于指令重排，volatile关键字可以查看[我的另外一篇文章：JMM](https://mikefighting.github.io/2017/12/20/note-concurrency-jmm-0/)。其实解决这种某个类只被创建一次的问题，也可以使用static来实现：

```java
    private static class ExpensiveObjHolder {
        public static ExpensiveObj instance = new ExpensiveObj();
    }
    public  static ExpensiveObj getInstance() {
        return ExpensiveObjHolder.instance; // 这里将导致ExpensiveObjHolder类被实例化
    }
```

这里在调用`getInstance`方法的时候，JVM会执行**类的初始化**，此时JVM回去获取一把锁，这个锁可以保证其它线程在此时不会使用该对象，其实发生了指令重拍，其它对象也不知道，因为它必须等JVM创建完成之后才可以使用。

## 对线程安全的类操作不一定都是线程安全的

比如AtomicInteger是线程安全的，但是下面的操作就不是线程安全了。

```java
AtomicInteger atomicInteger = new AtomicInteger(3);
    if (atomicInteger.get() == 3) { // 1
        atomicInteger.set(4); // 2
        //...
    }
```

因为在线程A执行步骤1的时候，线程B也可能进入这个判断中，那么此时线程B也会再执行一次2，如果这个方法内部还有一些其它需要互斥的操作，那么就可能造成线程不安全的一系列问题。也是就是说：

> 线程安全操作 + 线程安全操作 ≠ 线程安全操作

## 增加CPU不能持续程序效率

根据Amdahl定律，增加处理器之后效率提升的最大幅度为：

>串行执行代码所占百分比的倒数。

关于Amdahl定律的详细解释请看我的[另外一篇文章](https://mikefighting.github.io/2017/12/14/note-java-concurrency-in-practice-2-2/)，增加处理器的个数和资源利用率的关系如下：

![amdahl_principle](http://upload-images.jianshu.io/upload_images/1513759-d927f9340e4f6710.png)

从中我们可以看出，串行部分占比越多增加CPU的个数越没有用。同时CPU的利用率下降的越快，也造成了越大的费用支出。

## 多线程不一定快

越多的线程就可能造成越多的上下文切换，上下文切换消耗的时间在0.1毫秒到1毫秒之间，这对CPU来说已经是巨大的损耗了，频繁的上下文切换所造成的性能损耗可能不如单线程运行的程序，同时多线程可能还涉及到加锁的需要，这就操作成了更大的性能损耗（因为某些锁的底层实现是需要进行相应的系统调用）

## 参考资料

 1. https://www.cnblogs.com/549294286/p/3766717.html
 2. 《Java并发编程的艺术》
 3. 《Java并发编程实战》
