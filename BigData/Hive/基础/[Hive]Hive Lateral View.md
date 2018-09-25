### 1. 语法
```
lateralView: LATERAL VIEW udtf(expression) tableAlias AS columnAlias (',' columnAlias)*
fromClause: FROM baseTable (lateralView)*
```
### 2. 描述

Lateral View一般与用户自定义表生成函数（如explode()）结合使用。 如[内置表生成函数](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF#LanguageManualUDF-Built-inTable-GeneratingFunctions(UDTF))中所述，UDTF为每个输入行生成零个或多个输出行。 Lateral View 首先将UDTF应用于基表的每一行，然后将结果输出行连接到输入行，以形成具有提供的表别名的虚拟表。

```
在Hive 0.6.0之前，Lateral View 不支持谓词下推优化。
在Hive 0.5.0和更早版本中，如果你使用WHERE子句，你的查询如果没有没有编译。
解决方法是在你查询之前添加 set hive.optimize.ppd = false 。
这个问题是在Hive 0.6.0中进行修复的， 请参阅https://issues.apache.org/jira/browse/HIVE-1056。
```

```
从Hive 0.12.0中，可以省略列别名。
在这种情况下，别名继承自从UTDF返回的StructObjectInspector的字段名称。
```

### 3. Example

考虑以下名为pageAds的基表。 它有两列：pageid（网页名称）和adid_list（网页上显示的广告数组）：

名称 | 类型
--- | ---
pageid | STRING
adid_list | Array<int>

具有两行数据的示例表：
pageid | adid_list
--- | ---
contact_page | [3, 4, 5]
front_page | [1, 2, 3]

而且用户希望统计广告在所有网页上展示的总次数。

Lateral View 与 explode()函数 结合使用可以将adid_list转换为单独的行：


```
hive> SELECT pageid, adid
    > FROM tmp_laterview LATERAL VIEW explode(adid_list) adTable AS adid;
OK
front_page	1
front_page	2
front_page	3
contact_page	3
contact_page	4
contact_page	5
Time taken: 0.132 seconds, Fetched: 6 row(s)
```

然后，为了计算特定广告的展示次数，使用如下命令：

```
hive> SELECT adid, count(1)
    > FROM tmp_laterview LATERAL VIEW explode(adid_list) adTable AS adid
    > GROUP BY adid;

OK
1	1
2	1
3	2
4	1
5	1
Time taken: 11.456 seconds, Fetched: 5 row(s)
```

### 4. Multiple Lateral Views

FROM子句可以有多个LATERAL VIEW子句。 后面的LATERAL VIEWS子句可以引用出现在LATERAL VIEWS左侧表的任何列。

例如，如下查询：
```
SELECT * FROM exampleTable
LATERAL VIEW explode(col1) myTable1 AS myCol1
LATERAL VIEW explode(col2) myTable2 AS myCol2;
```
LATERAL VIEW子句按照它们出现的顺序应用。 例如使用以下基表：

Array<int> pageid_list  |  Array<string> adid_list
--- | ---
[1, 2, 3] | ["a", "b", "c"]
[3, 4] | ["c", "d"]


单个Lateral View查询
```
hive> SELECT pageid_list, adid
    > FROM tmp_laterview
    > LATERAL VIEW explode(adid_list) adTable AS adid;
OK
[1,2,3]	a
[1,2,3]	b
[1,2,3]	c
[4,5]	c
[4,5]	d
```

多个Lateral View查询：
```
hive> SELECT pageid, adid
    > FROM tmp_laterview
    > LATERAL VIEW explode(adid_list) adTable AS adid
    > LATERAL VIEW explode(pageid_list) adTable AS pageid;
OK
1	a
2	a
3	a
1	b
2	b
3	b
1	c
2	c
3	c
4	c
5	c
4	d
5	d
```
### 5. Outer Lateral Views

```
在Hive0.12.0版本后引入
```
用户可以指定可选的OUTER关键字来生成行，即使LATERAL VIEW通常不会生成行。当所使用的UDTF不产生任何行时（使用explode()函数时，explode的列为空时，很容易发生上述这种情况）。 在这种情况下，源数据行不会出现在结果中。如果想让源数据行继续出现在结果中，可以使用 OUTER可以用来阻止关键字，并且来自UDTF的列使用NULL值代替。


例如，以下查询返回空结果：
```
hive> SELECT * FROM tmp_laterview LATERAL VIEW explode(array()) C AS a;
OK
Time taken: 0.077 seconds
```
但是使用OUTER关键词：
```
hive> SELECT * FROM tmp_laterview LATERAL VIEW OUTER explode(array()) C AS a;
OK
[1,2,3]	["a","b","c"]	NULL
[4,5]	["c","d"]	NULL
Time taken: 0.053 seconds, Fetched: 2 row(s)
```

原文：https://cwiki.apache.org/confluence/display/Hive/LanguageManual+LateralView
