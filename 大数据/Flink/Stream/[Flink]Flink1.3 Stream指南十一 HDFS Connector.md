此连接器提供一个Sink，将分区文件写入Hadoop FileSystem支持的任何文件系统。要使用此连接器，添加以下依赖项：
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-filesystem_2.10</artifactId>
  <version>1.4-SNAPSHOT</version>
</dependency>
```
备注:

streaming 连接器目前还不是二进制发布包的一部分，请参阅[此处](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/linking.html)来了解有关如何将程序与Libraries打包以进行集群执行的信息。

### 文件分桶的Sink(Bucketing File Sink)

分桶(Bucketing)行为以及写入数据操作都可以配置，我们稍后会讲到。下面展示了如何通过默认配置创建分桶的Sink，输出到按时间切分的滚动文件中：

Java版本:
```java
DataStream<String> input = ...;
input.addSink(new BucketingSink<String>("/base/path"));
```

Scala版本:
```
val input: DataStream[String] = ...
input.addSink(new BucketingSink[String]("/base/path"))
```

这里唯一必需的参数是这些分桶文件存储的基本路径`/base/path`。可以通过指定自定义bucketer，writer和batch大小来进一步配置sink。

默认情况下，分桶sink通过元素到达时当前系统时间来切分的，并使用`yyyy-MM-dd--HH`时间格式来命名这些分桶。这个时间格式与当前的系统时间传入`SimpleDateFormat`来形成一个桶的路径。每当遇到一个新的时间就会创建一个新的桶。例如，如果你有一个包含分钟的最细粒度时间格式，那么你将会每分钟获得一个新桶。每个桶本身就是一个包含part文件的目录(Each bucket is itself a directory that contains several part files)：Sink的每个并行实例都将创建自己的part文件，当part文件变得太大时，会紧挨着其他文件创建一个新的part文件。当一个桶在最近没有被写入数据时被视为非活跃的。当桶变得不活跃时，打开的part文件将被刷新(flush)并关闭。默认情况下，sink每分钟都会检查非活跃的桶，并关闭一分钟内没有写入数据的桶。可以在`BucketingSink上`使用`setInactiveBucketCheckInterval()`和`setInactiveBucketThreshold()`配置这些行为。

你还可以使用`BucketingSink上`的`setBucketer()`指定自定义bucketer。如果需要，bucketer可以使用元素或元组的属性来确定bucket目录。

默认的writer是`StringWriter`。对传入的元素调用`toString()`，并将它们写入part文件，用换行符分隔。要在`BucketingSink`上指定一个自定义的writer，使用`setWriter()`方法即可。如果要写入Hadoop SequenceFiles文件中，可以使用提供的`SequenceFileWriter`，并且可以配置使用压缩格式。

最后一个配置选项是batch大小。这指定何时关闭part文件，并开启一个新文件。(默认part文件大小为384MB)。

```java
DataStream<Tuple2<IntWritable,Text>> input = ...;

BucketingSink<String> sink = new BucketingSink<String>("/base/path");
sink.setBucketer(new DateTimeBucketer<String>("yyyy-MM-dd--HHmm"));
sink.setWriter(new SequenceFileWriter<IntWritable, Text>());
sink.setBatchSize(1024 * 1024 * 400); // this is 400 MB,

input.addSink(sink);
```

Scala版本:
```
val input: DataStream[Tuple2[IntWritable, Text]] = ...

val sink = new BucketingSink[String]("/base/path")
sink.setBucketer(new DateTimeBucketer[String]("yyyy-MM-dd--HHmm"))
sink.setWriter(new SequenceFileWriter[IntWritable, Text]())
sink.setBatchSize(1024 * 1024 * 400) // this is 400 MB,

input.addSink(sink)
```
上面例子将创建一个sink，写入遵循下面格式的分桶文件中：
```
/base/path/{date-time}/part-{parallel-task}-{count}
```
其中date-time是从日期/时间格式获得的字符串，parallel-task是并行sink实例的索引，count是由于batch大小而创建的part文件的运行编号。


备注:
```
Sink版本:1.3
```

原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/connectors/filesystem_sink.html#hdfs-connector
