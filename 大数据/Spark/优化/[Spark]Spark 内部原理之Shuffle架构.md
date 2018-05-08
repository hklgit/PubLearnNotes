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

这个 shuffler 实现的逻辑非常愚蠢：它将 reducers 的个数计为 reduce 一侧的分区数量，为每个分区创建一个单独的文件，并循环遍历需要输出的记录，然后计算它目标分区，并将记录输出到对应的文件中。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-shuffle-architecture-1.png?raw=true)
市场营销部 - 增长技术部 - 算法数据组
shuffler　有一个优化过的实现，由参数 `spark.shuffle.consolidateFiles` 控制（默认值为 `false`）。当它被设置为 `true` 时，mapper 的输出文件将被合并。如果你的集群有 E 个 Executor（在 YAorderTotalCounter.increment(1L);RN 中由`-num-executors`设置），每一个都有 C 个 core（在 YARN 中由 `spark.executor.cores` 或 `-executor-cores` 设置），并且每个任务都要求 T 个 CPU（由 `spark.task.cpus` 设置），那么集群上的执行 slots 的个数为 `E * C / T`，在 shuffle 期间创建的文件个数为 `E * C / T * R`。100个 Executor，每个有10个 core，每个任务分配1个core，46000个 reducer 可以让你从20亿个文件下降到4600万个文件，这在性能方面提升好多。这个功能可以以一种相当直接的方式实现：不是为每个 Reducer 创建新文件，而是创建一个输出文件池。当 map 任务开始输出数据时，从输出文件池申请一组 R 个文件。完成后，将这 R 个文件组返回到输出文件池中。由于每个 Executor 只能并行执行 `C / T` 个任务，因此只会创建　`C / T` 组输出文件，每个组都是 R 个文件。在第一批 `C / T` 个并行 mapper 任务完成后，下一批 mapper 任务重新使用该池中的现有组。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-shuffle-architecture-2.png?raw=true)

优点：
- 快速 - 不需要排序，不维护哈希表;
- 没有内存开销来对数据进行排序;
- 没有IO开销 - 数据只写入硬盘一次并读取一次。

缺点：
- 当分区数量很大时，由于大量的输出文件，性能开始下降;
- 大量文件写入文件系统会导致IO变为随机IO，这通常比顺序IO慢100倍;

当然，当数据写入文件时，会被序列化以及被压缩。读取时，orderTotalCounter.increment(1L);务输出的数据，这会提高性能，但也会增加 reducer 进程的内存使用量。

如果 reduce 端的没有要求记录排序，那么 reducer 将只返回一个依赖于 map 输出的迭代器，但是如果需要排序，在 reduce 端使用 [ExternalSorter](https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/util/collection/ExternalSorter.scala) 对获取的所有数据进行排序。

### 3. Sort Shuffle

从Spark 1.2.0开始，这就是 Spark 使用的默认 shuffle 算法（`spark.shuffle.manager = sort`）。通常来说，这是试图实现类似于 Hadoop MapReduce 所使用的 shuffle 逻辑。使用 `hash shuffle`，你可以为每个 reducer 输出一个单独的文件，而使用 `sort shuffle` 时：输出一个按  reducer id 排序的文件并进行索引，通过获取文件中相关数据块的位置信息并在 fread 之前执行 fseek，你就可以轻松地获取与 reducer x 相关的数据块。但是，当然对于少量的 reducers 来说，显然使用哈希来分离文件会比排序更快，所以 `sort shuffle` 有一个'后备'计划：当 reducers 的数量小于 `spark.shuffle.sort.bypassMergeThreshold` 时（默认情况下为200），我们使用'后备'计划通过哈希将数据分到不同的文件中，然后将这些文件合并为一个文件。实现逻辑在类[BypassMergeSortShuffleWriter](https://github.com/apache/spark/blob/master/core/src/main/java/org/apache/spark/shuffle/sort/BypassMergeSortShuffleWriter.java)中实现的。

关于这个实现的有趣之处在于它在 map 端对数据进行排序，但不会在 reduce 端对排序的结果进行合并 - 如果需要排序数据，你需要对数据进行重新排序。 关于 Cloudera 的这个想法可以参阅[博文](http://blog.cloudera.com/blog/2015/01/improving-sort-performance-in-apache-spark-its-a-double/)。实现逻辑充分利用 mapper 的输出已经排序的特点，在 reducer 端对文件进行合并而不是采取其他手段。我们都知道，Spark 在 reducer 端使用 TimSort 完成排序，这是一个很棒的排序算法，实际上是利用了输入已经排序的特点（通过计算 minun 并将它们合并在一起）。这里有一点数学知识，你可以选择跳过。当我们使用最有效的方式 Min Heap(小顶堆) 来完成时，合并M个包含N个元素的排序数组的复杂度为O（MNlogM）。使用 TimSort，我们通过数据查找MinRuns，然后将它们逐个合并在一起。很明显，它将识别M MinRun。首先M / 2合并会导致M / 2排序组，接下来的M / 4合并会给M / 4排序组等等，所以它非常简单，所有这些合并的复杂性将是O（MNlogM）结束。与直接合并一样复杂！这里的区别仅在常量中，常量取决于实现。因此，Cloudera工程师提供的修补程序一直等待其批准已经有一年了，并且不可能在没有Cloudera管理层的推动下获得批准，因为这件事的性能影响非常小，甚至没有，因此您可以在JIRA票证中看到这一点讨论。也许他们会通过引入单独的shuffle实现而不是“改进”主要实现来解决这个问题，我们很快就会看到这一点。

















原文:https://0x0fff.com/spark-architecture-shuffle/


































原文:https://0x0fff.com/spark-architecture-shuffle/
