---
layout: post
author: sjf0115
title: Flink1.4 使用状态
date: 2018-01-16 20:32:17
tags:
  - Flink

categories: Flink
---

### 1. Keyed State 与 Operator State

`Flink`有两种基本的状态：`Keyed State`和`Operator State`。

#### 1.1 Keyed State

`Keyed State`总是与`key`相关，只能在`KeyedStream`的函数和运算符中使用。

备注:
```
KeyedStream继承DataStream，表示根据指定的key进行分组的数据流。使用DataStream提供的KeySelector根据key对其上的算子State进行分区。
DataStream支持的典型操作也可以在KeyedStream上进行，除了诸如shuffle，forward和keyBy之类的分区方法之外。

一个KeyedStream可以通过调用DataStream.keyBy()来获得。而在KeyedStream上进行任何transformation都将转变回DataStream。
```

你可以将 `Keyed State` 视为已经分区或分片的`Operator State`，每个 `key` 对应一个状态分区。每个`Keyed State`在逻辑上只对应一个 `<并行算子实例，key>`，并且由于每个 `key` "只属于" 一个`Keyed Operator`的一个并行实例，我们可以简单地认为成 `<operator，key>`。

`Keyed State` 被进一步组织成所谓的 `Key Group`。`Key Group` 是 `Flink` 可以重新分配 `Keyed State` 的最小单位；`Key Group`的数量与最大并行度一样多。在执行期间，`Keyed Operator`的每个并行实例都与一个或多个`Key Group`的`key`一起工作。

#### 1.2 Operator State

使用`Operator State` (或非`Keyed State`)，每个算子状态都绑定到一个并行算子实例。`Kafka Connector` 是在`Flink`中使用算子状态的一个很好的例子。`Kafka`消费者的每个并行实例都要维护一个`topic`分区和偏移量的map作为其`Operator State`。

在并行度发生变化时，`Operator State`接口支持在并行算子实例之间进行重新分配状态。可以有不同的方案来处理这个重新分配。

### 2. Raw State 与 Managed State

`Keyed State`和`Operator State`以两种形式存在：托管状态`Managed State`和原始状态`Raw State`。

`Managed State`由`Flink`运行时控制的数据结构表示，如内部哈希表或`RocksDB`。例如`ValueState`，`ListState`等。`Flink`的运行时对状态进行编码并将它们写入检查点。

`Raw State`是指算子保持在它们自己数据结构中的状态。当检查点时，他们只写入一个字节序列到检查点。`Flink`对状态的数据结构一无所知，只能看到原始字节。

所有数据流函数都可以使用`Managed State`，但`Raw State`接口只能在实现算子时使用。建议使用`Managed State`（而不是`Raw State`），因为在`Managed State`下，`Flin`k可以在并行度发生变化时自动重新分配状态，并且还可以更好地进行内存管理。

备注:
```
如果你的Managed State需要自定义序列化逻辑，请参阅相应的指南以确保将来的兼容性。Flink的默认序列化器不需要特殊处理。
```

### 3. Managed Keyed State

`Managed Keyed State`接口提供了对不同类型状态的访问，这些状态的作用域为当前输入元素的`key`。这意味着这种类型的状态只能用于`KeyedStream`，可以通过`stream.keyBy（...）`创建。

现在，我们先看看可用状态的不同类型，然后我们将看到如何在一个程序中使用它们。可用状态是：
- `ValueState <T>`：保存了一个可以更新和检索的值（如上所述，作用域为输入元素的`key`，所以操作看到的每个`key`可能有一个值）。该值可以使用`update（T）`来设置，使用`T value（）`来检索。
- `ListState <T>`：保存了一个元素列表。可以追加元素并检索当前存储的所有元素的`Iterable`。使用`add（T）`添加元素，可以使用`Iterable <T> get（）`来检索`Iterable`。
- `ReducingState <T>`：保存一个单一的值，表示添加到状态所有值的聚合。接口与`ListState`相同，但使用`add（T）`添加的元素，使用指定的`ReduceFunction`转换为聚合。
- `AggregatingState <IN，OUT>`：保存一个单一的值，表示添加到状态所有值的聚合。与`ReducingState`不同，聚合后的类型可能与添加到状态的元素类型不同。接口与`ListState`相同，但使用`add（IN）`添加到状态的元素使用指定的`AggregateFunction`进行聚合。
- `FoldingState <T，ACC>`：保存一个单一的值，表示添加到状态所有值的聚合。与`ReducingState`不同，聚合后类型可能与添加到状态的元素类型不同。接口与`ListState`相同，但使用`add（T）`添加到状态的元素使用指定的`FoldFunction`折叠成聚合。
- MapState <UK，UV>：保存了一个映射列表。可以将键值对放入状态，并检索当前存储的所有映射的`Iterable`。使用`put（UK，UV）`或`putAll（Map <UK，UV>）`添加映射。与用户`key`相关的值可以使用`get（UK）`来检索。映射，键和值的迭代视图可分别使用`entries（）`，`keys（）`和`values（）`来检索。

所有类型的状态都有一个`clear（）`方法，它清除了当前活跃`key`的状态，即输入元素的`key`。

备注:
```
FoldingState和FoldingStateDescriptor已经在Flink 1.4中被弃用，将来会被彻底删除。请改用AggregatingState和AggregatingStateDescriptor。
```

请记住，这些状态对象仅能用于状态接口，这一点很重要。状态没有必要存储在内存中，也可以驻留在磁盘或其他地方。第二件要记住的是，你从状态获得的值取决于输入元素的`key`。因此，如果所涉及的`key`不同，那你在用户函数调用中获得的值可能与另一个调用中的值不同。

为了得到一个状态句柄，你必须创建一个`StateDescriptor`。它包含了状态的名字（我们将在后面看到，你可以创建多个状态，必须有唯一的名称，以便引用它们），状态值的类型，以及用户自定义函数，如`ReduceFunction`。根据要检索的状态类型，你可以创建一个`ValueStateDescriptor`，`ListStateDescriptor`，`ReducingStateDescriptor`，`FoldingStateDescriptor`或`MapStateDescriptor`。

使用`RuntimeContext`来访问状态，所以只能在`Rich`函数中使用。请参阅[这里](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/api_concepts.html#rich-functions)了解有关信息，我们会很快看到一个例子。 在`RichFunction`中可用的`RuntimeContext`具有下面访问状态的方法：
- ValueState<T> getState(ValueStateDescriptor<T>)
- ReducingState<T> getReducingState(ReducingStateDescriptor<T>)
- ListState<T> getListState(ListStateDescriptor<T>)
- AggregatingState<IN, OUT> getAggregatingState(AggregatingState<IN, OUT>)
- FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)
- MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)

下面是`FlatMapFunction`的一个例子：

Java版本:
```java
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count 个数
        currentSum.f0 += 1;

        // add the second field of the input value 总和
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
```

Scala版本:
```
class CountWindowAverage extends RichFlatMapFunction[(Long, Long), (Long, Long)] {

  private var sum: ValueState[(Long, Long)] = _

  override def flatMap(input: (Long, Long), out: Collector[(Long, Long)]): Unit = {

    // access the state value
    val tmpCurrentSum = sum.value

    // If it hasn't been used before, it will be null
    val currentSum = if (tmpCurrentSum != null) {
      tmpCurrentSum
    } else {
      (0L, 0L)
    }

    // update the count
    val newSum = (currentSum._1 + 1, currentSum._2 + input._2)

    // update the state
    sum.update(newSum)

    // if the count reaches 2, emit the average and clear the state
    if (newSum._1 >= 2) {
      out.collect((input._1, newSum._2 / newSum._1))
      sum.clear()
    }
  }

  override def open(parameters: Configuration): Unit = {
    sum = getRuntimeContext.getState(
      new ValueStateDescriptor[(Long, Long)]("average", createTypeInformation[(Long, Long)])
    )
  }
}


object ExampleCountWindowAverage extends App {
  val env = StreamExecutionEnvironment.getExecutionEnvironment

  env.fromCollection(List(
    (1L, 3L),
    (1L, 5L),
    (1L, 7L),
    (1L, 4L),
    (1L, 2L)
  )).keyBy(_._1)
    .flatMap(new CountWindowAverage())
    .print()
  // the printed output will be (1,4) and (1,5)

  env.execute("ExampleManagedState")
}
```

这个例子实现了一个穷人的计数窗口。我们通过第一个字段键入元组（在这个例子中都有相同的`key`为`1`）。该函数将计数和总和存储在`ValueState`中。一旦计数达到2，就输出平均值并清除状态，以便我们从0开始。注意，如果我们元组第一个字段具有不同值，那将为每个不同的输入`key`保持不同的状态值。

#### 3.1 Scala DataStream API中的状态

除了上面介绍的接口之外，Scala API还具有在`KeyedStream`上使用单个`ValueState`的有状态`map（）`或`flatMap（）`函数的快捷方式。用户函数可以在`Option`获取`ValueState`的当前值，并且必须返回将用于更新状态的更新值。
```
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
```

### 4. Managed Operator State

要使用`Managed Operator State`，有状态函数可以实现更通用的`CheckpointedFunction`接口或`ListCheckpointed <T extends Serializable>`接口。

#### 4.1 CheckpointedFunction

`CheckpointedFunction`接口提供了使用不同的重分配方案对非`Ked State`的访问。它需要实现一下两种方法：
```
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
```

每当执行检查点时，将调用`snapshotState（）`。每当用户自定义函数被初始化时，对应的`initializeState（）`都被调用，或当函数被初始化时，或者当函数实际上从早期的检查点恢复时被调用(The counterpart, initializeState(), is called every time the user-defined function is initialized, be that when the function is first initialized or be that when the function is actually recovering from an earlier checkpoint. )。鉴于此，`initializeState（）`不仅是初始化不同类型的状态的地方，而且还包括状态恢复逻辑的位置。

目前支持列表式的`Managed Operator State`。状态应该是一个可序列化的对象列表，相互间彼此独立，因此可以在扩展时重新分配。换句话说，这些对象可以在非`Keyed State`中重新分配比较细的粒度。根据状态访问方法，定义了以下重新分配方案：
- 均分再分配: Each operator returns a List of state elements. The whole state is logically a concatenation of all lists. On restore/redistribution, the list is evenly divided into as many sublists as there are parallel operators. Each operator gets a sublist, which can be empty, or contain one or more elements. As an example, if with parallelism 1 the checkpointed state of an operator contains elements element1 and element2, when increasing the parallelism to 2, element1 may end up in operator instance 0, while element2 will go to operator instance 1.
- 合并再分配: Each operator returns a List of state elements. The whole state is logically a concatenation of all lists. On restore/redistribution, each operator gets the complete list of state elements.

下面是一个有状态的`SinkFunction`的例子，它使用`CheckpointedFunction`在将元素输出到外部之前进行缓冲元素。它演示了基本的均分再分配列表状态：

Java版本:
```java
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction,
                   CheckpointedRestoring<ArrayList<Tuple2<String, Integer>>> {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }

    @Override
    public void restoreState(ArrayList<Tuple2<String, Integer>> state) throws Exception {
        // this is from the CheckpointedRestoring interface.
        this.bufferedElements.addAll(state);
    }
}
```

Scala版本:
```
class BufferingSink(threshold: Int = 0)
  extends SinkFunction[(String, Int)]
    with CheckpointedFunction
    with CheckpointedRestoring[List[(String, Int)]] {

  @transient
  private var checkpointedState: ListState[(String, Int)] = _

  private val bufferedElements = ListBuffer[(String, Int)]()

  override def invoke(value: (String, Int)): Unit = {
    bufferedElements += value
    if (bufferedElements.size == threshold) {
      for (element <- bufferedElements) {
        // send it to the sink
      }
      bufferedElements.clear()
    }
  }

  override def snapshotState(context: FunctionSnapshotContext): Unit = {
    checkpointedState.clear()
    for (element <- bufferedElements) {
      checkpointedState.add(element)
    }
  }

  override def initializeState(context: FunctionInitializationContext): Unit = {
    val descriptor = new ListStateDescriptor[(String, Int)](
      "buffered-elements",
      TypeInformation.of(new TypeHint[(String, Int)]() {})
    )

    checkpointedState = context.getOperatorStateStore.getListState(descriptor)

    if(context.isRestored) {
      for(element <- checkpointedState.get()) {
        bufferedElements += element
      }
    }
  }

  override def restoreState(state: List[(String, Int)]): Unit = {
    bufferedElements ++= state
  }
}
```

The initializeState method takes as argument a FunctionInitializationContext. This is used to initialize the non-keyed state “containers”. These are a container of type ListState where the non-keyed state objects are going to be stored upon checkpointing.

Note how the state is initialized, similar to keyed state, with a StateDescriptor that contains the state name and information about the type of the value that the state holds:

Java版本:
```Java
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
```

Scala版本:
```
val descriptor = new ListStateDescriptor[(String, Long)](
    "buffered-elements",
    TypeInformation.of(new TypeHint[(String, Long)]() {})
)

checkpointedState = context.getOperatorStateStore.getListState(descriptor)
```

The naming convention of the state access methods contain its redistribution pattern followed by its state structure. For example, to use list state with the union redistribution scheme on restore, access the state by using getUnionListState(descriptor). If the method name does not contain the redistribution pattern, e.g. getListState(descriptor), it simply implies that the basic even-split redistribution scheme will be used.

After initializing the container, we use the isRestored() method of the context to check if we are recovering after a failure. If this is true, i.e. we are recovering, the restore logic is applied.

As shown in the code of the modified BufferingSink, this ListState recovered during state initialization is kept in a class variable for future use in snapshotState(). There the ListState is cleared of all objects included by the previous checkpoint, and is then filled with the new ones we want to checkpoint.

As a side note, the keyed state can also be initialized in the initializeState() method. This can be done using the provided FunctionInitializationContext.

#### 4.2 ListCheckpointed

`ListCheckpointed`接口是`CheckpointedFunction`进行限制的一种变体，它只支持在还原时使用均分再分配方案的列表样式状态。还需要实现以下两种方法：

```
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
```

On snapshotState() the operator should return a list of objects to checkpoint and restoreState has to handle such a list upon recovery. If the state is not re-partitionable, you can always return a Collections.singletonList(MY_STATE) in the snapshotState().

#### 4.2.1 Stateful Source Functions

与其他算子相比，有状态的数据源需要得到更多的关注。为了能更新状态以及输出集合的原子性（在失败/恢复时需要一次性语义），用户需要从数据源的上下文中获取锁。

Java版本:
```java
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  current offset for exactly once semantics */
    private Long offset;

    /** flag for job cancellation */
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }

    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
```

Scala版本:
```
class CounterSource
       extends RichParallelSourceFunction[Long]
       with ListCheckpointed[Long] {

  @volatile
  private var isRunning = true

  private var offset = 0L

  override def run(ctx: SourceFunction.SourceContext[Long]): Unit = {
    val lock = ctx.getCheckpointLock

    while (isRunning) {
      // output and state update are atomic
      lock.synchronized({
        ctx.collect(offset)

        offset += 1
      })
    }
  }

  override def cancel(): Unit = isRunning = false

  override def restoreState(state: util.List[Long]): Unit =
    for (s <- state) {
      offset = s
    }

  override def snapshotState(checkpointId: Long, timestamp: Long): util.List[Long] =
    Collections.singletonList(offset)

}
```

备注:
```
Flink版本:1.4
```

原文:https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/stream/state/state.html
