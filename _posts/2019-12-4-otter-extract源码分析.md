---
layout: post
title:  "otter extract源码分析"
description: otter extract源码分析
date:   2019-12-4 09:26:00 +000
categories: otter 数据一致性
tags: otter 数据一致性
---

### 前言

SETL是otter的核心组件，我们将逐个模块分析和了解otter的实现机制。我们先对E(extract)的组件做分析。

### 核心组件

`ExtractTask` 是extract的工作线程。

![ExtractTask类图](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20191204093804.png)

`GlobalTask` 是SETL的父线程。封装了工作线程需要的一些公共属性。

```java
public abstract class GlobalTask extends Thread {

    protected final Logger              logger  = LoggerFactory.getLogger(this.getClass());
    //控制线程运行状态
    protected volatile boolean          running = true;
    //从源端到目标端的整个过程描述，主要由一些同步映射过程组成。简单来说，一个同步规则就是一个pipeline
    protected Pipeline                  pipeline;
    //pipeline对应的唯一ID
    protected Long                      pipelineId;
    //仲裁事件相关控制(HA)
    protected ArbitrateEventService     arbitrateEventService;
    //行数据(binlog)?
    protected RowDataPipeDelegate       rowDataPipeDelegate;
    //线程池
    protected ExecutorService           executorService;
    //客户端配置?
    protected ConfigClientService       configClientService;
    protected StageAggregationCollector stageAggregationCollector;
    protected Map<Long, Future>         pendingFuture;
```

### 源码分析





