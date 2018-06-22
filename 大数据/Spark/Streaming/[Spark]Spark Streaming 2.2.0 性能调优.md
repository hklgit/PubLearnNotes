---
layout: post
author: sjf0115
title: Spark Streaming 2.2.0 性能调优
date: 2018-04-06 11:28:01
tags:
  - Spark
  - Spark Stream

categories: Spark
permalink: spark-streaming-performance-tuning
---

在集群上的 Spark Streaming 应用程序中获得最佳性能需要一些调整。本节介绍了可调整的多个 parameters （参数）和 configurations （配置）提高你的应用程序性能.在高层次上, 你需要考虑两件事情:

通过有效利用集群资源, Reducing the processing time of each batch of data （减少每批数据的处理时间）.

设置正确的 batch size （批量大小）, 以便 batches of data （批量的数据）可以像 received （被接收）处理一样快（即 data processing （数据处理）与 data ingestion （数据摄取）保持一致）.

### 接收数据的并行级别

通过网络接收数据（如Kafka，Flume，Socket等）需要将数据反序列化并存储在Spark中。如果数据接收成为系统的瓶颈，则考虑并行化接收数据。请注意，每个输入 DStream 会创建一个接收单个数据流的接收器（运行在 Worker 节点上）。因此，接收多个数据流可以通过创建多个输入 DStream 并将其配置为从数据源接收不同的数据流分区。例如，接收两个 Topic 数据的单个 Kafka 输入 DStream 可以拆分成两个 Kafka 输入流，每个输入流只接收一个 Topic。这将运行两个接收器，允许并行接收数据，从而提高整体吞吐量。这些多个 DStream 可以整合在一起组成一个 DStream。然后，应用在单个输入 DStream 上的转换可以应用在统一流上（union之后的流）。这如下完成。

Java版本：
```java
int numStreams = 5;
List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<>(numStreams);
for (int i = 0; i < numStreams; i++) {
  kafkaStreams.add(KafkaUtils.createStream(...));
}
JavaPairDStream<String, String> unifiedStream = streamingContext.union(kafkaStreams.get(0), kafkaStreams.subList(1, kafkaStreams.size()));
unifiedStream.print();
```
Scala版本：
```scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```
Python版本：
```python
numStreams = 5
kafkaStreams = [KafkaUtils.createStream(...) for _ in range (numStreams)]
unifiedStream = streamingContext.union(*kafkaStreams)
unifiedStream.pprint()
```

另一个应该考虑的参数是接收器的 `block interval`，由配置参数 `spark.streaming.blockInterval` 决定。对于大多数接收器，接收到的数据在存储到 Spark 内存之前会合并为数据块。每个 batch 中块的个数决定了在处理类似 map 转换操作中处理接收到数据的任务个数。每个接收器每个 batch 的任务个数将近似于 `batch interval / block interval`。例如，`block interval` 为 200 ms，`batch interval` 为 2s 的 batch 会创建10个任务。如果任务个数太少（即少于每台机器的 core 的数量），那么效率将会很低，因为所有可用 core 不会全都用于处理数据。要增加给定 `batch interval` 的任务个数，需要减小 `block interval`。但是，建议的 `block interval` 最小值大约为50毫秒，低于该值时，任务启动开销可能会有问题。

使用多输入流/接收器接收数据的另一种方法是显式对输入数据流重新分区（使用 `inputStream.repartition（<分区个数>）`）。这会在进一步处理之前将收到的批量数据分布到集群中指定数量的机器上。












原文：
