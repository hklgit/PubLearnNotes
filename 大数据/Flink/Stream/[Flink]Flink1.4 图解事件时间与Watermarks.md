---
layout: post
author: sjf0115
title: Flink1.4 图解事件时间戳与Watermarks
date: 2018-01-15 14:47:01
tags:
  - Flink

categories: Flink
---

如果你正在构建实时流处理应用程序，那么事件时间处理是你迟早必须使用的功能之一。因为在现实世界的大多数用例中，消息到达都是无序的，应该有一些方法，通过你建立的系统知道消息可能延迟到达，并且有相应的处理方案。在这篇博文中，我们将看到为什么我们需要事件时间处理，以及我们如何在ApacheFlink中使用它。

`EventTime`是事件在现实世界中发生的时间，`ProcessingTime`是Flink系统处理该事件的时间。要了解事件时间处理的重要性，我们首先要建立一个基于处理时间的系统，看看它的缺点。

我们创建一个大小为10秒的滑动窗口，每5秒滑动一次，在窗口结束时，系统将发送在此期间收到的消息数。 一旦了解了`EventTime`处理在滑动窗口如何工作，那么了解其在滚动窗口中如何工作也就不是难事。所以让我们开始吧。

### 1. 基于处理时间的系统

在这个例子中，我们期望消息具有一定格式的值，时间戳就是消息的那个值，同时时间戳是在源产生此消息的时间。由于我们正在构建基于处理时间的系统，因此以下代码忽略了时间戳部分。

我们需要知道消息中应包含消息产生时间是很重要的。Flink或任何其他系统不是一个魔术盒，可以以某种方式自己生成这个产生时间。稍后我们将看到，事件时间处理提取此时间戳信息来处理延迟消息。

```
val text = senv.socketTextStream("localhost", 9999)
val counts = text.map {(m: String) => (m.split(",")(0), 1) }
    .keyBy(0)
    .timeWindow(Time.seconds(10), Time.seconds(5))
    .sum(1)
counts.print
senv.execute("ProcessingTime processing example")
```

#### 1.1 消息到达无延迟

假设源分别在第13秒产生两个类型a的消息以及在第16秒产生一个消息。(小时和分钟不重要，因为窗口大小只有10秒)。

![](http://img.blog.csdn.net/20171029180052930?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

这些消息将落入如下所示窗口中。前两个在第13秒产生的消息将落入窗口1`[5s-15s]`和窗口2`[10s-20s]`中，第三个在第16秒产生的消息将落入窗口2`[10s-20s]`和窗口3`[15s-25s]`中。每个窗口发出的最终计数分别为(a，2)，(a，3)和(a，1)。

![](http://img.blog.csdn.net/20171029180104306?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

该输出跟预期的输出是一样的。现在我们看看当一个消息延迟到达系统时会发生什么。

#### 1.2 消息延迟到达

现在假设其中一条消息(在第13秒产生)延迟到达6秒(第19秒到达)，可能是由于某些网络拥塞。你能猜测这个消息会落入哪个窗口？

![](http://img.blog.csdn.net/20171029180114828?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

延迟的消息落入窗口2和窗口3中，因为19在10-20和15-25之间。在窗口2中计算没有任何问题(因为消息本应该落入这个窗口)，但是它影响了窗口1和窗口3的计算结果。我们现在将尝试使用`EventTime`处理来解决这个问题。

### 2. 基于EventTime的系统

要使用`EventTime`处理，我们需要一个时间戳提取器，从消息中提取事件时间信息。请记住，消息是有格式值，时间戳。 `extractTimestamp`方法获取时间戳部分并将其作为Long类型返回。现在忽略`getCurrentWatermark`方法，我们稍后再回来：

```
class TimestampExtractor extends AssignerWithPeriodicWatermarks[String] with Serializable {
  override def extractTimestamp(e: String, prevElementTimestamp: Long) = {
    e.split(",")(1).toLong
  }
  override def getCurrentWatermark(): Watermark = {
      new Watermark(System.currentTimeMillis)
  }
}
```

现在我们需要设置这个时间戳提取器，并将`TimeCharactersistic`设置为`EventTime`。其余的代码与`ProcessingTime`的情况保持一致：

```
senv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
val text = senv.socketTextStream("localhost", 9999)
                .assignTimestampsAndWatermarks(new TimestampExtractor)
val counts = text.map {(m: String) => (m.split(",")(0), 1) }
      .keyBy(0)
      .timeWindow(Time.seconds(10), Time.seconds(5))
      .sum(1)
counts.print
senv.execute("EventTime processing example")
```
运行上述代码的结果如下图所示：

![](http://img.blog.csdn.net/20171029180023621?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

结果看起来更好一些，窗口2和3现在是正确的结果，但是窗口1仍然是有问题的。Flink没有将延迟的消息分配给窗口3，因为现在检查消息的事件时间，并且知道它不应该在那个窗口中。但是为什么没有将消息分配给窗口1？原因是当延迟的信息到达系统时(第19秒)，窗口1的评估(
evaluation)已经完成了(第15秒)。现在让我们尝试通过使用Watermark来解决这个问题。

### 3. Watermark

Watermark是一个非常重要同时也是非常有趣的想法，我将尽力给你一个简短的概述。如果你有兴趣了解更多信息，你可以从Google中观看这个令人敬畏的[演讲](https://www.youtube.com/watch?v=3UfZN59Nsk8)，还可以从dataArtisans那里阅读此[博客](https://data-artisans.com/blog/how-apache-flink-enables-new-streaming-applications-part-1)。 Watermark本质上是一个时间戳。当Flink中的算子(operator)接收到Watermark时，它明白它不会看到比该时间戳更早的消息。因此，在`EventTime`中，Watermark也可以被认为是告诉Flink它有多远的一种方式(Hence watermark can also be thought of as a way of telling Flink how far it is, in the “EventTime”.)。

在这个例子中Watermark的目的，就是把它看作是告诉Flink一个消息延迟多少的方式。在最后一次尝试中，我们将Watermark设置为当前系统时间。因此，不会有任何延迟的消息。我们现在将Watermark设置为当前时间减去5秒，这告诉Flink希望消息最多延迟5秒钟，这是因为每个窗口仅在Watermark通过时被评估。由于我们的Watermark是当前时间减去5秒，所以第一个窗口[5s-15s]将会在第20秒被评估。类似地，窗口[10s-20s]将会在第25秒进行评估，依此类推(译者注:窗口延迟评估)。

```
override def getCurrentWatermark(): Watermark = {
      new Watermark(System.currentTimeMillis - 5000)
}
```
通常最好保持接收到的最大时间戳，并创建最大时间减去预期延迟时间的Watermark，而不是用当前系统时间减去延迟时间(create the watermark with max - expected delay, instead of subtracting from the current system time)。

进行上述更改后运行代码的结果是：

![](http://img.blog.csdn.net/20171029180041319?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

最后我们得到了正确的结果，所有这三个窗口现在都按照预期的方式发送计数，就是(a，2)，(a，3)和(a，1)。

我们也可以使用`AllowedLateness`功能设置消息的最大允许延迟时间来解决这个问题。

### 4. 结论

实时流处理系统的重要性日益增长，延迟消息的处理是你构建任何此类系统的一部分。在这篇博文中，我们看到延迟到达的消息会影响系统的结果，以及如何使用ApacheFlink的事件时间功能来解决它们。


原文:http://vishnuviswanath.com/flink_eventtime.html
