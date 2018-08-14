我们在使用 Spark Streaming 读取 Kafka 中持续不断的流时，我们不得不考虑 Kafka Offset 的管理，以便在出现故障时可以恢复流应用程序。在这篇文章中，我们将介绍几种管理 Offset 的方法。

### 1. Offset 管理简介

Spark Streaming 与 Kafka 的集成允许用户从单个Kafka Topic 或多个 Kafka Topic 中读取消息。Kafka Topic 从分布式分区集合中接收消息。每个分区按顺序维护已接收的消息，由 offset 进行标识，也称为位置。开发人员可以利用 Offsets 来控制他们的 Spark Streaming 作业读取的位置，但它确实需要偏移管理。

Offsets 管理对于保证流式应用在整个生命周期中数据的连贯性是非常有益的。例如，如果在应用停止或者报错退出之前没有将offset保存在持久化数据库中，那么offset rangges就会丢失。更进一步说，如果没有保存每个分区已经读取的offset，那么Spark Streaming就没有办法从上次断开（停止或者报错导致）的位置继续读取消息。


### 2. 存储 Offset 到外部系统

### 3. 不管理 Offset



























原文：http://blog.cloudera.com/blog/2017/06/offset-management-for-apache-kafka-with-apache-spark-streaming/
