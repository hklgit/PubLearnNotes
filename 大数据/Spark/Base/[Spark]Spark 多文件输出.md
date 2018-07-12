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

```
[xiaosi@ying ~]$  sudo -uxiaosi hadoop fs -ls tmp/data_group/order_feature/
Found 3 items
-rw-r--r--   3 xiaosi xiaosi          0 2018-07-12 16:24 tmp/data_group/order_feature/_SUCCESS
-rw-r--r--   3 xiaosi xiaosi     723754 2018-07-12 16:23 tmp/data_group/order_feature/have_order
-rw-r--r--   3 xiaosi xiaosi     799216 2018-07-12 16:23 tmp/data_group/order_feature/no_order
```




























https://www.iteblog.com/archives/1281.html
https://gist.github.com/silasdavis/d1d1f1f7ab78249af462
https://stackoverflow.com/questions/23995040/write-to-multiple-outputs-by-key-spark-one-spark-job
