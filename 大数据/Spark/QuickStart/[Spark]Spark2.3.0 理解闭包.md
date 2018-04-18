---
layout: post
author: sjf0115
title: Spark2.3.0 理解闭包
date: 2018-04-10 19:28:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-understanding-closures
---

Spark的难点之一是理解跨集群执行代码时变量和方法的作用域和生命周期。修改作用域之外的变量的RDD操作经常会造成混淆。在下面的例子中，我们将看看使用 `foreach（）` 来增加计数器的代码，其他操作也会出现类似的问题。

### 1. Example

考虑下面简单的 RDD 元素求和示例，以下行为可能会有所不同，具体取决于是否在同一个 JVM 内执行。一个常见的例子是以本地模式运行Spark（`--master = local [n]`）并将 Spark 应用程序部署到集群中（例如，通过 spark-submit 到 YARN）：

Java版本:
```java
int counter = 0;
JavaRDD<Integer> rdd = sc.parallelize(data);

// Wrong: Don't do this!!
rdd.foreach(x -> counter += x);

println("Counter value: " + counter);
```
Scala版本:
```scala
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
```
Python：
```python
counter = 0
rdd = sc.parallelize(data)

# Wrong: Don't do this!!
def increment_counter(x):
    global counter
    counter += x
rdd.foreach(increment_counter)

print("Counter value: ", counter)
```

### 2. Local模式与cluster模式

上述代码的行为是不确定的，并且可能无法按预期工作。为了执行作业，Spark 将 RDD 操作的处理分解为任务，每个任务由执行程序执行。 在执行之前，Spark会计算任务的关闭。 闭包是那些执行程序在RDD上执行其计算（在本例中为foreach（））时必须可见的变量和方法。 该封闭序列化并发送给每个执行者。

































原文：http://spark.apache.org/docs/2.3.0/rdd-programming-guide.html#understanding-closures-
