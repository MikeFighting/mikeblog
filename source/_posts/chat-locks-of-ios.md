---
title: 如何选择IOS中的锁？
date: 2017-12-01 17:34:14
tags: Objective-C
---

关于IOS中的锁很多文章都曾谈过，也有很多基本的用法，比如：[IOS中的各种锁](http://www.cocoachina.com/ios/20161129/18216.html),[探讨IOS开发中各种锁](https://www.jianshu.com/p/6773757a6cd5),[ibireme的不再安全的OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)，这里面对每种锁的用法和加解锁的时间都给出了说明，但是对每种锁的适用场景和在此场景下各种锁的优劣，都没有给予说明，导致我们虽然知道锁的基本用法，但是在选择的时候还是无从下手，本文就是为了解决这个问题的。

## IOS中各种锁的试用场景

### OSSpinLock和dispatch_semaphore

从ibireme的测试中，我们知道自旋锁的加锁和解锁时间是最快的，为什么自旋锁会这么快呢？我们知道当多线程为了争夺一个锁，这时没有争到锁的线程可能会挂起，如果锁被释放了，那么其它被挂起的线程就会被重新唤醒来重新竞争锁。在线程被挂起和重新唤醒的过程中会发生上下文切换，上下文切换是很消耗性能的。如果说每次获取锁之后执行的任务时间很短，那么这些上下文切换造成的性能损耗就会很大。而OSSpinLock这种锁，不会让线程挂起，而是会让线程自旋，不断的去判断锁的状态，如果锁可用就去获得锁，如果不可用就会一直自旋。这样就没有了上下文切换造成的性能损耗，既然自旋锁的加锁开锁速度最快，那么是不是说所有涉及线程安全的地方都应该使用自旋锁呢？不是的。如果说加锁之后任务执行的时间很长，那么所有的没有得到锁的线程都在自循空等，那么就会造成CPU资源的浪费，这些浪费还会影响执行任务线程的执行。这时就可以考虑使用信号量来作为锁，信号量也是非常快的，但是它会让线程休眠而不是空旋，并且在现在比较智能的处理器上，如果使用信号量来加锁，那么它会先让各自的线程先自旋一段时间，如果这个时间之内，线程还没有获得锁，那么就让其休眠。

>如果任务执行的时间非常短（比如给对象的属性赋值），并且资源的竞争很激烈，那么最好使用OSSpinLock，否则使用dispatch_semaphore较好

### NSCondition和NSConditionLock

条件锁是为了解决生产者消费者问题而产生的一类锁。也就是说在生产者消费者模式中，生产者的生产量达到了消费者可以消费的水平时就释放锁，否则就持有锁。消费者只有在生产者释放锁之后才可以进行消费，并且在消费完成之后就也释放对生产者的锁，然后生产者就可以重新进行生产。也就是说条件锁解决了多个相互关联任务之间的执行顺序问题。

### @synchronized

用@synchronized包住需要进行同步的代码是非常简单的方法同时也不必考虑加锁之后不能释放锁的误操作，但是利用这种方式加锁和开锁需要较大的性能损耗，

## 参考资料

http://www.cocoachina.com/ios/20161129/18216.html</br>
https://stackoverflow.com/questions/195853/spinlock-versus-semaphore</br>
https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1</br>