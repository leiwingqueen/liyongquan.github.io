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

### 参考

[官网](https://bookkeeper.apache.org/docs/4.10.0/getting-started/installation/)

[github](https://github.com/apache/bookkeeper)