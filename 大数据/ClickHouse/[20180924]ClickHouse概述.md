---
layout: post
author: sjf0115
title: ClickHouse概述
date: 2018-09-24 20:16:01
tags:
  - ClickHouse

categories: ClickHouse
permalink: what-is-clickhouse
---

`ClickHouse` 是一个面向列的数据库管理系统（DBMS），用于在线分析处理查询（OLAP）。

在`正常`的面向行的DBMS中，数据按以下顺序存储：

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/ClickHouse/what-is-clickhouse-1.png?raw=true)

换句话说，与一行相关的所有值都物理存储在相邻的位置。面向行的 DBMS 的示例是 MySQL，Postgres 和 MS SQL Server。在面向列的DBMS中，数据存储如下：

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/ClickHouse/what-is-clickhouse-2.png?raw=true)

这些示例仅显示数据的排列顺序。不同列的值分别存储，同一列的数据一起存储。面向列的 DBMS 的示例有：`Vertica`，`Paraccel`（Actian Matrix和Amazon Redshift），`Sybase IQ`，`Exasol`，`Infobright`，`InfiniDB`，`MonetDB`（VectorWise和Actian Vector），`LucidDB`，`SAP HANA`，`Google Dremel`，`Google PowerDrill`，`Druid`和`KDB+`。

不同的存储顺序适合于不同的应用场景。数据访问场景是指执行什么样查询，查询频率，每种类型的查询读取多少数据 - 行，列和字节；读取和更新数据之间的关系；数据的大小以及它在本地的使用情况；是否使用事务，以及它们是如何隔离的；数据副本和逻辑完整性；每种类型的查询的延迟和吞吐量等。

系统上的负载越高，场景化的系统定制就越重要，定制化就越具体。没有一个系统能够适用于不同的应用场景。如果一个系统可以适应多种场景，那么在高负载情况下，系统处理所有场景表现都会很差，或者仅其中一种场景表现良好。

对于OLAP（联机分析处理）场景的主要特点：
- 绝大多数请求都是读请求。
- 更新时每次更新相当大的批次（> 1000行），而不是更新一行；或者根本不更新。
- 数据添加到数据库后基本不怎么修改。
- 对于读请求，会从数据库中提取大量的数据，但只读取一小部分列（一个小的子集）。
- `宽`表意味着它包含很多列。
- 查询相对较少（每台服务器通常只有数百个查询或更少）。
- 对于简单查询，允许大约50ms的延迟。
- 列值比较小 - 数字和短字符串（例如，每个URL 60个字节）。
- 处理单个查询时需要高吞吐量（每台服务器每秒高达数十亿行）。
- 不需要事务。
- 对数据一致性要求低
- 每个查询都有一个大表，其他所有表都是小表。
- 查询结果显著小于源数据。也就是说，数据被过滤或聚合。结果可以放在单个服务器的内存中。

很容易看出，OLAP场景与其他常见场景（如OLTP或Key-Value访问）有很大不同。所以，如果你想获得不错的表现，尝试使用OLTP或Key-Value数据库来处理分析查询是没有任何意义的。例如，如果你尝试使用 MongoDB 或 Redis 进行分析，与OLAP数据库相比，你的性能会很差。

### 2. 为什么面向列的数据库更适合OLAP场景

面向列的数据库更适合于OLAP场景：对于大多数查询，处理速度至少提高了100倍。原因在下面详细解释，但事实上更容易在视觉上展示：

面向行的数据库：
![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/ClickHouse/what-is-clickhouse-3.gif?raw=true)

面向列的数据库：
![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/ClickHouse/what-is-clickhouse-4.gif?raw=true)

看到不同了？

#### 2.1 输入/输出

- 对于分析查询，只需要读取少量的列。在面向列的数据库中，你可以只读取所需的数据。例如，如果你需要100列中的5列数据，则I/O可能会减少20倍。
- 由于数据是以数据包的形式读取的，因此压缩比较容易。列中的数据也更容易压缩。这进一步减少了I/O量。
- 由于I/O的减少，更多的数据可以装载进系统缓存中。

例如，查询'统计每个广告平台的记录数量'需要读取一个'广告平台ID'列，未压缩时占用1个字节。如果大多数流量不是来自广告平台，那么你可以期望至少有10倍的压缩比。当使用快速压缩算法时，数据解压缩速度可以达到每秒解压缩至少几千兆字节的未压缩数据。换句话说，这个查询可以在一台服务器上以每秒大约几十亿行的速度处理。这个速度实际上是在实践中是容易实现的。

Example:
```
$ clickhouse-client
ClickHouse client version 0.0.52053.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.52053.

:) SELECT CounterID, count() FROM hits GROUP BY CounterID ORDER BY count() DESC LIMIT 20
```
输出：

CounterID|count()
---|---
114208|56057344
115080 | 51619590
3228 | 44658301
38230 | 42045932
145263 | 42042158
91244 | 38297270
154139 | 26647572
150748 | 24112755
242232 | 21302571
338158 | 13507087
62180 | 12229491
82264 | 12187441
232261 | 12148031
146272 | 11438516
168777 | 11403636
4120072 | 11227824
10938808 | 10519739
74088 |  9047015
115079 |  8837972
337234 |  8205961

看一下花费的时间：
```
20 rows in set. Elapsed: 0.153 sec. Processed 1.00 billion rows, 4.00 GB (6.53 billion rows/s., 26.10 GB/s.
```

#### 2.2 CPU

由于执行查询需要处理大量的行，因此它有助于为整个向量而不是单独的行指调度所有操作，或者实现查询引擎，以便没有调度成本。如果你不这样做，任何半象限的磁盘子系统，查询解释器不可避免地中断CPU。将数据存储在列中并在可能的情况下按列处理是有意义的。

有两种方法可以做到这一点：
- 向量引擎。所有的操作都是以向量形式写入，而不是单独的值。这意味着你不需要频繁调用操作，调度成本可以忽略不计。操作代码包含一个优化的内部循环。
- 代码生成。为查询生成的代码具有所有的间接调用。

这不是可以在'普通''数据库中完成的，因为运行简单查询没有任何意义。但是，也有例外。例如，MemSQL 在处理SQL查询时使用代码生成来减少延迟。（为了比较，分析性DBMS需要优化吞吐量，而不是延迟。）

请注意，为了提高CPU效率，查询语言必须是声明式的（SQL或MDX），或者至少是一个向量`（J，K）`。查询应该只包含隐式循环，以便优化。

原文：https://clickhouse.yandex/docs/en/
