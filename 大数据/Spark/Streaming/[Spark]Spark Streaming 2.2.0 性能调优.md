---
layout: post
author: sjf0115
title: Spark Streaming 2.2.0 性能调优
date: 2018-04-06 11:28:01
tags:
  - Spark
  - Spark Stream

categories: Spark
permalink: spark-streaming-performance-tuning
---

在集群上的 Spark Streaming 应用程序中获得最佳性能需要一些调整。本节介绍了可调整的多个 parameters （参数）和 configurations （配置）提高你的应用程序性能.在高层次上, 你需要考虑两件事情:

通过有效利用集群资源, Reducing the processing time of each batch of data （减少每批数据的处理时间）.

设置正确的 batch size （批量大小）, 以便 batches of data （批量的数据）可以像 received （被接收）处理一样快（即 data processing （数据处理）与 data ingestion （数据摄取）保持一致）.






















原文：
