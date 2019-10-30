---
title: 显式锁
date: 2017-12-16 10:59:57
tags: Notes
---

## 内置锁和显示锁

在Java5中`ReentrantLock`相比内置锁有很大的性能提升，但是在Java6中内置锁的性能有了很大的提升，他们的性能相差不大。并且由于`synchronized`是JVM的内置属性，所以它在未来可以进行更深层次的优化。
内置锁相比显式锁拥有以下的优势：

* 结构更加紧凑
* 使用内置锁不用手动进行解锁，所以具有更低的危险性
* 在线程转储(thread dumps)过程中能给出哪些帧获取了哪些锁(`ReentrantLock`在Java6之后也支持)
* 可以检测和识别反生死锁的线程

>当使用某些内置锁不能满足的高级功能时，比如：可定时的，可轮询的，可中断的锁，公平队列，以及非结构的锁时。则需要考虑使用`ReentrantLock`。

## 公平锁和非公平锁

公平锁就是说每个请求锁的线程按照先到先得的顺序获得锁，后到的线程在它前一个线程没有释放锁时是不能获取锁的，没有获取的锁的线程就被放到队列里挂起。非公平锁就是说每个请求锁的线程，如果发现该锁没有被占用就去请求锁，而不管其前面有没有线程在等待，这将提升吞吐量。由于线程的挂起和重新唤醒，线程的调度会带来很大的性能损耗，所以，如果请求锁的平均时间间隔非常短，那么最好使用非公平锁。反之，如果持有锁的时间较长，或者请求锁的时间间隔较长，那么性能的瓶颈就不在切换线程上了，这时就应该使用公平锁，使用非公平锁"插队"的方式所带来吞吐量的提升将会是非常微弱的。内置锁和`ReentrantLock`都没有提供确定的公平性保证，因为一般来说实现总体的公平性已经足够了。

## 条件队列

首先为什么会出现条件队列呢？我们知道要保证某个多线程的方法被调用成功可以使用线程休眠的方式，但是这种方式会有问题：如果休眠的时间过短，那么会导致CPU的资源消耗过高，如果休眠的时间很长，这时如果其它某个线程修改了判断条件，这时休眠的线程不能及时响应，只有在休眠之后才会响应。比如：

```java
@ThreadSafe
public class SleepyBoundedBuffer<V> extends BaseBoundedBuffer<V> {
public SleepyBoundedBuffer(int size) {
    super(size);
}
public void put(V v) throws InterruptedException {
    while (true) {
    synchronized (this) {
        if (!isFull()) {
            doPut(v);
            return;
        }
    }
     Thread.sleep( SLEEP_GRANULARITY );
  }
}
public V take() throws InterruptedException {
    while (true) {
    synchronized (this) {
        if (!isEmpty())
            return doTake();
    }
     Thread.sleep( SLEEP_GRANULARITY ); }
  }
 }
```

使用条件队列就可以解决这种问题：条件队列就是说在条件不满足时候线程休眠(调用wait方法)，当条件满足的时候线程会收到通知并且被唤醒(调用notify方法)，这样就不会产生多余的开销。</br>
调用`wait`方法之后会发生以下事情：

1. 释放锁
2. 阻塞当前线程并等待直到超时
3. 线程被终端或者被一个通知唤醒
4. 唤醒后，wait在返回前还要重新获取锁

在步骤3和步骤4之间可能有另外一个线程获取了锁，并且改变了对象的标志，这个时候其实条件已经变成假的了。或者这个被唤醒的原因并不是条件变成了真，而是其它的线程的某个条件变成了真，那个线程调用了notify或者notifyAll方法。所以说即使线程被notify唤醒了，并不一定是因为条件满足了，所以在唤醒之后还要继续检查条件，这时要将wati放在一个。如下所示：

```java
void stateDependentMethod() throws InterruptedException {
// condition predicate must be guarded by lock
synchronized(lock) {
while (!conditionPredicate()){
    lock.wait();
    // object is now in desired state
  }
}
```