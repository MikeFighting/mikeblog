---
title: Java中的活跃性
date: 2017-12-12 13:15:34
tags: Notes
---

在多线程开发中，我们往往为了安全性而去加锁，如果锁过多，就可能出现**顺序死锁**。如果不适用锁，使用信号量和线程池来限制对资源的访问，那么又可能出现**资源死锁**。那么究竟怎样判断死锁？死锁的种类都有哪些？怎样避免死锁呢?

## 死锁

我们如果把每个线程都想像成有向图中的一个点，如果线程A等待线程B所占用的资源，那么就从A向B画一条直线，如果最终这个图形成了一个环形，那么就出现了资源的相互依赖，就造成了死锁。

## 死锁之后的处理形式

死锁之后该怎么处理分为两种方式，第一种方式就是什么都不做不了，应用程序将到此结束（也可能是某个子系统停止或者性能降低），直到重新启动，才会解除本次死锁。第二种方式就是干涉死锁。比如数据库操作在两个事务之间出现了死锁，那么数据库服务器会选择一个牺牲者并且放弃这个事务。作为牺牲者的事务将放弃它的所有资源，从而使其它事务继续进行。让后等待其它任务执行完成之后再去执行这个被牺牲了的任务。

## 顺序死锁

如果有left和right两把锁，同时有A线程和B线程去访问，如果按照下面的顺序就可能造成死锁。

![order_dead_lock](http://upload-images.jianshu.io/upload_images/1513759-56118e0094e4a64d.png)

线程A持有了left锁，再获取了right锁的时候才可以进行下一步的执行，并且只有获得了right锁才可以释放掉left锁。线程B已经获取了right锁，在获取了left锁的时候才可以进行下一步的执行，也只有获取了right锁才可能释放掉right锁。所以就造成了最后的死锁。这个死锁引起的原因就是锁的顺序不一致，也就是说在使用锁进行同步的过程中如果有两把锁，那么锁的顺序需要保持一致，否则就可能造成死锁。代码如下

```java
public class LeftRightDeadLock {

    private final Object left = new Object();
    private final Object right = new Object();

    public void leftRight() {
        synchronized(left) {
            synchronized(right) {
                doSomething();
            }

        }
    }

    public void rightLeft() {
        synchronized(right) {
            synchronized(left) {
                doSomethingElse();
            }
        }
    }
}
```

## 动态的锁顺序死锁

有些时候我们没有很明确的在两个不同的方法中使用两把锁，但是仍然可能造成死锁，这种死锁往往不容易被发现，比如我们要给将账户A的钱转给账户B，那么我们可以使用下面的方法来确保转账的原子性，比如：

```java

public void transferMoney(Account fromAccount, Account toAccount,
DollarAmount amount) throws InsufficientFundsException {
synchronized (fromAccount) {
    synchronized (toAccount) {
if (fromAccount.getBalance().compareTo(amount) < 0) {
    throw new InsufficientFundsException();
    }
else {
    fromAccount.debit(amount); toAccount.credit(amount);
        }
     }
  }
}
```

这个死锁的原因就是可能调用方在两个线程中使用的参数顺序可能相反，这就造成死锁，因为我们不能确定调用方是怎么调用我们写的接口的。比如下面的调用：

```java
A: transferMoney(myAccount, yourAccount,10);
B: transferMoney(yourAccount, myAccount,20);
```

这时就造成了死锁，并且这种死锁是在一般情况下是不会发生的，这就造成了难以排查的错误。这时该如何去做呢？我们的目的是想让外界两个参数的改变不会影响到内部锁的顺序，所以，我们可以拿两个入参的`identityHashCode`去作为判断条件，根据hash值的大小来改变加锁的顺序。当然，这里面可能有哈希碰撞的情况（这种情况发生的几率是非常低的），如果有这种情况的出现，那么就给这两个同步操作外部再加一个锁，这样来确保这个操作的原子性，就不会有死锁的情况了。这里面如果加锁的两个对象有唯一的键值，那么就可以直接用其键值，这样就不必再使用额外的锁了。

## 协作对象之间发生的死锁

比如下面的Taxi和Dispatcher对象都使用了锁，并且它们之前是相互协作的。

```java
class Taxi {
@GuardedBy("this") private Point location, destination; private final Dispatcher dispatcher;
public Taxi(Dispatcher dispatcher) {
    this.dispatcher = dispatcher;
    }
public synchronized Point getLocation() {
    return location;
   }
public synchronized void setLocation(Point location) {
    this.location = location;
if (location.equals(destination)) {
    dispatcher .notifyAvailable (this );
 }

   }
}
class Dispatcher {
@GuardedBy("this") private final Set<Taxi> taxis; @GuardedBy("this") private final Set<Taxi> availableTaxis;
public Dispatcher() {
       taxis = new HashSet<Taxi>(); availableTaxis = new HashSet<Taxi>();
    }
public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis .add(taxi);
    }
public synchronized Image getImage() {
    Image image = new Image();
    for (Taxi t : taxis) {
     image.drawMarker(t.getLocation()); return image;
  }

   }
}
```

在这里面`setLocation`方法需要先获取`Taxi`的锁，再获取`Dispatcher`的锁。而`getImage`方法需要先获取`Dispatcher`的锁，然后再获取`Taxi`的锁，这样就可能造成上文中所说的顺序锁的问题。并且这种锁是更加难以排查的。所以最好不要使用`synchronize`(管程)。

>在持有锁的过程中调用某个外部方法，那么将可能会出现活跃性的问题。

## 丢失信号死锁

多线程访问某个资源，在有条件谓词作为前置条件，如果条件为假，那么我们会调用wait方法将线程阻塞。如果某个线程将条件变为了真，并且这个wait的线程没有收到这个信号。那么原来wait的线程将会永远等待下去，进而导致死锁。也就是说，线程A通知了一个条件队列，而线程B随后进入这个条件队列，但是线程B将被阻塞而不能执行，因为其需要等待另外一个通知的到来。

## 开放调用

之所以出现上述协作对象之间发生的死锁，是因为在调用另外一个对象的方法的过程中，已经持有了一把锁。这种调用称作不开放，所谓的开放调用就是指：在调用某个方法的时候不需要持有锁。通常来说开放调用要比非开放调用更加安全，更加不容易产生死锁，所以我们要尽可能地使用开放调用。我们可以使用开放调用的方法来解决上述遇到的问题：

```java
@ThreadSafe
class Taxi {
@GuardedBy("this")
private Point location, destination; private final Dispatcher dispatcher;
public synchronized Point getLocation() {
        return location;
    }
public void setLocation(Point location) {
    boolean reachedDestination; synchronized (this) {
    }
}
    this.location = location;
    reachedDestination = location.equals(destination);
}
if (reachedDestination) {
    dispatcher .notifyAvailable (this );
}

@ThreadSafe
class Dispatcher {
@GuardedBy("this") private final Set<Taxi> taxis;
@GuardedBy("this") private final Set<Taxi> availableTaxis;
public synchronized void notifyAvailable(Taxi taxi) {
availableTaxis .add(taxi); }
public Image getImage() {
    Set<Taxi> copy;
  }
}
synchronized (this) {
copy = new HashSet<Taxi>(taxis);
}
Image image = new Image(); for (Taxi t : copy)
image.drawMarker(t.getLocation()); return image;
```

这样就可以将多个锁区分开来，从而在多个对象调用的时候就不会死锁了。

## 资源死锁

资源死锁的起因也是由于访问资源的原子性和访问资源的顺序所造成的互相牵制。比如：线程A已经建立了和数据库D1的链接，正在尝试连接数据库D2；与此同时，线程B已经建立了和数据库D2的连接，正在尝试连接数据库D1。这时就造成了资源死锁。（当然这和数据库同时连接的个数，以及资源的大小有关。资源越大，连接的个数越多，那么出现死锁的可能性就越少。）
在资源死锁中，还有一种线程饥饿死锁的情况，比如下面的代码：

```java
public class ThreadDeadlock {
ExecutorService exec = Executors.newSingleThreadExecutor();
public class RenderPageTask implements Callable<String> {
    public String call() throws Exception {
        Future<String> header, footer;
        header = exec.submit(new LoadFileTask("header.html"));
        footer = exec.submit(new LoadFileTask("footer.html"));
        String page = renderBody();
        // Will deadlock -- task waiting for result of subtask
        return header.get() + page + footer.get();
   }
 }
}
```

这会出现死锁，因为`header.get()`和`fotter.get()`是阻塞的，它将会等待exec执行完毕，而exec想要执行必须要等到`header.get() + page + footer.get();`执行完毕，这样就造成了线程饥饿死锁。(RenderPageTask是任务1，header.get() + page + footer.get()是任务2)。

## 饥饿

饥饿就是指某个线程始终不能获取其所需要的资源，导致它不能继续执行，引起饥饿的最常见资源就是CPU的时钟周期。Java中提供了10中线程的优先级，然而对应到操作系统中，可能某些优先级会重合（因为操作系统可能没有这么多的优先级）。并且设置线程的优先级可能不会起到明显的效果，反而可能因为优先级翻转而造成死锁，所以我们尽量不要去改动线程的优先级。但是这种情况也不是绝对的，比如有一个CPU密集的后台任务在执行，那么这个任务很可能会和主线程去抢占CPU资源，从而导致主线程响应性降低，为了解决这个问题，我们可以将后台线程的优先级降低，从而提高主线程的响应性。

>尽量避免使用线程优先级，因为这会增加平台依赖性，并且可能会导致活跃性问题。在大多数并发应用程序中，都应该使用默认的线程优先级。

## 活锁

活锁是指没有发生死锁，但是程序一直在重试，并且重试一直错误，导致程序不能正常往下执行。（产生这种情况的原因是对错误的估计不对：本来是不能解决的错误，却以为可以通过重试解决）。
同时，多个线程之间的协作也可能造成死锁，因为可能两个协作的线程都对彼此进行响应，响应完之后使得任何一个线程都不能继续执行，解决这种活锁的问题可以通过在重试机制中引入随机性，也就是说某个重试完之后，另一个线程在随机的时间段之后再进行重试，从而避免了和之前线程的碰撞。