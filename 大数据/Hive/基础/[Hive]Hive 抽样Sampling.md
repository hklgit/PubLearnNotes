---
layout: post
author: sjf0115
title: Hive 抽样Sampling
date: 2018-05-12 19:16:01
tags:
  - Hive

categories: Hive
permalink: hive-base-how-to-use-sampling
---

### 1. Block 抽样

Block 抽样功能在 Hive 0.8 版本开始引入。具体参阅JIRA - [Input Sampling By Splits](https://issues.apache.org/jira/browse/HIVE-2121)

```
block_sample: TABLESAMPLE (n PERCENT)
```
该语句允许至少抽取 n% 大小的数据（注意：不是行数，而是数据大小）做为输入，仅支持 CombineHiveInputFormat ，不能够处理一些特殊的压缩格式。如果抽样失败，MapReduce 作业的输入将是整个表或者是分区的数据。由于在 HDFS 块级别进行抽样，所以抽样粒度为块大小。例如如果块大小为 256MB，即使 n% 的输入仅为100MB，那也会得到 256MB 的数据。

在下面例子查询中输入大小的 0.1% 或更多：
```sql
SELECT *
FROM source TABLESAMPLE(0.1 PERCENT) s;
```
如果希望在不同的块中抽取相同大小的数据，可以改变下面的参数：
```
set hive.sample.seednumber=<INTEGER>;
```
或者可以指定要读取的总长度，但与 PERCENT 抽样具有相同的限制。（从Hive 0.10.0开始 - https://issues.apache.org/jira/browse/HIVE-3401）
```
block_sample: TABLESAMPLE (ByteLengthLiteral)

ByteLengthLiteral : (Digit)+ ('b' | 'B' | 'k' | 'K' | 'm' | 'M' | 'g' | 'G')
```
在下面例子查询中输入大小为 100M 或更多：
```sql
SELECT *
FROM source TABLESAMPLE(100M) s;
```
Hive 还支持按行数对输入进行限制，但它与上述两种行为不同。首先，它不需要 `CombineHiveInputFormat`，这意味着这可以在non-native表上使用。其次，用户给定的行数应用到每个 split 上。 因此总行数还取决于输入 split 的个数（不同 split 个数得到的总行数也会不一样）。（从Hive 0.10.0开始 - https://issues.apache.org/jira/browse/HIVE-3401）
```
block_sample: TABLESAMPLE (n ROWS)
```
例如，以下查询将从每个输入 split 中取前10行：
```sql
SELECT * FROM source TABLESAMPLE(10 ROWS);
```

### 2.

```
table_sample: TABLESAMPLE (BUCKET x OUT OF y [ON colname])
```

TABLESAMPLE子句允许用户编写用于数据抽样而不是整个表的查询，该子句出现FROM子句中，可用于任何表中。桶编号从1开始，colname表明抽取样本的列，可以是非分区列中的任意一列，或者使用rand()表明在整个行中抽取样本而不是单个列。在colname上分桶的行随机进入1到y个桶中，返回属于桶x的行。下面的例子中，返回32个桶中的第3个桶中的行：





原文:https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Sampling
