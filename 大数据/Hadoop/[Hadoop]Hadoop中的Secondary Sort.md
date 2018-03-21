---
layout: post
author: sjf0115
title: Hadoop中的Secondary Sort
date: 2017-12-04 14:15:17
tags:
  - Hadoop

categories: Hadoop
permalink: hadoop-basics-secondary-sort-in-mapreduce
---



### 1. Query

### 2. 理解Shuffle阶段

### 3. 二次排序

#### 3.1 Composite Key

#### 3.2 Sort Comparator

#### 3.3 Partitioner

如果我们使用多个 `reducer`，会发生什么？ 默认分区器 `HashPartitioner` 将根据 `CompositeKey` 对象的 `hashCode` 值将其分配给 `reducer`。无论我们是否重写 `hashcode()` 方法（正确使用所有属性的哈希）还是不重写（使用默认 Object 的实现，使用内存中地址），都将 "随机" 对所有 keys 进行分区。

二次排序不会这样的。因为合并来自 mappers 的所有分区后，reducer 的 key 可能会像这样（第一列）：

![]()

在第一个输出列中，在一个 reducer 内，对于给定 `state` 的数据按城市名称排序，然后按总捐赠量降序排列。但这种排序没有什么意义，因为有些数据丢失了。例如，`Reducer 0` 有2个排序的 `Los Angeles` key，但来自 `Reducer 1` 的 `Los Angeles` 条目应该放在两个 key 之间。

因此，当使用多个 reducers 时，我们想要的是将具有相同 `state` 的所有 （key，value） 键值对发送给同一个 reducer，就像第二列显示的那样。最简单的方法是创建我们自己的 `NaturalKeyPartitioner`，类似于默认的 `HashPartitioner`，但仅基于 `state`  的 `hashCode`，而不是完整的 `CompositeKey` 的 `hashCode`：
```java
import org.apache.hadoop.mapreduce.Partitioner;
import data.writable.DonationWritable;

public class NaturalKeyPartitioner extends Partitioner<CompositeKey, DonationWritable> {

    @Override
    public int getPartition(CompositeKey key, DonationWritable value, int numPartitions) {

        // Automatic n-partitioning using hash on the state name
        return Math.abs(key.state.hashCode() & Integer.MAX_VALUE) % numPartitions;

    }
}
```
我们使用 `job.setPartitionerClass（NaturalKeyPartitioner.class）` 将此类设置为作业的分区器。

#### 3.4 Group Comparator

`Group Comparator` 决定每次调用 `reduce` 方法时如何对这些值分组。

继续使用上图中的 `Reducer 0` 的例子。如果合并分区后，一个 reducer 中的（key，value）键值对必须如下处理：

![]()



### 4. 作业运行与结果


### 5. 结论


























































原文： http://blog.ditullio.fr/2015/12/28/hadoop-basics-secondary-sort-in-mapreduce/
