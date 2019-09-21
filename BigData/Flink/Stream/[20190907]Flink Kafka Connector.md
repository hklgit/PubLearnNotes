---
layout: post
author: sjf0115
title: Flink Kafka Connector
date: 2019-09-10 12:14:17
tags:
  - Flink
  - Flink Stream

categories: Flink
permalink: flink-kafka-connector
---

## 1. 简介

Flink 提供了指定的 Kafka Connector 用于从 Kafka Topic 中读取数据以及写入数据到 Kafka Topic 中。Flink Kafka 消费者与 Flink 的检查点机制集成，提供 Exactly-Once 处理语义。为了达到这一目的，Flink 不需要完全依赖 Kafka 的消费组偏移量追踪，而是在内部跟踪和检查这些偏移。

根据你的用例和环境选择一个包（maven artifact id）和类名。 对于大多数用户来说，选择 FlinkKafkaConsumer08（flink-connector-kafka 的一部分）就足够了。

Maven|开始支持版本|消费者与生产者类名|Kafka版本|备注
---|---|---|---|---
flink-connector-kafka-0.8_2.11|1.0.0|FlinkKafkaConsumer08 FlinkKafkaProducer08|0.8.x|使用 [SimpleConsumer](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example) API。偏移量被提交到ZooKeeper上。
flink-connector-kafka-0.9_2.11|1.0.0|FlinkKafkaConsumer09 FlinkKafkaProducer09|0.9.x|使用新版 [Consumer](http://kafka.apache.org/documentation.html#newconsumerapi) API。
flink-connector-kafka-0.10_2.11|1.2.0|FlinkKafkaConsumer010 FlinkKafkaProducer010|0.10.x|这个连接器支持生产与消费的[带时间戳的Kafka消息](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message)。
flink-connector-kafka-0.11_2.11|1.4.0|FlinkKafkaConsumer011 FlinkKafkaProducer011|0.11.x| Kafka 0.11.x 版本不支持 scala 2.10 版本。此连接器支持 [Kafka 事务消息](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging) 可以为生产者提供 Exactly-Once 语义。
flink-connector-kafka_2.11 | 1.7.0 | FlinkKafkaConsumer
FlinkKafkaProducer | >= 1.0.0 | 这是一个通用的 Kafka 连接器，会跟踪最新版本的 Kafka 客户端。连接器使用的客户端版本可能会在 Flink 版本之间发生变化。从 Flink 1.9 版本开始，使用 Kafka 2.2.0 客户端。新版本 Kafka 客户端可以向后兼容 0.10.0 版本或更高版本，可以使用这个连接器。但是如果使用旧版本的 Kafka(0.11、0.10、0.9或0.8)，则应该使用与 kafka 版本对应的连接器。

然后，导入 Maven 项目中的连接器:
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka_2.11</artifactId>
  <version>1.9.0</version>
</dependency>
```
请注意，目前流连接器不是二进制包的一部分。

## 2. 安装Kafka

- 根据 [Kafka 安装与启动](http://smartsi.club/kafka-setup-and-run.html) 说明下载代码并启动服务器（每次启动应用程序前都需要启动 Zookeeper 和 Kafka 服务器）。
- 如果 Kafka 和 Zookeeper 服务器运行在远程机器上，那么 config/server.properties 文件中的 `advertised.host.name` 必须设置为计算机的IP地址。

## 3. Kafka 1.0.0+ Connector

从 Flink 1.7 版本开始，有一个新的通用 Kafka 连接器，不需要绑定特定版本的 Kafka。相反，绑定的是Flink 发布时最新版本的 Kafka。

如果你的 Kafka Broker 版本是1.0.0或更高版本，那么应使用这个 Kafka 连接器。 如果使用旧版本的 Kafka（0.11,0.10,0.9或0.8），则应使用与 Broker 对应的连接器。

### 3.1 兼容性

通过 Kafka 客户端 API 和 Broker 的兼容性保证，通用 Kafka 连接器与老版本以及新版本的 Kafka Broker 兼容。

### 3.2 将Kafka连接器从0.11版本迁移到通用连接器

为了迁移，请参阅[升级作业和Flink版本指南](https://ci.apache.org/projects/flink/flink-docs-release-1.9/ops/upgrading.html)以及:
- 使用Flink 1.9或更新版本。
- 不要同时升级 Flink 和算子。
- 确保作业中使用的 Kafka 消费者以及 Kafka 生产者分配了唯一标识符(uid)。
- 使用 SavePoint 功能停止作业。

### 3.3 用法

要使用通用 Kafka 连接器，请为其添加依赖关系：
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka_2.11</artifactId>
  <version>1.9.0</version>
</dependency>
```
然后实例化新的数据源(FlinkKafkaConsumer)和接收器(FlinkKafkaProducer)。 除了从模块和类名中删除特定的 Kafka 版本之外，API向后兼容 Kafka 0.11 连接器。

## 4. Kafka消费者

Flink 的 Kafka 消费者称为`FlinkKafkaConsumer08`(0.9.0.x 版本为`FlinkKafkaConsumer09`，大于等于 1.0.0 版本为 `FlinkKafkaConsumer`)。提供了可以访问一个或多个 Kafka Topic 的功能。

Kafka 消费者的构造函数接受如下参数:
- Kafka Topic 名称或者Kafka Topic名称列表
- 用于反序列化 Kafka 数据的`DeserializationSchema`/`KafkaDeserializationSchema`
- Kafka消费者的配置。需要以下属性：`bootstrap.servers`(逗号分隔的Kafka broker列表、`zookeeper.connect`(逗号分隔的Zookeeper服务器)(对于Kafka 0.8是必需的)、`group.id`(消费组的Id)。

Java版本:
```Java
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
```

Scala版本:
```
val properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
stream = env.addSource(new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties)).print
```

### 4.1 DeserializationSchema

Flink Kafka 消费者需要知道如何将 Kafka 中的二进制数据转换为 Java/Scala 对象。`DeserializationSchema` 允许用户指定这样的范式。每个 Kafka 消息都会调用 `T deserialize(byte[] message)` 方法。

从 `AbstractDeserializationSchema` 开始通常很有帮助，它负责将生成的 Java/Scala 类型描述为 Flink 的类型系统。实现 `vanilla DeserializationSchema` 的用户需要自己实现 `getProducedType(...)` 方法。

为了访问 Kafka 消息的键，值和元数据，`KafkaDeserializationSchema` 具有 `T deserialize(ConsumerRecord<byte[], byte[]> record)` 反序列化方法。为方便起见，Flink 提供以下 schema:
- `TypeInformationSerializationSchema`(以及 `TypeInformationKeyValueSerializationSchema`) 会基于 Flink 的 TypeInformation 创建 schema。对 Flink 读写数据会非常有用。此 schema 是其他通用序列化方法的高性能替代方案。
- `JsonDeserializationSchema`(以及 `JSONKeyValueDeserializationSchema`)将序列化的 JSON 转换为 ObjectNode 对象，可以使用 `objectNode.get("field").as(Int/String/...)()` 来访问某个字段。KeyValue objectNode 包含一个"key"和"value"字段，这包含了所有字段，以及一个可选的"元数据"字段，可以用来查询此消息的偏移量/分区/主题。
- `AvroDeserializationSchema` 可以使用静态模式读取 Avro 格式序列化的数据。可以从 Avro 生成的类( `AvroDeserializationSchema.forSpecific(...)`) 中推断出 schema，也可以GenericRecords 使用手动提供的模式（with AvroDeserializationSchema.forGeneric(...)）。此反序列化架构要求序列化记录不包含嵌入式架构。

### 4.2 Kafka消费者起始位置配置

Flink Kafka Consumer 允许配置如何确定 Kafka 分区的起始位置。

Java版本:
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer08<String> myConsumer = new FlinkKafkaConsumer08<>(...);
// 从最早的记录开始消费
myConsumer.setStartFromEarliest();
// 从最近的记录开始消费
myConsumer.setStartFromLatest();
// 从指定时间戳(毫秒)开始消费
myConsumer.setStartFromTimestamp(...);
// 默认行为 从指定消费组偏移量开始消费
myConsumer.setStartFromGroupOffsets();
DataStream<String> stream = env.addSource(myConsumer);
...
```
Scala版本:
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val myConsumer = new FlinkKafkaConsumer08[String](...)
myConsumer.setStartFromEarliest()      // start from the earliest record possible
myConsumer.setStartFromLatest()        // start from the latest record
myConsumer.setStartFromTimestamp(...)  // start from specified epoch timestamp (milliseconds)
myConsumer.setStartFromGroupOffsets()  // the default behaviour

val stream = env.addSource(myConsumer)
...
```
Flink 所有版本的 Kafka Consumer 都具有上述配置起始位置的方法。
- `setStartFromGroupOffsets`（默认行为）：从消费者组(在消费者属性中 `group.id` 配置)提交到 Kafka Broker(Kafka 0.8版本提交到 ZooKeeper)的偏移量开始读取分区。如果找不到分区的偏移量，将使用 `auto.offset.reset` 属性中的设置。
- `setStartFromEarliest()/ setStartFromLatest()`：从最早/最新记录开始读取。在这个模式下，Kafka 提交的偏移量不用起任何作用，可以忽略。
- `setStartFromTimestamp(long)`：从指定的时间戳开始。对于每个分区，时间戳大于或等于指定时间戳的记录会被用作起始位置。如果分区的最新记录早于时间戳，则只会从分区中读取最新记录。在这个模式下，Kafka 提交的偏移量不用起任何作用，可以忽略。

你还可以为每个分区指定消费者应该开始的确切偏移量。

Java版本:
```java
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);

myConsumer.setStartFromSpecificOffsets(specificStartOffsets);
```
Scala版本:
```scala
val specificStartOffsets = new java.util.HashMap[KafkaTopicPartition, java.lang.Long]()
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L)

myConsumer.setStartFromSpecificOffsets(specificStartOffsets)
```
上面的示例配置消费者从主题 myTopic 的0,1和2分区指定偏移量开始消费。偏移量应该是消费者从每个分区读取的下一条记录。请注意，如果消费者需要读取的分区在提供的偏移量 Map 中没有指定的偏移量，那么自动转换为默认的消费组偏移量(例如，`setStartFromGroupOffsets()`)。

请注意，当作业从故障中自动恢复或使用保存点手动恢复时，这些起始位置配置方法不会影响起始位置。在恢复时，每个 Kafka 分区的起始位置由存储在保存点或检查点中的偏移量确定。

### 4.3 Kafka消费者与容错

启用 Flink 的检查点后，Flink Kafka 消费者会从 Topic 中消费记录，并以一致的方式定期检查(checkpoint)其所有 Kafka 偏移量以及其他算子的状态。如果作业失败，Flink 会将流式程序恢复到最新检查点的状态，并从存储在检查点中的偏移量重新开始消费来自 Kafka 的记录。

因此，检查点间隔定义了程序在发生故障时最多可以回退多少。要使用容错的 Kafka 消费者，需要在运行环境中启用拓扑的检查点。

Java版本:
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 每5s进行一次checkpoint
env.enableCheckpointing(5000);
```
Scala版本:
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()
// 每5s进行一次checkpoint
env.enableCheckpointing(5000)
```
请另外注意，如果有足够的 slot 可用于重新启动拓扑，那么 Flink 才能重新启动拓扑。因此，如果拓扑由于与 TaskManager 断开而失败，那么仍必须有足够的可用 slot。如果未启用检查点，Kafka 消费者可以定期向 Zookeeper 提交偏移量。

### 4.4 Kafka 消费者分区发现

#### 4.4.1 分区发现

Flink Kafka 消费者支持动态发现创建的 Kafka 分区，并使用 Exactly-Once 处理语义来消费。在初始检索分区元数据（即，当作业开始运行时）发现的所有分区会从最早的偏移量开始消费。

默认情况下，分区发现是禁用的。如果要启用它，需要设置 `flink.partition-discovery.interval-millis` 非负值，表示以毫秒为单位的发现间隔。

> 当使用 Flink 1.3.x 之前的版本，消费者从保存点恢复时，无法在恢复的运行启用分区发现。如果要启用，恢复将失败并抛出异常。在这种情况下，为了使用分区发现，需要在 Flink 1.3.x 版本中生成保存点，然后再从中恢复。

#### 4.4.2 Topic发现

Flink Kafka 消费者还能够使用正则表达式基于 Topic 名称的模式匹配来发现Topic。请参阅以下示例：
```

```

### 4.5 Kafka Consumers Offset Committing Behaviour Configuration

### 4.6 Kafka Consumers and Timestamp Extraction/Watermark Emission

### 3. Kafka生产者

#### 3.1 Kafka Producers and Fault Tolerance






原文:[Apache Kafka Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/connectors/kafka.html)
