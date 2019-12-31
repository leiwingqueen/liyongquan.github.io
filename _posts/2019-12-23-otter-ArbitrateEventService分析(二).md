---
layout: post
title:  "otter ArbitrateEventService分析(二)"
description: otter ArbitrateEventService分析(二)
date: 2019-12-23 09:40:00 +000
categories: otter
tags: otter
---

这一篇尝试从单元测试切入来了解ArbitrateEventService的运行原理。

重点分析`ArbitrateAllTest` 这个类。

### 环境搭建

- zk环境

1. 本地安装zk并启动，默认端口为2181
2. 初始化结点

```shell
#初始化
create /otter
create /otter/channel
create /otter/node
#查看结点信息
[zk: localhost:2181(CONNECTED) 15] ls /otter
[channel, node]
```

- 修改单元测试zk端口号

```java
public class BaseEventTest extends BaseOtterTest {

    private String    cluster1  = "127.0.0.1:2181";
    private String    cluster2  = "127.0.0.1:2181,127.0.0.1:2181";
```

- 运行输出

在每个不同的阶段分别输出日志

```shell
#select await 并行度为3
SelectServiceDemo await...processId:0
SelectServiceDemo await...processId:1
SelectServiceDemo await...processId:2
#extract 启动
SelectServiceDemo single...processId:2
ExtractServiceDemo await...processId:2
SelectServiceDemo single...processId:0
ExtractServiceDemo await...processId:0
SelectServiceDemo single...processId:1
ExtractServiceDemo await...processId:1
ExtractServiceDemo single...processId:2
#transform 启动
TransformServiceDemo await...processId:2
ExtractServiceDemo single...processId:0
TransformServiceDemo await...processId:0
ExtractServiceDemo single...processId:1
TransformServiceDemo await...processId:1
TransformServiceDemo single...processId:2
TransformServiceDemo single...processId:0
#load 启动
LoadServiceDemo await...processId:0
LoadServiceDemo single...processId:0
SelectServiceDemo await...processId:3
TransformServiceDemo single...processId:1
LoadServiceDemo await...processId:1
LoadServiceDemo single...processId:1
LoadServiceDemo await...processId:2
SelectServiceDemo await...processId:4
SelectServiceDemo single...processId:3
ExtractServiceDemo await...processId:3
SelectServiceDemo single...processId:4
ExtractServiceDemo await...processId:4
LoadServiceDemo single...processId:2
SelectServiceDemo await...processId:5
ExtractServiceDemo single...processId:3
TransformServiceDemo await...processId:3
ExtractServiceDemo single...processId:4
TransformServiceDemo await...processId:4
SelectServiceDemo single...processId:5
ExtractServiceDemo await...processId:5
TransformServiceDemo single...processId:4
ExtractServiceDemo single...processId:5
TransformServiceDemo await...processId:5
TransformServiceDemo single...processId:3
LoadServiceDemo await...processId:3
TransformServiceDemo single...processId:5
LoadServiceDemo single...processId:3
LoadServiceDemo await...processId:4
SelectServiceDemo await...processId:6
SelectServiceDemo single...processId:6
ExtractServiceDemo await...processId:6
LoadServiceDemo single...processId:4
LoadServiceDemo await...processId:5
SelectServiceDemo await...processId:7
LoadServiceDemo single...processId:5
SelectServiceDemo await...processId:8
ExtractServiceDemo single...processId:6
TransformServiceDemo await...processId:6
SelectServiceDemo single...processId:7
ExtractServiceDemo await...processId:7
SelectServiceDemo single...processId:8
ExtractServiceDemo await...processId:8
TransformServiceDemo single...processId:6
LoadServiceDemo await...processId:6
ExtractServiceDemo single...processId:8
TransformServiceDemo await...processId:8
ExtractServiceDemo single...processId:7
TransformServiceDemo single...processId:8
LoadServiceDemo single...processId:6
```

### 调用链跟踪

SelectRpcArbitrateEvent仲裁调度逻辑主要依赖`AbstractProcessListener`

```java
/**
     * 阻塞方法，获取对应可以被处理的processId，支持中断处理
     */
    public Long waitForProcess() throws InterruptedException {
        // take和history.put操作非原子，addReply操作时会出现并发问题，同一个processId插入两次
        Long processId = (Long) replyProcessIds.take();
        logger.debug("## {} get reply id [{}]", ClassUtils.getShortClassName(this.getClass()), processId);
        return processId;
    }

    protected synchronized void addReply(Long processId) {
        boolean isSuccessed = replyProcessIds.offer(processId);
        ...
    }
```

内部维护了一个队列`replyProcessIds`。我们重点关注下addReply的方法在哪里触发

`ProcessMonitor.init` > `ProcessMonitor.initProcess` > `ProcessMonitor.processChanged`



### 参考文献

[Otter调度模型](https://www.bookstack.cn/read/otter/2.md)



