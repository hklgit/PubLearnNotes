---
layout: post
author: sjf0115
title: Hadoop MapReduce 2 工作原理
date: 2017-12-14 19:58:01
tags:
  - Hadoop

categories: Hadoop
---

对于节点数超出4000的大型集群，`MapReduce 1`的系统开始面领着扩展性的瓶颈。在2010年雅虎的一个团队开始设计下一代`MapReduce`.由此，`YARN`(`Yet Another Resource Negotiator`的缩写或者为`YARN Application Resource Neforiator`的缩写)应运而生。

`YARN` 将 `Jobtracker` 的职能划分为多个独立的实体，从而改善了'经典的'`MapReduce`面临的扩展瓶颈问题。`Jobtracker`负责作业调度和任务进度监视、追踪任务、重启失败或过慢的任务和进行任务登记，例如维护计数器总数。

`YARN`将这两种角色划分为两个独立的守护进程：管理集群上资源使用的资源管理器(`ResourceManager`)和管理集群上运行任务生命周期的应用管理器(`ApplicationMaster`)。基本思路是：应用服务器与资源管理器协商集群的计算资源：容器(每个容器都有特定的内存上限)，在这些容器上运行特定应用程序的进程。容器由集群节点上运行的节点管理器(`NodeManager`)监视，以确保应用程序使用的资源不会超过分配给它的资源。

与`jobtracker`不同，应用的每个实例（这里指一个`MapReduce`作业）有一个专用的应用`master`(`ApplicationMaster`)，它运行在应用的运行期间。这种方式实际上和最初的`Google`的`MapReduce`论文里介绍的方法很相似，该论文描述了`master`进程如何协调在一组`worker`上运行的`map`任务和`reduce`任务。

如前所述，`YARN`比`MapReduce`更具一般性，实际上`MapReduce`只是`YARN`应用的一种形式。有很多其他的`YARN`应用(例如能够在集群中的一组节点上运行脚本的分布式shell)以及其他正在开发的程序。`YARN`设计的精妙之处在于不同的`YARN`应用可以在同一个集群上共存。例如，一个`MapReduce`应用可以同时作为MPI应用运行，这大大提高了可管理性和集群的利用率。

此外，用户甚至有可能在同一个`YARN`集群上运行多个不同版本的`MapReduce`，这使得`MapReduce`升级过程更容易管理。注意，`MapReduce`的某些部分(比如作业历史服务器和shuffle处理器)以及`YARN`本身仍然需要在整个集群上升级。

`YARN`上的`MapReduce`比经典的`MapReduce`包括更多的实体：
- 提交`MapReduce`作业的客户端
- `YARN`资源管理器(`ResourceManager`)，负责协调集群上计算资源的分配
- `YARN`节点管理器(`NodeManager`)，负责启动和监视集群中机器上的计算容器(`container`)
- `MapReduce`应用程序`master`(`ApplicationMaster`)，负责协调运行`MapReduce`作业的任务。它和`MapReduce`任务在容器中运行，这些容器由资源管理器分配并由节点管理器进行管理
- 分布式文件系统（一般为`HDFS`），用来与其他实体见共享作业文件

作业运行过程如下图所示:


















来源于: `Hadoop 权威指南`
