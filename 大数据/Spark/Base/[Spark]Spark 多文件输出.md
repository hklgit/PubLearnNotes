---
layout: post
author: sjf0115
title: Spark 多文件输出
date: 2018-07-12 19:01:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-multiple-output-format
---

在[Hadoop 多文件输出MultipleOutputFormat](http://smartsi.club/2017/03/15/hadoop-base-multiple-output-format/)中我介绍了如何在Hadoop中根据Key或者Value的不同将属于不同的类型记录写到不同的文件中。在里面用到了MultipleOutputFormat这个类。

因为Spark内部写文件方式其实调用的都是Hadoop相关API，所以我们也可以通过Spark实现多文件输出。不过遗憾的是，Spark内部没有多文件输出的函数供大家直接调用，不过不用担心，我们可以轻松的实现这个功能。我们可以通过调用saveAsHadoopFile函数并自定义MultipleOutputFormat类即可，如下所示：
```java
package com.qunar.search.util;
import org.apache.hadoop.mapred.lib.MultipleTextOutputFormat;
/**
 * Spark 多文件输出
 * @author sjf0115
 * @Date Created in 下午4:09 18-7-12
 */
public class RDDMultipleTextOutputFormat<K, V> extends MultipleTextOutputFormat<K, V> {
    @Override
    protected String generateFileNameForKeyValue(K key, V value, String name) {
        return key.toString();
    }
}
JavaPairRDD<String, String> pairRDD = ...;
pairRDD.saveAsHadoopFile(outputPath, String.class, String.class, RDDMultipleTextOutputFormat.class);
```
RDDMultipleTextOutputFormat类中的 `generateFileNameForKeyValue` 函数有三个参数，key和value是RDD对应的Key和Value，而name参数是每个Reduce的编号。上面例子中没有使用该参数，而是直接将同一个Key的数据输出到同一个文件中：
```
[xiaosi@ying ~]$  sudo -uxiaosi hadoop fs -ls tmp/data_group/example/output/price
Found 3 items
-rw-r--r--   3 xiaosi xiaosi          0 2018-07-12 16:24 tmp/data_group/example/output/price/_SUCCESS
-rw-r--r--   3 xiaosi xiaosi     723754 2018-07-12 16:23 tmp/data_group/example/output/price/adr
-rw-r--r--   3 xiaosi xiaosi     799216 2018-07-12 16:23 tmp/data_group/example/output/price/ios
```
我们可以看到输出已经根据RDD的key将属于不同的类型记录写到不同的文件中，每个key对应一个文件，如果想每个key对应多个文件输出，如下代码所示：
```java
public class RDDMultipleTextOutputFormat<K, V> extends MultipleTextOutputFormat<K, V> {
    @Override
    protected String generateFileNameForKeyValue(K key, V value, String name) {
        return key.toString() + "/" + name;
    }
}
```
输出如下所示:
```
[xiaosi@ying ~]$  sudo -uxiaosi hadoop fs -ls tmp/data_group/example/output/price/
Found 3 items
-rw-r--r--   3 xiaosi xiaosi          0 2018-07-16 10:00 tmp/data_group/example/output/price/_SUCCESS
drwxr-xr-x   - xiaosi xiaosi          0 2018-07-16 10:00 tmp/data_group/example/output/price/adr
drwxr-xr-x   - xiaosi xiaosi          0 2018-07-16 10:00 tmp/data_group/example/output/price/ios
[xiaosi@ying ~]$
[xiaosi@ying ~]$  sudo -uxiaosi hadoop fs -ls tmp/data_group/example/output/price/adr/
Found 2 items
-rw-r--r--   3 xiaosi xiaosi 23835 2018-07-16 10:00 tmp/data_group/example/output/price/adr/part-00000
-rw-r--r--   3 xiaosi xiaosi      22972 2018-07-16 10:00 tmp/data_group/example/output/price/adr/part-00001
```



























https://www.iteblog.com/archives/1281.html
https://gist.github.com/silasdavis/d1d1f1f7ab78249af462
https://stackoverflow.com/questions/23995040/write-to-multiple-outputs-by-key-spark-one-spark-job
