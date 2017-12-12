Flink提供了特定的Kafka Connector(连接器)，用于从Kafka Topic中读取以及写入数据到Kafka Topic。Flink Kafka消费者与Flink的检查点机制集成，提供一次性处理语义(exactly-once processing semantics)。为了达到这一目的，Flink并不完全依赖Kafka的消费者组偏移跟踪(Kafka’s consumer group offset tracking)，同时跟踪和检查点内部的这些偏移( tracks and checkpoints these offsets internally)。

请为您的用例和环境选择一个package（maven artifact id）和类名。 对于大多数用户来说，选择FlinkKafkaConsumer08（flink-connector-kafka的一部分）就足够了。

Maven|开始支持版本|消费者与生产者类名|Kafka版本|备注
---|---|---|---|---
flink-connector-kafka-0.8_2.10|1.0.0|FlinkKafkaConsumer08 FlinkKafkaProducer08|0.8.x|Uses the SimpleConsumer API of Kafka internally. Offsets are committed to ZK by Flink.
flink-connector-kafka-0.9_2.10|1.0.0|FlinkKafkaConsumer09 FlinkKafkaProducer09|0.9.x|Uses the new Consumer API Kafka.
flink-connector-kafka-0.10_2.10|1.2.0|FlinkKafkaConsumer010 FlinkKafkaProducer010|0.10.x|This connector supports Kafka messages with timestamps both for producing and consuming.
flink-connector-kafka-0.11_2.11|1.4.0|FlinkKafkaConsumer011 FlinkKafkaProducer011|0.11.x|Since 0.11.x Kafka does not support scala 2.10. This connector supports Kafka transactional messaging to provide exactly once semantic for the producer.

### 1. Maven依赖

```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8_2.10</artifactId>
  <version>1.4-SNAPSHOT</version>
</dependency>
```
请注意，流连接器当前不是二进制分发的一部分。 在这里查看如何链接到它们进行集群执行。

### 2. Kafka消费者

Flink的Kafka消费者称为`FlinkKafkaConsumer08`(或Kafka 0.9.0.x版本为`FlinkKafkaConsumer09`)。可以访问一个或多个Kafka Topic。

Kafka消费者的构造函数接受如下参数:

(1) Kafka Topic名称 或者 Kafka Topic名称列表

(2) 用于反序列化Kafka数据的`DeserializationSchema`/`KeyedDeserializationSchema`

(3) Kafka消费者的配置。需要以下属性：
- bootstrap.servers(逗号分隔的Kafka broker列表
- zookeeper.connect(逗号分隔的Zookeeper服务器)(仅仅是Kafka 0.8的必需属性)
- group.id(消费者group的id)


Example:

Java版本:
```Java
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
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

#### 2.1 DeserializationSchema

Flink Kafka 消费者需要知道如何将Kafka中的二进制数据转换为Java/Scala对象。DeserializationSchema允许用户指定这样的范式。对每个Kafka消息都会调用`T deserialize(byte[] message)`方法，继续传递来自Kafka的值。



#### 2.2 Kafka Consumers Start Position Configuration

#### 2.3 Kafka Consumers and Fault Tolerance

#### 2.4 Kafka Consumers Partition Discovery

#### 2.5 Kafka Consumers Offset Committing Behaviour Configuration

#### 2.6 Kafka Consumers and Timestamp Extraction/Watermark Emission

### 3. Kafka生产者

#### 3.1 Kafka Producers and Fault Tolerance
