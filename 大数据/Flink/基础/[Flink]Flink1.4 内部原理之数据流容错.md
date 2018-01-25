---
layout: post
author: sjf0115
title: Flink1.4 内部原理之数据流容错
date: 2018-01-24 14:39:01
tags:
  - Flink

categories: Flink
---

### 1. 概述

`Apache Flink`提供了一个容错机制来持续恢复数据流应用程序的状态。该机制确保即使在出现故障的情况下，程序的状态也将最终反映每条来自数据流的记录严格一次`exactly once`。 请注意，有一个开关可以降级为保证至少一次(`least once`)（如下所述）。

容错机制连续生成分布式流数据流的快照。对于状态较小的流式应用程序，这些快照非常轻量级，可以频繁生成，而不会对性能造成太大影响。流应用程序的状态存储在可配置的位置（例如主节点或`HDFS`）。

如果应用程序发生故障（由于机器，网络或软件故障），`Flink`会停止分布式流式数据流。然后系统重新启动算子并将其重置为最新的成功检查点。输入流被重置为状态快照的时间点。作为重新启动的并行数据流处理的任何记录都保证不属于先前检查点状态的一部分。

注意:默认情况下，检查点被禁用。有关如何启用和配置检查点的详细信息，请参[阅检查点](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/stream/state/checkpointing.html)。

为了实现这个机制的保证，数据流源（如消息队列或代理）需要能够将流重放到定义的最近时间点。`Apache Kafka`有这个能力，而`Flink`的Kafka连接器就是利用这个能力。有关`Flink`连接器提供的保证的更多信息，请参阅[数据源和接收器的容错保证](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/connectors/guarantees.html)。

因为`Flink`的检查点是通过分布式快照实现的，所以我们交替使用`快照`和`检查点`两个概念。

### 2. Checkpointing

`Flink`的容错机制的核心部分是生成分布式数据流和算子状态的一致性快照。这些快照作为一个一致性检查点，在系统发生故障时可以回溯。`Flink`的生成这些快照的机制在[分布式数据流的轻量级异步快照](https://arxiv.org/abs/1506.08603)中进行详细的描述。它受分布式快照`Chandy-Lamport`算法的启发，并且专门针对`Flink`的执行模型量身定制。

#### 2.1 Barriers

`Flink`分布式快照的一个核心元素是数据流`Barriers`。这些`Barriers`被放入数据流中，并作为数据流的一部分与记录一起流动。`Barriers`永远不会超越记录，严格按照相对顺序流动。`Barriers`将数据流中的记录分成进入当前快照的记录集合和进入下一个快照的记录集合。每个`Barriers`都携带前面快照的ID。`Barriers`不会中断流的流动，因此非常轻。来自不同快照的多个`Barriers`可以同时在流中，这意味着不同快照可以同时发生。

![]()

`Barriers`在数据流源处被放入的并行数据流。快照`n`放入`Barriers`的位置（我们称之为`Sn`）是快照覆盖数据的源流中的位置。例如，在`Apache Kafka`中，这个位置是分区中最后一个记录的偏移量。该位置`Sn`会报告给检查点协调员（`Flink`的`JobManager`）。

`Barriers`向下游流动。当中间算子从其所有输入流中接收到快照`n`的`Barriers`时，它会将快照`n`的`Barriers`发送到其所有输出流中。一旦`Sink`算子（流式`DAG`的末尾）从其所有输入流中接收到`Barriers n`，就向检查点协调器确认快照`n`。在所有`Sink`确认了快照之后，才被确认已经完成。

一旦快照`n`完成，作业将不会再向数据源询问`Sn`之前的记录，因为那时这些记录（以及它们的后代记录）已经通过了整个数据流拓扑。

#### 2.2 State

#### 2.3 Exactly Once vs. At Least Once

#### 2.4























































原文:https://ci.apache.org/projects/flink/flink-docs-release-1.4/internals/stream_checkpointing.html
