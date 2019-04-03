---
layout: post
author: sjf0115
title: Flink如何管理Kafka的消费偏移量
date: 2019-04-03 19:39:01
tags:
  - Flink
  - Flink Stream

categories: Flink
permalink: how-flink-manages-kafka-consumer-offsets
---

在这篇文章中我们将结合例子逐步讲解 Flink 是如何与 Kafka 工作来确保将 Kafka Topic 中的消息以 Exactly-Once 语义处理。

检查点(Checkpoint)是一种能使 Flink 从故障恢复的内部机制。检查点是 Flink 应用程序状态的一致性副本，包括了输入的读取位点。如果发生故障，Flink 通过从检查点加载应用程序状态来恢复应用程序，并从恢复的读取位点继续处理，就好像什么事情都没发生一样。你可以把检查点理解为电脑游戏的存档。如果你在游戏中存档之后发生了什么事情，你可以随时读档重来一次。

检查点使 Flink 具有容错能力，并确保在发生故障时也能保证流应用程序的语义。检查点每隔固定的间隔来触发，该间隔可以在应用中配置。

Flink 中的 Kafka 消费者是一个有状态的算子(operator)并且集成了 Flink 的检查点机制，它的状态是所有 Kafka 分区的读取偏移量。当一个检查点被触发时，每一个分区的偏移量都保存到这个检查点中。Flink 的检查点机制保证了所有算子任务的存储状态都是一致的，即它们存储状态都是基于相同的输入数据。当所有的算子任务成功存储了它们的状态，一个检查点才成功完成。因此，当从潜在的系统故障中恢复时，系统提供了 Excatly-Once 的状态更新语义。

下面我们将一步步的介绍 Flink 如何对 Kafka 消费偏移量（Offset）做检查点的。在本文的例子中，数据存储在 Flink 的 JobMaster 中。值得注意的是，在 POC 或生产用例下，这些数据通常是存储到一个外部文件系统（如HDFS或S3）中。

### 1. 第一步

如下所示，一个 Kafka topic，有两个partition，每个partition都含有 “A”, “B”, “C”, ”D”, “E” 5条消息。我们将两个partition的偏移量（offset）都设置为0.



### 2. 第二步
### 3. 第三步
### 4. 第四步
### 5. 第五步
### 6. 第六步














原文:[How Apache Flink manages Kafka consumer offsets](https://www.ververica.com/blog/how-apache-flink-manages-kafka-consumer-offsets)
