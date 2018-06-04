---
layout: post
author: sjf0115
title: Spark 如何使用累加器Accumulator
date: 2018-06-04 20:28:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-how-to-use-accumulator
---

Accumulator 是 spark 提供的累加器，累加器可以用来实现计数器（如在 MapReduce 中）或者求和。Spark 本身支持数字类型的累加器，程序员可以添加对新类型的支持。

### 1. 内置累加器

在 Spark2.0.0 版本之前，我们可以通过调用 `SparkContext.intAccumulator()` 或 `SparkContext.doubleAccumulator()` 来创建一个 Int 或 Double 类型的累加器：
```java
Accumulator<Double> doubleAccumulator = sparkContext.doubleAccumulator(0.0, "Double Accumulator");
Accumulator<Integer> intAccumulator = sparkContext.intAccumulator(0, "Int Accumulator");
Accumulator<Double> doubleAccumulator2 = sparkContext.accumulator(0.0, "Double Accumulator 2");
Accumulator<Integer> intAccumulator2 = sparkContext.accumulator(0, "Int Accumulator 2");java
```

在 Spark２.0.0 之后的版本中，之前的的 Accumulator 已被废除，用 AccumulatorV2 代替:
```
@deprecated("use AccumulatorV2", "2.0.0")
class Accumulator[T] private[spark] (
    // SI-8813: This must explicitly be a private val, or else scala 2.11 doesn't compile
    @transient private val initialValue: T,
    param: AccumulatorParam[T],
    name: Option[String] = None,
    countFailedValues: Boolean = false)
  extends Accumulable[T, T](initialValue, param, name, countFailedValues)

// Int
@deprecated("use sc().longAccumulator(String)", "2.0.0")
def intAccumulator(initialValue: Int, name: String): Accumulator[java.lang.Integer] =
  sc.accumulator(initialValue, name)(IntAccumulatorParam)
    .asInstanceOf[Accumulator[java.lang.Integer]]

@deprecated("use sc().longAccumulator(String)", "2.0.0")
def accumulator(initialValue: Int, name: String): Accumulator[java.lang.Integer] =
  intAccumulator(initialValue, name)

// Double
@deprecated("use sc().doubleAccumulator(String)", "2.0.0")
def doubleAccumulator(initialValue: Double, name: String): Accumulator[java.lang.Double] =
  sc.accumulator(initialValue, name)(DoubleAccumulatorParam)
    .asInstanceOf[Accumulator[java.lang.Double]]

@deprecated("use sc().doubleAccumulator(String)", "2.0.0")
def accumulator(initialValue: Double, name: String): Accumulator[java.lang.Double] =
  doubleAccumulator(initialValue, name)    
```
我们可以通过调用 `sparkContext.sc().longAccumulator()` 或 `sparkContext.sc().doubleAccumulator()` 来创建一个 Long 或 Double 类型的累加器：
```java
DoubleAccumulator doubleAccumulator = sparkContext.sc().doubleAccumulator("Double Accumulator");
LongAccumulator longAccumulator = sparkContext.sc().longAccumulator("Long Accumulator");
```
看一下这两个方法具体的实现：
```java
/**
 * Create and register a long accumulator, which starts with 0 and accumulates inputs by `add`.
 */
def longAccumulator: LongAccumulator = {
  val acc = new LongAccumulator
  register(acc)
  acc
}

/**
 * Create and register a double accumulator, which starts with 0 and accumulates inputs by `add`.
 */
def doubleAccumulator: DoubleAccumulator = {
  val acc = new DoubleAccumulator
  register(acc)
  acc
}
```

通过源码我们知道分别通过创建 `LongAccumulator` 和 `DoubleAccumulator` 对象，然后进行注册来创建一个累加器。所以我们也可以使用如下方式创建一个Long类型的累加器：
```java
LongAccumulator longAccumulator = new LongAccumulator();
sparkContext.sc().register(longAccumulator, "Long Accumulator");
```
> LongAccumulator DoubleAccumulator 都继承自 AccumulatorV2

Spark内置了数值型累加器(例如，Long，Double类型)，我们还可以通过继承 AccumulatorV2 来创建我们自己类型的累加器。

### 2. 自定义累加器

自定义累加器类型的功能在 1.x 版本中就已经提供了，但是使用起来比较麻烦，在 Spark 2.0.0 版本后，累加器的易用性有了较大的改进，而且官方还提供了一个新的抽象类：AccumulatorV2 来提供更加友好的自定义类型累加器的实现方式。官方同时给出了一个实现的示例：CollectionAccumulator，这个类允许以集合的形式收集 Spark 应用执行过程中的一些信息。例如，我们可以用这个类收集 Spark 处理数据过程中的非法数据或者引起异常的异常数据，这对我们处理异常时很有帮助。当然，由于累加器的值最终要汇聚到 Driver 端，为了避免 Driver 端的出现 OOM，需要收集的数据规模不宜过大。

实现自定义类型累加器需要继承 AccumulatorV2 并覆盖下面几个方法：
- reset 将累加器重置为零
- add 将另一个值添加到累加器中
- merge 将另一个相同类型的累加器合并到该累加器中。

下面这个累加器可以用于在程序运行过程中收集一些异常或者非法数据，最终以 `List[String]` 的形式返回：
```java
package com.sjf.open.spark;

import com.google.common.collect.Lists;
import org.apache.spark.util.AccumulatorV2;

import java.util.ArrayList;
import java.util.List;

/**
 * 自定义累加器 CollectionAccumulator
 * @author sjf0115
 * @Date Created in 下午2:11 18-6-4
 */
public class CollectionAccumulator<T> extends AccumulatorV2<T, List<T>> {

    private List<T> list = Lists.newArrayList();

    @Override
    public boolean isZero() {
        return list.isEmpty();
    }

    @Override
    public AccumulatorV2<T, List<T>> copy() {
        CollectionAccumulator<T> accumulator = new CollectionAccumulator<>();
        synchronized (accumulator) {
            accumulator.list.addAll(list);
        }
        return accumulator;
    }

    @Override
    public void reset() {
        list.clear();
    }

    @Override
    public void add(T v) {
        list.add(v);
    }

    @Override
    public void merge(AccumulatorV2<T, List<T>> other) {
        if(other instanceof CollectionAccumulator){
            list.addAll(((CollectionAccumulator) other).list);
        }
        else {
            throw new UnsupportedOperationException("Cannot merge " + this.getClass().getName() + " with " + other.getClass().getName());
        }
    }

    @Override
    public List<T> value() {
        return new ArrayList<>(list);
    }
}
```
下面我们在数据处理过程中收集非法坐标为例，来看一下我们自定义的累加器如何使用:
```java
package com.sjf.open.spark;

import com.google.common.collect.Lists;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.VoidFunction;

import java.io.Serializable;
import java.util.List;

/**
 * 自定义累加器示例
 * @author sjf0115
 * @Date Created in 下午2:11 18-6-4
 */
public class CustomAccumulatorExample implements Serializable{

    public static void main(String[] args) {
        String appName = "CustomAccumulatorExample";
        SparkConf conf = new SparkConf().setAppName(appName);
        JavaSparkContext sparkContext = new JavaSparkContext(conf);

        List<String> list = Lists.newArrayList();
        list.add("27.34832,111.32135");
        list.add("34.88478,185.17841");
        list.add("39.92378,119.50802");
        list.add("94,119.50802");

        CollectionAccumulator<String> collectionAccumulator = new CollectionAccumulator<>();
        sparkContext.sc().register(collectionAccumulator, "Illegal Coordinates");
        // 原始坐标
        JavaRDD<String> sourceRDD = sparkContext.parallelize(list);
        // 过滤非法坐标
        JavaRDD<String> resultRDD = sourceRDD.filter(new Function<String, Boolean>() {
            @Override
            public Boolean call(String str) throws Exception {
                String[] coordinate = str.split(",");
                double lat = Double.parseDouble(coordinate[0]);
                double lon = Double.parseDouble(coordinate[1]);
                if(Math.abs(lat) > 90 || Math.abs(lon) > 180){
                    collectionAccumulator.add(str);
                    return true;
                }
                return false;
            }
        });
        // 输出
        resultRDD.foreach(new VoidFunction<String>() {
            @Override
            public void call(String coordinate) throws Exception {
                System.out.println("[Data]" + coordinate);
            }
        });
        // 查看异常坐标
        for (String coordinate : collectionAccumulator.value()) {
            System.out.println("[Illegal]: " + coordinate);
        }
    }

}
```
结果输出:
```
[Illegal]: 94,119.50802
[Illegal]: 34.88478,185.17841
```

### 3. 累加器注意事项

累加器只能在 action 中进行更新，Spark 会保证每个任务对累加器只更新一次，即使重新启动的任务也不会重新更新该值。在 transformations 操作中，我们需要知道，如果任务或作业 stage 被重新执行，每个任务的更新操作可能会执行多次。

累加器不会改变 Spark 的懒加载（Lazy）执行模型。如果它们在 RDD 上的某个操作中进行更新，那么只有在 RDD 作为 action 操作的一部分进行计算时才会对它们进行更新。因此，在一个像 map() 这样的 transformation 操作时，累加器的更新并没有执行。

```java
package com.sjf.open.spark;

import com.google.common.collect.Lists;
import org.apache.spark.Accumulator;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.VoidFunction;
import org.apache.spark.util.CollectionAccumulator;
import org.apache.spark.util.DoubleAccumulator;
import org.apache.spark.util.LongAccumulator;

import java.io.Serializable;
import java.util.List;

/**
 * 累加器陷阱
 * @author sjf0115
 * @Date Created in 下午2:11 18-6-4
 */
public class AccumulatorTrap implements Serializable{

    public static void main(String[] args) {
        String appName = "AccumulatorTrap";
        SparkConf conf = new SparkConf().setAppName(appName);
        JavaSparkContext sparkContext = new JavaSparkContext(conf);

        LongAccumulator evenAccumulator = sparkContext.sc().longAccumulator("Even Num Accumulator");
        LongAccumulator oddAccumulator = sparkContext.sc().longAccumulator("Odd Num Accumulator");

        /*LongAccumulator evenAccumulator = new LongAccumulator();
        LongAccumulator oddAccumulator = new LongAccumulator();

        sparkContext.sc().register(evenAccumulator, "Even Num Accumulator");
        sparkContext.sc().register(oddAccumulator, "Odd Num Accumulator");*/

        List<Integer> numList = Lists.newArrayList();
        for(int i = 0;i < 10;i++){
            numList.add(i);
        }
        JavaRDD<Integer> numRDD = sparkContext.parallelize(numList);

        // transform
        JavaRDD<Integer> resultRDD = numRDD.map(new Function<Integer, Integer>() {
            @Override
            public Integer call(Integer num) throws Exception {
                if (num % 2 == 0) {
                    evenAccumulator.add(1L);
                    return 0;
                } else {
                    oddAccumulator.add(1L);
                    return 1;
                }
            }
        });

        // the first action
        resultRDD.count();
        System.out.println("Odd Num Count : " + oddAccumulator.value());
        System.out.println("Even Num Count : " + evenAccumulator.value());

        // the second action
        resultRDD.foreach(new VoidFunction<Integer>() {
            @Override
            public void call(Integer num) throws Exception {
                System.out.println(num);
            }
        });
        System.out.println("Odd Num Count : " + oddAccumulator.value());
        System.out.println("Even Num Count : " + evenAccumulator.value());
    }
}
```





















....
