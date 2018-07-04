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

```
SELECT a, b, SUM(c) FROM tab1 GROUP BY a, b GROUPING SETS ( (a,b) )
```
等价于:
```
SELECT a, b, SUM(c) FROM tab1 GROUP BY a, b
```

```
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS ( (a,b), a)
```
等价于:
```
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b
UNION
SELECT a, null, SUM( c ) FROM tab1 GROUP BY a
```

```
SELECT a,b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS (a,b)
```
等价于:
```
SELECT a, null, SUM( c ) FROM tab1 GROUP BY a
UNION
SELECT null, b, SUM( c ) FROM tab1 GROUP BY b
```

```
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b GROUPING SETS ( (a, b), a, b, ( ) )
```
等价于:
```
SELECT a, b, SUM( c ) FROM tab1 GROUP BY a, b
UNION
SELECT a, null, SUM( c ) FROM tab1 GROUP BY a, null
UNION
SELECT null, b, SUM( c ) FROM tab1 GROUP BY null, b
UNION
SELECT null, null, SUM( c ) FROM tab1
```

### 3. Grouping__ID

```sql
SELECT GROUPING__ID, dt, platform, channel, SUM(pv), COUNT(DISTINCT userName)
FROM tmp_read_pv
GROUP BY dt, platform, channel GROUPING SETS ( dt, (dt, platform), (dt, channel), (dt, platform, channel));
```

GROUPING__ID|日期|平台|浏览量|用户数
---|---|---|---|---
1|20180627|NULL|NULL|242.0|8
5|20180627|NULL|toutiao|149.0|6
5|20180627|NULL|uc|93.0|6
3|20180627|adr|NULL|137.0|6
7|20180627|adr|toutiao|82.0|5
7|20180627|adr|uc|55.0|4
3|20180627|ios|NULL|105.0|2
7|20180627|ios|toutiao|67.0|1
7|20180627|ios|uc|38.0|2
1|20180628|NULL|NULL|282.0|9
3|20180628|NULL|NULL|26.0|1
5|20180628|NULL|NULL|96.0|1
5|20180628|NULL|toutiao|89.0|7
5|20180628|NULL|uc|97.0|6
7|20180628|NULL|uc|26.0|1
3|20180628|adr|NULL|96.0|4
7|20180628|adr|toutiao|35.0|4
7|20180628|adr|uc|61.0|3
3|20180628|ios|NULL|160.0|4
7|20180628|ios|NULL|96.0|1
7|20180628|ios|toutiao|54.0|3
7|20180628|ios|uc|10.0|2

当我们没有统计某一列时，它的值显示为NULL，这可能与列本身就有NULL值冲突，这就需要一种方法区分是没有统计还是值本来就是NULL。

#### 简单方式

如果数据中本身的值没有为NULL，我们是可以使用如下方式：
```

```

#### GROUPING__ID

如果数据中本身的值有为NULL的，这就与我们没有统计这一列而造成的NULL形成冲突，我们可以借助GROUPING__ID如下统计：

```sql
SELECT GROUPING__ID, dt, platform, channel, SUM(pv), COUNT(DISTINCT userName)
FROM tmp_read_pv
GROUP BY dt, platform, channel GROUPING SETS ( dt, (dt, platform), (dt, channel), (dt, platform, channel));
```

GROUPING__ID|类型|日期|平台|浏览量|用户数
---|---|---|---|---|---
1|dt|20180627|NULL|NULL|242.0|8
1|dt|20180628|NULL|NULL|282.0|9
3|dt_platform|20180628|NULL|NULL|26.0|1
3|dt_platform|20180628|ios|NULL|160.0|4
3|dt_platform|20180628|adr|NULL|96.0|4
3|dt_platform|20180627|ios|NULL|105.0|2
3|dt_platform|20180627|adr|NULL|137.0|6
5|dt_channel|20180628|NULL|toutiao|89.0|7
5|dt_channel|20180628|NULL|NULL|96.0|1
5|dt_channel|20180627|NULL|uc|93.0|6
5|dt_channel|20180627|NULL|toutiao|149.0|6
5|dt_channel|20180628|NULL|uc|97.0|6
7|dt_platform_channel|20180628|NULL|uc|26.0|1
7|dt_platform_channel|20180628|ios|NULL|96.0|1
7|dt_platform_channel|20180628|ios|toutiao|54.0|3
7|dt_platform_channel|20180627|ios|uc|38.0|2
7|dt_platform_channel|20180627|adr|uc|55.0|4
7|dt_platform_channel|20180627|adr|toutiao|82.0|5
7|dt_platform_channel|20180628|ios|uc|10.0|2
7|dt_platform_channel|20180628|adr|uc|61.0|3
7|dt_platform_channel|20180628|adr|toutiao|35.0|4
7|dt_platform_channel|20180627|ios|toutiao|67.0|1

GROUPING__ID 的值与 GROUP BY 表达式中列的取值和顺序有关，所以如果重新排列 GROUPING__ID 对应的含义也会变化。
```sql
SELECT GROUPING__ID, dt, platform, channel, SUM(pv), COUNT(DISTINCT userName)
FROM tmp_read_pv
GROUP BY dt, platform, channel GROUPING SETS ( dt, (dt, platform), (dt, channel), (dt, platform, channel));
```

日期|平台|渠道|GROUPING__ID|二进制
---|---|---|---|---
20180628 |ios |uc |7 | 1 1 1
NULL |ios |uc |6 | 0 1 1
NULL |NULL |uc |4 | 0 0 1
20180628 |ios |NULL |3 | 1 1 0
NULL |ios |NULL |2 | 0 1 0
20180628 |NULL |NULL |1 | 1 0 0





参考：https://stackoverflow.com/questions/29577887/grouping-in-hive
