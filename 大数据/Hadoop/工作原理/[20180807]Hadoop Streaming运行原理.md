---
layout: post
author: sjf0115
title: Hadoop Streaming 运行机制
date: 2018-08-07 20:29:01
tags:
  - Hadoop
  - Hadoop 内部原理

categories: Hadoop
permalink: hadoop-internal-anatomy-of-hadoop-streaming
---

Hadoop 提供了 MapReduce 的 API，允许你使用非 Java 的其他语言来写自己的 Map 和 Reduce 函数。Hadoop Streaming 使用 Unix 标准流作为 Hadoop 和应用程序之间的接口，所以我们可以使用任何编程语言通过标准输入/输出来写 MapReduce 程序。允许用户将任何可执行文件或者脚本作为 Map 和 Reduce 函数，这大大提高了程序员的开发效率。

Streaming 天生适合用于文本处理。Map 的输入数据通过标准输入流传递给 Map 函数，并且是一行一行的传输，最后将结果行写入到标准输出中。Map 输出的键值对是一个制表符分割的行，Reduce 函数的输入格式与之相同（通过制表符分割的键值对）并通过标准输入流进行传输。Reduce 函数从标准输入流中读取输入行，该输入已由 Hadoop 框架根据键排过序，最后将结果写入标准输出。

### 1. Example

```python
#! /usr/bin/env python

import re
import sys

for vid in sys.stdin:
  if vid.startswith("60") :
    print "adr"
  elif vid.startswith("80"):
    print "ios"
  else:
    print "other"
```
程序通过程序块读取 STDIN 中的每一行来迭代执行标准输入中的每一行。该程序块从输入的每一行中判断是否包含`60`和`80`，如果包含`60`表示平台为`adr`，如果包含`80`表示平台为`ios`，否则为`other`。

> 值得一提的是 Streaming 和 Java MapReduce

### 2. 运行机制

Streaming 运行特殊的 Map 任务和 Reduce 任务，目的是运行用户提供的可执行程序，并与之通信，如下图所示：

![]()

Streaming 任务使用标准输入和输出流与进程（可以使用任何语言编写）进行通信。在任务执行过程中，Java 进程都会把输入键值对传递给外部的进程，后者通过用户定义的 Map 函数和 Reduce 函数来执行它并把输出键值对传回 Java 进程。从节点管理器的角度看，就像其子进程自己在运行 Map 或 Reduce 代码一样。

























原文：Hadoop权威指南
