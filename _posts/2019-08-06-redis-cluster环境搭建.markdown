---
layout: post
title:  "redis cluster环境搭建"
description: redis cluster环境搭建
date:   2019-08-06 21:03:36 +0530
categories: redis 分布式
---
#### 一、入门
为了更好地了解redis的源码和工作原理，花了两天时间把redis cluster的环境给搭建起来。主要有以下几步：
- vmware
- ubuntu
- redis 
[下载地址](http://download.redis.io/releases/)
我这次选择的是最新的版本 5.0.5
- ruby
redis cluster需要有ruby的环境
- jemalloc
redis默认的内存管理包

#### 二、配置
##### 1.redis配置文件
这里只把核心修改的配置标出来。

```xml
# Accept connections on the specified port, default is 6379 (IANA #815344).
# If port 0 is specified Redis will not listen on a TCP socket.
port 6381

# Creating a pid file is best effort: if Redis is not able to create it
# nothing bad happens, the server will start and run normally.
pidfile /var/run/redis_6381.pid

# Specify the log file name. Also the empty string can be used to force
# Redis to log on the standard output. Note that if you use standard
# output for logging but daemonize, logs will be sent to /dev/null
logfile /usr/local/redis/log/redis.6381.log

# The filename where to dump the DB
dbfilename dump.6381.rdb

# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir /usr/local/redis/data/6381

# The name of the append only file (default: "appendonly.aof")

appendfilename "appendonly.6381.aof"

# Normal Redis instances can't be part of a Redis Cluster; only nodes that are
# started as cluster nodes can. In order to start a Redis instance as a
# cluster node enable the cluster support uncommenting the following:
cluster-enabled yes

# Every cluster node has a cluster configuration file. This file is not
# intended to be edited by hand. It is created and updated by Redis nodes.
# Every Redis Cluster node requires a different cluster configuration file.
# Make sure that instances running in the same system do not have
# overlapping cluster configuration file names.
#
cluster-config-file /usr/local/redis/conf/nodes_6381.conf

# Cluster node timeout is the amount of milliseconds a node must be unreachable
# for it to be considered in failure state.
# Most other internal time limits are multiple of the node timeout.
#
cluster-node-timeout 5000
```

我们把配置拷贝6份，从6080~6085，一共6份配置，分别起6个redis实例
##### 2.集群配置
redis-trib.rb命令已在新版本废弃，已统一使用redis-cli命令，我们尝试创建6个实例的redis集群，复制因子为1(一个从结点)。

> /usr/local/redis/redis-5.0.5/src$ ./redis-cli --cluster create 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 127.0.0.1:6385 --cluster-replicas 1

```shell
liyongquan@liyongquan-virtual-machine:/usr/local/redis/redis-5.0.5/src$ ./redis-cli --cluster create 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 127.0.0.1:6385 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:6384 to 127.0.0.1:6380
Adding replica 127.0.0.1:6385 to 127.0.0.1:6381
Adding replica 127.0.0.1:6383 to 127.0.0.1:6382
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 211ec1ff747f4214ae4a1c7d15bc8c4b05e8c184 127.0.0.1:6380
   slots:[0-5460] (5461 slots) master
M: b6c7c9661a737b3e44b1729557faacb9691f6e0b 127.0.0.1:6381
   slots:[5461-10922] (5462 slots) master
M: 6f37f18df0588b3fdc5a09281841ef5f38ca51e9 127.0.0.1:6382
   slots:[10923-16383] (5461 slots) master
S: ebc2e194c8c77c3a62e6717bd9d4eacc327878ee 127.0.0.1:6383
   replicates 211ec1ff747f4214ae4a1c7d15bc8c4b05e8c184
S: e1bcda2b1d69dfcb47547427e881aaf2dddc2a56 127.0.0.1:6384
   replicates b6c7c9661a737b3e44b1729557faacb9691f6e0b
S: e78420bbe2c61569b518b24f9cc6ab00ee8b8cf4 127.0.0.1:6385
   replicates 6f37f18df0588b3fdc5a09281841ef5f38ca51e9
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 127.0.0.1:6380)
M: 211ec1ff747f4214ae4a1c7d15bc8c4b05e8c184 127.0.0.1:6380
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: e78420bbe2c61569b518b24f9cc6ab00ee8b8cf4 127.0.0.1:6385
   slots: (0 slots) slave
   replicates 6f37f18df0588b3fdc5a09281841ef5f38ca51e9
M: b6c7c9661a737b3e44b1729557faacb9691f6e0b 127.0.0.1:6381
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 6f37f18df0588b3fdc5a09281841ef5f38ca51e9 127.0.0.1:6382
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: e1bcda2b1d69dfcb47547427e881aaf2dddc2a56 127.0.0.1:6384
   slots: (0 slots) slave
   replicates b6c7c9661a737b3e44b1729557faacb9691f6e0b
S: ebc2e194c8c77c3a62e6717bd9d4eacc327878ee 127.0.0.1:6383
   slots: (0 slots) slave
   replicates 211ec1ff747f4214ae4a1c7d15bc8c4b05e8c184
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

```
我们看到redis的16384个slot已经平均分配到3主3从的结点中，后面我们将重点了解下redis cluster的slot的分配算法。
#### 四、测试
我们简单做一个get set的测试，通过redis cli命令连接到集群，进行测试。
![测试](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/14814047-f3914e9a886ef15d.png)

我们再尝试下redis集群的高可用，我们尝试把其中一段的slot主从结点都停掉，整个集群会变成不可用。

![停止一段slot](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/14814047-509860680b0d3b29.png)



![集群不可用](https://leiwingqueen-1300197911.cos.ap-guangzhou.myqcloud.com/14814047-24b380cb80b7d1f6.png)






