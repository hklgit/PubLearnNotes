---
layout: post
author: sjf0115
title: Stream 主流流处理框架比较
date: 2018-01-10 15:34:01
tags:
  - Stream

categories: Stream
---


分布式流处理是对无边界数据集进行连续不断的处理、聚合和分析。它跟`MapReduce`一样是一种通用计算，但我们期望延迟在毫秒或者秒级别。这类系统一般采用有向无环图（`DAG`）。

`DAG`是任务链的图形化表示，我们用它来描述流处理作业的拓扑。如下图，数据从`sources`流经处理任务链到`sinks`。单机可以运行`DAG`，但本篇文章主要聚焦在多台机器上运行`DAG`的情况。
