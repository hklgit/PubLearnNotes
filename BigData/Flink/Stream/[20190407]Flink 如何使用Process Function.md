
### 1. ProcessFunction

ProcessFunction 函数是低层次的流处理算子，可访问所有（非循环）流应用程序的基本构建块(basic building blocks)：
- Events (数据流元素)
- State (容错和一致性)
- Timers (事件时间和处理时间)

ProcessFunction 可以被认为是一个 FlatMapFunction，可以访问 KeyedState 和 Timers。为输入流中接收的每个事件调用来此函数来处理事件。

对于容错状态，ProcessFunction 可以通过 RuntimeContext 访问 KeyedState，类似于其他有状态函数访问 KeyedState。

Timers 允许应用程序对处理时间和事件时间的变化作出反应。每次调用函数 `processElement（...）` 都会获得一个 Context 对象，通过该对象可以访问元素的事件时间戳和 TimerService。TimerService 可以为将来的事件时间/处理时间实例注册回调。当到达 Timers 的某个特定时刻时，会调用 `onTimer（...）` 方法。在调用期间，所有状态再次限定为 Timers 创建的键，允许 Timers 操纵 KeyedState。

> 如果要访问 KeyedState 和 Timers，则必须在 KeyedStream 上使用 ProcessFunction。

```java
stream.keyBy(...).process(new MyProcessFunction())
```

### 2. 低层次Join

要在两个输入上实现低层次操作，应用程序可以使用 CoProcessFunction。此函数绑定到两个不同的输入，并为来自两个不同输入的记录分别调用 `processElement1（...）` 和 `processElement2（...）`。

实现低层次 Join 通常遵循以下模式：
- 为一个输入（或两者）创建状态对象。
- 从输入接收元素时更新状态。
- 从其他输入接收元素后，探测状态并生成 Join 结果。

例如，你可能会将客户数据与金融交易数据进行 Join，同时保存客户数据的状态。如果你比较关心无序事件 Join 的完整性和确定性，那么当客户数据流的 Watermark 已经超过交易时间时，你可以使用 Timers 来评估和发出交易的 Join。

### 3. Example

在以下示例中，KeyedProcessFunction 为每个 key 维护一个计数，每当 key 在一分钟（事件时间）内没有更新时就会发送 key/count 键值对：
- count，key 和 last-modification-timestamp 存储在 ValueState 中，ValueState 由 key 隐含定义。
- 对于每条记录，KeyedProcessFunction 增加计数器并修改最后的时间戳。
- 该函数还会在未来一分钟内调用回调（事件时间）。
- 在每次回调时，都会检查存储计数的最后修改时间与回调的事件时间时间戳，如果匹配则发送 key/count（即在一分钟内没有更新）

> 这个简单的例子可以用会话窗口实现。在这里使用 KeyedProcessFunction 只是用来说明它提供的基本模式。

Java版本：
```java
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.ProcessFunction.Context;
import org.apache.flink.streaming.api.functions.ProcessFunction.OnTimerContext;
import org.apache.flink.util.Collector;


// the source data stream
DataStream<Tuple2<String, String>> stream = ...;

// apply the process function onto a keyed stream
DataStream<Tuple2<String, Long>> result = stream
    .keyBy(0)
    .process(new CountWithTimeoutFunction());

/**
 * The data type stored in the state
 */
public class CountWithTimestamp {

    public String key;
    public long count;
    public long lastModified;
}

/**
 * The implementation of the ProcessFunction that maintains the count and timeouts
 */
public class CountWithTimeoutFunction
        extends KeyedProcessFunction<Tuple, Tuple2<String, String>, Tuple2<String, Long>> {

    /** The state that is maintained by this process function */
    private ValueState<CountWithTimestamp> state;

    @Override
    public void open(Configuration parameters) throws Exception {
        state = getRuntimeContext().getState(new ValueStateDescriptor<>("myState", CountWithTimestamp.class));
    }

    @Override
    public void processElement(
            Tuple2<String, String> value,
            Context ctx,
            Collector<Tuple2<String, Long>> out) throws Exception {

        // retrieve the current count
        CountWithTimestamp current = state.value();
        if (current == null) {
            current = new CountWithTimestamp();
            current.key = value.f0;
        }

        // update the state's count
        current.count++;

        // set the state's timestamp to the record's assigned event time timestamp
        current.lastModified = ctx.timestamp();

        // write the state back
        state.update(current);

        // schedule the next timer 60 seconds from the current event time
        ctx.timerService().registerEventTimeTimer(current.lastModified + 60000);
    }

    @Override
    public void onTimer(
            long timestamp,
            OnTimerContext ctx,
            Collector<Tuple2<String, Long>> out) throws Exception {

        // get the state for the key that scheduled the timer
        CountWithTimestamp result = state.value();

        // check if this is an outdated timer or the latest timer
        if (timestamp == result.lastModified + 60000) {
            // emit the state on timeout
            out.collect(new Tuple2<String, Long>(result.key, result.count));
        }
    }
}
```
Scala版本:
```Scala
import org.apache.flink.api.common.state.ValueState
import org.apache.flink.api.common.state.ValueStateDescriptor
import org.apache.flink.api.java.tuple.Tuple;
import org.apache.flink.streaming.api.functions.ProcessFunction
import org.apache.flink.streaming.api.functions.ProcessFunction.Context
import org.apache.flink.streaming.api.functions.ProcessFunction.OnTimerContext
import org.apache.flink.util.Collector

// the source data stream
val stream: DataStream[Tuple2[String, String]] = ...

// apply the process function onto a keyed stream
val result: DataStream[Tuple2[String, Long]] = stream
  .keyBy(0)
  .process(new CountWithTimeoutFunction())

/**
  * The data type stored in the state
  */
case class CountWithTimestamp(key: String, count: Long, lastModified: Long)

/**
  * The implementation of the ProcessFunction that maintains the count and timeouts
  */
class CountWithTimeoutFunction extends KeyedProcessFunction[Tuple, (String, String), (String, Long)] {

  /** The state that is maintained by this process function */
  lazy val state: ValueState[CountWithTimestamp] = getRuntimeContext
    .getState(new ValueStateDescriptor[CountWithTimestamp]("myState", classOf[CountWithTimestamp]))


  override def processElement(
      value: (String, String),
      ctx: KeyedProcessFunction[Tuple, (String, String), (String, Long)]#Context,
      out: Collector[(String, Long)]): Unit = {

    // initialize or retrieve/update the state
    val current: CountWithTimestamp = state.value match {
      case null =>
        CountWithTimestamp(value._1, 1, ctx.timestamp)
      case CountWithTimestamp(key, count, lastModified) =>
        CountWithTimestamp(key, count + 1, ctx.timestamp)
    }

    // write the state back
    state.update(current)

    // schedule the next timer 60 seconds from the current event time
    ctx.timerService.registerEventTimeTimer(current.lastModified + 60000)
  }

  override def onTimer(
      timestamp: Long,
      ctx: KeyedProcessFunction[Tuple, (String, String), (String, Long)]#OnTimerContext,
      out: Collector[(String, Long)]): Unit = {

    state.value match {
      case CountWithTimestamp(key, count, lastModified) if (timestamp == lastModified + 60000) =>
        out.collect((key, count))
      case _ =>
    }
  }
}
```

> 在 Flink 1.4.0 版本之前，当调用处理时间 Timers 时，`ProcessFunction.onTimer（）` 方法会将当前处理时间设置为事件时间时间戳。此行为非常微小，用户可能会注意不到。但是这是有问题的，因为处理时间时间戳是不确定的，不与 Watermark 对齐。此外，用户实现的逻辑依赖于这个错误的时间戳，很可能是出乎意料的错误。所以我们决定解决它。 升级到1.4.0后，使用不正确的事件时间戳的作业会失败，用户应将作业调整为正确的逻辑。

### 4. KeyedProcessFunction

KeyedProcessFunction 作为 ProcessFunction 的扩展，可以在 `onTimer（...）` 方法中访问 Timers 的 key。

Java版本:
```java
@Override
public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception {
    K key = ctx.getCurrentKey();
    // ...
}
```
Scala版本:
```Scala
override def onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[OUT]): Unit = {
  var key = ctx.getCurrentKey
  // ...
}
```
### 5. Timers

两种类型的 Timers（处理时间和事件时间）由 TimerService 在内部维护并排队执行。

TimerService 删除每个 key 和时间戳重复的 Timers，即每个 key 和时间戳最多有一个 Timers。如果为同一时间戳注册了多个 Timers，则只会调用一次 `onTimer（）` 方法。

> Flink同步调用 `onTimer（）`和 `processElement（）`。 因此，用户不必担心并发修改状态。

#### 5.1 容错

Timers 具有容错能力，并且与应用程序的状态一起 checkpoint。如果故障恢复或从保存点启动应用程序，则会恢复 Timers。

> 在恢复之前应该点火的检查点处理时间计时器将立即触发。 当应用程序从故障中恢复或从保存点启动时，可能会发生这种情况。

>  除了 RocksDB后端/增量快照/基于堆的定时器的组合（将使用FLINK-10026解析）之外，定时器始终是异步检查点。 请注意，大量的计时器可以增加检查点时间，因为计时器是检查点状态的一部分。

#### 5.2 Timers合并

由于 Flink 每个键每个时间戳只保留一个 Timers，因此可以通过降低 Timers 的分辨率来合并它们并减少 Timers 的数量。

对于分辨率1秒的 Timers（事件时间或处理时间），我们可以将目标时间向下舍入为整秒数。Timers 最多提前1秒触发，但不会迟于我们的要求，精确到毫秒。结果，每个键每秒最多有一个 Timers。

Java版本:
```java
long coalescedTime = ((ctx.timestamp() + timeout) / 1000) * 1000;
ctx.timerService().registerProcessingTimeTimer(coalescedTime);
```
Scala版本:
```Scala
val coalescedTime = ((ctx.timestamp + timeout) / 1000) * 1000
ctx.timerService.registerProcessingTimeTimer(coalescedTime)
```
由于事件时间  Timers 仅当 Watermark 到达时触发，因此我们可以将当前的 Timers 与下一个 Watermark 的那些 Timers 调度以及合并。

Java版本:
```java
long coalescedTime = ctx.timerService().currentWatermark() + 1;
ctx.timerService().registerEventTimeTimer(coalescedTime);
```
Scala版本:
```Scala
val coalescedTime = ctx.timerService.currentWatermark + 1
ctx.timerService.registerEventTimeTimer(coalescedTime)
```
可以使用下方式停止和删除 Timers：
Java版本:
```java
long timestampOfTimerToStop = ...
ctx.timerService().deleteProcessingTimeTimer(timestampOfTimerToStop);
```
Scala版本:
```Scala
val timestampOfTimerToStop = ...
ctx.timerService.deleteProcessingTimeTimer(timestampOfTimerToStop)
```
停止处理时间 Timers：
             Java版本：
```java
long timestampOfTimerToStop = ...
ctx.timerService().deleteEventTimeTimer(timestampOfTimerToStop);
```
Scala版本：
```scala
val timestampOfTimerToStop = ...
ctx.timerService.deleteEventTimeTimer(timestampOfTimerToStop)
```
> 如果没有注册给定时间戳的 Timers，那么停止 Timers 没有任何效果。                               

> Flink版本:1.8

原文:[Process Function (Low-level Operations)](https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/operators/process_function.html)
