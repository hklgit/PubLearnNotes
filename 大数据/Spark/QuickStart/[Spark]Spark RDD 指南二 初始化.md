### 1. 初始化

Spark程序必须做的第一件事是创建一个`JavaSparkContext`对象(Scala和Python中是`SparkContext`对象)，它告诉Spark如何访问集群。 要创建`SparkContext`，您首先需要构建一个包含有关应用程序信息的`SparkConf`对象。


Java版本：
```
private static String appName = "JavaWordCountDemo";
private static String master = "local";

// 初始化Spark
private static SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
private static JavaSparkContext sc = new JavaSparkContext(conf);
```

Scala版本：
```
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```

==备注==

每个JVM只有一个SparkContext可能是活跃的。 在创建新的SparkContext之前，必须先调用`stop()`方法停止之前活跃的SparkContext。

Python版本：
```
conf = SparkConf().setAppName(appName).setMaster(master)
sc = SparkContext(conf=conf)
```

appName参数是应用程序在集群UI上显示的名称。 master是Spark，Mesos或YARN集群URL，或以本地模式运行的特殊字符串“local”。 实际上，当在集群上运行时，您不需要在程序中写死`master`，而是使用spark-submit启动应用程序并以参数传递进行接收。但是，对于本地测试和单元测试，你可以通过“local”来运行Spark进程。


### 2. 使用Shell

在 Spark shell 中，已经为你创建了一个专有的 `SparkContext`，可以通过变量`sc`访问。你自己创建的 `SparkContext` 将无法工作。可以用 `--master` 参数来设置 `SparkContext` 要连接的集群，用 `--jars` 来设置需要添加到 classpath 中的 JAR 包，如果有多个 JAR 包使用逗号分割符连接它们。你还可以通过向`--packages`参数提供逗号分隔的maven坐标列表，将依赖关系（例如Spark Packages）添加到shell会话中。 可能存在依赖关系的其他存储库（例如Sonatype）可以传递给`--repositories`参数。例如：在一个拥有 4 核的环境上运行 bin/spark-shell，使用：

```
./bin/spark-shell --master local[4]
```
或者，还可以将code.jar添加到其classpath中，请使用：
```
 ./bin/spark-shell --master local[4] --jars code.jar
```
使用maven坐标来包含依赖关系：
```
 ./bin/spark-shell --master local[4] --packages "org.example:example:0.1"
```
可以执行 `spark-shell --help` 获取完整的选项列表。其背后，`spark-shell`调用的是更常用的[`spark-submit`脚本](http://blog.csdn.net/sunnyyoona/article/details/55271395)(Behind the scenes, spark-shell invokes the more general spark-submit script.)。

原文：http://spark.apache.org/docs/latest/programming-guide.html#initializing-spark