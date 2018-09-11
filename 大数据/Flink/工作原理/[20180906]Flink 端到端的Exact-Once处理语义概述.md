
这篇文章改编自2017年柏林Flink Forward上Piotr Nowojski的[演讲](https://berlin.flink-forward.org/kb_sessions/hit-me-baby-just-one-time-building-end-to-end-exactly-once-applications-with-flink/)。你可以在Flink Forward Berlin网站上找到幻灯片和演示文稿。

2017年12月发布的Apache Flink 1.4.0为Flink为流处理引入了一个重要特性：`TwoPhaseCommitSinkFunction` 的新功能（此处为相关的[Jira](https://issues.apache.org/jira/browse/FLINK-7210)），提取了两阶段提交协议的公共部分，使用Flink和一系列数据源和接收器（包括Apache Kafka 0.11 版本以及更高版本）构建端到端的Exact-Once语义的应用程序成为可能。它提供了一个抽象层，用户只需实现几个方法就可以实现端到端的Exact-Once语义。

如果这就是你需要了解的全部内容，可以去这个[地方](https://ci.apache.org/projects/flink/flink-docs-release-1.4/api/java/org/apache/flink/streaming/api/functions/sink/TwoPhaseCommitSinkFunction.html)了解有关如何使用 `TwoPhaseCommitSinkFunction`。或者你可以直接去看实现Exact-Once语义的[Kafka 0.11 producer](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/connectors/kafka.html#kafka-011)的文档，这也是在 `TwoPhaseCommitSinkFunction` 之上实现的。

如果你想了解更多信息，我们将在这篇文章中去深入了解一下新特性以及Flink幕后发生的事情。

纵览全篇，有以下几点：
- 描述一下Flink检查点在Flink应用程序中保证Exact-Once语义的作用。
- 展现Flink如何通过两阶段提交协议与数据源（source）和数据接收器（sink）交互，以提供端到端的Exact-Once语义保证。
- 通过一个简单的示例，了解如何使用 `TwoPhaseCommitSinkFunction` 实现一个Exact-Once语义的文件接收器。

### 1. Flink程序的Exactly-Once语义

当我们说`Exactly-Once语义`时，我们的意思是每个传入的事件只会对最终结果影响一次。即使机器或软件出现故障，也没有重复数据，也没有丢失的数据。

Flink长期以来在Flink应用程序中提供Exactly-Once语义。在过去几年中，我们已经深入探讨了Flink的检查点，这是Flink提供精确一次语义的能力的核心。 Flink文档还提供了该功能的全面概述。

在继续之前，这里是检查点算法的快速摘要，因为理解检查点对于理解这个更广泛的主题是必要的。























原文：https://data-artisans.com/blog/end-to-end-exactly-once-processing-apache-flink-apache-kafka
