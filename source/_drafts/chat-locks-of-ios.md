---
title: 如何选择iOS中的锁？
date: 2017-12-01 17:34:14
tags: Objective-C
---

关于IOS中的锁很多文章都曾谈过，也有很多基本的用法，比如：[iOS中的各种锁](http://www.cocoachina.com/ios/20161129/18216.html),[探讨iOS开发中各种锁](https://www.jianshu.com/p/6773757a6cd5),[ibireme的不再安全的OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)，这里面对每种锁的用法和加解锁的时间都给出了说明，但是对每种锁的适用场景和在此场景下各种锁的优劣，都没有给予说明，导致我们虽然知道锁的基本用法，但是在选择的时候还是无从下手，本文就是为了解决这个问题的。

## iOS中各种锁的试用场景

### OSSpinLock和dispatch_semaphore

从ibireme的测试中，我们知道自旋锁的加锁和解锁时间是最快的，为什么自旋锁会这么快呢？我们知道当多线程为了争夺一个锁，这时没有争到锁的线程可能会挂起，如果锁被释放了，那么其它被挂起的线程就会被重新唤醒来重新竞争锁。在线程被挂起和重新唤醒的过程中会发生上下文切换，同时需要重新载入线程所属的TCB（Thread Control Block），上下文切换是很消耗性能的，这通常需要 10 微秒左右，而且至少需要两次切换。。如果说每次获取锁之后执行的任务时间很短，那么这些上下文切换造成的性能损耗就会很大。而OSSpinLock这种锁，不会让线程挂起，而是会让线程自旋，不断的去判断锁的状态，如果锁可用就去获得锁，如果不可用就会一直自旋，它的内部其实是使用CAS这种无锁化编程来实现的，伪代码如下：

```objc
// 一开始没有锁上，任何线程都可以申请锁  
bool lock = false;
do {
    while(test_and_set(&lock);  //test_and_set是一个CAS操作
        Critical section  // 临界区
    lock = false; // 相当于释放锁，这样别的线程可以进入临界区
        Reminder section // 不需要锁保护的代码
}
```

这样就没有了上下文切换造成的性能损耗，既然自旋锁的加锁开锁速度最快，那么是不是说所有涉及线程安全的地方都应该使用自旋锁呢？不是的。如果说加锁之后任务执行的时间很长，那么所有的没有得到锁的线程都将处于忙等状态，这会造成CPU资源的浪费。这时就可以考虑使用信号量来作为锁，信号量也是非常快的，但是它会让线程休眠而不是空旋，并且在现在比较智能的处理器上，如果使用信号量来加锁，那么它会先让各自的线程先自旋一段时间，如果这个时间之内，线程还没有获得锁，那么就让其休眠。[信号量的内部实现是`futex`进行的](http://man7.org/linux/man-pages/man2/futex.2.html)。

>如果任务执行的时间非常短（比如给对象的属性赋值），并且资源的竞争很激烈，那么最好使用OSSpinLock，否则使用dispatch_semaphore较好

### NSCondition和NSConditionLock

条件锁是为了解决生产者消费者问题而产生的一类锁。也就是说在生产者消费者模式中，生产者的生产量达到了消费者可以消费的水平时就释放锁，否则就持有锁。消费者只有在生产者释放锁之后才可以进行消费，并且在消费完成之后就也释放对生产者的锁，然后生产者就可以重新进行生产。也就是说条件锁解决了多个相互关联任务之间的执行顺序问题。

### pthread

`pthread`可以实现读写锁，pthread_mutex的`pthread_mutexattr_settype`可以设置互斥锁的各种属性

```objc
/*
 * Mutex type attributes
 */

#define PTHREAD_MUTEX_NORMAL       0
#define PTHREAD_MUTEX_ERRORCHECK   1
#define PTHREAD_MUTEX_RECURSIVE    2
#define PTHREAD_MUTEX_DEFAULT      PTHREAD_MUTEX_NORMAL

pthread_mutexattr_t attr;  
pthread_mutexattr_init(&attr);  
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL);  // 定义锁的属性
pthread_mutex_t mutex;  
pthread_mutex_init(&mutex, &attr) // 创建锁
pthread_mutex_lock(&mutex); // 申请锁  
// 临界区
pthread_mutex_unlock(&mutex); // 释放锁
```

这里可以使用`PTHREAD_MUTEX_RECURSIVE`来创建可重入的递归锁。

### @synchronized

用@synchronized包住需要进行同步的代码是非常简单的方法同时也不必考虑加锁之后不能释放锁的误操作，但是利用这种方式加锁和开锁需要较大的性能损耗，并且很可能导致死锁。[关于@synchronized，这儿比你想要知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

### 互斥锁

互斥锁的底层实现也是`sys_futex`这个系统调用。

### NSThread

NSThread内部其实就是通过`pthread_mutex_lock`来实现的。

```objc
#define    MLOCK \
- (void) lock\
{\
  int err = pthread_mutex_lock(&_mutex);\
  // 错误处理 ……
}
```

### NSCondition

NSCondition也是使用pthread来进行实现的：

```objc
- (void) signal {
  pthread_cond_signal(&_condition);
}
// 其实这个函数是通过宏来定义的，展开后就是这样
- (void) lock {
  int err = pthread_mutex_lock(&_mutex);
}
```

### NSRecursiveLock

NSRecursiveLock也是使用pthread来实现的。

总结：iOS中的锁，其实只有两种信号量底层的`lll_futex_wait`（信号量），`os_unfair_lock_t`(OSSpinLock)以及`pthread_mutex_lock`这三种锁，其实`futext`要比`pthread`性能要高，因为它很多情况下是在用户空间的，它会造成最少的用户态和内核态之间的切换。

### 公平锁和非公平锁？

公平锁是说，如果某个锁被持有，或者有线程在阻塞队列中，那么新来的线程就必须被放到队列中，这就是说不允许插队。在非公平锁中，只有当锁被某个线程持有的时候才会把新来的线程放到队列中，也就是说，如果某一时刻锁被释放了，那么新来的线程就可以插队到最前面去争夺这把锁。这种可以**插队**就是所谓的不公平。在非公平锁中可能会出现饥渴或者优先级翻转的问题，但是非公平锁是更快的。

## 参考资料

http://www.cocoachina.com/ios/20161129/18216.html</br>
https://stackoverflow.com/questions/195853/spinlock-versus-semaphore</br>
https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW1</br>
https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW2
https://bestswifter.com/ios-lock/#
https://developer.apple.com/documentation/os/1646466-os_unfair_lock_lock?language=objc

