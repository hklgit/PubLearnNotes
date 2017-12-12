
本节介绍如何在Flink中配置程序的并行执行。一个Flink程序由多个Task(transformations/operators，data sources和sinks)组成。一个任务被分成多个并发实例来执行，每个并发实例只处理Task输入数据的一个子集。一个Task的并发实例的个数称为并发度`parallelism`。

如果你想使用[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/savepoints.html)，应该考虑设置最大并发度。从保存点恢复时，可以更改特定算子或整个程序的并发度，并且此配置指定了并发的上限(change the parallelism of specific operators or the whole program and this setting specifies an upper bound on the parallelism)。This is required because Flink internally partitions state into key-groups and we cannot have +Inf number of key-groups because this would be detrimental to performance.

### 1. 设置并发度

任务的并发度可以在Flink中指定不同级别。

#### 1.1 算子级别

单个算子，数据源，sink可以通过调用`setParallelism()`方法来定义并发度。例如，像这样：

Java版本:
```java
DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = text
    .flatMap(new LineSplitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5);

wordCounts.print();

env.execute("Word Count Example");
```

Scala版本:
```
val env = StreamExecutionEnvironment.getExecutionEnvironment

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5)
wordCounts.print()

env.execute("Word Count Example")
```

#### 1.2 执行环境级别

如[这](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/api_concepts.html#anatomy-of-a-flink-program)所述，Flink程序是在执行环境的上下文中执行的。执行环境为它执行的所有算子，数据源和数据sink提供了默认的并发度。执行环境的并发度可以通过显式配置一个算子的并发度来覆盖。

执行环境的默认并发度可以通过调用`setParallelism()`方法来指定。要为执行所有算子，数据源和sink设置并发度为3，请按如下方式设置执行环境的默认并发度：

Java版本:
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(3);

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = [...]
wordCounts.print();

env.execute("Word Count Example");
```

Scala版本:
```
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(3)

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1)
wordCounts.print()

env.execute("Word Count Example")
```

#### 1.3 客户端级别

在向Flink提交作业时，可以在客户端设置并发度。客户端可以是Java或Scala程序。这种客户端的一个例子是Flink的命令行界面(CLI)。

对于CLI客户端，可以使用-p指定并发度参数。 例如：
```
./bin/flink run -p 10 ../examples/*WordCount-java*.jar
```
在Java/Scala程序中，并发度设置如下：

Java版本:
```java
try {
    PackagedProgram program = new PackagedProgram(file, args);
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123");
    Configuration config = new Configuration();

    Client client = new Client(jobManagerAddress, config, program.getUserCodeClassLoader());

    // set the parallelism to 10 here
    client.run(program, 10, true);

} catch (ProgramInvocationException e) {
    e.printStackTrace();
}
```

Scala版本:
```
try {
    PackagedProgram program = new PackagedProgram(file, args)
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123")
    Configuration config = new Configuration()

    Client client = new Client(jobManagerAddress, new Configuration(), program.getUserCodeClassLoader())

    // set the parallelism to 10 here
    client.run(program, 10, true)

} catch {
    case e: Exception => e.printStackTrace
}
```

#### 1.4 系统级别

可以通过在`./conf/flink-conf.yaml`中设置`parallelism.default`属性来为所有执行环境定义全系统默认并发度。详细信息请参阅[配置文档](https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/config.html)。


### 2. 设置最大并发度

最大并发度可以在可以设置并发度的地方设置(客户端级别和系统级别除外)。你可以调用`setMaxParallelism()`取代`setParallelism()`方法来设置最大并发度。

最大并发度的默认设置大致为`operatorParallelism +（operatorParallelism / 2）`，下限为127，上限为32768。

备注:
```
将最大并发度设置为非常大的数值可能会对性能造成不利影响，因为一些后端状态必须保持在内部数据结构，而这些内部数据结构随key-groups(这是可扩展状态的内部实现机制)的数量进行扩展。(some state backends have to keep internal data structures that scale with the number of key-groups (which are the internal implementation mechanism for rescalable state).)
```

原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/parallel.html
