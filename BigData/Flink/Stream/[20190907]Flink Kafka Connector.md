
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

为了访问 Kafka 消息的键，值和元数据，`KafkaDeserializationSchema` 具有以下反序列化方法 `T deserialize(ConsumerRecord<byte[], byte[]> record)`。为方便起见，Flink 提供以下模式:
- TypeInformationSerializationSchema（和TypeInformationKeyValueSerializationSchema）创建基于Flink的模式TypeInformation。如果Flink编写和读取数据，这将非常有用。此模式是其他通用序列化方法的高性能Flink替代方案。
- JsonDeserializationSchema（和JSONKeyValueDeserializationSchema）将序列化的JSON转换为ObjectNode对象，可以使用objectNode.get（“field”）作为（Int / String / ...）（）从中访问字段。KeyValue objectNode包含一个“key”和“value”字段，其中包含所有字段，以及一个可选的“元数据”字段，用于公开此消息的偏移量/分区/主题。
- AvroDeserializationSchema它使用静态提供的模式读取使用Avro格式序列化的数据。它可以从Avro生成的类（AvroDeserializationSchema.forSpecific(...)）中推断出模式，也可以GenericRecords 使用手动提供的模式（with AvroDeserializationSchema.forGeneric(...)）。此反序列化架构要求序列化记录不包含嵌入式架构。


### 4.2 Kafka Consumers Start Position Configuration

### 4.3 Kafka Consumers and Fault Tolerance

### 4.4 Kafka Consumers Partition Discovery

### 4.5 Kafka Consumers Offset Committing Behaviour Configuration

### 4.6 Kafka Consumers and Timestamp Extraction/Watermark Emission

### 3. Kafka生产者

#### 3.1 Kafka Producers and Fault Tolerance






原文:[Apache Kafka Connector](https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/connectors/kafka.html)
