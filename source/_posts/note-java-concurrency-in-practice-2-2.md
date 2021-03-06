---
title: Java并发编程-性能
date: 2017-12-14 16:52:07
tags: Notes
---

并发的目的就是提高系统的性能和响应速度，但是这些都要建立在安全性的基础之上的，也就是说先要保证系统能够正常的运行，满足现有的业务需求，然后再考虑性能。并且在提升性能的同时往往会增加系统的复杂性，因为很多性能的优化需要牺牲掉代码的可理解性和面向对象的原则（同时可能会出现活跃性的问题），增加系统的维护成本，并且有时不会增加系统的性能反而会降低系统的性能。

## 多线程带来的开销

多线程的引入会造成一些额外的开销，比如：线程的创建和销毁，线程的调度，上下文的切换，线程之间的协调（例如加锁，触发信号以及内存同步）。如果这些性能开销大于吞吐量，响应性所带来的性能提升那么就会得不偿失。

## 可伸缩性

什么是可伸缩性？可伸缩性指的是通过增加计算机的资源（例如CPU，内存，存储容量或IO带宽），程序的吞吐量或者处理能力能相应的增加。但是这种伸缩性往往是有极限的，比如下面提到的Amdahl定律。

## Amdahl定律

增加CPU的个数也不能一直提高性能。Amdahl定律就指明了这一点：
![Amdahl公式](http://upload-images.jianshu.io/upload_images/1513759-d38318f874db2020.png)
在这个公式中：*F表示串行执行的部分占所有部分的比重，N代处理器的个数，最后的结果代表和单一处理器时速度的比例。*我们假设两个极限：

* 在处理器个数等于一时，其数值是1，也就是说它的速度没有提升。
* 在处理器个数为正无穷的时候，其数值为1/F。

所以只提高处理器的个数是不不能一直提高执行速度的。我们来举个例子：

放假了班主任给小明同学布置了两份作业：

一、9张英语作业：将英语单词第一单元抄写九遍
二、数学第二单元的课后习题做完

假设做数学习题所用的时间是1小时，写英语单词的时间为每单元1小时，所以说如果有小明一个人来做需要10个小时才能完成。小明很贪玩，到了周日晚上才发现自己作业没有写，小明急了，于是就找同学帮他写英语单词，因为英语单词是抄写9份，所以他找了9个小伙伴给他抄，他自己做数学习题（假设数学题不能多个人来做），这样本来是个小时完成的任务，小明最后用了2小时完成了，按照Amdahl定律，其F是0.1，N是10，所以最后的数值是5.26倍。然后我们假设小明有无数个小伙伴都来帮他抄写英语单词，那么最后他需要的时间就接近于1小时，做以按照Amdahl定律，其F实0.1，N是正无穷，所以最后数值是10，也就是说他的速度是原来的10倍。也就是说无论他找多少个小伙伴都不能突破一小时的时间，**因为这一小时只能是串行执行的，不能由其他的处理器来协助完成。**

然而要能够使用Amdahl定律需要首先估算出串行执行的部分所占的比例。
同时如果增加了处理器的个数，那么每个处理器的利用率都会下降，其中串行所占比例越重的系统，其利用率下降的越厉害。

## 内存的同步

在使用`synchronized`和`volatile`以提供可见性的同时也引入了需要内存同步的问题，因为其使用了内存栅栏（Memory Barrier），使用内存栅栏是可以刷新缓存，使得缓存无效的。某个线程的同步会影响到其它线程的性能，因为同不会增加内存总线上的通信量，总线的带宽是有限的，并且所有处理器都共享这个带宽。但是现在的JVM会对代码做相应的优化，比如下面两段代码：

```java
synchronized (new Object()) {
           // do something
}
```

这里永远不会有任何两个线程会去竞争这个锁的，因为每次进入这个方法都会新建一把锁。

```java
public String getStoogeNames() {
List<String> stooges = new Vector<String>();
stooges.add("Moe");
stooges.add("Larry");
stooges.add("Curly");
return stooges.toString();
}
```

这里使用`Vector`可能会进行四次获取锁和释放锁的操作，但是由于这个`stooges`是一个局部变量，局部变量是属于线程栈的，每个线程中的不一样，所以没有必要加锁。在这两种情况下，如果编译器够智能，就会进行优化从而去掉锁。并且如果编译器没有进行逸出分析，那么也可能进行锁的粒度粗化。也就是说将原来需要加四次锁的地方改为加一次锁。也就是说再非竞争同步的时候，我们就不必担心，JVM已经帮我们做了优化，我们需要关心的就是可能引发竞争的地方。

>在并发程序中，对可伸缩性的最主要威胁就是独占方式的资源锁。

## 锁分段和减少锁的粒度

锁分段的意思是有一个数据集合A,B,C,D,那么线程T1和线程T2一般情况不会同时访问A，而是访问不同的数据，所以在T1和T2访问不同的数据时就是没有必要加锁的，这时可以根据数据集合的量配置若干个锁，比如ConcurrentHashMap中使用的是16个锁来增大吞吐量。而减少锁的粒度是如果之前很多个资源都用同一个锁，并且各个资源之间没有相互的依赖，那么可以将原来的一个锁变为多个锁。锁分段往往将锁放入Collection中，而减少锁的粒度往往是声明若干个锁。*其实锁分段也是一种减少锁粒度的方式。*

## 不能滥用对象池

有时候某个对象会重复的使用，我们为了不重新new对象（事实上Java的分配操作要比C语言的malloc的调用速度还要快）以及垃圾回收带来的开销，常常会建一个池子将已经创建的对象保存进池子以备后用。但是这样做需要考虑以下几点：对象池的大小很重要，如果对象池很小，那么将不会起到相应的作用；如果对象池很大，那么将会占用很多内存资源，这会对垃圾回收器带来压力。如果用在多线程，那么会造成更加严重的性能问题，因为如果不用线程池，那么每个新建的对象就会在线程本地的内存块中。如果使用多线程，那么情况会更加糟糕，因为需要协调每个线程之间的调用，在协调的过程中可能导致某个线程的死锁。并且多个线程之间的同步，如果使用锁，那么及时没有竞争，只是加锁和解锁所带来的性能损耗要比new对象带来的损耗大得多。这看似一个性能优化的技术点，但实际上会导致可伸缩性的问题。

>同步的开销要比new对象的开销少的得多。

## 上下文切换

上下文切换是什么？
锁竞争的过程中会发生上下文切换，越多的上下文切换将会造成越低的吞吐量。如果某个执行IO操作的线程被阻塞了，同时这个线程还持有一把锁，那么如果有越来越多的线程来请求这个锁，也将被阻塞而挂起，这就会增加上下文切换的次数。所以要尽量减少持有锁的时间来减少上下文切换。

参考资料：

1. [线程上下文切换和进程上下文](https://stackoverflow.com/questions/5440128/thread-context-switch-vs-process-context-switch)

2. [上下文切换的定义](http://www.linfo.org/context_switch.html)

3. 


