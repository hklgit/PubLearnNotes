---
layout: post
author: sjf0115
title: Flink1.4 内部原理之作业与调度
date: 2018-01-29 17:31:01
tags:
  - Flink

categories: Flink
permalink: flink_internals_job_scheduling
---


### 1. 调度

`Flink`中的执行资源是通过任务槽定义的。每个`TaskManager`将有一个或多个任务槽，每个任务槽可以运行一个并行任务的管道。 流水线由多个连续的任务组成，例如一个MapFunction的第n个并行实例和一个ReduceFunction的第n个并行实例。 请注意，Flink经常同时执行连续的任务：对于任何情况下发生的流式处理程序，对于批处理程序也经常发生。


























备注:
```
Flink版本:1.4
```

原文:https://ci.apache.org/projects/flink/flink-docs-release-1.4/internals/job_scheduling.html
