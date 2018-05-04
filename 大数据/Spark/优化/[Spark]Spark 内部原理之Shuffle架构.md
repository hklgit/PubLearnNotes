---
layout: post
author: sjf0115
title: Spark 内部原理之Shuffle架构
date: 2018-05-02 11:33:01
tags:
  - Spark
  - Spark 内部原理

categories: Spark
permalink: spark-internal-shuffle-architecture
---

我们遵循 MapReduce 命名约定。在 shuffle 操作中，在 Source Executor 中发送数据的任务称之为 `mapper`，将数据拉取到 Target Executor 的任务称之为 `reducer`，它们之间发生的是 `shuffle`。

一般来说，shuffle 有2个重要的压缩参数：
- `spark.shuffle.compress` - 是否压缩 shuffle 输出
- `spark.shuffle.spill.compress` - 是否压缩中间shuffle溢出文件。
默认值都为 true，并且两者都会使用 `spark.io.compression.codec` 编解码器来压缩数据，默认值为 snappy。

Spark 中有许多 shuffle 实现。使用哪个具体实现取决于 `spark.shuffle.manager` 参数。该参数有有三种可选择的值：`hash`, `sort`, `tungsten-sort`，从 Spark 1.2.0 开始 `sort` 是默认选项。

### 2. Hash Shuffle

在 Spark 1.2.0 之前，Shuffle 的默认选项为 `Hash` (`spark.shuffle.manager = hash`)。但它有许多缺点，主要是由它[创建的文件数量](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F13/projects/reports/project16_report.pdf)引起的 - 每个 mapper 任务为每个 reducer 创建一个文件，从而在集群上会产生 `M * R` 个总文件，其中 M 是 mapper 的数量，R 是 reducer 的数量。使用大量的 mapper 和 reducer 会产生比较大的问题，包括输出缓冲区大小，文件系统上打开文件的数量，创建和删除所有这些文件的速度。这有一个雅虎如何面对这些问题时一个很好的[例子](http://spark-summit.org/2013/wp-content/uploads/2013/10/Li-AEX-Spark-yahoo.pdf)，46k个 mapper 和 46k个 reducer 在集群上会生成20亿个文件。

这个 shuffler 实现的逻辑非常愚蠢：它将 reducers 的数量计算为 reduce 一侧的分区数量，为每个分区创建一个单独的文件，并循环遍历需要输出的记录，然后计算它 目标分区，并将记录输出到相应的文件。

























原文:https://0x0fff.com/spark-architecture-shuffle/


































原文:https://0x0fff.com/spark-architecture-shuffle/
