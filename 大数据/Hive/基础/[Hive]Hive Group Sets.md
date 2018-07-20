---
layout: post
author: sjf0115
title: Hive Group Sets
date: 2018-07-05 19:16:01
tags:
  - Hive

categories: Hive
permalink: hive-base-group-sets
---

GROUPING SETS 子句是 SELECT 语句的 GROUP BY 子句的扩展。通过 GROUPING SETS 子句，您可采用多种方式对结果分组，而不必使用多个 SELECT 语句来实现这一目的。这就意味着，能够减少响应时间并提高性能。

```
20180627  adr uc  d918  30
20180627	adr	toutiao	d918	7
20180628	adr	uc	d918	15
20180628	adr	toutiao	d918	10
20180627	ios	uc	828b	16
20180628	ios	uc	828b	6
20180628	ios	toutiao	828b	18
20180627	adr	toutiao	cece	5
20180628	adr	toutiao	cece	8
20180627	ios	toutiao	6428	67
20180627	ios	uc	6428	22
20180627	adr	uc	e962	9
20180627	adr	uc	e962	8
20180628	ios	toutiao	953c	13
20180628	ios	toutiao	953c	7
20180627	adr	toutiao	f930	54
20180628	adr	toutiao	f930	8
20180627	adr	uc	f930	4
20180628	adr	uc	f930	40
20180627	adr	uc	2bfa	4
20180627	adr	toutiao	2bfa	7
20180628	adr	uc	2bfa	6
20180628	adr	toutiao	2bfa	9
20180627	adr	toutiao	2f3d	5
20180627	adr	toutiao	2f3d	4
20180628	ios	\N	2f3d	62
20180628	ios	\N	2f3d	34
20180628	\N	uc	5f02	12
20180628	\N	uc	5f02	14
20180628	ios	uc	f215	4
20180628	ios	toutiao	f215	16
```

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS tmp_read_pv (
  dt string,
  platform string,
  channel string,
  userName string,
  pv string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
LOCATION '/user/wirelessdev/tmp/data_group/example/input/read_pv/';
```

### 2. Example

```sql
SELECT a, b, SUM(c) FROM tab1 GROUP BY a, b GROUPING SETS ( (a,b) )
```
等价于:
```sql
SELECT a, b, SUM(c) FROM tab1 GROUP BY a, b
```

```sql
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS ( (a,b), a)
```
等价于:
```sql
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b
UNION
SELECT a, null, SUM( c ) FROM tab1 GROUP BY a
```

```sql
SELECT a,b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS (a,b)
```
等价于:
```sql
SELECT a, null, SUM( c ) FROM tab1 GROUP BY a
UNION
SELECT null, b, SUM( c ) FROM tab1 GROUP BY b
```

```sql
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS ( (a, b), a, b, ( ) )
```
等价于:
```sql
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b
UNION
SELECT a, null, SUM( c ) FROM tab1 GROUP BY a, null
UNION
SELECT null, b, SUM( c ) FROM tab1 GROUP BY null, b
UNION
SELECT null, null, SUM( c ) FROM tab1
```

### 3. Grouping__ID

GROUPING SETS会对GROUP BY子句中的列进行多维组合，结果整体展现，对于没有参与GROUP BY的那一列置为NULL值。如果列本身值就为NULL，则可能会发生冲突。这样我们就没有办法去区分该列显示的NULL值是列本身值就是NULL值，还是因为该列没有参与GROUP BY而被置为NULL值。所以需要一些方法来识别列中的NULL，GROUPING__ID函数就是解决方案。

此函数返回一个位向量，与每列是否存在对应。用二进制形式中的每一位来标示对应列是否参与GROUP BY，如果某一列参与了GROUP BY，对应位就被置为`1`，否则为`0`。这可以用于区分数据中是否存在空值。

> GROUPING__ID 的值与 GROUP BY 表达式中列的取值和顺序有关，所以如果重新排列 GROUPING__ID 对应的含义也会变化。

```sql
SELECT GROUPING__ID, dt, platform, channel, SUM(pv), COUNT(DISTINCT userName)
FROM tmp_read_pv
GROUP BY dt, platform, channel GROUPING SETS ( dt, (dt, platform), (dt, channel), (dt, platform, channel));
```

序号|GROUPING__ID|二进制|日期|平台|渠道|浏览量|用户数
---|---|---|---|---|---|---|---
1|1|100|20180627|NULL|NULL|242.0|8
2|1|100|20180628|NULL|NULL|282.0|9
3|3|110|20180627|adr|NULL|137.0|6
4|3|110|20180627|ios|NULL|105.0|2
5|3|110|20180628|NULL|NULL|26.0|1
6|3|110|20180628|adr|NULL|96.0|4
7|3|110|20180628|ios|NULL|160.0|4
8|5|101|20180627|NULL|toutiao|149.0|6
9|5|101|20180627|NULL|uc|93.0|6
10|5|101|20180628|NULL|NULL|96.0|1
11|5|101|20180628|NULL|toutiao|89.0|7
12|5|101|20180628|NULL|uc|97.0|6
13|7|111|20180627|adr|toutiao|82.0|5
14|7|111|20180627|adr|uc|55.0|4
15|7|111|20180627|ios|toutiao|67.0|1
16|7|111|20180627|ios|uc|38.0|2
17|7|111|20180628|NULL|uc|26.0|1
18|7|111|20180628|adr|toutiao|35.0|4
19|7|111|20180628|adr|uc|61.0|3
20|7|111|20180628|ios|NULL|96.0|1
21|7|111|20180628|ios|toutiao|54.0|3
22|7|111|20180628|ios|uc|10.0|2

例如上面的第5，10，17，20行所示，有些字段本身值就为NULL。

如果没有参与GROUP BY的列不显示NULL而是显示一个其他值（例如，`total`），对于列本身值没有为NULL的情况，可以使用如下简单方式来实现：
```

```

### 4. Grouping函数

Grouping函数用来表示GROUP BY子句中的表达式是否对给定行进行聚合。`0` 表示该列作为 GROUPING SETS 的一部分，而 `1` 表示该列不属于 GROUPING SETS 一部分，即没有使用该列对该行进行聚合。具体请看以下查询：
```

```





参考：https://stackoverflow.com/questions/29577887/grouping-in-hive

https://cwiki.apache.org/confluence/display/Hive/Enhanced+Aggregation%2C+Cube%2C+Grouping+and+Rollup
