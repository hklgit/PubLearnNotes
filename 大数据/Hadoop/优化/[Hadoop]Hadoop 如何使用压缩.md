---
layout: post
author: sjf0115
title: Hadoop 如何使用压缩
date: 2018-04-23 19:32:17
tags:
  - Hadoop
  - Hadoop 优化

categories: Hadoop
permalink: hadoop-how-to-use-compression
---

就如上一篇文章介绍的那样，如果输入文件是压缩文件，当 MapReduce 程序读取压缩文件时，根据文件名的后缀来选择 codes，输入文件自动解压缩（我们不需要指定压缩文件是哪一种压缩格式）。

下面我们列出了一些代码，为 Hadoop 中常用的压缩格式设置输出压缩。

### 1. 常用压缩格式
#### 1.1 Gzip

对于最终输出，我们可以使用FileOutputFormat上的静态方便方法来设置属性：
```java
FileOutputFormat.setCompressOutput(job, true);
FileOutputFormat.setOutputCompressorClass(job, GzipCodec,class);
```
或者
```java
Configuration conf = new Configuration();
conf.setBoolean("mapred.output.compress", true);
conf.setClass("mapred.output.compression.codec", GzipCodec.class, CompressionCodec.class);
```
对于 Map 输出：
```java
Configuration conf = new Configuration();
conf.setBoolean("mapred.compress.map.output",true);
conf.setClass("mapred.map.output.compression.codec", GzipCodec.class, CompressionCodec.class);
Job job = Job.getInstance(conf);
```

#### 1.2 LZO

对于最终输出：
```java
FileOutputFormat.setCompressOutput(conf, true);
FileOutputFormat.setOutputCompressorClass(conf, LzoCodec.class);
```
> 为了使LZO可分割，我们需要生成一个LZO索引文件。

对于 Map 输出：
```java
Configuration conf = new Configuration();
conf.setBoolean("mapred.compress.map.output",true);
conf.setClass("mapred.map.output.compression.codec", LzoCodec.class, CompressionCodec.class);
Job job = Job.getInstance(conf);
```

#### 1.3 Snappy

对于最终输出：
```java
conf.setOutputFormat(SequenceFileOutputFormat.class);
SequenceFileOutputFormat.setOutputCompressionType(conf, CompressionType.BLOCK);
SequenceFileOutputFormat.setCompressOutput(conf, true);
conf.set("mapred.output.compression.codec","org.apache.hadoop.io.compress.SnappyCodec");
```
对于Map输出：
```java
Configuration conf = new Configuration();
conf.setBoolean("mapred.compress.map.output", true);
conf.set("mapred.map.output.compression.codec","org.apache.hadoop.io.compress.SnappyCodec");
```
### 2. 实验与结果

文件系统计数器用于分析实验结果。以下是典型的内置文件系统计数器。





原文：http://comphadoop.weebly.com/how-to-use-compression.html
