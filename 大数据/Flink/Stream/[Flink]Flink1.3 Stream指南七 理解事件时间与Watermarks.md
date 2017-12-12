Flink实现了数据流模型(Dataflow Model)中许多技术。如果想对事件时间(event time)和水位线(watermarks)更详细的了解，请参阅下面的文章:
- [The world beyond batch: Streaming 101
](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)
- [The Dataflow Model](http://www.vldb.org/pvldb/vol8/p1792-Akidau.pdf)

支持事件时间的流处理器需要一种方法来衡量事件时间的进度。例如，一个构建按小时处理窗口的窗口算子，当事件时间超过一小时末尾时需要被通知(a window operator that builds hourly windows needs to be notified when event time has passed beyond the end of an hour)，以便操作员可以关闭正在进行的窗口。

事件时间可以独立于处理时间来运行。例如，在一个程序中，算子(operator)的当前事件时间可以略微落后于处理时间(考虑接收事件的延迟)，而两者以相同的速度继续前行。另一方面，另一个流式处理程序可以运行几个星期的事件时间，但是处理只需几秒钟(another streaming program might progress through weeks of event time with only a few seconds of processing)，通过快速转发已经在Kafka Topic中缓冲的一些历史数据。

Flink中测量事件时间进度的机制是水位线(watermarks)。水位线作为数据流的一部分流动，并携带时间戳t。`Watermark(t)`声明在数据流中事件时间已达到时间t，这意味着流不再有时间戳t'<= t(即时间戳老于或等于水印的事件)的元素。

下图显示了具有时间戳(逻辑上)和内嵌watermark的事件流。在这个例子中，事件是顺序的（相对于它们的时间戳），这意味着水位线只是数据流中的周期性标记。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/stream_watermark_in_order.svg)

watermark对于乱序数据流至关重要，如下图所示，事件并未按照时间戳进行排序。通常，watermark是数据流中一个点的声明，到达某一时间戳的所有事件都应该已经到达这一点(watermark is a declaration that by that point in the stream, all events up to a certain timestamp should have arrived)。一旦watermark达到算子，算子就可以将其内部的事件时间时钟提前到watermark的值。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/stream_watermark_out_of_order.svg)


### 1. 数据流中的并行Watermarks

watermarks是直接通过源函数(source functions)生成的或在源函数之后生成的。源函数的每个并行子任务通常独立生成watermarks。这些watermarks在该特定并行源中定义事件时间。

watermarks贯穿整个流处理程序，他们会在到达的算子处将事件时间提前(they advance the event time at the operators where they arrive)。无论算子提前事件时间到何时，它都会为下游的后续算子生成一个新的watermarks(Whenever an operator advances its event time, it generates a new watermark downstream for its successor operators.)。

一些算子消耗多个输入流;union操作，例如后面跟着keyBy(...)函数或者partition(...)函数。这样的算子的当前事件时间是其输入流的事件时间的最小值。由于输入流更新了事件时间，因此算子也是如此。

下图显示了流过并行流的事件和watermarks的示例，以及跟踪事件时间的算子:

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/parallel_streams_watermarks.svg)

### 2. 延迟元素

某些元素可能违反watermarks条件，这意味着即使在`watermarks(t)`发生之后，还是会出现很多的时间戳t'<= t的元素。事实上，在现实世界的许多设置中，某些元素可以被任意地延迟，因此指定一个时间，在这个时间内所有在一个特定事件时间戳的事件都会发生是不可能的(making it impossible to specify a time by which all elements of a certain event timestamp will have occurred)。 此外，即使延迟可以被限制，但通常也不希望延迟太多的watermarks，因为它在事件时间窗口的评估中导致太多的延迟。

因此，流处理程序中可能会明确地指定一些延迟元素。延迟元素是在系统的事件时钟（由水印发出信号）之后已经通过了延迟元素时间戳的时间之后到达的元素(Late elements are elements that arrive after the system’s event time clock (as signaled by the watermarks) has already passed the time of the late element’s timestamp.)。 有关如何处理事件时间窗口中的晚期元素的更多信息，请参阅允许的延迟。

### 3. 调试Watermarks

请参阅[调试Windows和事件时间](https://ci.apache.org/projects/flink/flink-docs-release-1.3/monitoring/debugging_event_time.html)部分，以便在运行时调试Watermarks。

原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/event_time.html#event-time-and-watermarks
