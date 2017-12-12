### 1. 抽象层次

Flink提供不同级别的抽象层次来开发流处理和批处理应用程序。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/levels_of_abstraction.svg)

(1) 最低级别的抽象只是提供有状态的数据流。通过`Process Function`集成到DataStream API中。它允许用户不受限制的处理来自一个或多个数据流的事件，并可以使用一致的容错状态(consistent fault tolerant state)。另外，用户可以注册事件时间和处理时间的回调函数，允许程序实现复杂的计算。

(2) 在实际中，大多数应用程序不需要上述描述的低级抽象，而是使用如`DataStream API`(有界/无界流)和`DataSet API`(有界数据集)的核心API进行编程。这些核心API提供了用于数据处理的通用构建模块，如用户指定的各种转换，连接，聚集，窗口，状态等。在这些API中处理的数据类型被表示为对应编程语言中的类。

低级别的`Process Function`与`DataStream API`集成在一起，使得可以对特定操作使用较低级别的抽象接口。`DataSet API`为有限数据集提供了额外的原语(primitives)，如循环/迭代。

(3) `Table API`是以表为核心的声明式DSL，可以动态地改变表(当表表示流数据时)。`Table API`遵循(扩展的)关系模型：每个表都有一个schema(类似于关系数据库中的表)，对应的API提供了类似的操作(offers comparable operations)，如`select`，`project`，`join`，`group-by`，`aggregate`等。`Table API`程序声明性地定义了如何在逻辑上实现操作，而不是明确指定操作实现的具体代码。尽管`Table API`可以通过各种类型的用户自定义函数进行扩展，它比核心API表达性要差一些，但使用上更简洁(编写代码更少)。另外，`Table API`程序也会通过一个优化器，在执行之前应用优化规则。

可以在表和`DataStream`/`DataSet`之间进行无缝转换，允许程序混合使用`Table API`和`DataStream`和`DataSet API`。

(4) Flink提供的最高级抽象是SQL。这种抽象在语法和表现力方面与`Table API`类似，但是是通过SQL查询表达式实现程序。SQL抽象与`Table API`紧密交互，SQL查询可以在`Table API`中定义的表上执行。

### 2. 程序与数据流

Flink程序的基本构建块是流和转换操作。

备注:
```
Flink的DataSet API中使用的数据集也是内部的流 - 稍后会介绍这一点。
```

从概念上讲，流是数据记录(可能是永无止境的)流，而转换是将一个或多个流作为输入，并产生一个或多个输出流。

执行时，Flink程序被映射到由流和转换算子组成的流式数据流(streaming dataflows)。每个数据流从一个或多个source开始，并在一个或多个sink中结束。数据流类似于有向无环图(DAG)。尽管通过迭代构造允许特殊形式的环，但是为了简单起见，大部分我们都会这样描述。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/program_dataflow.svg)

程序中的转换与数据流中的算子通常是一一对应的。然而，有时候，一个转换可能由多个转换算子组成。

### 3. 并行数据流图

Flink中的程序本质上是分布式并发执行的。在执行过程中，一个流有一个或多个流分区，每个算子有一个或多个算子子任务。算子子任务之间相互独立，并且在不同的线程中执行，甚至有可能在不同的机器或容器上执行。

算子子任务的数量是该特定算子的并发数。流的并发数总是产生它的算子的并发数。同一程序的不同算子可能具有不同的并发级别。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/parallel_dataflow.svg)

在两个算子之间的流可以以一对一模式或重新分发模式传输数据:

(1) 一对一流(例如上图中的Source和map()算子之间的流)保留了元素的分区和排序。这意味着将会在map()算子的子任务[1]中看到在Source算子的子任务[1]中产生的相同元素，并且具有相同的顺序。

(2) 重分发流(例如上图的的`map()`和`keyBy()/window()/apply()`之间，以及在`keyBy()/window()/apply()`和`Sink`之间的数据流)改变了流的分区。每个算子子任务根据所选的转换操作将数据发送到不同的目标子任务。比如`keyBy()`(根据key的哈希值重新分区)，`broadcast()`，或者`rebalance()`(随机重新分区)。在重新分配交换中，只会在每对发送与接受子任务(比如，`map()`的子任务[1]与`keyBy()/window()/apply()`的子任务[2])中保留元素间的顺序。在上图的例子中，尽管在子任务之间每个 key 的顺序都是确定的，但是由于程序的并发引入了不确定性，最终到达`Sink`的元素顺序就不能保证与一开始的元素顺序完全一致。

关于配置并发的更多信息可以参阅[并发执行文档](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/parallel.html)。

### 4. 窗口

### 5. 时间

### 6. 有状态操作

### 7. 容错性检查点

### 8. 批处理操作














































译者：sjf0115
链接：
原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/concepts/programming-model.html
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
