---
layout: post
title:  "Disruptor-源码分析(三)-生产者"
description: Disruptor-源码分析(三)-生产者
date:   2019-10-16 14:00:00 +000
categories: mq 高并发 disruptor
tags: mq 高并发 disruptor
---

上一篇blog我们尝试分析了消费者的逻辑，这次重点分析生产者的写消息的逻辑。

disruptor有两种生产模式。

- 单生产者
- 多生产者

通过[disruptor wiki]( https://github.com/LMAX-Exchange/disruptor/wiki/Getting-Started )我们可以看到这两者的区别。

>  One of the best ways to improve performance in concurrent systems is to adhere to the [Single Writer Principle](http://mechanical-sympathy.blogspot.co.nz/2011/09/single-writer-principle.html), this applies to the Disruptor. If you are in the situation where there will only ever be a single thread producing events into the Disruptor, then you can take advantage of this to gain additional performance. 

单生产者的模式是能够让队列的写性能得到额外的提升。下面是官方提供的一个测试数据。[OneToOneSequencedThroughputTest.java]( https://github.com/LMAX-Exchange/disruptor/blob/master/src/perftest/java/com/lmax/disruptor/sequenced/OneToOneSequencedThroughputTest.java )

多生产者

```java
Run 0, Disruptor=26,553,372 ops/sec
Run 1, Disruptor=28,727,377 ops/sec
Run 2, Disruptor=29,806,259 ops/sec
Run 3, Disruptor=29,717,682 ops/sec
Run 4, Disruptor=28,818,443 ops/sec
Run 5, Disruptor=29,103,608 ops/sec
Run 6, Disruptor=29,239,766 ops/sec
```

单生产者

```java
Run 0, Disruptor=89,365,504 ops/sec
Run 1, Disruptor=77,579,519 ops/sec
Run 2, Disruptor=78,678,206 ops/sec
Run 3, Disruptor=80,840,743 ops/sec
Run 4, Disruptor=81,037,277 ops/sec
Run 5, Disruptor=81,168,831 ops/sec
Run 6, Disruptor=81,699,346 ops/sec
```

写入性能，单生产者相比多生产者有2~3倍的提升。

### 源码分析



### 参考

[disruptor wiki]( https://github.com/LMAX-Exchange/disruptor/wiki/Getting-Started )

