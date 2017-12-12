Flink提供了抽象类，允许程序员自己可以分配时间戳并发出watermarks。更具体地说，可以通过`AssignerWithPeriodicWatermarks`或`AssignerWithPunctuatedWatermarks`接口来实现，具体实现取决于用户情况。简而言之，第一个将周期性的发出watermarks，而第二个则基于传入记录的某些属性，例如，当在流中遇到特殊元素时。

为了进一步缓解这些任务的编程工作，Flink带有一些内置的时间戳分配器。除了开箱即用的功能外，它们的实现也可以作为自定义实现的一个例子。

### 1. 递增时间戳的Assigners

周期性生成watermarks最简单的特例是时间戳在给定源任务中以递增出现(timestamps seen by a given source task occur in ascending order)。在这种情况下，由于没有比这还早的时间戳到达，所以当前时间戳可以始终充当watermarks。

请注意，每个并行数据源任务的时间戳只需要升序。例如，如果在特定设置中，一个并行数据源实例读取一个Kafka分区，那么只需要在每个Kafka分区内时间戳是升序的。每当并行数据流被洗牌，联合，连接或合并时，Flink的watermark合并机制能够产生正确的watermark。

```
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignAscendingTimestamps( _.getCreationTime )
```


### 2. Assigners allowing a fixed amount of lateness

周期性生成watermark的另一个例子是当watermark滞后于在一段时间内数据流中看到的最大(事件时间)时间戳(the watermark lags behind the maximum (event-time) timestamp seen in the stream by a fixed amount of time.)。这种情况包括事先知道流中可能遇到的最大延迟的情况(the maximum lateness that can be encountered in a stream is known in advance)，例如，创建每个元素都带有时间戳的自定义数据源，在一段时间内进行测试时。对于这些情况，Flink提供了`BoundedOutOfOrdernessTimestampExtractor`，带有一个`maxOutOfOrderness`参数，即在计算给定窗口的最终结果时一个元素在被被忽略之前允许延迟的最大时间。延迟对应于`t-t_w`的结果，其中t是元素的(事件时间)时间戳，t_w是先前watermark的时间戳。如果延迟大于0，则该元素被认为是迟到的，并且在计算其相应窗口的作业结果时默认为忽略该元素。

```
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[MyEvent](Time.seconds(10))( _.getCreationTime ))
```


原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/event_timestamp_extractors.html











原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/event_timestamp_extractors.html
