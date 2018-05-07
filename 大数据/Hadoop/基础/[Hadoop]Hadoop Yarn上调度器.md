---
layout: post
author: sjf0115
title: Hadoop Yarn上的调度器
date: 2018-05-07 20:01:01
tags:
  - Hadoop
  - Hadoop 基础

categories: Hadoop
permalink: hadoop-scheduler-of-yarn
---

### 1. 引言

Yarn在Hadoop的生态系统中担任了资源管理和任务调度的角色。在讨论其构造器之前先简单了解一下Yarn的架构。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Hadoop/Hadoop%E4%B8%8B%E4%B8%80%E4%BB%A3MapReduce-yarn-architecture.gif?raw=true)

上图是Yarn的基本架构，其中ResourceManager是整个架构的核心组件，它负责整个集群中包括内存、CPU等资源的管理；ApplicationMaster负责应用程序在整个生命周期的任务调度；NodeManager负责本节点上资源的供给和隔离；Container可以抽象的看成是运行任务的一个容器。本文讨论的调度器是在ResourceManager组建中进行调度的，接下来就一起研究一下包括FIFO调度器、Capacity调度器、Fair调度器在内的三个调度器。

### 2. FIFO调度器
