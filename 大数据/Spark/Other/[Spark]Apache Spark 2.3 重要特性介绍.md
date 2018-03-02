---
layout: post
author: sjf0115
title: Apache Spark 2.3 重要特性介绍
date: 2018-02-28 17:17:17
tags:
  - Spark

categories: Spark
permalink: introducing-apache-spark-2-3
---

为了继续实现 Spark 更快，更轻松，更智能的目标，Spark 2.3 在许多模块都做了重要的更新，比如 `Structured Streaming` 引入了低延迟的持续处理；支持 `stream-to-stream joins`；通过改善 `pandas UDFs` 的性能来提升 PySpark；支持第四种调度引擎 `Kubernetes clusters`（其他三种分别是自带的独立模式Standalone，YARN、Mesos）。除了这些比较具有里程碑的重要功能外，Spark 2.3 还有以下几个重要的更新：
- 引入 `DataSource v2 APIs` [SPARK-15689, SPARK-20928]
- 矢量化的 `ORC reader` [SPARK-16060]
- `Spark History Server v2 with K-V store` [SPARK-18085]
- 基于 `Structured Streaming` 的机器学习管道API模型 [SPARK-13030, SPARK-22346, SPARK-23037]
- `MLlib` 增强 [SPARK-21866, SPARK-3181, SPARK-21087, SPARK-20199]
- `Spark SQL` 增强 [SPARK-21485, SPARK-21975, SPARK-20331, SPARK-22510, SPARK-20236]

这篇文章将简单地介绍上面一些高级功能和改进，更多的特性请参见 [Spark 2.3 release notes](https://spark.apache.org/releases/spark-release-2-3-0.html)。


### 1. 毫秒延迟的持续流处理

`Apache Spark 2.0` 的 `Structured Streaming` 将微批次处理（micro-batch processing）从它的高级 APIs 中解耦出去，原因有两个：首先，开发人员更容易学习这些 API，不需要考虑这些 APIs 的微批次处理情况；其次，它允许开发人员将一个流视为一个无限表，他们查询流的数据，就像他们查询静态表一样简便。

但是，为了给开发人员提供不同的流处理模式，社区引入了一种新的毫秒级低延迟（millisecond low-latency）模式：持续模式（continuous mode）。

在内部，结构化的流引擎逐步执行微批中的查询计算，执行周期由触发器间隔决定，这个延迟对大多数真实世界的流应用程序来说是可以容忍的。



译文：
原文： https://databricks.com/blog/2018/02/28/introducing-apache-spark-2-3.html
