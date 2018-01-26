---
layout: post
author: sjf0115
title: Flink1.4 可查询状态
date: 2018-01-26 19:11:17
tags:
  - Flink

categories: Flink
permalink: flink_stream_queryable_state
---


### 1. 架构

备注:
```
可查询状态的客户端API目前处于迭代状态，对提供的接口的稳定性没有任何保证。在即将到来的Flink版本中，API很可能会在客户端方面发生较大更改。
```

简而言之，此功能向外界公开了Flink的托管键控（分区）状态（请参见使用状态），并允许用户从Flink外部查询作业的状态。对于某些情况，可查询状态消除了对外部系统（如键值存储）的分布式操作/事务的需求，而这些系统往往是实践中的瓶颈。另外，这个特性对于调试目的可能特别有用。

注意：查询状态对象时，该对象是从一个并发线程访问的，没有任何同步或复制。这是一个设计选择，因为任何上述都会导致工作延迟增加，这是我们想避免的。由于任何使用Java堆空间的状态后端，例如MemoryStateBackend或FsStateBackend在检索值时不能与副本一起使用，而是直接引用存储的值，读取 - 修改 - 写入模式不安全，并且可能导致可查询状态服务器由于并发修改而失败。 RocksDBStateBackend是安全的这些问题。

### 2. 启用可查询状态
### 3. 使状态可查询
#### 3.1 可查询状态Stream
#### 3.2 Managed Keyed State
### 4 . 查询状态
#### 4.1 Example
### 5. 配置
#### 5.1 State Server
#### 5.2 Proxy
### 6. 使用限制














































原文:https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/stream/state/queryable_state.html
