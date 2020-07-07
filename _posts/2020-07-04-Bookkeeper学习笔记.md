---
layout: post
title:  "bookeeper学习笔记"
description: bookeeper学习笔记
date:   2020-07-04 11:00:00 +000
categories: pulsar
tags: pulsar
---
### Bookeeper是什么

[官网](https://bookkeeper.apache.org/docs/4.10.0/getting-started/installation/)

> A scalable, fault-tolerant, and low-latency storage service optimized for real-time workloads

- 可扩展
- 容错
- 低延时
- 实时工作?

应用场景：

- WAL (Write-Ahead-Logging), e.g. HDFS NameNode.
- Message Store, e.g. Apache Pulsar.
- Offset/Cursor Store, e.g. Apache Pulsar.
- Object/Blob Store, e.g. storing state machine snapshots.

其中以pulsar比较出名。

### 安装

本地运行

```shell
bin/bookkeeper localbookie 10
```

### 核心组件

#### bookkeeper做了什么？

> BookKeeper is a service that provides persistent storage of streams of log [entries](https://bookkeeper.apache.org/docs/latest/getting-started/concepts/#entries)—aka *records*—in sequences called [ledgers](https://bookkeeper.apache.org/docs/latest/getting-started/concepts/#ledgers). BookKeeper replicates stored entries across multiple servers.

- 持久化日志数据
- 分布式存储

#### 基本概念

- entry/record 

每个日志/流水记录

- ledger(中文翻译成 账本？)

一组流式日志记录(streams of log entries)

- bookies

存储ledger的服务器

#### ledgers

> Ledgers are sequences of entries, while each entry is a sequence of bytes. Entries are written to a ledger:

- sequentially, and
- at most once.

ledger负责存储entry，保证顺序存储，最多只保存一次。ledger是一个逻辑概念，一个ledger可能会在不同的bookie上有多个副本。另外entry写入的顺序正确性是由客户端去保证的。

eg.假设有B1~B4,4个bookie，可能会有如下映射关系

ledger1:[B1,B2,B3]

ledger2:[B2,B3,B4]

#### Metadata storage(元数据存储)

使用zookeeper实现

#### Bookies数据管理

- Journals

bookkeeper的WAL。

数据写入ledger前会先写入一个事务日志，事务日志由Journals维护

- Entry logs





### 参考

[官网](https://bookkeeper.apache.org/docs/4.10.0/getting-started/installation/)

[github](https://github.com/apache/bookkeeper)