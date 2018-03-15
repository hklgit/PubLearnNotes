---
layout: post
author: 加米谷大数据
title: Spark内部原理之运行原理一
date: 2018-03-14 11:33:01
tags:
  - Spark
  - Spark 内部原理

categories: Spark
permalink: spark-internal-operating-principle-one
---

在大数据领域，只有深挖数据科学领域，走在学术前沿，才能在底层算法和模型方面走在前面，从而占据领先地位。

Spark的这种学术基因，使得它从一开始就在大数据领域建立了一定优势。无论是性能，还是方案的统一性，对比传统的 Hadoop，优势都非常明显。Spark 提供的基于 RDD 的一体化解决方案，将 MapReduce、Streaming、SQL、Machine Learning、Graph Processing 等模型统一到一个平台下，并以一致的API公开，并提供相同的部署方案，使得 Spark 的工程应用领域变得更加广泛。

### 1. Spark 专业术语定义

#### 1.1 Application：Spark应用程序

指的是用户编写的Spark应用程序，包含了Driver功能代码和分布在集群中多个节点上运行的Executor代码。

Spark应用程序，由一个或多个作业JOB组成，如下图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-1.jpg?raw=true)

#### 1.2 Driver：驱动程序

Spark 中的 Driver 即运行上述 Application 的 Main() 函数并且创建 SparkContext，其中创建 SparkContext 的目的是为了准备 Spark 应用程序的运行环境。在 Spark 中由 SparkContext 负责和 ClusterManager 通信，进行资源的申请、任务的分配和监控等；当 Executor 部分运行完毕后，Driver 负责将 SparkContext 关闭。通常 SparkContext 代表 Driver，如下图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-2.jpg?raw=true)

#### 1.3 Cluster Manager：资源管理器

指的是在集群上获取资源的外部服务，常用的有：Standalone，Spark 原生的资源管理器，由 Master 负责资源的分配；Haddop Yarn，由 Yarn 中的 ResearchManager 负责资源的分配；Messos，由 Messos 中的 Messos Master 负责资源管理，如下图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-3.jpg?raw=true)

#### 1.4 Executor：执行器

Application 运行在 Worker 节点上的一个进程，该进程负责运行 Task，并且负责将数据存在内存或者磁盘上，每个 Application 都有各自独立的一批 Executor，如下图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-4.jpg?raw=true)

#### 1.5 Worker：计算节点

集群中任何可以运行 Application 代码的节点，类似于 Yarn 中的 NodeManager 节点。在Standalone模式中指的就是通过Slave文件配置的Worker节点，在Spark on Yarn模式中指的就是NodeManager节点，在Spark on Messos模式中指的就是Messos Slave节点，如下图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-5.jpg?raw=true)

#### 1.6 RDD：弹性分布式数据集

Resillient Distributed Dataset，Spark的基本计算单元，可以通过一系列算子进行操作（主要有Transformation和Action操作），如下图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-6.jpg?raw=true)

#### 1.7 窄依赖

父RDD每一个分区最多被一个子RDD的分区所用；表现为一个父RDD的分区对应于一个子RDD的分区，或两个父RDD的分区对应于一个子RDD 的分区。如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-7.jpg?raw=true)

#### 1.8 宽依赖

父RDD的每个分区都可能被多个子RDD分区所使用，子RDD分区通常对应所有的父RDD分区。如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-8.jpg?raw=true)

常见的窄依赖有：map、filter、union、mapPartitions、mapValues、join（父RDD是hash-partitioned ：如果JoinAPI之前被调用的RDD API是宽依赖(存在shuffle), 而且两个join的RDD的分区数量一致，join结果的rdd分区数量也一样，这个时候join api是窄依赖）。

常见的宽依赖有groupByKey、partitionBy、reduceByKey、join（父RDD不是hash-partitioned ：除此之外的，rdd 的join api是宽依赖）。

#### 1.9 DAG：有向无环图

Directed Acycle graph，反应RDD之间的依赖关系，如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-9.jpg?raw=true)

#### 1.10 DAGScheduler：有向无环图调度器

基于 DAG 划分 Stage 并以 TaskSet 的形势把 Stage 提交给 TaskScheduler；负责将作业拆分成不同阶段的具有依赖关系的多批任务；最重要的任务之一就是：计算作业和任务的依赖关系，制定调度逻辑。在 SparkContext 初始化的过程中被实例化，一个 SparkContext 对应创建一个 DAGScheduler。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-10.jpg?raw=true)

#### 1.11 TaskScheduler：任务调度器

将 Taskset 提交给 worker（集群）运行并回报结果；负责每个具体任务的实际物理调度。如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-11.jpg?raw=true)

#### 1.12 Job：作业

由一个或多个调度阶段所组成的一次计算作业；包含多个Task组成的并行计算，往往由Spark Action催生，一个JOB包含多个RDD及作用于相应RDD上的各种Operation。如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-12.jpg?raw=true)

#### 1.13 Stage：调度阶段

一个任务集对应的调度阶段；每个Job会被拆分很多组Task，每组任务被称为Stage，也可称TaskSet，一个作业分为多个阶段；Stage分成两种类型ShuffleMapStage、ResultStage。如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-13.jpg?raw=true)


1.14 TaskSet：任务集

由一组关联的，但相互之间没有Shuffle依赖关系的任务所组成的任务集。如图所示。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-14.jpg?raw=true)

提示：

1）一个Stage创建一个TaskSet；

2）为Stage的每个Rdd分区创建一个Task,多个Task封装成TaskSet

#### 1.15 Task：任务

被送到某个Executor上的工作任务；单个分区数据集上的最小处理流程单元。如图所示

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Spark/spark-internal-operating-principle-one-15.jpg?raw=true)



























原文： https://www.toutiao.com/i6511498014832460301/?tt_from=weixin&utm_campaign=client_share&timestamp=1520998005&app=news_article&utm_source=weixin&iid=26380623414&utm_medium=toutiao_android&wxshare_count=1
