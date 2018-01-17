---
layout: post
author: sjf0115
title: Flink1.4 数据流类型与转换关系
date: 2018-01-04 15:47:01
tags:
  - Flink

categories: Flink
---

Flink 为流处理和批处理分别提供了 DataStream API 和 DataSet API。正是这种高层的抽象和 flunent API 极大地便利了用户编写大数据应用。不过很多初学者在看到官方文档中那一大坨的转换时，常常会蒙了圈，文档中那些只言片语也很难讲清它们之间的关系。所以本文将介绍几种关键的数据流类型，它们之间是如何通过转换关联起来的。下图展示了 Flink 中目前支持的主要几种流的类型，以及它们之间的转换关系。
