一些转换(例如，join，coGroup，keyBy，groupBy)要求在一组元素上定义一个key。其他转换(Reduce，GroupReduce，Aggregate，Windows)允许在使用这些函数之前对数据进行分组。

一个DataSet进行分组如下:
```java
DataSet<...> input = // [...]
DataSet<...> reduced = input.groupBy(/*define key here*/).reduceGroup(/*do something*/);
```
而使用DataStream可以指定一个key:
```java
DataStream<...> input = // [...]
DataStream<...> windowed = input.keyBy(/*define key here*/).window(/*window specification*/);
```
Flink的数据模型不是基于键值对。因此，你不需要将数据集类型物理上打包成keys和values。keys是"虚拟"：它们只是被定义在实际数据之上的函数，以指导分组算子使用。

备注:
```
在下面的讨论中，我们将使用DataStream API和keyBy。对于DataSet API，你只需要替换为DataSet和groupBy即可。
```

下面介绍几种Flink定义keys方法。

### 1. 为Tuples类型定义keys

最简单的情况是在元组的一个或多个字段上对元组进行分组：

下面是在元组的第一个字段(整数类型)上进行分组。

Java版本:
```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0)
```

Scala版本:
```
val input: DataStream[(Int, String, Long)] = // [...]
val keyed = input.keyBy(0)
```
下面，我们将在复合key上对元组进行分组，复合key包含元组的第一个和第二个字段:

Java版本:
```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1)
```

Scala版本:
```
val input: DataSet[(Int, String, Long)] = // [...]
val grouped = input.groupBy(0,1)
```

如果你有一个包含嵌套元组的DataStream，例如：
```
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
```
如果指定keyBy(0)，则使用整个Tuple2作为key(以Integer和Float为key)。如果要到嵌套的Tuple2的某个字段中，则必须使用下面说明的字段表达式指定keys。


### 2. 使用字段表达式定义keys

你可以使用基于字符串的字段表达式来引用嵌套字段以及定义用于分组，排序，连接或coGrouping的key。

字段表达式可以非常容易地选择(嵌套)复合类型(如Tuple和POJO类型)中的字段。

在下面的例子中，我们有一个WC POJO，它有两个字段`word`和`count`。如果想通过字段`word`分组，我们只需将`word`传递给keyBy()函数即可。

```Java
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
```

字段表达式语法:

(1) 按其字段名称选择POJO字段。例如，`user`是指POJO类型的`user`字段。

(2) 通过字段名称或0到字段索引偏移量之间的数值选择元组字段(field name or 0-offset field index)。例如，`f0`和`5`分别指Java元组类型的第一和第六字段。

(3) 你可以在POJO和元组中选择嵌套字段。例如，`user.zip`是指存储在POJO类型的`user`字段中的POJO类型的`zip`字段。支持POJO和Tuples的任意嵌套和混合，如`f1.user.zip`或`user.f3.1.zip`。

(4) 你可以使用`*`通配符表达式选择完整类型。这也适用于不是元组或POJO类型的类型。

Example:
```Java
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
```
下面是上述示例代码的有效字段表达式：
```
count：WC类中的count字段。
complex：递归地选择复合字段POJO类型ComplexNestedClass的所有字段。
complex.word.f2：选择嵌套字段Tuple3的最后一个字段。
complex.hadoopCitizen：选择Hadoop IntWritable类型。
```

### 3. 使用key选择器函数定义keys

定义key的另一种方法是`key选择器`函数。key选择器函数将单个元素作为输入，并返回元素的key。key可以是任何类型的。

以下示例显示了一个key选择器函数，它只返回一个对象的字段：

Java版本:
```java
public class WC {
  public String word; public int count;
}

DataStream<WC> words = // [...]
KeyedStream<WC> kyed = words.keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
});
```

Scala版本:
```
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val keyed = words.keyBy( _.word )
```

备注:
```
Flink版本:1.3
```


原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/api_concepts.html#specifying-keys











ddd
