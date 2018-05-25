### 1. java.lang.NoClassDefFoundError: org/apache/spark/Logging


#### 1.1 问题
```
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/spark/Logging
    at java.lang.ClassLoader.defineClass1(Native Method)
    at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    at java.net.URLClassLoader.defineClass(URLClassLoader.java:467)
    at java.net.URLClassLoader.access$100(URLClassLoader.java:73)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:368)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:362)
    at java.security.AccessController.doPrivileged(Native Method)I
    at java.net.URLClassLoader.findClass(URLClassLoader.java:361)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    at org.apache.spark.streaming.twitter.TwitterUtils$.createStream(TwitterUtils.scala:44)
    at TwitterStreamingApp$.main(TwitterStreamingApp.scala:42)
    at TwitterStreamingApp.main(TwitterStreamingApp.scala)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:729)
    at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:185)
    at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:210)
    at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:124)
    at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.ClassNotFoundException: org.apache.spark.Logging
    at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    ... 23 more
```
#### 1.2 解决方案
先看一下我们的Maven依赖：
```
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.1.0</version>
</dependency>

<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming-kafka_2.11</artifactId>
    <version>1.6.3</version>
</dependency>
```
我们看到我们Spark版本为2.1.0版本，然而在spark 1.5.2版本之后 org/apache/spark/Logging 已经被移除了。
由于spark-streaming-kafka 1.6.3版本中使用到了logging，所以我们替换一个依赖：

```
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming-kafka-0-8_2.11</artifactId>
    <version>2.1.0</version>
</dependency>
```


### 2. Task not serializable
#### 2.1 问题描述
```
Exception in thread "main" org.apache.spark.SparkException: Task not serializable
	at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:298)
	at org.apache.spark.util.ClosureCleaner$.org$apache$spark$util$ClosureCleaner$$clean(ClosureCleaner.scala:288)
	at org.apache.spark.util.ClosureCleaner$.clean(ClosureCleaner.scala:108)
	at org.apache.spark.SparkContext.clean(SparkContext.scala:2094)
	at org.apache.spark.streaming.dstream.DStream$$anonfun$map$1.apply(DStream.scala:546)
	at org.apache.spark.streaming.dstream.DStream$$anonfun$map$1.apply(DStream.scala:546)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:151)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:112)
	at org.apache.spark.SparkContext.withScope(SparkContext.scala:701)
	at org.apache.spark.streaming.StreamingContext.withScope(StreamingContext.scala:264)
	at org.apache.spark.streaming.dstream.DStream.map(DStream.scala:545)
	at org.apache.spark.streaming.api.java.JavaDStreamLike$class.map(JavaDStreamLike.scala:157)
	at org.apache.spark.streaming.api.java.AbstractJavaDStreamLike.map(JavaDStreamLike.scala:42)
	at com.sjf.open.spark.stream.SparkStreamTest.receiveKafkaData(SparkStreamTest.java:76)
	at com.sjf.open.spark.stream.SparkStreamTest.startStream(SparkStreamTest.java:71)
	at com.sjf.open.spark.stream.SparkStreamTest.main(SparkStreamTest.java:48)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
Caused by: java.io.NotSerializableException: com.sjf.open.spark.stream.SparkStreamTest
Serialization stack:
	- object not serializable (class: com.sjf.open.spark.stream.SparkStreamTest, value: com.sjf.open.spark.stream.SparkStreamTest@1e6cc850)
	- element of array (index: 0)
	- array (class [Ljava.lang.Object;, size 1)
	- field (class: java.lang.invoke.SerializedLambda, name: capturedArgs, type: class [Ljava.lang.Object;)
	- object (class java.lang.invoke.SerializedLambda, SerializedLambda[capturingClass=class com.sjf.open.spark.stream.SparkStreamTest, functionalInterfaceMethod=org/apache/spark/api/java/function/Function.call:(Ljava/lang/Object;)Ljava/lang/Object;, implementation=invokeSpecial com/sjf/open/spark/stream/SparkStreamTest.handleMessage:(Lscala/Tuple2;)Ljava/lang/String;, instantiatedMethodType=(Lscala/Tuple2;)Ljava/lang/String;, numCaptured=1])
	- writeReplace data (class: java.lang.invoke.SerializedLambda)
	- object (class com.sjf.open.spark.stream.SparkStreamTest$$Lambda$8/1868987089, com.sjf.open.spark.stream.SparkStreamTest$$Lambda$8/1868987089@3c0036b)
	- field (class: org.apache.spark.api.java.JavaPairRDD$$anonfun$toScalaFunction$1, name: fun$1, type: interface org.apache.spark.api.java.function.Function)
	- object (class org.apache.spark.api.java.JavaPairRDD$$anonfun$toScalaFunction$1, <function1>)
	at org.apache.spark.serializer.SerializationDebugger$.improveException(SerializationDebugger.scala:40)
	at org.apache.spark.serializer.JavaSerializationStream.writeObject(JavaSerializer.scala:46)
	at org.apache.spark.serializer.JavaSerializerInstance.serialize(JavaSerializer.scala:100)
	at org.apache.spark.util.ClosureCleaner$.ensureSerializable(ClosureCleaner.scala:295)
	... 20 more
```

#### 2.2 解决方案

发现我们的SparkStreamTest类没有进行序列化：
```
public class SparkStreamTest{
    ...
}
```
修改为：
```
public class SparkStreamTest implements Serializable{
    ...
}
```

### 3.java.lang.NoSuchMethodError: com.fasterxml.jackson.core.JsonFactory.requiresPropertyOrdering()Z

#### 3.1 问题
```
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at org.apache.spark.metrics.MetricsSystem$$anonfun$registerSinks$1.apply(MetricsSystem.scala:200)
	at org.apache.spark.metrics.MetricsSystem$$anonfun$registerSinks$1.apply(MetricsSystem.scala:194)
	at scala.collection.mutable.HashMap$$anonfun$foreach$1.apply(HashMap.scala:99)
	at scala.collection.mutable.HashMap$$anonfun$foreach$1.apply(HashMap.scala:99)
	at scala.collection.mutable.HashTable$class.foreachEntry(HashTable.scala:230)
	at scala.collection.mutable.HashMap.foreachEntry(HashMap.scala:40)
	at scala.collection.mutable.HashMap.foreach(HashMap.scala:99)
	at org.apache.spark.metrics.MetricsSystem.registerSinks(MetricsSystem.scala:194)
	at org.apache.spark.metrics.MetricsSystem.start(MetricsSystem.scala:102)
	at org.apache.spark.SparkContext.<init>(SparkContext.scala:522)
	at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:2313)
	at org.apache.spark.sql.SparkSession$Builder$$anonfun$6.apply(SparkSession.scala:868)
	at org.apache.spark.sql.SparkSession$Builder$$anonfun$6.apply(SparkSession.scala:860)
	at scala.Option.getOrElse(Option.scala:121)
	at org.apache.spark.sql.SparkSession$Builder.getOrCreate(SparkSession.scala:860)
	at com.sjf.open.spark.JavaWordCount.test1(JavaWordCount.java:37)
	at com.sjf.open.spark.JavaWordCount.main(JavaWordCount.java:96)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
Caused by: java.lang.NoSuchMethodError: com.fasterxml.jackson.core.JsonFactory.requiresPropertyOrdering()Z
	at com.fasterxml.jackson.databind.ObjectMapper.<init>(ObjectMapper.java:537)
	at com.fasterxml.jackson.databind.ObjectMapper.<init>(ObjectMapper.java:448)
	at org.apache.spark.metrics.sink.MetricsServlet.<init>(MetricsServlet.scala:48)
	... 26 more
```

#### 3.2 解决方案

引入的ES包中也包含com.fasterxml.jackson.core版本为2.8.1
，版本较高，找不到那个方法．
所以需要显示指明com.fasterxml.jackson.core版本:
```
<jackson.version>2.6.5</jackson.version>
<!-- jackson spark sql 与 es 冲突　手动指定-->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>${jackson.version}</version>
</dependency>
```

### 4. Unable to instantiate SparkSession with Hive support because Hive classes are not found

#### 4.1 问题描述

在使用Spark 实现对Hive的支持时
```
String warehouseLocation = "spark-warehouse";
SparkSession session = SparkSession.builder()
        .appName("Java Spark Hive Example")
        .config("spark.sql.warehouse.dir", warehouseLocation)
        .enableHiveSupport()
        .getOrCreate();
session.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)");
session.sql("LOAD DATA LOCAL INPATH 'SparkDemo/src/main/resources/kv1.txt' INTO TABLE src");

session.sql("SELECT * FROM src").show();
```
报如下异常：
```
Exception in thread "main" java.lang.IllegalArgumentException: Unable to instantiate SparkSession with Hive support because Hive classes are not found.
	at org.apache.spark.sql.SparkSession$Builder.enableHiveSupport(SparkSession.scala:815)
	at com.sjf.open.sql.DataSourceDemo.hiveTable(DataSourceDemo.java:104)
	at com.sjf.open.sql.DataSourceDemo.main(DataSourceDemo.java:114)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
```
#### 4.2 解决方案

添加如下依赖：
```
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-hive_2.11</artifactId>
    <version>2.1.1</version>
    <scope>provided</scope>
</dependency>
```

### 5. loadTable is not implemented

#### 5.1　问题描述

```
Exception in thread "main" java.lang.UnsupportedOperationException: loadTable is not implemented
	at org.apache.spark.sql.catalyst.catalog.InMemoryCatalog.loadTable(InMemoryCatalog.scala:316)
	at org.apache.spark.sql.catalyst.catalog.SessionCatalog.loadTable(SessionCatalog.scala:319)
	at org.apache.spark.sql.execution.command.LoadDataCommand.run(tables.scala:302)
	at org.apache.spark.sql.execution.command.ExecutedCommandExec.sideEffectResult$lzycompute(commands.scala:58)
	at org.apache.spark.sql.execution.command.ExecutedCommandExec.sideEffectResult(commands.scala:56)
	at org.apache.spark.sql.execution.command.ExecutedCommandExec.doExecute(commands.scala:74)
	at org.apache.spark.sql.execution.SparkPlan$$anonfun$execute$1.apply(SparkPlan.scala:114)
	at org.apache.spark.sql.execution.SparkPlan$$anonfun$execute$1.apply(SparkPlan.scala:114)
	at org.apache.spark.sql.execution.SparkPlan$$anonfun$executeQuery$1.apply(SparkPlan.scala:135)
	at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:151)
	at org.apache.spark.sql.execution.SparkPlan.executeQuery(SparkPlan.scala:132)
	at org.apache.spark.sql.execution.SparkPlan.execute(SparkPlan.scala:113)
	at org.apache.spark.sql.execution.QueryExecution.toRdd$lzycompute(QueryExecution.scala:87)
	at org.apache.spark.sql.execution.QueryExecution.toRdd(QueryExecution.scala:87)
	at org.apache.spark.sql.Dataset.<init>(Dataset.scala:185)
	at org.apache.spark.sql.Dataset$.ofRows(Dataset.scala:64)
	at org.apache.spark.sql.SparkSession.sql(SparkSession.scala:592)
	at com.sjf.open.sql.DataSourceDemo.hiveTable(DataSourceDemo.java:107)
	at com.sjf.open.sql.DataSourceDemo.main(DataSourceDemo.java:114)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
```

#### 5.2 解决方案

### 6. kafka.common.MessageSizeTooLargeException

#### 6.1 问题描述
```
Reported error Error handling message; exiting - kafka.common.MessageSizeTooLargeException: Found a message larger than the maximum fetch size of this consumer on topic logs.logger.piao.mp_piao partition 4 at fetch offset 18397184. Increase the fetch size, or decrease the maximum message size the broker will allow.
```
#### 6.2 问题解决












....
