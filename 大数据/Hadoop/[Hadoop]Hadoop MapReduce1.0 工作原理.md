---
layout: post
author: sjf0115
title: Hadoop MapReduce1.0 工作原理
date: 2017-12-14 13:03:01
tags:
  - Hadoop

categories: Hadoop
---

下面解释一下作业在经典的`MapReduce 1.0`中运行的工作原理。最顶层包含4个独立的实体:

- 客户端，提交`MapReduce`作业。
- `JobTracker`，协调作业的运行。`JobTracker`是一个Java应用程序，它的主类是`JobTracker`。
- `TaskTracker`，运行作业划分后的任务。`TaskTracker`是一个Java应用程序，它的主类是`TaskTracker`。
- 分布式文件系统(一般为`HDFS`)，用来在其他实体间共享作业文件。

### 1. 作业提交

`Job`的`submit()`方法创建一个内部的`JobSunmmiter`实例，并且调用其`submitJobInternal()`方法。提交作业后，`waitForCompletion()`每秒轮询作业的进度，如果发现自上次报告后有改变，便把进度报告到控制台。作业完成后，如果成功，就显示作业计数器。如果失败，导致作业失败的错误被记录到控制台。

`JobSunmmiter`所实现的作业提交过程如下:

(1) 向`jobtracker`请求一个新的作业ID(通过调用JobTracker的getNewJobId()方法获取）。参见步骤2.
        检查作业的输出说明。例如，如果没有指定输出目录或输出目录已经存在，作业就不提交，错误抛回给MapReduce程序。
        计算作业的输入分片。如果分片无法计算，比如因为输入路径不存在，作业不提交，错误返回给MapReduce程序。
        将运行作业所需要的资源（包括作业JAR文件、配置文件和计算所得的输入分片）复制到一个以作业ID命名的目录下jobtracker        的 文件系统中。作业JAR中副本较多（由mapred.submit.replication属性控制，默认值为10），因此在运行作业的任务时， 集          群中有很多个副本可供tasktracker访问。步骤3.
        告知jobtracker作业准备执行（通过调用JobTracker的submitJob()方法实现）。参见步骤4.











































来源于: `Hadoop 权威指南`
