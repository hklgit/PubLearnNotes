---
layout: post
author: sjf0115
title: Flink1.4 内部原理之内存管理
date: 2018-02-02 11:40:01
tags:
  - Flink
  - Flink内部原理

categories: Flink
permalink: flink-batch-internals-memory-management
---

### 1. 概述

Flink中的内存管理用于控制特定运行时操作使用的内存量。内存管理用于所有累积（可能很大）了一定数量记录的操作。
这种操作的典型例子是：
- 排序 `Sorting` - 排序用于对分组，连接后的记录进行排序以及生成排序结果。
- 哈希表 `Hash Tables` - 哈希表用于连接 `join` 中以及迭代中的解集(使用它们进行分组和聚合)。
- 缓存 `Caching` - 缓存数据对于迭代算法以及作业恢复期间的检查点非常重要。
- `(Block-)Nested-Loop-Join` - 此算法用于数据集之间的笛卡尔乘积。

如果没有管理/控制内存的方法，当要排序（或哈希）的数据大于 `JVM` 可以备用的内存（通常会抛出 `OutOfMemoryException`）时，这些操作将会失败。内存管理是一种非常精确地控制每个算子使用多少内存的方法，并且通过将一些数据移动到磁盘上来让他们高效地脱离内核操作。究竟发生了什么取决于具体的操作/算法（见下文）。

内存管理还允许在同一个 `JVM` 中为消耗内存的不同算子划分内存。这样，`Flink` 可以确保不同的算子在同一个 `JVM` 中相互临近运行，不会相互干扰。

备注:
```
目前为止，内存管理仅在批处理算子中使用。流处理算子遵循不同的理念。
```

### 2. Flink的内存管理

从概念上讲，`Flink` 将堆分成三个区域：
- `Network buffers` ： 一定数量的 32KB 大小的缓冲区 `buffer`，网络堆栈使用它缓冲网络传输（`Shuffle`，`Broadcast`）的记录。在 `TaskManager` 启动时分配。默认分配 2048 个缓冲区，但可以通过 `taskmanager.network.numberOfBuffers` 参数进行调整。
- `Memory Manager pool` : 一个大的缓冲区 `buffer` （32 KB）集合（译者注：可理解为缓冲池），每当需要缓冲记录时，所有运行时算法（`Sorting`，`Hashing`，`Caching`）使用这些缓冲区缓冲记录。记录以序列形式存储在这些块中。内存管理器在启动时分配这些缓冲区。
- `Remaining (Free) Heap` : 这部分堆留给用户代码和 `TaskManager` 的数据结构使用。由于这些数据结构比较小，所以这部分内存基本都是给用户代码使用的。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Flink/flink-batch-internals-memory-management-1.png?raw=true)

在分配 `Network buffers` 和 `MemoryManager buffers` 的同时，`JVM` 通常会执行一个或多个 `Full GC`。这会导致 `TaskManager` 的启动时多增加一些时间，但是在执行任务时会节省垃圾回收时间（saves in garbage collection time later）。`Network buffers` 和 `MemoryManager buffers` 在 `TaskManager` 的整个生命周期中存在。他们转移到 `JVM` 的内部存储区域的老年代，成为存活更长时间，不被回收的对象。

> 注意
- 缓冲区的大小可以通过 `taskmanager.network.bufferSizeInBytes` 进行调整，在大多数情况下，`32K` 已经是一个不错的大小了。
- 有想法去如何统一 `NetworkBuffer Pool`和 `Memory Manager` 区域。
- 有想法添加一个模式，由 `MemoryManager` 惰性分配（需要时分配）内存缓冲区。这会减少 `TaskManager` 的启动时间，但是当实际分配缓冲区时，会导致更多的垃圾回收。

### 3. Memory Segments

`Flink` 将其所有内存表示为内存段的集合。内存段表示一个内存区域（默认为32 KB），并提供根据偏移量访问数据的各种方法（get/put longs，int，bytes，段和数组之间的复制...）。你可以将其视为专用于 `Flink` 的 `java.nio.ByteBuffer` 的一个版本（请参阅下面为什么我们不使用 `java.nio.ByteBuffer`）。

每当 `Flink` 在某个地方存储记录时，它实际上将其序列化到一个或多个内存段中。系统可以在另一个数据结构中存储一个指向该记录的 `指针`（通常也构建为内存段的集合）。这意味着 `Flink` 依靠高效的序列化来识别页面以及实现记录跨越不同页面。为了这个目的，`Flink` 带来了自己的类型信息系统和序列化堆栈。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Flink/flink-batch-internals-memory-management-2.png?raw=true)

序列化到 `Memory Segments` 的一组记录

序列化格式由 `Flink` 的序列化器定义，并知道记录的各个字段。尽管这个特性目前还没有广泛使用（此博客写于2015年），但它允许在处理过程中部分反序列化记录，以提高性能。

为了进一步提高性能，使用 `Memory Segments` 的算法尝试可以在序列化的数据上工作。这是通过可扩展类型实用类 [TypeSerializer](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/typeutils/TypeSerializer.java) 和 [TypeComparator](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/typeutils/TypeComparator.java) 实现的。例如，分类器中的大多数比较可以归结为比较某些页面的字节（如memcmp）即可。这样，使用一起使用序列化和序列化数据的具有更高性能，同时也能够控制分配的内存量。



### 3. 对垃圾回收的影响

### 4. 内存管理算法

































原文: https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=53741525
