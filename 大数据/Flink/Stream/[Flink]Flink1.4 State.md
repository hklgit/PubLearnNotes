---
layout: post
author: sjf0115
title: Flink1.4 State
date: 2018-01-16 20:32:17
tags:
  - Flink

categories: Flink
---

### 1. Keyed State 与 Operator State

Flink有两种基本的`State`：`Keyed State`和`Operator State`。

#### 1.1 Keyed State

`Keyed State`总是与`key`相关，只能在`KeyedStream`的函数和运算符中使用。

备注:
```
KeyedStream继承DataStream，表示根据指定的key进行分组的数据流。使用DataStream提供的KeySelector根据key对其上的算子State进行分区。
DataStream支持的典型操作也可以在KeyedStream上进行，除了诸如shuffle，forward和keyBy之类的分区方法之外。

一个KeyedStream可以通过调用DataStream.keyBy()来获得。而在KeyedStream上进行任何transformation都将转变回DataStream。
```

您可以将键控状态视为已经分区或分区的操作员状态，每个键只有一个状态分区。 每个键控状态都被逻辑地绑定到<parallel-operator-instance，key>的唯一组合上，并且由于每个键“只属于”一个键控操作符的一个并行实例，我们可以简单地把它想象成<operator，key>。

键控状态被进一步组织成所谓的密钥组。 密钥组是Flink可以重新分配键控状态的原子单位; 有确定的最大并行度的密钥组就是一样多的。 在执行期间，键控操作符的每个并行实例都与一个或多个键组的键一起工作。

#### 1.2 Operator State

### 2. Raw State 与 Managed State

### 3. Managed Keyed State

### 4. Managed Operator State



原文:https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/stream/state/state.html
