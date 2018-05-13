---
layout: post
author: sjf0115
title: Hadoop Streaming使用指南
date: 2018-05-10 13:15:00
tags:
  - Hadoop
  - Hadoop 基础

categories: Hadoop
permalink: hadoop-streaming-use-guide
---

### 1. 概述

Hadoop Streaming 是Hadoop发行版附带的实用工具。该工具允许你使用任何可执行文件或脚本作为 mapper 或 reducer 来创建和运行 Map/Reduce 作业。 例如：
```
mapred streaming \
  -input myInputDirs \
  -output myOutputDir \
  -mapper /bin/cat \
  -reducer /usr/bin/wc
```

### 2. 如何工作

在上面的例子中，mapper 和 reducer 都是可执行文件，它们从 stdin （逐行读取）输入，并将输出发送到 stdout。Hadoop Streaming 将创建一个 Map/Reduce作业，并将作业提交到集群上，同时监视作业的进度。

当 mapper 指定为可执行文件时，在 mapper 初始化时每个 mapper 任务将可执行文件作为一个单独的进程进行启动。当 mapper 任务运行时，将输入转换为行并提供给 stdin 进程。与此同时，mapper 从 stdout 进程收集以行格式的输出，并将每行转换为一个键/值对，将其作为 mapper 的输出。默认情况下，每行第一个制表符之前的前缀（包括制表符本身）为 key，而剩下的为 value。如果行中没有制表符，则整行被认为是 key，value 为空。但是，这可以通过 `-inputformat` 命令选项来设置，稍后讨论。

当 reducer 指定为可执行文件时，每个 reducer 任务将可执行文件作为一个单独的进程进行启动，然后对 reducer 进行初始化。 在 reducer 任务运行时，将输入键/值对转换为行并将提供给 stdin 进程。同时，reducer 从 stdout 进程收集以行格式的输出，并将每行转换为键/值对，作为 reducer 的输出。默认情况下，每行第一个制表符之前的前缀（包括制表符本身）为 key，而剩下的为 value。但是，这可以通过 `-outputformat` 命令选项来设置，稍后讨论。

这是 Map/Reduce 框架和 streaming mapper/reducer 之间通信协议的基础。

用户可以指定 `stream.non.zero.exit.is.failure` 为 true 或 false，以使 streaming 任务的非零状态分别标识为 Failure 或 Success。默认情况下，以非零状态退出的流式处理任务被视为失败任务。

### 3. Streaming Command Options

Streaming 支持流式命令选项以及[通用命令选项](http://hadoop.apache.org/docs/r3.1.0/hadoop-streaming/HadoopStreaming.html#Generic_Command_Options)。一般的命令行语法如下所示。

> 确保通用命令选项在流式命令选项之前放置，否则命令将失效。有关示例，请参阅使[Making Archives Available to Tasks](http://hadoop.apache.org/docs/r3.1.0/hadoop-streaming/HadoopStreaming.html#Making_Archives_Available_to_Tasks)。

```
mapred streaming [genericOptions] [streamingOptions]
```

Parameter|Optional/Required|Description
---|---|---
-input directoryname or filename|Required|Input location for mapper
-output directoryname|Required|Output location for reducer
-mapper executable or JavaClassName|Optional|Mapper executable. If not specified, IdentityMapper is used as the default
-reducer executable or JavaClassName|Optional|Reducer executable. If not specified, IdentityReducer is used as the default
-file filename	Optional	Make the mapper, reducer, or combiner executable available locally on the compute nodes
-inputformat JavaClassName	Optional	Class you supply should return key/value pairs of Text class. If not specified, TextInputFormat is used as the default
-outputformat JavaClassName	Optional	Class you supply should take key/value pairs of Text class. If not specified, TextOutputformat is used as the default
-partitioner JavaClassName	Optional	Class that determines which reduce a key is sent to
-combiner streamingCommand or JavaClassName	Optional	Combiner executable for map output
-cmdenv name=value	Optional	Pass environment variable to streaming commands
-inputreader	Optional	For backwards-compatibility: specifies a record reader class (instead of an input format class)
-verbose	Optional	Verbose output
-lazyOutput	Optional	Create output lazily. For example, if the output format is based on FileOutputFormat, the output file is created only on the first call to Context.write
-numReduceTasks	Optional	Specify the number of reducers
-mapdebug	Optional	Script to call when map task fails
-reducedebug	Optional	Script to call when reduce task fails














> Hadoop版本:3.1.0

原文：http://hadoop.apache.org/docs/r3.1.0/hadoop-streaming/HadoopStreaming.html
