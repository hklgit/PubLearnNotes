离散流或者DStreams是Spark Streaming提供的基本抽象，它代表一个连续的数据流。从源中获取输入流，或者是输入流通过转换算子生成的处理后的数据流。在内部，DStreams由一系列连续的 RDD组成。这是Spark对不可变，分布式数据集的抽象（更多细节参见Spark编程指南）。 DStream中的每个RDD包含来自特定间隔的数据，如下图所示：

![image](http://spark.apache.org/docs/latest/img/streaming-dstream.png)

对DStream应用的任何操作都会转换为DStream隐含的RDD的操作。 例如，在指南一示例将行数据流转换单词数据流例子中，flatMap操作应用于lines这个DStreams的每个RDD，生成words这个DStreams的 RDD。过程如下图所示：

![image](http://spark.apache.org/docs/latest/img/streaming-dstream-ops.png)

这些隐含RDD转换操作由Spark引擎计算。 DStream操作隐藏了大部分细节，并为开发人员提供了更高级别的API以方便使用。 这些操作将在后面的章节中详细讨论。

