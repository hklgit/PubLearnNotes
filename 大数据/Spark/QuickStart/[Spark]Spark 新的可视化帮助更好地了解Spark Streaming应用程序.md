---
layout: post
author: sjf0115
title: Spark UI 之 Streaming 标签页
date: 2018-04-19 11:28:01
tags:
  - Spark
  - Spark Stream

categories: Spark
permalink: spark-streaming-ui-streaming-tab
---

这篇博文将重点介绍为理解 Spark Streaming 应用程序而引入的新的可视化功能。我们已经更新了 Spark UI 中的 Streaming 标签页来显示以下信息：
- 时间轴视图和事件率统计，调度延迟统计以及以往的批处理时间统计
- 每个批次中所有JOB的详细信息
此外，为了理解在 Streaming 操作上下文中作业的执行情况，有向无环执行图的可视化增加了 Streaming 的信息。

让我们通过一个从头到尾分析Streaming应用程序的例子详细看一下上面这些新的功能。

### 1. 处理趋势的时间轴和直方图

当我们调试一个 Spark Streaming 应用程序的时候，我们更希望看到数据正在以什么样的速率被接收以及每个批次的处理时间是多少。Streaming标签页中新的UI能够让你很容易的看到目前的值和之前1000个批次的趋势情况。当你在运行一个 Streaming 应用程序的时候，如果你去访问 Spark UI 中的 Streaming 标签页，你将会看到类似下面图一的一些东西（红色的字母，例如[A]，是我们的注释，并不是UI的一部分）。

第一行（标记为 [A]）展示了Streaming应用程序当前的状态；在这个例子中，应用已经以1秒的批处理间隔运行了将近40分钟;在它下面是输入速率（Input rate）的时间轴（标记为 [B]），显示了Streaming应用从它所有的源头以大约49 events每秒的速度接收数据。在这个例子中，时间轴显示了在中间位置（标记为[C]）平均速率有明显的下降，在时间轴快结束的地方应用又恢复了。如果你想得到更多详细的信息，你可以点击 Input Rate旁边（靠近[B]）的下拉列表来显示每个源头各自的时间轴，正如下面图2所示：





















原文：
译文：https://www.csdn.net/article/2015-07-15/2825214
