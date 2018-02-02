---
title: Java多线程中的常见问题
date: 2018-02-02 21:27:44
tags: Java
---

## 一、64位的long和double不具有顺序一致性

 CPU和内存中的通信是通过总线进行的，总线的宽度是固定的，如果有多个CPU同时请求总线进行读/写总线事务的话，会由总线仲裁（Bus Arbitration）进行裁决，然后获胜的CPU才可以进行数据传递。并且在某个CPU占用了总线之后，其它CPU要请求总线仲裁是要被拒的。这种机制就保证了CPU对内存的访问是以串行的方式进行的。如果有一个32位的处理器，但是要进行64位的写数据操作，这时CPU会将数据分为两个32位的写操作，并且这两个32位数据在请求总线仲裁的时候可能被分配到了不同的总线事务中，所以此时这个64位的写操作就不具有原子性了。这时如果处理器A写入了高32位，在写入低32位期间，处理器B对该数据进行了访问，那么就会产生错误的数据。（Java5之前读写都可以被拆分，Java5之后写操作可以被拆分，但是读操作必须是原子的）。如果要将变量前面加上volatile修饰符，那么对其操作将具有原子性，因为加上volatile之后，对其进行读写操作就像是“加锁”了一样。


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