---
layout: post
author: sjf0115
title: Spark Persist,Cache以及Checkpoint
date: 2018-07-11 12:55:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-persist-cache-checkpoint
---

要重用RDD（弹性分布式数据集），Apache Spark提供了许多选项，包括：
- Persisting
- Caching
- Checkpointing

了解每个的用法非常重要，下面我们将了解每一个的用法。重用意味着将计算和数据存储在内存中，并在不同的算子中多次重复使用。通常，在处理数据时，我们需要多次使用相同的数据集。例如，许多机器学习算法（如K-Means）在生成模型之前会对数据进行多次迭代。如果处理过程中的中间结果没有持久存储在内存中，这意味着你需要将中间结果存储在磁盘上，这会降低整体性能，因为与RAM相比，从磁盘访问数据就像是从隔壁或从其他国家获取内容。下面我们看一下在不同存储设备上的访问时间：

![]()

注意硬盘访问时间和RAM访问时间。这就是为什么Hadoop MapReduce与Spark相比速度慢的原因，因为每个MapReduce迭代都会在磁盘上读取或写入数据。Spark在内存中处理数据，如果使用不当将导致作业执行期间性能下降。让我们首先从持久RDD到内存开始，但首先我们看看为什么我们需要这个。

假设我们执行以下Spark语句：
```scala
val textFile = sc.textFile("file:///c://fil.txt")
textFile.first()
textFile.count()
```
第一行读取内存中的文件内容，读取操作是Transformation操作，因此不会有任何执行。Spark直到遇到Action操作才会惰性地执行DAG。接下来的两行是Action操作，它们为每个Action操作生成一个单独的作业。第二行得到RDD的第一个文本行并打印出来。第三行计算RDD中的行数。这两个Action都会产生结果，但内部发生的事情是Spark为每个Action生成一个单独的作业，因此RDD计算了两次。现在让我们执行以下语句：
```scala
val textFile = sc.textFile("file:///c://fil.txt")
textFile.cache()
textFile.first()
textFile.count()
// again execute the same set of commands
textFile.first()
textFile.count()
```
我们来看看Shell应用程序UI界面。如果你正在运行Spark Shell，那么默认情况下，可以通过URL `http://localhost:4040` 访问此接口：

![]()

每个Action都会在Spark中生成一个单独的作业 从底部开始，前两组记录是作为first（）和count（）动作执行的作业。 中间2记录再次是相同的操作，但在此之前，RDD持久存储在RAM中。 由于Spark必须在第一个语句中重新计算RDD，因此持续时间没有改善。 但请注意在RDD保留在RAM中后执行的前2个作业。 这次完成每个作业的持续时间显着减少，这是因为Spark没有从磁盘重新计算RDD，而是处理RAM中持久存在的RDD，并且与硬盘访问相比RAM访问时间较少 我们在完成相同工作的时间较短。 现在让我们关注持久化，缓存和检查点


















原文：https://www.linkedin.com/pulse/persist-cache-checkpoint-apache-spark-shahzad-aslam/
