---
layout: post
title:  "DruidDataSource-InterruptedException异常排查"
description: DruidDataSource-InterruptedException异常排查
date:   2019-10-22 14:00:00 +000
categories: DruidDataSource 连接池 ReentrantLock
tags: DruidDataSource 连接池 ReentrantLock
---

 最近在线上环境发现Druid连接池出现了InterruptedException异常，其实涉及到锁的使用，这篇文章我们尝试分析下连接池和可重入锁的实现，来对这个问题进行一个比较全面的分析。

异常堆栈内容如下：

```java
	java.lang.InterruptedException
at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1220)
at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
at com.alibaba.druid.pool.DruidDataSource.recycle(DruidDataSource.java:1281)
at com.alibaba.druid.pool.DruidPooledConnection.recycle(DruidPooledConnection.java:297)
at com.alibaba.druid.pool.DruidPooledConnection.close(DruidPooledConnection.java:247)
at com.kugou.fanxing.revenue.persist.DefaultNsqMessageDao.insertOrUpdateByMessageIdAndAppId(DefaultNsqMessageDao.java:98)
at com.kugou.fanxing.revenue.persist.DefaultNsqMessageDao.insertOrUpdateByMessageIdAndAppId(DefaultNsqMessageDao.java:31)
at com.kugou.fanxing.revenue.persist.MessagePersistCommand.run(MessagePersistCommand.java:40)
at com.kugou.fanxing.revenue.persist.MessagePersistCommand.run(MessagePersistCommand.java:15)
at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:302)
at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:298)
```

这里我们主要提出几个问题。

- 为什么会抛这个异常
- 回收连接池吞掉这个异常是否有问题？
- 可重入锁ReentrantLock的作用？

根据这几个问题，我们再去分析下连接池回收连接的逻辑。

### 参考文献

https://www.cnblogs.com/lingyejun/p/9064114.html