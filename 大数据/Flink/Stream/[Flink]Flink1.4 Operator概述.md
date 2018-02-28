---
layout: post
author: sjf0115
title: Flink1.4 Operator概述
date: 2018-02-28 10:25:17
tags:
  - Flink
  - Flink Stream

categories: Flink
permalink: flink-stream-operators-overall
---

算子(`Operator`)将一个或多个 `DataStream` 转换为新的 `DataStream`。程序可以将多个转换组合成复杂的数据流拓扑。

本节将介绍基本转换(`transformations`)操作，应用这些转换后的有效物理分区以及深入了解 Flink 算子链。

### 1. DataStream Transformations

#### 1.1 Map  

```
DataStream → DataStream
```

输入一个元素并生成一个对应的元素。下面是一个将输入流的值加倍的 `map` 函数：

Java版本：
```Java
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
```

scala版本：
```scala
dataStream.map { x => x * 2 }
```

#### 1.2 FlatMap

```
DataStream → DataStream
```
输入一个元素并生成零个，一个或多个元素。下面是个将句子拆分为单词的 `flatMap` 函数：

Java版本：
```java
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
```

#### 1.3 Filter
```
DataStream → DataStream
```
为每个元素计算一个布尔值的函数并保留函数返回 `true` 的那些元素。下面是一个筛选出零值的 `filter` 函数：

Java版本:
```java
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
```

#### 1.4 KeyBy
```
DataStream → KeyedStream
```
逻辑上将一个流分成不相交的分区，每个分区包含相同键的元素。在内部，这是通过哈希分区实现的。参阅博文[Flink1.4 定义keys的几种方法](http://smartsi.club/2018/01/04/flink-how-to-specifying-keys/)来了解如何指定键。这个转换返回一个 `KeyedStream`。

Java版本:
```java
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
```

Scala版本:
```

```

> 备注

> 在以下情况，不能指定为key：
> - POJO类型，但没有覆盖hashCode()方法并依赖于Object.hashCode()实现。
> - 任意类型的数组。

#### 1.5 Reduce
```
KeyedStream → DataStream
```



### 2. Physical partitioning

### 3. Task chaining and resource groups






























































> 备注：

> Flink 版本： 1.4

原文： https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/stream/operators/index.html
