---
layout: post
title:  "RateLimiter-源码分析2"
description: RateLimiter-源码分析2
date:   2019.07.29 22:37:56 +0530
categories: 分布式 限流 RateLimiter
---
####  一、前言
上一篇结尾的时候我提了两个问题，这一次主要针对这两个问题进行思考和讨论。
![ticket更新](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/20190915094151.png)

- 问题1
这里的returnValue为什么不是直接返回nextFreeTicketMicros，而是直接获取nextFreeTicketMicros更新前的值？
- 问题2
如果timeout设置为0，但是存量的ticket不足的情况下，需要freshPermits来填充，那么最终不是也会sleep吗？

#### 二、代码测试
![增加日志](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/20190915095032.png)
我尝试在sleep之前增加一条日志，用于观察程序睡眠的时间。
简单编写一个测试类用于测试ticket不足的场景。

```java
public static void main(String[] args) {
    RateLimiter rateLimiter=RateLimiter.create(5);
    for(int i=0;i<5;i++) {
      boolean acquire = rateLimiter.tryAcquire(10);
      System.out.println("result:"+acquire);
    }
  }
```
最终运行的结果：
```java
sleep0ms...
result:true
result:false
result:false
result:false
result:false

Process finished with exit code 0
```
我们可以知道
- 当有可用ticket，但是ticket不足的时候会返回成功，并且不会sleep
- 不足的ticket会更新下一次的nextFreeTicketMicros，导致下一次tryAcquire失败。如测试用例所示，nextFreeTicketMicros会修改为2s后(10/5=2)。

我们再来总结下，只要你手头上至少有一个ticket(当前时间>=nextFreeTicketMicros)，不管请求者是需要多少个ticket，我们都一并满足给他，返回true，并且把ticket更新为负数(往后更新nextFreeTicketMicros)。但是这种模式后面来的人就遭殃了，假如前面一个用户申请了很多ticket，那么后来者需要等待很长的时间才能申请到一个ticket。

#### 四、总结
RateLimiter的SmoothBursty实现我们已经大致讨论完了，可以看得出来其实SmoothBursty的实现会存在一些弊端，比如说可能会出现比较严重的突刺流量，另外一个实现类SmoothWarmingUp是否能比较好地解决这部分问题？这个有待讨论







