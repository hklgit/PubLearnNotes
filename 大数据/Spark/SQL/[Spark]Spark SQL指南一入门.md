### 1. 概述

Spark SQL是用于结构化数据处理的Spark模块。 与基本的Spark RDD API不同，Spark SQL提供的接口为Spark提供了有关数据和正在执行计算的结构更多信息。在内部，Spark SQL使用这些额外的信息执行额外的优化。 Spark提供了几种与Spark SQL进行交互的方法，包括`SQL`和`Dataset API`。 当使用相同的执行引擎计算结果时，与你用来表达计算的API/语言无关(When computing a result the same execution engine is used, independent of which API/language you are using to express the computation. )。 这种统一意味着开发人员可以轻松地在不同API之间来回切换，从而提供了表达给定转换操作最自然的方式．


#### 1.1 SQL

`Spark SQL`的一个用途是执行SQL查询。 `Spark SQL`也可以从现有已安装的Hive中读取数据。 有关如何配置此功能的更多信息，请参阅`Hive Tables`部分。当使用另一种编程语言运行SQL时，结果将以`Dataset`/`DataFrame`返回。你还可以使用命令行或`JDBC`/`ODBC`与SQL接口进行交互。

#### 1.2 Datasets 与 DataFrames

`Dataset`是分布式数据集合。 `Dataset`是Spark 1.6中新增加的一个接口，它提供了RDD的优势（强类型，使用强大的lambda函数的功能），并具有Spark SQL优化的执行引擎的优点(provides the benefits of RDDs　with the benefits of Spark SQL’s optimized execution engine)。可以从JVM对象构建`Dataset`，然后使用转换操作函数（map，flatMap，filter等）进行操作。`Dataset` API 可用于Scala和Java，但 Python不支持`Dataset` API。但是由于Python是动态语言，`Dataset` API的许多优点已经在Python实现了（例如，可以通过名称访问行的字段row.columnName）。R的情况类似。

`DataFrame`是一个以命名列方式组织的分布式数据集。 它在概念上等同于关系数据库中的一个表或R/Python中的`data frames`，但是进行了更多优化。 `DataFrames`可以从各种各样的源构建，例如：结构化数据文件，Hive中的表，外部数据库或现有RDD。 DataFrame API在Scala，Java，Python和R中可用。在Scala和Java中，`DataFrame`是行的数据集。 在Scala API中，DataFrame只是`Dataset[Row]`的一个类型别名。 而在Java API中，用户需要使用`Dataset<Row>`来表示DataFrame。

Throughout this document, we will often refer to Scala/Java Datasets of Rows as DataFrames.

### 2. 入门

#### 2.1 SparkSession

Spark中所有功能的入口点是`SparkSession`类。 要创建基本的`SparkSession`，只需使用`SparkSession.builder（）`：

Java版本：
```
import org.apache.spark.sql.SparkSession;

SparkSession session = SparkSession.builder()
    .appName("Java Spark SQL basic example")
    .config("spark.master", "local")
    .getOrCreate();
```
Scala版本：
```
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()

// For implicit conversions like converting RDDs to DataFrames
import spark.implicits._
```
Python版本：
```
from pyspark.sql import SparkSession

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL basic example") \
    .config("spark.some.config.option", "some-value") \
    .getOrCreate()
```

Spark 2.0中的SparkSession为Hive功能提供内置支持，包括使用HiveQL编写查询，访问Hive UDF以及从Hive表读取数据的功能。 要使用这些功能，你没有必要安装Hive。

#### 2.2 创建DataFrames

使用SparkSession，应用程序可以从现有的RDD，Hive表或Spark数据源创建DataFrames。

作为示例，以下将基于JSON文件的内容创建DataFrame：

Json文件内容：
```
{"name":"Michael"}
{"name":"Andy", "age":30}
{"name":"Justin", "age":19}
```
创建DataFrames：
```
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

Dataset<Row> df = session.read().json("SparkDemo/src/main/resources/people.json");
df.show();

/*+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+*/
```
#### 2.3 DataFrame操作

`DataFrames`为`Scala`，`Java`，`Python`和`R`中的结构化数据操作提供了一种特定领域的语言(DSL)。

如上所述，在Spark 2.0中，DataFrames只是Scala和Java API中的行数据集(DataSet of Rows)。 与来自强类型Scala/Java数据集的“类型转换`typed transformations`”相反，这些操作也称为“无类型转换`untyped transformations`”(These operations are also referred as “untyped transformations” in contrast to “typed transformations” come with strongly typed Scala/Java Datasets.)。

这里我们列举了使用Datasets的结构化数据处理的一些基本示例：
```
Dataset<Row> dataFrame = session.read().json("SparkDemo/src/main/resources/people.json");

// 以树格式输出范式
dataFrame.printSchema();
// 只选择name列数据
dataFrame.select("name").show();
// 选择name age列 age加一
dataFrame.select(col("name"), col("age").plus(1)).show();
// 过滤age大于21岁
dataFrame.filter(col("age").gt(21)).show();
// 按age分组求人数
dataFrame.groupBy("age").count().show();
```
对应输出结果：
```
(1)
root
    |-- age: long (nullable = true)
    |-- name: string (nullable = true)

(2)
+-------+
|   name|
+-------+
|Michael|
|   Andy|
| Justin|
+-------+
   
(3)
+-------+---------+
|   name|(age + 1)|
+-------+---------+
|Michael|     null|
|   Andy|       31|
| Justin|       20|
+-------+---------+

(4)
+---+----+
|age|name|
+---+----+
| 30|Andy|
+---+----+

(5)
+----+-----+
| age|count|
+----+-----+
|  19|    1|
|null|    1|
|  30|    1|
+----+-----+
```
有关可在数据集上执行的操作类型的完整列表，请参阅[API文档](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/Dataset.html)。

除了简单的列引用和表达式，Datasets还具有丰富的函数库，包括字符串操作，日期算术，常用的数学运算等。 完整列表可参阅在[DataFrame函数参考](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$)。

#### 2.4 编程方式运行SQL查询

`SparkSession`上的sql函数使应用程序能以编程方式运行SQL查询，并将结果以`DataSet<Row>`返回。

```
// 创建DataFrame
Dataset<Row> dataFrame = session.read().json("SparkDemo/src/main/resources/people.json");
// 将DataFrame注册为SQL临时视图
dataFrame.createOrReplaceTempView("people");
// 使用SQL查询数据
Dataset<Row> sqlDataFrame = session.sql("SELECT name, age FROM people where age < 30");
// 输出结果
sqlDataFrame.show();
```
输出结果：
```
+------+---+
|  name|age|
+------+---+
|Justin| 19|
+------+---+
```

#### 2.5 全局临时视图

`Spark SQL`中的临时视图是会话范围的，如果创建它的会话终止，临时视图也将消失。 如果要在所有会话之间共享临时视图，并保持活动状态直到Spark应用程序终止，你可以创建一个全局临时视图。全局临时视图与系统保留的数据库`global_temp`相关联，我们必须使用限定名称来引用它。 

例如：
```
SELECT * FROM global_temp.view1。
```

```
// 创建DataFrame
Dataset<Row> dataFrame = session.read().json("SparkDemo/src/main/resources/people.json");
// 将DataFrame注册为全局临时视图
dataFrame.createGlobalTempView("people");
// 全局临时视图与系统保留的数据库global_temp有关
Dataset<Row> sqlDataFrame = session.sql("SELECT name, age FROM global_temp.people where age < 30");
// 输出结果
sqlDataFrame.show();

// 全局临时视图是跨会话的
Dataset<Row> sqlDataFrame2 = session.newSession().sql("SELECT name, age FROM global_temp.people where age < 30");
sqlDataFrame2.show();
```
输出结:
```
+------+---+
|  name|age|
+------+---+
|Justin| 19|
+------+---+

+------+---+
|  name|age|
+------+---+
|Justin| 19|
+------+---+
```

#### 2.6 创建数据集DataSet

`DataSet`与`RDD`类似，但是，不是使用Java序列化或Kryo，而是使用专门的编码器来序列化对象以进行网络处理或传输。虽然编码器和标准序列化都将可以将对象转换成字节，但是编码器是动态生成的代码(code generated dynamically)，并使用一种让Spark可以执行多个操作（如过滤，排序和散列），而无需将字节反序列化对象的格式(encoders are code generated dynamically and use a format that allows Spark to perform many operations like filtering, sorting and hashing without deserializing the bytes back into an object.)。

```
import java.io.Serializable;

public class Person implements Serializable{
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

```
import java.util.Arrays;
import java.util.Collections;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;

// 创建Person对象
Person person = new Person();
person.setName("Andy");
person.setAge(32);

// 根据Java Bean创建Encoders
Encoder<Person> personEncoder = Encoders.bean(Person.class);
Dataset<Person> dataSet = session.createDataset(
    Collections.singletonList(person),
    personEncoder
);
dataSet.show();

// 大多数常见类型的编码器都在Encoders中提供
Encoder<Integer> integerEncoder = Encoders.INT();
Dataset<Integer> primitiveDS = session.createDataset(Arrays.asList(1, 2, 3), integerEncoder);
Dataset<Integer> transformedDS = primitiveDS.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer call(Integer value) throws Exception {
        return value + 1;
    }
}, integerEncoder);
transformedDS.collect(); // Returns [2, 3, 4]

// DataFrames可以通过提供的类来转换为DataSet 基于名称映射
// 创建DataFrame
Dataset<Row> dataFrame = session.read().json("SparkDemo/src/main/resources/people.json");
// DataFrame转换为DataSet
Dataset<Person> peopleDS = dataFrame.as(personEncoder);
peopleDS.show();
```


#### 2.7 与RDD交互

Spark SQL支持两种不同的方法将现有`RDD`转换为`Datasets`。
- 第一种方法使用反射来推断包含特定类型对象的RDD的模式。 当您在编写Spark应用程序时，您已经知道架构，这种基于反射的方法会导致更简洁的代码，并且运行良好。
- 第二种方法是通过编程接口来创建`DataSet`，允许构建一个范式，并将其应用到现有的RDD。虽然此方法更详细，但构造Datasets时，直到运行时才知道列及其类型( While this method is more verbose, it allows you to construct Datasets when the columns and their types are not known until runtime.)。


##### 2.7.1 使用反射推导范式

`Spark SQ`L支持自动将JavaBeans的RDD转换为`DataFrame`。 使用反射获取的Bean信息定义了表的范式。 目前为止，`Spark SQL`还不支持包含Map字段的JavaBean。 但是支持嵌套的JavaBeans和List或Array字段。你可以通过创建一个实现`Serializable`的类并为其所有字段设置`getter`和`setter`方法来创建一个JavaBean。

```
// 从文本文件创建一个Person对象的RDD
JavaRDD<Person> peopleRDD = session.read().textFile(textPath).javaRDD().map(new Function<String, Person>() {
    @Override
    public Person call(String line) throws Exception {
        String[] parts = line.split(",");
        Person person = new Person();
        person.setName(parts[0]);
        if(parts.length > 1){
            person.setAge(Integer.parseInt(parts[1].trim()));
        }
        return person;
    }
});

// 将范式应用到JavaBeans的RDD上获取DataFrame
Dataset<Row> peopleDataFrame = session.createDataFrame(peopleRDD, Person.class);
// 将DataFrame注册为临时视图
peopleDataFrame.createOrReplaceTempView("people");
// 使用由spark提供的sql方法来运行SQL语句
Dataset<Row> teenagersDataFrame = session.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19");

// 可以通过字段索引访问结果行的列
Encoder<String> stringEncoder = Encoders.STRING();
Dataset<String> teenagerNamesByIndexDF = teenagersDataFrame.map(new MapFunction<Row, String>() {
    @Override
    public String call(Row row) throws Exception {
        return "Name: " + row.getString(0);
    }
}, stringEncoder);
teenagerNamesByIndexDF.show();

// 可以通过字段名称访问结果行的列
Dataset<String> teenagerNamesByFieldDF = teenagersDataFrame.map(new MapFunction<Row, String>() {
    @Override
    public String call(Row row) throws Exception {
        return "Name: " + row.<String>getAs("name");
    }
}, stringEncoder);
teenagerNamesByFieldDF.show();
```
输出结果:
```
+------------+
|       value|
+------------+
|Name: Justin|
+------------+

+------------+
|       value|
+------------+
|Name: Justin|
+------------+
```

##### 2.7.2 使用编程接口指定范式

当JavaBean类不能提前定义时（例如，记录的结构以字符串编码，或者文本数据集将被解析，对于不同的用户来说，字段的投影方式不同），可以通过编程方式创建`DataSet<Row>`有如下三个步骤：
- 从原始RDD(例如，JavaRDD<String>)创建`Rows`的RDD(JavaRDD<Row>);
- 创建由StructType表示的范式，与步骤1中创建的RDD中的Rows结构相匹配。
- 通过SparkSession提供的`createDataFrame`方法将范式应用于行的RDD。


```
import java.util.ArrayList;
import java.util.List;

import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.function.Function;

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

// 创建RDD
JavaRDD<String> peopleRDD = session.sparkContext().textFile(textPath, 1).toJavaRDD();
// 在字符串中编码范式
String schemaString = "name age";

// 根据字符串定义的范式生成范式
List<StructField> fields = new ArrayList<>();
for (String fieldName : schemaString.split(" ")) {
    StructField field = DataTypes.createStructField(fieldName, DataTypes.StringType, true);
    fields.add(field);
}
StructType schema = DataTypes.createStructType(fields);

// 将RDD(people)的记录转换为Rows
JavaRDD<Row> rowRDD = peopleRDD.map(new Function<String, Row>() {
    @Override
    public Row call(String record) throws Exception {
        String[] attributes = record.split(",");
        return RowFactory.create(attributes[0], attributes[1].trim());
    }
});

// 将范式应用到RDD
Dataset<Row> peopleDataFrame = session.createDataFrame(rowRDD, schema);
// 将DataFrame注册为SQL临时视图
peopleDataFrame.createOrReplaceTempView("people");

// SQL可以在使用DataFrames创建的临时视图中运行
Dataset<Row> results = session.sql("SELECT name FROM people");

// SQL查询的结果是DataFrames，并支持所有正常的RDD操作
// 结果中的行的列可以通过字段索引或字段名称访问
Dataset<String> namesDS = results.map(new MapFunction<Row, String>() {
    @Override
    public String call(Row row) throws Exception {
        return "Name: " + row.getString(0);
    }
}, Encoders.STRING());
namesDS.show();
```
输出结果：
```
+-------------+
|        value|
+-------------+
|Name: Michael|
|   Name: Andy|
| Name: Justin|
+-------------+
```


#### 2.8 聚合

内置的`DataFrames`函数提供了常用的聚合函数，例如count()，countDistinct()，avg()，max()，min()等。虽然这些函数是为DataFrames设计的，但Spark SQL也具有类型安全的版本，在Scala和Java中其中有一些使用在强类型数据集上(Spark SQL also has type-safe versions for some of them in Scala and Java to work with strongly typed Datasets)。此外，用户不必限于预定义的聚合函数，可以自己创建。

##### 2.8.1 非类型安全的用户自定义聚合函数

自定义聚合函数
```
import org.apache.spark.sql.Row;
import org.apache.spark.sql.expressions.MutableAggregationBuffer;
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction;
import org.apache.spark.sql.types.DataType;
import org.apache.spark.sql.types.DataTypes;
import org.apache.spark.sql.types.StructField;
import org.apache.spark.sql.types.StructType;

import java.util.ArrayList;
import java.util.List;

public class MyAverage extends UserDefinedAggregateFunction{

    private StructType inputSchema;
    private StructType bufferSchema;

    public MyAverage() {
        List<StructField> inputFields = new ArrayList<>();
        inputFields.add(DataTypes.createStructField("inputColumn", DataTypes.LongType, true));
        inputSchema = DataTypes.createStructType(inputFields);

        List<StructField> bufferFields = new ArrayList<>();
        bufferFields.add(DataTypes.createStructField("sum", DataTypes.LongType, true));
        bufferFields.add(DataTypes.createStructField("count", DataTypes.LongType, true));
        bufferSchema = DataTypes.createStructType(bufferFields);
    }

    // 聚合函数输入参数数据类型
    @Override
    public StructType inputSchema() {
        return inputSchema;
    }

    // 聚合函数缓冲数据的数据类型
    @Override
    public StructType bufferSchema() {
        return bufferSchema;
    }

    // 返回值的数据类型
    @Override
    public DataType dataType() {
        return DataTypes.DoubleType;
    }

    // 这个函数是否总是在相同的输入上返回相同的输出
    @Override
    public boolean deterministic() {
        return true;
    }

    // 初始化聚合缓冲区 缓冲区本身是一个Row
    // 除了提供标准方法　如在索引中检索值(例如 get() getBoolean())外
    // 还提供了更新其值的机会　缓冲区内的数组和映射仍然是不可变的
    @Override
    public void initialize(MutableAggregationBuffer buffer) {
        buffer.update(0, 0L);
        buffer.update(1, 0L);
    }

    // 使用input的新输入数据更新给定的聚合缓冲区buffer
    @Override
    public void update(MutableAggregationBuffer buffer, Row input) {
        if (!input.isNullAt(0)) {
            long updatedSum = buffer.getLong(0) + input.getLong(0);
            long updatedCount = buffer.getLong(1) + 1;
            buffer.update(0, updatedSum);
            buffer.update(1, updatedCount);
        }
    }

    // 合并两个聚合缓冲区并将更新的缓冲区值存储回buffer1
    @Override
    public void merge(MutableAggregationBuffer buffer1, Row buffer2) {
        long mergedSum = buffer1.getLong(0) + buffer2.getLong(0);
        long mergedCount = buffer1.getLong(1) + buffer2.getLong(1);
        buffer1.update(0, mergedSum);
        buffer1.update(1, mergedCount);
    }

    // 计算最终结果
    @Override
    public Object evaluate(Row buffer) {
        return ((double) buffer.getLong(0)) / buffer.getLong(1);
    }
}

```

使用：
```
// 注册自定义聚合函数
session.udf().register("myAverage", new MyAverage());
// 创建DataFrame
Dataset<Row> df = session.read().json(employeePath);
// 将DataFrame注册为SQL临时视图
df.createOrReplaceTempView("employees");
df.show();

Dataset<Row> result = session.sql("SELECT myAverage(salary) as average_salary FROM employees");
result.show();
```
输出结果：
```
+-------+------+
|   name|salary|
+-------+------+
|Michael|  3000|
|   Andy|  4500|
| Justin|  3500|
|  Berta|  4000|
+-------+------+

+--------------+
|average_salary|
+--------------+
|        3750.0|
+--------------+
```

##### 2.8.2 类型安全的用户自定义聚合函数

强类型数据集的用户自定义聚合围绕`Aggregator`抽象类开展工作。 例如，类型安全的用户自定义的平均值聚合函数可以如下所示：

Average：
```
package com.sjf.open.model;

import java.io.Serializable;

/**
 * Created by xiaosi on 17-6-7.
 */
public class Average implements Serializable{

    private long sum;
    private long count;

    public Average(){

    }

    public Average(long sum, long count) {
        this.sum = sum;
        this.count = count;
    }

    public long getSum() {
        return sum;
    }

    public void setSum(long sum) {
        this.sum = sum;
    }

    public long getCount() {
        return count;
    }

    public void setCount(long count) {
        this.count = count;
    }
}

```
Employee：
```
package com.sjf.open.model;

import java.io.Serializable;

/**
 * Created by xiaosi on 17-6-7.
 */
public class Employee implements Serializable{

    private String name;
    private long salary;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public long getSalary() {
        return salary;
    }

    public void setSalary(long salary) {
        this.salary = salary;
    }
}

```
自定义聚合函数MyAverage2：
```
package com.sjf.open.model;

import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.expressions.Aggregator;

/**
 * Created by xiaosi on 17-6-7.
 */
public class MyAverage2 extends Aggregator<Employee, Average, Double>{

    // 聚合的零值 应满足b + zero = b的属性
    @Override
    public Average zero() {
        return new Average(0L, 0L);
    }

    // 组合两个值以产生一个新值 为了性能,函数可以修改buffer并返回,而不是构造一个新的对象
    @Override
    public Average reduce(Average buffer, Employee employee) {
        long newSum = buffer.getSum() + employee.getSalary();
        long newCount = buffer.getCount() + 1;
        buffer.setSum(newSum);
        buffer.setCount(newCount);
        return buffer;
    }

    // 合并两个中间值
    @Override
    public Average merge(Average b1, Average b2) {
        long mergedSum = b1.getSum() + b2.getSum();
        long mergedCount = b1.getCount() + b2.getCount();
        b1.setSum(mergedSum);
        b1.setCount(mergedCount);
        return b1;
    }

    // 转换聚合的输出
    @Override
    public Double finish(Average reduction) {
        return ((double) reduction.getSum()) / reduction.getCount();
    }

    // 指定中间值类型的编码器
    @Override
    public Encoder<Average> bufferEncoder() {
        return Encoders.bean(Average.class);
    }

    // 指定最终输出值类型的编码器
    @Override
    public Encoder<Double> outputEncoder() {
        return Encoders.DOUBLE();
    }
}

```
使用：
```
Encoder<Employee> employeeEncoder = Encoders.bean(Employee.class);
Dataset<Employee> ds = session.read().json(employeePath).as(employeeEncoder);
ds.show();

MyAverage2 myAverage2 = new MyAverage2();
// 将函数转换为TypedColumn 并赋予一个名称
TypedColumn<Employee, Double> averageSalary = myAverage2.toColumn().name("average_salary");
Dataset<Double> result = ds.select(averageSalary);
result.show();
```

输出结果：
```
+-------+------+
|   name|salary|
+-------+------+
|Michael|  3000|
|   Andy|  4500|
| Justin|  3500|
|  Berta|  4000|
+-------+------+

+--------------+
|average_salary|
+--------------+
|        3750.0|
+--------------+
```



原文：http://spark.apache.org/docs/latest/sql-programming-guide.html#overview

