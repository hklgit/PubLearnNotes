---
layout: post
author: sjf0115
title: Spark2.3.0 持久化
date: 2018-03-16 11:28:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-rdd-persistence
---

### 1. 概述

Spark 中最重要的功能之一是在操作之间将数据集持久化(缓存)在内存中。当持久化一个 RDD 时，每个节点都会将其计算的任何分区存储在内存中，并将其重用于该数据集（或从其派生的数据集）的其他行动操作(each node stores any partitions of it that it computes in memory and reuses them in other actions on that dataset (or datasets derived from it))。这样可以使以后的动作操作执行的更快（通常超过10倍）。 缓存是迭代算法和快速交互使用的关键工具。

可以使用RDD上的`persist()`或`cache()`方法来标记要持久化的RDD(执行persist和cache方法不会持久化RDD)。 当RDD第一次在动作操作中计算时，它将持久化(缓存)到节点的内存中。Spark的缓存是可容错的 - 如果RDD的任何分区丢失，它将使用最初创建的转换操作自动重新计算。

### 2. 存储级别

除此之外，可以使用不同的持久化级别来存储每个持久化的RDD，从而允许你将数据集保留在磁盘上，或者将其以序列化的Java对象存储在内存中(以节省空间)，或者将其复制到所有节点上( to persist the dataset on disk, persist it in memory but as serialized Java objects (to save space), replicate it across nodes)。通过将StorageLevel对象传递给persist()方法来设置持久化级别。 cache()方法使用默认存储级别，即`StorageLevel.MEMORY_ONLY`。

持久化级别 | 说明
---|---
MEMORY_ONLY|将RDD以非序列化的Java对象存储在JVM中。 如果没有足够的内存存储RDD，则某些分区将不会被缓存，每次需要时都会重新计算。 这是默认级别。
MEMORY_AND_DISK | 将RDD以非序列化的Java对象存储在JVM中。如果数据在内存中放不下，则溢写到磁盘上．需要时则会从磁盘上读取
MEMORY_ONLY_SER (Java and Scala) | 将RDD以序列化的Java对象(每个分区一个字节数组)的方式存储．这通常比非序列化对象(deserialized objects)更具空间效率，特别是在使用快速序列化的情况下，但是这种方式读取数据会消耗更多的CPU。
MEMORY_AND_DISK_SER (Java and Scala) | 与`MEMORY_ONLY_SER`类似，但如果数据在内存中放不下，则溢写到磁盘上，而不是每次需要重新计算它们。
DISK_ONLY | 将RDD分区存储在磁盘上。
MEMORY_ONLY_2, MEMORY_AND_DISK_2等 | 与上面的储存级别相同，只不过将持久化数据存为两份，备份每个分区存储在两个集群节点上。
OFF_HEAP（实验中）| 与`MEMORY_ONLY_SER`类似，但将数据存储在堆内存中。 这需要启用堆内存。

==备注==

在Python中，存储对象将始终使用`Pickle`库进行序列化，持久化级别默认值就是以序列化后的对象存储在JVM堆空间中，因此选择什么样的序列化级别是无关紧要的。 当我们把数据写到磁盘或者堆外存储上时，也总是使用序列化后的数据．Python中的可用存储级别包括`MEMORY_ONLY`，`MEMORY_ONLY_2`，`MEMORY_AND_DISK`，`MEMORY_AND_DISK_2`，`DISK_ONLY`和`DISK_ONLY_2`。


在Shuffle操作中(例如，reduceByKey)，即使用户没有主动对数据进行持久化，Spark也会对一些中间数据进行持久化。 这样做是为了避免重新计算整个输入，如果一个节点在Shuffle过程中发生故障。 如果要重用，我们仍然建议用户对生成的RDD进行持久性。

### 3. 选择存储级别

Spark的存储级别旨在提供内存使用率和CPU效率之间的不同权衡。 我们建议通过以下过程来选择一个：
- 如果你的RDD适合于默认存储级别（MEMORY_ONLY），那就保持不动。 这是CPU效率最高的选项，允许RDD上的操作尽可能快地运行。
- 如果不是，请尝试使用`MEMORY_ONLY_SER`并选择一个快速的序列化库，这种方式更加节省空间，并仍然能够快速访问。 （Java和Scala）
- 不要溢写到磁盘，除非在数据集上的计算操作成本较高，或者需要过滤大量的数据。 否则，重新计算分区可能与从磁盘读取分区一样快。
- 如果要快速故障恢复（例如，使用Spark为Web应用程序提供服务），请使用副本存储级别`replicated storage levels`。 所有存储级别通过重新计算丢失的数据来提供完整的容错能力，但副本数据可让你继续在RDD上运行任务，而无需重新计算丢失的分区。

### 4. 清除数据

Spark会自动监视每个节点的缓存使用情况，并以最近最少使用（LRU）方式丢弃旧的数据分区。 如果您想手动删除RDD，而不是等待它自动从缓存中删除，请使用`RDD.unpersist()`方法。

> Spark版本: 2.3.0

原文：http://spark.apache.org/docs/2.3.0/rdd-programming-guide.html#rdd-persistence
