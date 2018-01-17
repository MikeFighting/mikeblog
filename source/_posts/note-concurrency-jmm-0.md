---
title: Java内存模型
date: 2017-12-20 10:22:33
tags: Note
---

## JVM不能保证64位的long型和double型具有顺序一致性

CPU和内存中的通信是通过总线进行的，如果有多个CPU同时请求总线进行读/写总线事务的话，会由总线仲裁（Bus Arbitration）进行裁决，然后获胜的CPU才可以进行数据传递。并且在某个CPU占用了总线之后，其它CPU要请求总线仲裁是要被拒的。这种机制就保证了CPU对内存的访问是以串行的方式进行的。如果有一个32位的处理器，但是要进行64位的写数据操作，这时CPU会将数据分为两个32位的写操作，并且这两个32位数据在请求总线仲裁的时候可能被分配到了不同的总线事务中，所以此时这个64位的写操作就不具有原子性了。这时如果处理器A写入了高32位，在写入低32位期间，处理器B对该数据进行了访问，那么就会产生错误的数据。（Java5之前读写都可以被拆分，Java5之后写操作可以被拆分，但是读操作必须是原子的）。

## volatile

### volatile对变量读写的影响

某个变量如果被声明为volatile，那么对它的读操作和写操作就具有原子性，但是如果是复合操作，那么就不具有原子性，比如：

```java
class VolatileFeaturesExample {
    volatile long v1 = 0L;
    public void set(long l){
        v1 = l;
    }

    public long get(){
        returen v1;
    }

    public void getAndIncrement(){
        v1 ++;
    }
}
```

在这里set和get方法的语义和加了锁是一样的，其中getAndIncrement其实可以分为以下三个部分：

```java
    public void getAndIncrement(){
        long tempt = get();
        temp += 1L;
        set(tempt);
    }
```

这其中get和set方法是加锁的，但是`tempt += 1L;`就不具有原子性了，也就是说volatile让单个读和写操作具有原子性，但是对于复合操作是没有原子性的。

### volatile变量读写的内存语义

>对volatile变量执行写操作，会将线程的本地内存值刷新到主存中；对volatile变量执行度操作，会将变量所在线程的本地内存置为无效，线程然后会从主内存中读取该volatile变量。


