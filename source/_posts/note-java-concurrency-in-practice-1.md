---
title: Java结构化并发应用程序
date: 2017-12-01 20:50:39
tags: Notes
---
# Java结构化并发应用程序

## 线程池和队列的关系

线程池和队列之间的关系是很紧密的。队列是用来放任务的，它有并行和串行之分。其中并行队列中的任务可以并发的执行；串行队列中的任务只能按照顺序一个一个执行，正是因为这个原因，串行队列也可以实现线程安全，也可以作为锁来用。而线程池就是很多的线程的容器，这些线程负责从队列中取出任务执行任务并且返回线程池以等待下个任务的到来。
在Java中通常会有如下几种创建线程池的方法：

```java
public static ExecutorService newFixedThreadPool(int nThreads) // 创建的线程数量是固定的
public static ExecutorService newWorkStealingPool(int parallelism) // 利用所有可用的处理器资源创建一个'工作密取'的线程池
public static ExecutorService newSingleThreadExecutor() // 创建一个单线程的线程池，放入线程池中的任务顺序执行
public static ExecutorService newCachedThreadPool() // 创建一个缓存的线程池，如果之前有可用的线程就用，如果没有就重新创建，如果执行的任务量小并且多的时候用这个线程池会提高性能。如果一个线程在60s之内没有被使用，那么这个线程将会被中断并且被移除线程池。所以说如果这个线程池如果一直是idel状态的时候，那么它不会消耗任何的资源
public static ScheduledExecutorService newScheduledExecutor() // 创建一个具有定时功能的线程池
//...
```

在IOS开发中GCD就是典型的例子：

```objc
    dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```

这个函数调用时候是这样子的：

```objc
dispatch_queue_t concruntQueue = dispatch_queue_create("come.mike.fighting0.com", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(concruntQueue, ^{

        NSLog(@"我正在做一项耗时的任务");

    });
```

这段代码就是说吧一个耗时的任务放到了一个并行的队列中，然后调用`dispatch_async`，在调用这个方法时系统会自动的给我们创建好一个线程池并且从其中取出一个线程，来执行我们的任务。这样，我们就不用自己再去创建并管理线程了，避免了不必要的错误并且避免了频繁创建线程所带来的开销，同时避免了任务到来的时候再去创建线程从而造成一定程度的响应延迟。

## Executor的生命周期

`Executor`的创建在上文中已经说明了，下面说下它的关闭。`Executor`的终止方式有以下两种：

* 缓慢关闭，让已经执行的任务执行完毕，然后不再接受正在等待的任务或者新来的其它任务(`shutdown`方法)
* 暴力关闭，直接关闭线程池，不管已经执行的任务(`shutdownNow`方法)

## Timer和SecheduledThreadPoolExecutor的对比

>因为Timer的调度机制是基于绝对时间而不是相对时间的，因此任务的执行对系统的时钟很敏感，而`SecheduledThreadPoolExecutor`是基于相对时间调度的，所以更加准确。</br>

* Timer会将所有的定时任务都放到一个线程中去执行，所以如果某个任务的执行时间长于所设定的时间间隔那么这个Timer就会不准确。而线程池就能很好的解决这个问题，因为它是在多个线程中执行不同的任务的，所以各个任务之间彼此没有影响。
* TimerTask如果抛出一个异常，那么Timer不会处理它，反而会终止所有的任务，包括正在执行的任务和将要执行的任务。在这之后也没有可以恢复Timer的方式。

那么问题来了，在Java中如果要实现自己的调度任务不使用Timer，该使用什么呢？应该使用`DelayQueue`，它内部的每个对象都有一个延迟时间的方法。

## 任务和线程处理中断的方式

虽然每个任务都在一个线程中执行，但是这个线程并不被这个任务所拥有。拥有这个线程和管理这个线程的*主人*是线程池，所以在遇到中断的时候，通常会将其抛出，然后让上层的代码来处理中断。举个例子：你在一个朋友家玩耍，这时忽然来了一个收租金的人大吵大闹要交房租（中断），这时你不应该处理，而是应该保留这个现场，并且把问题抛给你的朋友，因为这是他的家。这也就是什么很多阻塞库框架都会在遇到中断的时候抛出来`InterruptedException`，以便上层代码进行处理（尽快的退出，并且将中断尽快的传递给上层也是最温和的响应策略）。也就是说任务本身对中断不应该做任何的处理，不应该对中断策略做任何的假想，除非这个框架的中断处理策略已经定了，不需要再将中断抛给上层代码了。除了将中断传递给上层的调用者之外，任务还需要保存中断的状态，以备后续上层代码的处理，保存状态的方式为：

```java
Thread.currentThread().interrupt();
```

调用之后就会保持线程的中断状态，恢复中断状态的目的就是让调用栈中更高层的代码看到引发了一个中断，并且这个线程的状态是`interrupted`的。

```java
Thread myThread =  new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    throw  new InterruptedException();
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                    System.out.println("thread status:" + Thread.currentThread().isInterrupted());
                    e.printStackTrace();
                }
            }
        });
myThread.run();
```

这段代码如果不加`Thread.currentThread().interrupt();`，那么下面的`Thread.currentThread().isInterrupted()`就将会返回`false`。如果没有确定上层代码是否要处理异常，那么切记不能catch中这个中止的异常而不做任何的事情。

## Executor的作用

既然已经有了线程，那么Executor的作用是什么呢？它是将任务的提交和任务的执行分离开了。也就是说把复杂的业务过程分割开了，这样就更加便于我们修改执行策略。
