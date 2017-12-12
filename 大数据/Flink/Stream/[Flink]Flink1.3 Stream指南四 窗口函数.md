在定义[窗口分配器](http://blog.csdn.net/sunnyyoona/article/details/78327447)之后，我们需要在每个窗口上指定我们要执行的计算。这是窗口函数的责任，一旦系统确定窗口准备好处理数据，窗口函数就处理每个窗口中的元素。

窗口函数可以是`ReduceFunction`，`FoldFunction`或`WindowFunction`其中之一。前两个函数执行更有效率，因为Flink可以在每个窗口中元素到达时增量地聚合。`WindowFunction`将获得一个包含在窗口中所有元素的迭代器以及元素所在窗口的附加元信息(gets an Iterable for all the elements contained in a window and additional meta information about the window to which the elements belong)。

使用`WindowFunction`的窗口转换不能像其他函数那么有效率，是因为Flink在调用函数之前必须在内部缓存窗口中的所有元素。这可以通过将`WindowFunction`与`ReduceFunction`或`FoldFunction`组合使用来缓解，以获得窗口元素的增量聚合以及`WindowFunction`接收的附加窗口元数据。

### 1. ReduceFunction

`ReduceFunction`指定如何组合输入数据的两个元素以产生相同类型的输出元素。Flink使用`ReduceFunction`增量聚合窗口的元素。

`ReduceFunction`可以如下定义和使用:

Java版本:
```java
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
    }
});
```

Scala版本:
```
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce { (v1, v2) => (v1._1, v1._2 + v2._2) }
```
上述示例获得窗口中的所有元素元组的第二个字段之和。

### 2. FoldFunction

`FoldFunction`指定窗口的输入元素如何与输出类型的元素合并。`FoldFunction`会被每一个加入到窗口中的元素和当前的输出值增量地调用(The FoldFunction is incrementally called for each element that is added to the window and the current output value)，第一个元素与一个预定义的输出类型的初始值合并。

`FoldFunction`可以如下定义和使用:

Java版本:
```java
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
});
```
上述示例将所有输入元素的Long值追加到初始化为空的字符串中。

备注:
```
fold()不能应用于会话窗口或者其他可合并的窗口中。
```

### 3. WindowFunction的一般用法

`WindowFunction`将获得一个包含窗口中所有元素的迭代器，并且拥有所有窗口函数的最大灵活性。但是这些是以性能和资源消耗为代价的，因为元素不能增量聚合，相反还需要在内部缓存，直到窗口做好准备处理。

`WindowFunction`的接口如下所示：

Java版本:
```java
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {

  /**
   * Evaluates the window and outputs none or several elements.
   *
   * @param key The key for which this window is evaluated.
   * @param window The window that is being evaluated.
   * @param input The elements in the window being evaluated.
   * @param out A collector for emitting elements.
   *
   * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
   */
  void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
```

Scala版本:
```
trait WindowFunction[IN, OUT, KEY, W <: Window] extends Function with Serializable {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key    The key for which this window is evaluated.
    * @param window The window that is being evaluated.
    * @param input  The elements in the window being evaluated.
    * @param out    A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def apply(key: KEY, window: W, input: Iterable[IN], out: Collector[OUT])
}
```

`WindowFunction`可以如下定义和使用:

Java版本:
```java
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction());

/* ... */

public class MyWindowFunction implements WindowFunction<Tuple<String, Long>, String, String, TimeWindow> {

  void apply(String key, TimeWindow window, Iterable<Tuple<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + window + "count: " + count);
  }
}
```

Scala版本:
```
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction())

/* ... */

class MyWindowFunction extends WindowFunction[(String, Long), String, String, TimeWindow] {

  def apply(key: String, window: TimeWindow, input: Iterable[(String, Long)], out: Collector[String]): () = {
    var count = 0L
    for (in <- input) {
      count = count + 1
    }
    out.collect(s"Window $window count: $count")
  }
}
```
该示例展示`WindowFunction`计算窗口中元素个数。 此外，窗口函数将窗口相关信息添加到输出中。

备注:
```
使用WindowFunction做简单的聚合操作如计数操作，性能是相当差的。下一章节我们将展示如何将ReduceFunction跟WindowFunction结合起来，来获取增量聚合以及WindowFunction的添加信息。
```

### 4. ProcessWindowFunction

在可以使用`WindowFunction`的地方，你也可以使用`ProcessWindowFunction`。这个与`WindowFunction`非常相似，只是该接口允许查询更多关于context的信息，context是窗口执行的地方。

`ProcessWindowFunction`的接口如下:

Java版本:
```Java
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {

    /**
     * Evaluates the window and outputs none or several elements.
     *
     * @param key The key for which this window is evaluated.
     * @param context The context in which the window is being evaluated.
     * @param elements The elements in the window being evaluated.
     * @param out A collector for emitting elements.
     *
     * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;

    /**
     * The context holding window metadata
     */
    public abstract class Context {
        /**
         * @return The window that is being evaluated.
         */
        public abstract W window();
    }
}
```

Scala版本:
```
abstract class ProcessWindowFunction[IN, OUT, KEY, W <: Window] extends Function {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key      The key for which this window is evaluated.
    * @param context  The context in which the window is being evaluated.
    * @param elements The elements in the window being evaluated.
    * @param out      A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  @throws[Exception]
  def process(
      key: KEY,
      context: Context,
      elements: Iterable[IN],
      out: Collector[OUT])

  /**
    * The context holding window metadata
    */
  abstract class Context {
    /**
      * @return The window that is being evaluated.
      */
    def window: W
  }
}
```

`ProcessWindowFunction`可以如下定义和使用:

Java版本:
```java
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .process(new MyProcessWindowFunction());
```

Scala版本:
```
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .process(new MyProcessWindowFunction())
```

### 5. 带有增量聚合的WindowFunction

`WindowFunction`可以与`ReduceFunction`或`FoldFunction`组合使用，以便在元素到达窗口时增量聚合元素。当窗口关闭时，`WindowFunction`提供聚合结果。这允许在访问`WindowFunction`的额外窗口元信息的同时增量计算窗口(This allows to incrementally compute windows while having access to the additional window meta information of the WindowFunction.)。

备注:
```
你也可以使用ProcessWindowFunction而不是WindowFunction进行增量窗口聚合。
```

#### 5.1 使用FoldFunction增量窗口聚合

以下示例展现了增量`FoldFunction`如何与`WindowFunction`组合以提取窗口中的事件数，并返回窗口的key和结束时间。

Java版本:
```java
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .fold(new Tuple3<String, Long, Integer>("",0L, 0), new MyFoldFunction(), new MyWindowFunction())

// Function definitions

private static class MyFoldFunction
    implements FoldFunction<SensorReading, Tuple3<String, Long, Integer> > {

  public Tuple3<String, Long, Integer> fold(Tuple3<String, Long, Integer> acc, SensorReading s) {
      Integer cur = acc.getField(2);
      acc.setField(2, cur + 1);
      return acc;
  }
}

private static class MyWindowFunction
    implements WindowFunction<Tuple3<String, Long, Integer>, Tuple3<String, Long, Integer>, String, TimeWindow> {

  public void apply(String key,
                    TimeWindow window,
                    Iterable<Tuple3<String, Long, Integer>> counts,
                    Collector<Tuple3<String, Long, Integer>> out) {
    Integer count = counts.iterator().next().getField(2);
    out.collect(new Tuple3<String, Long, Integer>(key, window.getEnd(),count));
  }
}
```

Scala版本:
```
val input: DataStream[SensorReading] = ...

input
 .keyBy(<key selector>)
 .timeWindow(<window assigner>)
 .fold (
    ("", 0L, 0),
    (acc: (String, Long, Int), r: SensorReading) => { ("", 0L, acc._3 + 1) },
    ( key: String,
      window: TimeWindow,
      counts: Iterable[(String, Long, Int)],
      out: Collector[(String, Long, Int)] ) =>
      {
        val count = counts.iterator.next()
        out.collect((key, window.getEnd, count._3))
      }
  )
```

#### 5.2 使用ReduceFunction增量窗口聚合

以下示例展现了如何将增量`ReduceFunction`与`WindowFunction`组合以返回窗口中的最小事件以及窗口的开始时间。

Java版本:
```java
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .reduce(new MyReduceFunction(), new MyWindowFunction());

// Function definitions

private static class MyReduceFunction implements ReduceFunction<SensorReading> {

  public SensorReading reduce(SensorReading r1, SensorReading r2) {
      return r1.value() > r2.value() ? r2 : r1;
  }
}

private static class MyWindowFunction
    implements WindowFunction<SensorReading, Tuple2<Long, SensorReading>, String, TimeWindow> {

  public void apply(String key,
                    TimeWindow window,
                    Iterable<SensorReading> minReadings,
                    Collector<Tuple2<Long, SensorReading>> out) {
      SensorReading min = minReadings.iterator().next();
      out.collect(new Tuple2<Long, SensorReading>(window.getStart(), min));
  }
}
```

Scala版本:
```
val input: DataStream[SensorReading] = ...

input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .reduce(
    (r1: SensorReading, r2: SensorReading) => { if (r1.value > r2.value) r2 else r1 },
    ( key: String,
      window: TimeWindow,
      minReadings: Iterable[SensorReading],
      out: Collector[(Long, SensorReading)] ) =>
      {
        val min = minReadings.iterator.next()
        out.collect((window.getStart, min))
      }
  )
```


备注:
```
Flink版本:1.3
```

原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/windows.html#window-functions
