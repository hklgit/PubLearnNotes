Spark的核心概念是弹性分布式数据集（RDD），RDD是一个可容错、可并行操作的分布式元素集合。有两种方法可以创建RDD对象：
- 在驱动程序中并行化操作集合对象来创建RDD
- 从外部存储系统中引用数据集（如：共享文件系统、HDFS、HBase或者其他Hadoop支持的数据源）。

### 1. 并行化集合

通过在驱动程序中的现有集合上调用`JavaSparkContext`的`parallelize`方法创建并行化集合(Parallelized collections)。集合的元素被复制以形成可以并行操作的分布式数据集。 例如，下面是如何创建一个包含数字1到5的并行化集合：

Java版本：
```
List<Integer> list = Arrays.asList(1,2,3,4,5);
JavaRDD<Integer> rdd = sc.parallelize(list);
```
Scala版本：
```
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
```
Python版本：
```
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
```
RDD一旦创建，分布式数据集（distData）可以并行操作。 例如，我们可以调用`distData.reduce（（a，b） - > a + b）`来实现对列表元素求和。 我们稍后介绍分布式数据集的操作。

并行化集合的一个重要参数是分区（partition）数，将分布式数据集分割成多少分区。Spark集群中每个分区运行一个任务(task)。典型场景下，一般为每个CPU分配2－4个分区。但通常而言，Spark会根据你集群的状况，自动设置分区数。当然，你可以给`parallelize`方法传递第二个参数来手动设置分区数（如：`sc.parallelize(data, 10)`）。注意：Spark代码里有些地方仍然使用分片（slice）这个术语(分区的同义词)，主要为了保持向后兼容。


### 2. 外部数据集

Spark可以从Hadoop支持的任何存储数据源创建分布式数据集，包括本地文件系统，HDFS，Cassandra，HBase，Amazon S3等。Spark可以支持文本文件，SequenceFiles以及任何其他Hadoop 输入格式。

文本文件RDD可以使用`SparkContext`的`textFile`方法创建。该方法根据URL获取文件（机器上的本地路径，或`hdfs://`，`s3n://`等等）. 下面是一个示例调用：

Java版本：
```
JavaRDD<String> distFile = sc.textFile("data.txt");
```
Scala版本：
```
scala> val distFile = sc.textFile("data.txt")
distFile: org.apache.spark.rdd.RDD[String] = data.txt MapPartitionsRDD[10] at textFile at <console>:26
```
Python版本：
```
>>> distFile = sc.textFile("data.txt")
```

一旦创建完成，distFiile 就能做数据集操作。例如，我们可以用下面的方式使用 map 和 reduce 操作将所有行的长度相加：
```
distFile.map(s -> s.length()).reduce((a, b) -> a + b);
```

Spark读文件时一些注意事项：

(1) 如果使用本地文件系统路径，在所有工作节点上该文件必须都能用相同的路径访问到。要么复制文件到所有的工作节点，要么使用网络的方式共享文件系统。

(2) Spark 所有基于文件的输入方法，包括 `textFile`，能很好地支持文件目录，压缩过的文件和通配符。例如，你可以使用:
```
textFile("/my/directory")
textFile("/my/directory/*.txt")
textFile("/my/directory/*.gz")
```

(3) `textFile` 方法也可以选择第二个可选参数来控制文件分区(partitions)数目，默认情况下，Spark为每一个文件块创建一个分区（HDFS中分块大小默认为128MB），你也可以通过传递一个较大数值来请求更多分区。 注意的是，分区数目不能少于分块数目。

除了文本文件，Spark的Java API还支持其他几种数据格式：

(1) JavaSparkContext.wholeTextFiles可以读取包含多个小文本文件的目录，并将它们以（文件名，内容）键值对返回。 这与textFile相反，textFile将在每个文件中每行返回一条记录。
```
JavaPairRDD<String, String> rdd = sc.wholeTextFiles("/home/xiaosi/wholeText");
List<Tuple2<String, String>> list = rdd.collect();
for (Tuple2<?, ?> tuple : list) {
    System.out.println(tuple._1() + ": " + tuple._2());
}
```
(2) 对于SequenceFiles，可以使用SparkContext的sequenceFile [K，V]方法，其中K和V是文件中的键和值的类型。 这些应该是Hadoop的Writable接口的子类，如IntWritable和Text。

(3) 对于其他Hadoop InputFormats，您可以使用JavaSparkContext.hadoopRDD方法，该方法采用任意JobConf和输入格式类，键类和值类。 将这些设置与使用输入源的Hadoop作业相同。 您还可以使用基于“新”MapReduce API（org.apache.hadoop.mapreduce）的InputFormats的JavaSparkContext.newAPIHadoopRDD。

(4) JavaRDD.saveAsObjectFile 和 SparkContext.objectFile 支持保存一个RDD，保存格式是一个简单的 Java 对象序列化格式。这是一种效率不高的专有格式，如 Avro，它提供了简单的方法来保存任何一个 RDD。


原文：http://spark.apache.org/docs/latest/programming-guide.html#resilient-distributed-datasets-rdds