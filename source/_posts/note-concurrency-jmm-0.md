---
title: Java内存模型
date: 2017-12-20 10:22:33
tags: Note
---

## JVM不能保证64位的long型和double型具有顺序一致性

CPU和内存中的通信是通过总线进行的，总线的宽度是固定的，如果有多个CPU同时请求总线进行读/写总线事务的话，会由总线仲裁（Bus Arbitration）进行裁决，然后获胜的CPU才可以进行数据传递。并且在某个CPU占用了总线之后，其它CPU要请求总线仲裁是要被拒的。这种机制就保证了CPU对内存的访问是以串行的方式进行的。如果有一个32位的处理器，但是要进行64位的写数据操作，这时CPU会将数据分为两个32位的写操作，并且这两个32位数据在请求总线仲裁的时候可能被分配到了不同的总线事务中，所以此时这个64位的写操作就不具有原子性了。这时如果处理器A写入了高32位，在写入低32位期间，处理器B对该数据进行了访问，那么就会产生错误的数据。（Java5之前读写都可以被拆分，Java5之后写操作可以被拆分，但是读操作必须是原子的）。

## volatile

### volatile对变量读写的影响

volatile也被称为轻量级锁，如果某个变量被声明为volatile，那么对它的读操作和写操作就具有原子性，**但是如果是复合操作，那么就不具有原子性**，比如：

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

这其中get和set方法是加锁的，但是`tempt += 1L;`在执行的时候可以有多个线程同时操作（比如两个线程同时判断了get()的值，然后在线程A对其进行改变的过程中，线程B也对其进行修改），所以不具有原子性，也就是说volatile让单个读和写操作具有原子性，但是对于复合操作是没有原子性的。

### volatile变量读写的内存语义

>什么是内存语义？内存语义就是说某段代码，在进行内存实际操作的时候是怎样的。 对volatile变量执行写操作，会将线程的本地内存值刷新到主存中；对volatile变量执行读操作，操作系统会将变量所在线程的本地内存置为无效，然后该线程就会从主内存中读取该volatile变量。这样就保证了写线程的操作对读线程是可见的了。

### volatile内存语义的实现原理

volatile内存

<div id="container"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
var gitment = new Gitment({
id: 'SDWebImage2', // 可选。默认为 location.href
owner: 'MikeFighting',
repo: 'https://github.com/MikeFighting/BlogComment',
oauth: {
client_id: '8e2f9680af3a9d41bc50',
client_secret: '7f7c1e9cce7dfbd453018631ab6233bbaf73ad86',
},
})
gitment.render('container')
</script>