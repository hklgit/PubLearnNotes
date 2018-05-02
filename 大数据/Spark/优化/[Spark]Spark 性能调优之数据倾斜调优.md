---
layout: post
author: sjf0115
title: Spark 性能调优之数据倾斜调优
date: 2018-04-05 11:28:01
tags:
  - Spark
  - Spark 优化

categories: Spark
permalink: spark-performance-data-skew-tuning
---

### 1. 为何要处理数据倾斜

#### 1.1 什么是数据倾斜

对Spark/Hadoop这样的大数据系统来讲，数据量大并不可怕，可怕的是数据倾斜。

何谓数据倾斜？数据倾斜指的是，并行处理的数据集中，某一部分（如Spark或Kafka的一个Partition）的数据显著多于其它部分，从而使得该部分的处理速度成为整个数据集处理的瓶颈。

#### 1.2 数据倾斜是如何造成的

在Spark中，同一个Stage的不同Partition可以并行处理，而具体依赖关系的不同Stage之间是串行处理的。假设某个Spark Job分为Stage 0和Stage 1两个Stage，且Stage 1依赖于Stage 0，那Stage 0完全处理结束之前不会处理Stage 1。而Stage 0可能包含N个Task，这N个Task可以并行进行。如果其中N-1个Task都在10秒内完成，而另外一个Task却耗时1分钟，那该Stage的总时间至少为1分钟。换句话说，一个Stage所耗费的时间，主要由最慢的那个Task决定。

由于同一个Stage内的所有Task执行相同的计算，在排除不同计算节点计算能力差异的前提下，不同Task之间耗时的差异主要由该Task所处理的数据量决定。

Stage的数据来源主要分为如下两类：
- 从数据源直接读取。如读取HDFS，Kafka
- 读取上一个Stage的Shuffle数据

### 2. 如何缓解/消除数据倾斜

#### 2.1 尽量避免数据源的数据倾斜

以Spark Stream通过DirectStream方式读取Kafka数据为例。由于Kafka的每一个Partition对应Spark的一个Task（Partition），所以Kafka内相关Topic的各Partition之间数据是否平衡，直接决定Spark处理该数据时是否会产生数据倾斜。

Kafka某一Topic内消息在不同Partition之间的分布，主要由Producer端所使用的Partition实现类决定。如果使用随机Partitioner，则每条消息会随机发送到一个Partition中，从而从概率上来讲，各Partition间的数据会达到平衡。此时源Stage（直接读取Kafka数据的Stage）不会产生数据倾斜。

但很多时候，业务场景可能会要求将具备同一特征的数据顺序消费，此时就需要将具有相同特征的数据放于同一个Partition中。一个典型的场景是，需要将同一个用户相关的PV信息置于同一个Partition中。此时，如果产生了数据倾斜，则需要通过其它方式处理。

#### 2.2 调整并行度分散同一个Task的不同Key

##### 2.2.1 原理

Spark在做Shuffle时，默认使用HashPartitioner（非Hash Shuffle）对数据进行分区。如果并行度设置的不合适，可能造成大量不相同的Key对应的数据被分配到了同一个Task上，造成该Task所处理的数据远大于其它Task，从而造成数据倾斜。

如果调整Shuffle时的并行度，使得原本被分配到同一Task的不同Key发配到不同Task上处理，则可降低原Task所需处理的数据量，从而缓解数据倾斜问题造成的短板效应。


















原文:http://www.infoq.com/cn/articles/the-road-of-spark-performance-tuning
