---
layout: post
title:  "RateLimiter-源码分析"
description: RateLimiter-源码分析
date:   2019.07.25 00:07:04 +0530
categories: 分布式 限流 RateLimiter
---
### 一、前言
在分布式系统中，实现高可用有三大利器：
- 限流
- 降级
- 熔断
我们先对限流来进行一个分析。

### 二、限流的实现
业界常用的限流的实现方式也有多种，我尝试做一个简单的总结：
- 计数器
统计单位时间内的请求数量，超过某个阈值进行限流
- 信号量
控制请求的最大并发数。Hystrix的信号量模式就是通过控制最大并发数来实现限流。
- 滑动窗口
我个人理解是计数器的升级版，通过统计多个连续的桶来得到一个比较精准和合理的值来进行限流。
- 漏桶算法
流量会先经过漏桶，从漏桶出来的流量会保证匀速地产生，如果漏桶处理不过来(溢出)的流量认为就是需要限流的流量。
- 令牌桶算法
跟令牌桶有点相反，有一个定时器匀速地产生token到桶里面，流量经过的时候会到桶去获取token，获取token失败的流量就被阻塞。如果token一直没有被消费，会累积到桶里面，当然桶本身也有大小，超过最大数量的token会被丢弃掉。

漏桶算法 和 令牌桶算法 最大的区别：
- 漏桶算法的处理的流量必然是匀速的，但是令牌桶算法是允许有一定的突刺流量(取决于桶的最大大小)

### 三、RateLimiter的创建
guawa的RateLimiter就是使用令牌桶算法实现的。
> RateLimiter

我们先分析构造函数，RateLimiter的create方法生成了一个SmoothBursty的类

```java
public static RateLimiter create(double permitsPerSecond) {
    /*
     * The default RateLimiter configuration can save the unused permits of up to one second. This
     * is to avoid unnecessary stalls in situations like this: A RateLimiter of 1qps, and 4 threads,
     * all calling acquire() at these moments:
     *
     * T0 at 0 seconds
     * T1 at 1.05 seconds
     * T2 at 2 seconds
     * T3 at 3 seconds
     *
     * Due to the slight delay of T1, T2 would have to sleep till 2.05 seconds, and T3 would also
     * have to sleep till 3.05 seconds.
     */
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
  }
@VisibleForTesting
  static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
  }
```
SmoothBursty其实就是RateLimiter的子类

![SmoothBursty类图](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/20190915094019.png)
继续往下看SmoothBursty，有一个maxBurstSeconds的属性，这个就是桶的最大大小，如果ratelimiter没有试用，桶最大的保存的秒数，默认值为1。

```java
static final class SmoothBursty extends SmoothRateLimiter {
    /** The work (permits) of how many seconds can be saved up if this RateLimiter is unused? */
    final double maxBurstSeconds;

    SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
      super(stopwatch);
      this.maxBurstSeconds = maxBurstSeconds;
    }
```
继续往下看maxPermits 是如何定义的，桶保存的最大秒数*每秒允许的请求数
```java
@Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
      double oldMaxPermits = this.maxPermits;
//重点关注这一行，最大允许的请求为maxBurstSeconds * permitsPerSecond。
      maxPermits = maxBurstSeconds * permitsPerSecond;
      if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = maxPermits;
      } else {
        storedPermits =
            (oldMaxPermits == 0.0)
                ? 0.0 // initial state
                : storedPermits * maxPermits / oldMaxPermits;
      }
    }
/** The maximum number of stored permits. */
  double maxPermits;
```
### 四、tryAcquire
我们重点关注tryAcquire，tryAcquire跟acquire的主要区别是tryAcquire不会阻塞，马上返回限流结果，而acquire则会阻塞到获取ticket成功为止(当然可以设置等待超时时间)。
```java
public boolean tryAcquire() {
    return tryAcquire(1, 0, MICROSECONDS);
  }

public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
    long timeoutMicros = max(unit.toMicros(timeout), 0);
    checkPermits(permits);
    long microsToWait;
    synchronized (mutex()) {
      long nowMicros = stopwatch.readMicros();
      if (!canAcquire(nowMicros, timeoutMicros)) {
        return false;
      } else {
        microsToWait = reserveAndGetWaitLength(permits, nowMicros);
      }
    }
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return true;
  }
```
下面来分析一下：
- mutex()
获取锁的逻辑。通过双重校验锁的方式来保证锁的唯一性。
```java
private Object mutex() {
    Object mutex = mutexDoNotUseDirectly;
    if (mutex == null) {
      synchronized (this) {
        mutex = mutexDoNotUseDirectly;
        if (mutex == null) {
          mutexDoNotUseDirectly = mutex = new Object();
        }
      }
    }
    return mutex;
  }
```
- canAcquire
是否获取token成功判断
```
private boolean canAcquire(long nowMicros, long timeoutMicros) {
    return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
  }
```
```java
@Override
  final long queryEarliestAvailable(long nowMicros) {
    return nextFreeTicketMicros;
  }
```
queryEarliestAvailable这里返回的是下一个空闲的ticket产生的时间(毫秒)，nowMicros是当前时间。
我们尝试把公式改一下，可能会好理解一些：

> queryEarliestAvailable(nowMicros) <= nowMicros+timeoutMicros

这里的意思是，如果当前时间+等待的超时时间都比下一个空闲token产生的时间要早，则直接认为获取token失败，因为即使等待超时，也必然获取不到token。
- reserveAndGetWaitLength
更新并获取ticket，并sleep一定时间，返回给调用方。下面重点介绍这个方法。

### 五、更新ticket
```java
/**
   * Reserves next ticket and returns the wait time that the caller must wait for.
   *
   * @return the required wait time, never negative
   */
  final long reserveAndGetWaitLength(int permits, long nowMicros) {
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    return max(momentAvailable - nowMicros, 0);
  }

@Override
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);

    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }
```
- resync
更新ticket的数量和时间
```java
/** Updates {@code storedPermits} and {@code nextFreeTicketMicros} based on the current time. */
  void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      storedPermits = min(maxPermits, storedPermits + newPermits);
      nextFreeTicketMicros = nowMicros;
    }
  }
```
> double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();

新增的ticket为 (当前时间-上一次最后更新时间)/coolDownIntervalMicros()
这里coolDownIntervalMicros如果选择的是SmoothBursty实现，则为每个ticket生成所需要的毫秒数。
eg.qps为20，则每个ticket生成所需要的时间为50ms。
> storedPermits = min(maxPermits, storedPermits + newPermits);

更新存量的ticket，maxPermits前面已经提到，相当于桶的最大大小，默认是1s的qps数量。

- storedPermitsToSpend和freshPermits
storedPermitsToSpend：存量的ticket消费
freshPermits：存量ticket不足，需要额外申请的ticket

> long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);

等待时间计算，storedPermitsToWaitTime的SmoothBursty实现直接返回0，freshPermits * stableIntervalMicros为额外需要等待的毫秒数。

![ticket更新](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/20190915094151.png)
- 问题1：
这里的returnValue为什么不是直接返回nextFreeTicketMicros，而是直接获取nextFreeTicketMicros更新前的值？
- 问题2：
如果timeout设置为0，但是存量的ticket不足的情况下，需要freshPermits来填充，那么最终不是也会sleep吗？

### 五、总结
通过对RateLimiter的源码分析，对令牌桶算法的实现应该有一个比较深刻的理解了。上面提到的两个问题，下一篇我们再去分析。










