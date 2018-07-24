---
layout: post
author: sjf0115
title: Hive ORC文件格式
date: 2018-07-18 12:22:01
tags:
  - Hive

categories: Hive
permalink: hive-base-orc-file-format
---

### 1. ORC文件格式

> 在Hive 0.11.0版本引入此功能

`ORC` 是 `Optimized Row Columnar` 的缩写，`ORC` 文件格式提供一种高效的方法来存储Hive数据。旨在解决其他Hive文件格式的局限。当Hive读取，写入和处理数据时，使用 `ORC` 文件格式可以提高性能。

例如，与 `RCFile` 文件格式相比，`ORC` 文件格式具有许多优点，例如：
- 每个任务输出文件只有一个，这样可以减轻 NameNode 的负载；
- 支持的Hive数据类型包括 datetime, decimal, 以及一些复杂类型(struct, list, map, union)；
- 存储在文件中的轻量级索引；
- 基于数据类型的块级别压缩：Integer类型的列用行程长度编码(Run Length Encoding)，String类型的列用字典编码(Dictionary Encoding)；
- 使用多个互相独立的RecordReaders并行读相同的文件；
- 无需扫描markers就可以分割文件；
- 绑定读写所需要的内存；
- 使用Protocol Buffers存储Metadata，可以支持添加和删除一些字段。

#### 1.1 文件结构

`ORC` 文件包含一组称为 `Stripe` 的行数据，以及 `file footer` 中的辅助信息。在文件的末尾，`postscript` 保存了压缩参数和压缩页脚的大小。

`Stripe` 默认大小为250MB。大的 `Stripe` 大小可实现HDFS的大量高效读取。

`file footer` 包含文件中的 `Stripe` 列表，每个 `Stripe` 有多少行以及每列的数据类型。还包了一些含列级聚合的计数，最小值，最大值以及总和。

下图说明了ORC文件结构：

![]()

#### 1.2 Stripe结构

从上图我们可以看出，每个 `Stripe` 都包含index data、row data以及stripe footer。Stripe footer包含流位置的目录；Row data在表扫描的时候会用到。
Index data包含每列的最大和最小值以及每列所在的行。行索引里面提供了偏移量，它可以跳到正确的压缩块位置。具有相对频繁的行索引，使得在stripe中快速读取的过程中可以跳过很多行，尽管这个stripe的大小很大。在默认情况下，最大可以跳过10000行。拥有通过过滤谓词而跳过大量的行的能力，你可以在表的 secondary keys 进行排序，从而可以大幅减少执行时间。比如你的表的主分区是交易日期，那么你可以对次分区（state、zip code以及last name）进行排序。

### 2. 语法

文件格式在表（或分区）级别指定。你可以使用HiveQL语句指定ORC文件格式，例如：
```sql
CREATE TABLE Addresses (
  name string,
  street string,
  city string,
  state string,
  zip int
)
STORED AS orc tblproperties ("orc.compress"="NONE");
```
除此之外，还可以为表指定压缩算法：
```sql
CREATE TABLE Addresses (
  name string,
  street string,
  city string,
  state string,
  zip int
)
STORED AS orc tblproperties ("orc.compress"="Zlib");
```
> 通常不需要设置压缩算法，因为Hive会设置默认的压缩算法 `hive.exec.orc.default.compress=ZLIB`。

我们通常的做法是将 HDFS 中的数据作为文本，在其上创建 Hive 外部表，然后将数据以 ORC 格式存储在Hive中：
```sql
CREATE TABLE Addresses_ORC STORED AS ORC AS SELECT * FROM Addresses_TEXT;
```

### 3. 高级设置

属性全部放在 `TBLPROPERTIES` 中。ORC具有通常不需要修改的属性。但是，对于特殊情况，你可以修改下表中列出的属性：

![]()

> 从Hive 0.14.0开始　`ALTER TABLE table_name [PARTITION partition_spec] CONCATENATE` 可用于将小的ORC文件合并为一个更大的文件。合并发生在 `Stripe` 级别，这可以避免对数据进行解压缩和解码。

参考：　https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC

https://www.iteblog.com/archives/1014.html#HiveORCFile
