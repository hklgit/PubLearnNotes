---
layout: post
author: sjf0115
title: Spark2.3.0 共享变量
date: 2018-04-10 19:28:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-shared-variables
---


通常，当在远程集群节点上执行传递给Spark操作（如map或reduce）的函数时，将在函数中使用的所有变量的单独副本上工作。这些变量被复制到每台机器上，并且在远程机器上对变量的更新不会回传给驱动程序。在任务之间支持通用的，可读写的共享变量是非常低效的。但是，Spark为两种常见的使用模式提供了共享变量：广播变量和累加器。

### 1. 广播变量

广播变量允许程序员在每台机器上缓存一个只读变量，而不是每个任务保存一份拷贝。例如，使用它们我们可以以更有效的方式将一个大的输入数据集的副本提供给每个节点。Spark还试图使用高效的广播算法来分发广播变量，以降低通信成本。

Spark action 通过一系列 stage 执行，由分布式的 `shuffle` 操作分隔。Spark 会自动广播每个阶段中任务所需的通用数据。以这种方式广播的数据以序列化的形式进行缓存并在运行每个任务之前反序列化。这意味着只有当跨多个 stage 的任务需要相同的数据或以反序列化形式缓存数据非常重要时，显式创建广播变量才是有用的。

广播变量通过调用 `SparkContext.broadcast（v）` 从变量 v 创建。广播变量是 v 的一个包装，广播变量的值可以通过调用 value 方法来访问。下面的代码显示了这一点：

Java版本：
```java
Broadcast<int[]> broadcastVar = sc.broadcast(new int[] {1, 2, 3});

broadcastVar.value();
// returns [1, 2, 3]
```
Scala版本:
```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```
Python版本：
```python
>>> broadcastVar = sc.broadcast([1, 2, 3])
<pyspark.broadcast.Broadcast object at 0x102789f10>

>>> broadcastVar.value
[1, 2, 3]
```

创建广播变量后，运行在集群上的任意函数中的值 v 可以使用广播变量来代替，以便 v 不会在多个节点上拷贝。另外，对象 v 在广播之后为了确保所有节点上的广播变量获得相同的值，对象 v 不应该被修改（例如，如果该变量稍后被传送到新节点）。

### 2. 累加器

累加器是一种仅通过关联和交换操作进行 `加` 的变量，因此可以在并行计算中得到高效的支持。它们可以用来实现计数器（如在 MapReduce 中）或者求和。Spark 本身支持数字类型的累加器，程序员可以添加对新类型的支持。

作为使用者，你可以创建命名或未命名的累加器。如下图所示，命名累加器（在此为 counter 实例）会在 Web UI 中展示。Spark显示了由 `任务` 表中的任务修改的每个累加器的值。

![]()

跟踪 UI 中的累加器对于理解运行的 stage　的进度很有用（注意：Python尚未支持）。

数值型累加器可以通过调用 `SparkContext.longAccumulator()` 或 `SparkContext.doubleAccumulator()` 来创建，分别累加 Long 或 Double 类型的值。 运行在集群上的任务可以使用 `add` 方法进行累加。但是，它们无法读取累加器的值。只有驱动程序可以通过使用 `value` 方法读取累加器的值。

下面的代码显示了一个累加器，用于累加数组的元素：

Java版本:
```java
LongAccumulator accum = jsc.sc().longAccumulator();

sc.parallelize(Arrays.asList(1, 2, 3, 4)).foreach(x -> accum.add(x));
// ...
// 10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

accum.value();
// returns 10
```

Scala版本:
```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```
此代码使用了内置的 Long 类型的累加器，我们还可以通过继承 `AccumulatorV2` 来创建我们自己的类型。

> 备注:
> 在2.0.0之前的版本中，通过继承AccumulatorParam来实现，而2.0.0之后的版本需要继承AccumulatorV2来实现自定义类型的累加器。

`AccumulatorV2` 抽象类有几个方法必须重写：
- `reset` 将累加器复位为零
- `add` 将另一个值添加到累加器中
- `merge` 将另一个相同类型的累加器合并到该累加器中。

其他必须被覆盖的方法包含在[API文档](http://spark.apache.org/docs/2.3.0/api/scala/index.html#org.apache.spark.util.AccumulatorV2)中。 例如，假设我们有一个表示数学向量的MyVector类，我们可以这样写：

Java版本:
```java
class VectorAccumulatorV2 implements AccumulatorV2<MyVector, MyVector> {

  private MyVector myVector = MyVector.createZeroVector();

  public void reset() {
    myVector.reset();
  }

  public void add(MyVector v) {
    myVector.add(v);
  }
  ...
}

// Then, create an Accumulator of this type:
VectorAccumulatorV2 myVectorAcc = new VectorAccumulatorV2();
// Then, register it into spark context:
jsc.sc().register(myVectorAcc, "MyVectorAcc1");
```
Scala版本：
```scala
class VectorAccumulatorV2 extends AccumulatorV2[MyVector, MyVector] {

  private val myVector: MyVector = MyVector.createZeroVector

  def reset(): Unit = {
    myVector.reset()
  }

  def add(v: MyVector): Unit = {
    myVector.add(v)
  }
  ...
}

// Then, create an Accumulator of this type:
val myVectorAcc = new VectorAccumulatorV2
// Then, register it into spark context:
sc.register(myVectorAcc, "MyVectorAcc1")
```
请注意，当程序员定义自己的 AccumulatorV2 类型时，结果类型可以与添加的元素的类型不同。

累加器更新只能在 action 中执行，Spark 保证每个任务对累加器的更新只会应用一次，即重新启动的任务不会更新该值。在 transformations 中，我们需要知道，如果任务或作业 stage 被重新执行，每个任务的更新可能会被应用多次。

累加器不会改变 Spark 的 Lazy 执行模型。如果它们在 RDD 上的某个操作中进行更新，那么只有在 RDD 作为 action 操作的一部分进行计算后才更新它们的值。因此，累加器更新不保证在像 `map()` 这样的 Lazy 转换中执行。下面的代码片段演示了这个属性：

Java版本:
```java
LongAccumulator accum = jsc.sc().longAccumulator();
data.map(x -> { accum.add(x); return f(x); });
// Here, accum is still 0 because no actions have caused the `map` to be computed.
```
Scala版本:
```scala
val accum = sc.longAccumulator
data.map { x => accum.add(x); x }
// Here, accum is still 0 because no actions have caused the map operation to be computed.
```
Python版本:
```python
accum = sc.accumulator(0)
def g(x):
    accum.add(x)
    return f(x)
data.map(g)
# Here, accum is still 0 because no actions have caused the `map` to be computed.
```

> Spark 版本:2.3.0

原文：http://spark.apache.org/docs/2.3.0/rdd-programming-guide.html#shared-variables
