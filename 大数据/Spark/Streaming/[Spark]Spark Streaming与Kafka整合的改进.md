---
layout: post
author: sjf0115
title: Spark Streaming 与 Kafka 整合的改进
date: 2018-03-16 11:28:01
tags:
  - Spark
  - Spark Stream

categories: Spark
permalink: improvements-to-kafka-integration-of-spark-streaming
---

Apache Kafka 正在迅速成为最受欢迎的开源流处理平台之一。我们在 Spark Streaming 中也看到了同样的趋势。因此，在 Apache Spark 1.3 中，我们专注于对 Spark Streaming 与 Kafka 集成进行重大改进。主要增加如下：
- 为 Kafka 新增了 Direct API - 这允许每个 Kafka 记录在发生故障时只处理一次，并且不使用 [Write Ahead Logs](https://databricks.com/blog/2015/01/15/improved-driver-fault-tolerance-and-zero-data-loss-in-spark-streaming.html)。这使得 Spark Streaming + Kafka 流水线更高效，同时提供更强大的容错保证。
- 为 Kafka 新增了 Python API - 这样你就可以在 Python 中处理 Kafka 数据。

在本文中，我们将更详细地讨论这些改进。

### 1. Direct API

Spark Streaming 自成立以来一直支持 Kafka，Spark Streaming 与 Kafka 在生产环境中的很多地方一起使用。但是，Spark 社区要求更好的容错保证和更强的可靠性语义。为了满足这一需求，Spark 1.2 引入了 `Write Ahead Logs`（WAL）。它可以确保在发生故障时从任何可靠的数据源（即Flume，Kafka和Kinesis等事务源）接收的数据不会丢失（即至少一次语义）。即使对于像 plain-old 套接字这样的不可靠（即非事务性）数据源，它也可以最大限度地减少数据的丢失。

然而，对于允许从数据流中的任意位置重放数据流的数据源（例如 Kafka），我们可以实现更强大的容错语义，因为这些数据源让 Spark Streaming 可以更好地控制数据流的消费。Spark 1.3 引入了 Direct API 概念，即使不使用 Write Ahead Logs 也可以实现 exactly-once 语义。让我们来看看集成 Apache Kafka 的 Spark Direct API 的细节。

### 2. 我们是如何构建它？

从高层次的角度看，之前的 Kafka 集成与 `Write Ahead Logs`（WAL）一起工作如下：

(1) 运行在 Spark workers/executors 上的 Kafka Receivers 连续不断地从 Kafka 中读取数据，这用到了 Kafka 高级消费者API。

(2) 接收到的数据存储在 Spark 的 worker/executor的内存上，同时写入到 WAL（拷贝到HDFS）上。Kafka Receiver 只有在数据保存到日志后才会更新 Zookeeper中的 Kafka 偏移量。

(3) 接收到的数据及其WAL存储位置信息也可靠地存储。在出现故障时，这些信息用于从故障中恢复，重新读取数据并继续处理。















































原文：https://databricks.com/blog/2015/03/30/improvements-to-kafka-integration-of-spark-streaming.html
