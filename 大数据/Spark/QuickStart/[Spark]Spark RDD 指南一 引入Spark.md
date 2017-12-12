### 1. Java版

Spark 2.1.1适用于Java 7及更高版本。 如果您使用的是Java 8，则Spark支持使用lambda表达式来简洁地编写函数，否则可以使用org.apache.spark.api.java.function包中的类。

请注意，从Spark 2.0.0开始，不支持Java 7，并且可能会在Spark 2.2.0中删除(Note that support for Java 7 is deprecated as of Spark 2.0.0 and may be removed in Spark 2.2.0)。

要在Java中编写Spark应用程序，您需要在Spark上添加依赖关系。 Spark可通过Maven 仓库获得：
```
groupId = org.apache.spark
artifactId = spark-core_2.11
version = 2.1.0
```

另外，如果希望访问HDFS集群，则需要根据你的HDFS版本添加hadoop-client的依赖：
```
groupId = org.apache.hadoop
artifactId = hadoop-client
version = <your-hdfs-version>
```
最后，您需要将一些Spark类导入到程序中。 添加以下行：
```
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
```

### 2. Scala版

默认情况下，Spark 2.1.1在Scala 2.11上构建并分布式运行(Spark 2.1.1 is built and distributed to work with Scala 2.11 by default)。 （Spark可以与其他版本的Scala一起构建。）要在Scala中编写应用程序，您将需要使用兼容的Scala版本（例如2.11.X）。

要在Java中编写Spark应用程序，您需要在Spark上添加依赖关系。 Spark可通过Maven 仓库获得：
```
groupId = org.apache.spark
artifactId = spark-core_2.11
version = 2.1.1
```

另外，如果希望访问HDFS集群，则需要根据你的HDFS版本添加hadoop-client的依赖：

```
groupId = org.apache.hadoop
artifactId = hadoop-client
version = <your-hdfs-version>
```
最后，您需要将一些Spark类导入到程序中。 添加以下行：
```
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
```

==备注==

在Spark 1.3.0之前，您需要显式导入`org.apache.spark.SparkContext._`才能启用基本的隐式转换。

原文：http://spark.apache.org/docs/latest/programming-guide.html#linking-with-spark