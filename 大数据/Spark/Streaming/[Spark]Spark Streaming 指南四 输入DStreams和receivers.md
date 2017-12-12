### 1. 输入DStream与Receiver

输入DStreams表示从源中获取输入数据流的DStreams。在指南一示例中，lines表示输入DStream，它代表从netcat服务器获取的数据流。每一个输入DStream(除 file stream)都 与一个接收器Receiver相关联，接收器从源中获取数据，并将数据存入Spark内存中来进行处理。
输入DStreams表示从数据源获取的原始数据流。Spark Spark Streaming提供了两类内置的流源（streaming sources）：
```
基本源(Basic sources) - ：StreamingContext API中直接可用的源。 示例：文件系统(file system)和套接字连接(socket connections)。
高级源(Advanced sources) - 例如Kafka，Flume，Kinesis等源可通过额外的实用程序类获得。 这些需要额外依赖。
```
我们将在本文稍后讨论这两类源。

请注意，如果希望在流应用程序中并行接收多个数据流，你可以创建多个输入DStream（在“性能调优”部分中进一步讨论）。这需要创建多个接收器（Receivers），来同时接收多个数据流。但请注意，Spark worker / executor是一个长期运行的任务，因此会占用分配给Spark Streaming应用程序的其中一个核（core）。因此，记住重要的一点，Spark Streaming应用程序需要分配足够的核心（或线程，如果在本地运行）以处理接收的数据，以及运行接收器。

**注意**

- 当在本地运行Spark Streaming程序时，不要使用“local”或“local [1]”作为master URL。 这两个都意味着只会有一个线程用于本地任务运行。 如果使用基于接收器（例如套接字，Kafka，Flume等）的输入DStream，那么唯一的那个线程会用于运行接收器，将不会有处理接收到的数据的线程。 因此，在本地运行时，始终使用"local [n]"作为 master URL，其中 n > 要运行的 receiver 数（有关如何设置主服务器的信息，请参阅Spark属性）。
- 将逻辑扩展到集群上运行，分配给Spark Streaming应用程序的核心数量必须大于 receiver 数量。 否则系统将只接收数据，而不处理。

### 2. 源

#### 2.1 基本源

在指南一 Example 中我们已经了解到 ssc.socketTextStream（...），它通过TCP套接字连接从数据服务器获取文本数据创建DStream。 除了套接字，StreamingContext API提供了把文件作为输入源创建DStreams的方法。

##### 2.1.1 File Streams

可以从与HDFS API兼容的任何文件系统（即HDFS，S3，NFS等）上的文件读取数据，DStream可以使用如下命令创建：

Java:
```
streamingContext.fileStream<KeyClass, ValueClass, InputFormatClass>(dataDirectory);
```
Scala:
```
streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
```
Spark Streaming会监视dataDirectory目录并处理在该目录中创建的任何文件（不支持嵌套目录中写入的文件）。

***注意***
```
1 所有文件必须具有相同的数据格式
2 所有文件必须在dataDirectory目录下创建，文件是自动移动到dataDirectory下,并重命名
3 移动到dataDirectory目录后，不得进行更改文件。 因此，如果文件被连续追加数据，新的数据将不会被读取。
```

对于简单的文本文件，有一个更简单的方法：
```
streamingContext.textFileStream（dataDirectory）
```
文件流不需要运行接收器（Receiver），因此不需要分配内核。

==Python API==： fileStream在Python API中不可用，只有textFileStream可用。

##### 2.1.2 Streams based on Custom Receivers

可以使用通过自定义接收器接收的数据流创建DStream。 有关详细信息，请参阅[自定义接收器指南](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)。

##### 2.1.3 Queue of RDDs as a Stream

要使用测试数据测试Spark Streaming应用程序，还可以使用streamingContext.queueStream（queueOfRDDs）基于RDD队列创建DStream。 推送到队列中的每个RDD将被视为DStream中的一批次数据，并像流一样处理。

#### 2.2 高级源


这类源需要外部的非Spark库接口（require interfacing with external non-Spark libraries），其中一些需要复杂依赖（例如，Kafka和Flume）。 因此，为了尽量减少依赖的版本冲突问题，从这些源创建DStreams的功能已经移动到可以在必要时显式链接单独的库（ the functionality to create DStreams from these sources has been moved to separate libraries that can be linked to explicitly when necessary）。

请注意，这些高级源在Spark Shell中不可用，因此基于这些高级源的应用程序无法在shell中测试。 如果你真的想在Spark shell中使用它们，那么你必须下载相应的Maven组件的JAR及其依赖项，并将其添加到类路径中。

介绍一下常用的高级源：

- Kafka：Spark Streaming 2.1.0与Kafka代理版本0.8.2.1或更高版本兼容。 有关更多详细信息，请参阅[Kafka集成指南](http://spark.apache.org/docs/latest/streaming-kafka-integration.html)。
- Flume：Spark Streaming 2.1.0与Flume 1.6.0兼容。 有关更多详细信息，请参阅[Flume集成指南](http://spark.apache.org/docs/latest/streaming-flume-integration.html)。
- Kinesis：Spark Streaming 2.1.0与Kinesis Client Library 1.2.1兼容。 有关更多详细信息，请参阅[Kinesis集成指南](http://spark.apache.org/docs/latest/streaming-kinesis-integration.html)。


### 3. 自定义源

这在Python中还不支持。

输入DStreams也可以从自定义数据源创建。 所有你需要做的是实现一个用户定义的接收器（Receiver），可以从自定义源接收数据，并推送到Spark。 有关详细信息，请参阅[自定义接收器指南](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)。

### 4. Receiver可靠性

基于Receiver的可靠性，可以分为两种数据源。
如Kafka和Flume之类的源允许传输的数据被确认。 如果从这些可靠源接收数据的系统正确地确认接收的数据，则可以确保不会由于任何种类的故障而丢失数据。 这导致两种接收器（Receiver）：

```
1 可靠的接收器 - 当数据已被接收并存储副本在Spark中时，可靠的接收器正确地向可靠的源发送确认。
2 不可靠的接收器 - 不可靠的接收器不会向源发送确认。 这可以用在不支持确认机制的源上，或者由于确认机制的复杂性时，使用可靠源但不发送确认。
```





