---
layout: post
author: sjf0115
title: Spark2.3.0 RDD操作
date: 2018-03-12 19:13:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-rdd-operations
---

RDD支持两种类型的操作：
- 转换操作(transformations): 从现有数据集创建一个新数据集
- 动作操作(actions): 在数据集上进行计算后将值返回给驱动程序

例如，map 是一个转换操作，传递给每个数据集元素一个函数并返回一个新 RDD 表示返回结果。另一方面，reduce 是一个动作操作，使用一些函数聚合 RDD 的所有元素并将最终结果返回给驱动程序（尽管还有一个并行的 reduceByKey 返回一个分布式数据集）。

在 Spark 中，所有的转换操作(transformations)都是惰性(lazy)的，它们不会马上计算它们的结果。相反，它们仅仅记录应用到基础数据集(例如一个文件)上的转换操作。只有当 action 操作需要返回一个结果给驱动程序的时候， 转换操作才开始计算。

这个设计能够让 Spark 运行得更加高效。例如，我们知道：通过 map 创建的新数据集将在 reduce 中使用，并且仅仅返回 reduce 的结果给驱动程序，而不必将比较大的映射后的数据集返回。

### 1. 基础

为了说明 RDD 基础知识，请考虑以下简单程序：

Java版本:
```java
JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(s -> s.length());
int totalLength = lineLengths.reduce((a, b) -> a + b);
```
Scala版本:
```scala
val lines = sc.textFile("data.txt")
val lineLengths = lines.map(s => s.length)
val totalLength = lineLengths.reduce((a, b) => a + b)
```
Python版本:
```python
lines = sc.textFile("data.txt")
lineLengths = lines.map(lambda s: len(s))
totalLength = lineLengths.reduce(lambda a, b: a + b)
```

第一行定义了一个来自外部文件的基本 RDD。这个数据集并未加载到内存中或做其他处理：lines 仅仅是一个指向文件的指针。第二行将 lineLengths 定义为 map 转换操作的结果。其次，由于转换操作的惰性(lazy)，lineLengths 并没有立即计算。最后，我们运行 reduce，这是一个动作操作。此时，Spark 把计算分成多个任务(task)，并让它们运行在多台机器上。每台机器都运行 map 的一部分以及本地 reduce。然后仅仅将结果返回给驱动程序。

如果稍后还会再次使用 lineLength，我们可以在运行 reduce 之前添加：

Java版本:
```java
lineLengths.persist(StorageLevel.MEMORY_ONLY());
```
Scala版本:
```scala
lineLengths.persist()
```
Python版本:
```python
lineLengths.persist()
```
这将导致 lineLength 在第一次计算之后被保存在内存中。

### 2. 传递函数给Spark

Spark 的 API 很大程度上依赖于运行在集群上的驱动程序中的函数。

#### 2.1 Java版本

在 Java 中，函数由 [org.apache.spark.api.java.function](http://spark.apache.org/docs/2.3.0/api/java/index.html?org/apache/spark/api/java/function/package-summary.html) 接口实现。创建这样的函数有两种方法：
- 在你自己类中实现 Function 接口，作为匿名内部类或命名内部类，并将其实例传递给Spark。
- 使用 [lambda 表达式](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html) 来简洁地定义一个实现。

虽然本指南的大部分内容都使用 lambda 语法进行简明说明，但很容易以长格式使用所有相同的API。例如，我们可以按照以下方式编写我们的代码：

匿名内部类
```java
JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(new Function<String, Integer>() {
  public Integer call(String s) { return s.length(); }
});
int totalLength = lineLengths.reduce(new Function2<Integer, Integer, Integer>() {
  public Integer call(Integer a, Integer b) { return a + b; }
});
```
或者命名内部类
```java
class GetLength implements Function<String, Integer> {
  public Integer call(String s) { return s.length(); }
}
class Sum implements Function2<Integer, Integer, Integer> {
  public Integer call(Integer a, Integer b) { return a + b; }
}

JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(new GetLength());
int totalLength = lineLengths.reduce(new Sum());
```

> 请注意，Java中的匿名内部类也可以访问封闭范围内的变量，只要它们标记为final。 Spark会将这些变量的副本发送给每个工作节点，就像其他语言一样。

#### 2.2 Scala版本

有两种推荐的的方法可以做到这一点：
- [匿名函数语法](http://docs.scala-lang.org/tour/basics.html#functions)，可用于短片段代码。
- 全局单例对象中的静态方法。例如，您可以定义对象 MyFunctions，然后传递 MyFunctions.func1，如下所示：

```scala
object MyFunctions {
  def func1(s: String): String = { ... }
}

myRdd.map(MyFunctions.func1)
```

> 虽然也可以在类实例中传递方法的引用（与单例对象相反），但这需要将包含该类的对象与方法一起发送。 例如，考虑：

```scala
class MyClass {
  def func1(s: String): String = { ... }
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
}
```


下表中列出一些基本的函数接口：

函数名 | 实现的方法 | 用途
---|---|---
Function<T,R> | R call(T) | 接收一个输入值并返回一个输出值，用于类似map()和filter()等操作中
Function2<T1,T2,R> | R call(T1,T2) |  接收两个输入值并返回一个输出值，用于类似aggregate()和fold()等操作中
FlatMapFunction<T,R> | Iterable<R> call(T) | 接收一个输入值并返回任意个输出，用于类似flatMap()这样的操作中


### 3. 使用键值对
虽然大多数Spark操作适用于包含任何类型对象的RDD上，但是几个特殊操作只能在键值对的RDD上使用。 最常见的是分布式“shuffle”操作，例如按键分组或聚合元素。

在Java中，使用Scala标准库中的scala.Tuple2类来表示键值对。 可以如下简单地调用：
```
new Tuple2（a，b）
```
来创建一个元组，然后用 `tuple._1（）` 和 `tuple._2（）` 访问它的字段。

键值对的RDD由JavaPairRDD类表示。 您可以使用特殊版本的map操作（如mapToPair和flatMapToPair）从JavaRDD来构建JavaPairRDD。 JavaPairRDD将具有标准的RDD的函数以及特殊的键值对函数。

例如，以下代码在键值对上使用reduceByKey操作来计算每行文本在文件中的出现次数：
```
JavaRDD<String> lines = sc.textFile("data.txt");
JavaPairRDD<String, Integer> pairs = lines.mapToPair(s -> new Tuple2(s, 1));
JavaPairRDD<String, Integer> counts = pairs.reduceByKey((a, b) -> a + b);
```
例如，我们也可以使用counts.sortByKey（）来按字母顺序来对键值对排序，最后将counts.collect（）作为对象数组返回到驱动程序。

注意：当使用一个自定义对象作为 key 在使用键值对操作的时候，你需要确保自定义 equals() 方法和 hashCode() 方法是匹配的。更加详细的内容，查看 Object.hashCode() 文档)中的契约概述。


### 4. 转换操作(Transformations)

下面列出了Spark支持的一些常见转换函数。 有关详细信息，请参阅RDD API文档（Scala，Java，Python，R）和RDD函数doc（Scala，Java）。

#### 4.1 map(func) 映射
将函数应用于RDD中的每个元素，将返回值构成新的RDD。
```
List<String> aList = Lists.newArrayList("a", "B", "c", "b");
JavaRDD<String> rdd = sc.parallelize(aList);
// 小写转大写
JavaRDD<String> upperLinesRDD = rdd.map(new Function<String, String>() {
    @Override
    public String call(String str) throws Exception {
        if (StringUtils.isBlank(str)) {
            return str;
        }
        return str.toUpperCase();
    }
});
// A B C B
```

#### 4.2 filter(func) 过滤
返回通过选择func返回true的元素形成的新RDD。
```
List<String> list = Lists.newArrayList("a", "B", "c", "b");
JavaRDD<String> rdd = sc.parallelize(list);
// 只返回以a开头的字符串
JavaRDD<String> filterRDD = rdd.filter(new Function<String, Boolean>() {
    @Override
    public Boolean call(String str) throws Exception {
        return !str.startsWith("a");
    }
});
// B c b
```
#### 4.3 flatMap(func) 一行转多行
类似于map函数，但是每个输入项可以映射为0个输出项或更多输出项（所以func应该返回一个序列而不是一个条目）。
```
List<String> list = Lists.newArrayList("a 1", "B 2");
JavaRDD<String> rdd = sc.parallelize(list);
// 一行转多行 以空格分割
JavaRDD<String> resultRDD = rdd.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public Iterator<String> call(String s) throws Exception {
        if (StringUtils.isBlank(s)) {
            return null;
        }
        String[] array = s.split(" ");
        return Arrays.asList(array).iterator();
    }
});
// a
// 1
// B
// 2
```
#### 4.4 distinct([numTasks]))
去重
```
List<String> aList = Lists.newArrayList("1", "3", "2", "3");
JavaRDD<String> aRDD = sc.parallelize(aList);
// 去重
JavaRDD<String> rdd = aRDD.distinct(); // 1 2 3
```
#### 4.5 union(otherDataset) 并集
生成一个包含两个RDD中所有元素的RDD.　如果输入的RDD中有重复数据，union()操作也会包含这些重复的数据．
```
List<String> aList = Lists.newArrayList("1", "2", "3");
List<String> bList = Lists.newArrayList("3", "4", "5");
JavaRDD<String> aRDD = sc.parallelize(aList);
JavaRDD<String> bRDD = sc.parallelize(bList);
// 并集
JavaRDD<String> rdd = aRDD.union(bRDD); // 1 2 3 3 4 5
```

#### 4.6 intersection(otherDataset) 交集
求两个RDD共同的元素的RDD.　intersection()在运行时也会去掉所有重复的元素，尽管intersection()与union()的概念相似，但性能却差的很多，因为它需要通过网络混洗数据来发现共同的元素．
```
List<String> aList = Lists.newArrayList("1", "2", "3");
List<String> bList = Lists.newArrayList("3", "4", "5");
JavaRDD<String> aRDD = sc.parallelize(aList);
JavaRDD<String> bRDD = sc.parallelize(bList);
// 交集
JavaRDD<String> rdd = aRDD.intersection(bRDD); // 3
```

#### 4.7 subtract(otherDataset) 差集
subtract接受另一个RDD作为参数，返回一个由只存在第一个RDD中而不存在第二个RDD中的所有元素组成的RDD
```
List<String> aList = Lists.newArrayList("1", "2", "3");
List<String> bList = Lists.newArrayList("3", "4", "5");
JavaRDD<String> aRDD = sc.parallelize(aList);
JavaRDD<String> bRDD = sc.parallelize(bList);
// 差集
JavaRDD<String> rdd = aRDD.subtract(bRDD); // 1 2
```
#### 4.8 groupByKey 分组
根据键值对的key进行分组．对（K，V）键值对的数据集进行调用时，返回（K，Iterable <V>）键值对的数据集。

**注意**

如果分组是为了在每个key上执行聚合（如求总和或平均值），则使用reduceByKey或aggregateByKey会有更好的性能。

默认情况下，输出中的并行级别取决于父RDD的分区数。 可以设置可选参数numTasks来设置任务数量（By default, the level of parallelism in the output depends on the number of partitions of the parent RDD. You can pass an optional numTasks argument to set a different number of tasks.）。

```
Tuple2<String, Integer> t1 = new Tuple2<String, Integer>("Banana", 10);
Tuple2<String, Integer> t2 = new Tuple2<String, Integer>("Pear", 5);
Tuple2<String, Integer> t3 = new Tuple2<String, Integer>("Banana", 9);
Tuple2<String, Integer> t4 = new Tuple2<String, Integer>("Apple", 4);
List<Tuple2<String, Integer>> list = Lists.newArrayList();
list.add(t1);
list.add(t2);
list.add(t3);
list.add(t4);
JavaPairRDD<String, Integer> rdd = sc.parallelizePairs(list);
// 分组
JavaPairRDD<String, Iterable<Integer>> groupRDD = rdd.groupByKey();

// Apple --- 4
// Pear --- 5
// Banana --- 10 9
```

#### 4.9 reduceByKey(func, [numTasks]) 分组聚合

当在（K，V）键值对的数据集上调用时，返回（K，V）键值对的数据集，其中使用给定的reduce函数func聚合每个键的值，该函数类型必须是（V，V）=> V。

类似于groupByKey，可以通过设置可选的第二个参数来配置reduce任务的数量。

```
Tuple2<String, Integer> t1 = new Tuple2<String, Integer>("Banana", 10);
Tuple2<String, Integer> t2 = new Tuple2<String, Integer>("Pear", 5);
Tuple2<String, Integer> t3 = new Tuple2<String, Integer>("Banana", 9);
Tuple2<String, Integer> t4 = new Tuple2<String, Integer>("Apple", 4);
List<Tuple2<String, Integer>> list = Lists.newArrayList();
list.add(t1);
list.add(t2);
list.add(t3);
list.add(t4);
JavaPairRDD<String, Integer> rdd = sc.parallelizePairs(list);
// 分组计算
JavaPairRDD<String, Integer> reduceRDD = rdd.reduceByKey(new Function2<Integer, Integer, Integer>() {
    @Override
    public Integer call(Integer v1, Integer v2) throws Exception {
        return v1 + v2;
    }
});

// Apple --- 4
// Pear --- 5
// Banana --- 19
```

### 5. 动作操作 (Action)

下面列出了Spark支持的一些常见操作。


#### 5.1 reduce

接收一个函数作为参数，这个函数要操作两个相同元素类型的RDD并返回一个同样类型的新元素．

```
List<String> aList = Lists.newArrayList("aa", "bb", "cc", "dd");
JavaRDD<String> rdd = sc.parallelize(aList);
String result = rdd.reduce(new Function2<String, String, String>() {
    @Override
    public String call(String v1, String v2) throws Exception {
        return v1 + "#" + v2;
    }
});
System.out.println(result); // aa#bb#cc#dd
```
#### 5.2 collect

将整个RDD的内容返回．

```
List<String> list = Lists.newArrayList("aa", "bb", "cc", "dd");
JavaRDD<String> rdd = sc.parallelize(list);
List<String> collect = rdd.collect();
System.out.println(collect); // [aa, bb, cc, dd]
```
#### 5.3 take(n)

返回RDD中的n个元素，并且尝试只访问尽量少的分区，因此该操作会得到一个不均衡的集合．需要注意的是，这些操作返回元素的顺序与你的预期可能不一样．

```
List<String> list = Lists.newArrayList("aa", "bb", "cc", "dd");
JavaRDD<String> rdd = sc.parallelize(list);
List<String> collect = rdd.take(3);
System.out.println(collect); // [aa, bb, cc]
```
#### 5.5 takeSample

有时需要在驱动器程序中对我们的数据进行采样，takeSample(withReplacement, num, seed)函数可以让我们从数据中获取一个采样，并指定是否替换．

> Spark版本:2.3.0


原文：http://spark.apache.org/docs/2.3.0/rdd-programming-guide.html#rdd-operations
