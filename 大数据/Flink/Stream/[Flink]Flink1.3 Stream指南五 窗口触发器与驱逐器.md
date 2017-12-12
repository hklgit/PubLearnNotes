### 1. 窗口触发器

触发器(Trigger)确定窗口(由[窗口分配器](http://blog.csdn.net/sunnyyoona/article/details/78327447)形成)何时准备好被[窗口函数](http://blog.csdn.net/sunnyyoona/article/details/78329150)处理。每个窗口分配器都带有默认触发器。如果默认触发器不满足你的要求，可以使用`trigger(...)`指定自定义触发器。

触发器接口有五种方法允许触发器对不同的事件做出反应：
```java
public abstract TriggerResult onElement(T element, long timestamp, W window, TriggerContext ctx) throws Exception;

public abstract TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception;

public abstract TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception;

public void onMerge(W window, OnMergeContext ctx) throws Exception {
	throw new UnsupportedOperationException("This trigger does not support merging.");
}

public abstract void clear(W window, TriggerContext ctx) throws Exception;
```
(1) onElement()方法，当每个元素被添加窗口时调用。

(2) onEventTime()方法，当注册的事件时间计时器(event-time timer)触发时调用。

(3) onProcessingTime()方法，当注册的处理时间计时器(processing-time timer)触发时调用。

(4) onMerge()方法，与状态触发器相关，并且在相应的窗口合并时合并两个触发器的状态。例如，使用会话窗口时。

(5) clear()方法，执行删除相应窗口所需的任何操作(performs any action needed upon removal of the corresponding window)。

以上方法有两件事要注意:

(1) 前三个函数决定了如何通过返回一个`TriggerResult`来对其调用事件采取行动。`TriggerResult`可以是以下之一：
```java
public enum TriggerResult {
  // 什么都不做
	CONTINUE(false, false),

  // 触发计算，然后清除窗口中的元素
	FIRE_AND_PURGE(true, true),

  // 触发计算
	FIRE(true, false),

  // 清除窗口中的元素
	PURGE(false, true);
}
```

(2) 上面任何方法都可以用于注册处理时间计时器或事件时间计时器以供将来的操作使用。

#### 1.1 触发与清除

一旦触发器确定窗口准备好处理数据，它将触发，例如，它返回`FIRE`或`FIRE_AND_PURGE`。这是窗口算子给当前窗口发送结果的信号。给定一个带有`WindowFunction`的窗口，所有的元素都被传递给`WindowFunction`(可能在将所有元素传递给evictor之后)。带有`ReduceFunction`或者`FoldFunction`的窗口只是简单地发出他们急切希望得到的聚合结果(Windows with ReduceFunction of FoldFunction simply emit their eagerly aggregated result.)。

触发器触发时，可以是`FIRE`或`FIRE_AND_PURGE`。当是`FIRE`时保留窗口的内容，当时`FIRE_AND_PURGE`时会删除其内容。默认情况下，内置的触发器只返回`FIRE`，不会清除窗口状态(the pre-implemented triggers simply FIRE without purging the window state)。

备注:
```
清除只是简单地删除窗口的内容，并留下关于窗口和任何触发状态的任何潜在元信息。
```
#### 1.2 窗口分配器的默认触发器

窗口分配器的默认触发器适用于许多情况。例如，所有的事件时间窗口分配器都有一个`EventTimeTrigger`作为默认触发器。一旦watermark到达窗口的末尾，这个触发器就会被触发。

备注:
```
全局窗口(GlobalWindow)的默认触发器是永不触发的NeverTrigger。因此，在使用全局窗口时，必须自定义一个触发器。
```

备注:
```
通过使用trigger()方法指定触发器，将会覆盖窗口分配器的默认触发器。例如，如果你为TumblingEventTimeWindows指定CountTrigger，则不会再根据时间的进度启动窗口函数，而只能通过计数。现在，如果你希望基于时间和个数进行触发，则必须编写自己的自定义触发器。
```

#### 1.3 内置触发器和自定义触发器

Flink带有一些内置触发器:

(1) `EventTimeTrigger`，根据watermarks测量的事件时间进度触发。

(2) `ProcessingTimeTrigger`，基于处理时间触发。

(3) `CountTrigger`，一旦窗口中的元素数量超过给定限制就会触发。

(4) `PurgingTrigger`将其作为另一个触发器的参数，并将其转换为带有清除功能(transforms it into a purging one)。

如果需要实现一个自定义的触发器，你应该看看[Trigger](https://github.com/apache/flink/blob/master//flink-streaming-java/src/main/java/org/apache/flink/streaming/api/windowing/triggers/Trigger.java)抽象类。请注意，API仍在发展中，在Flink未来版本中可能会发生改变。

### 2. 窗口驱逐器

Flink的窗口模型允许在窗口分配器和触发器之外指定一个可选的驱逐器(Evictor)。这可以使用`evictor(...)`方法来完成。驱逐器能够在触发器触发之后，在应用窗口函数之前或之后从窗口中移除元素，也可以之前之后都删除元素( The evictor has the ability to remove elements from a window after the trigger fires and before and/or after the window function is applied.)。为此，Evictor接口有两种方法：

```java
/**
 * Optionally evicts elements. Called before windowing function.
 *
 * @param elements The elements currently in the pane.
 * @param size The current number of elements in the pane.
 * @param window The {@link Window}
 * @param evictorContext The context for the Evictor
 */
void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

/**
 * Optionally evicts elements. Called after windowing function.
 *
 * @param elements The elements currently in the pane.
 * @param size The current number of elements in the pane.
 * @param window The {@link Window}
 * @param evictorContext The context for the Evictor
 */
 void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);
```

`evictBefore()`包含驱逐逻辑，在窗口函数之前应用。而`evictAfter()`在窗口函数之后应用。在应用窗口函数之前被逐出的元素将不被处理。

Flink带有三个内置的驱逐器:

(1) CountEvictor：保持窗口内用户指定数量的元素，如果多于用户指定的数量，从窗口缓冲区的开头丢弃剩余的元素。

(2) DeltaEvictor：使用`DeltaFunction`和阈值，计算窗口缓冲区中的最后一个元素与其余每个元素之间的delta值，并删除delta值大于或等于阈值的元素(computes the delta between the last element in the window buffer and each of the remaining ones, and removes the ones with a delta greater or equal to the threshold)。

(3) TimeEvictor：以毫秒为单位的时间间隔作为参数，对于给定的窗口，找到元素中的最大的时间戳`max_ts`，并删除时间戳小于`max_ts - interval`的所有元素。

备注:
```
默认情况下，所有内置的驱逐器在窗口函数之前应用。
```

备注:
```
指定驱逐器可阻止任何的预聚合(pre-aggregation)，因为窗口的所有元素必须在应用计算之前传递给驱逐器。
```

备注:
```
Flink不保证窗口内元素的顺序。 这意味着虽然驱逐者可以从窗口的开头移除元素，但这些元素不一定是先到的还是后到的。
```

备注:
```
Flink版本:1.3
```

原文: https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/windows.html#triggers
