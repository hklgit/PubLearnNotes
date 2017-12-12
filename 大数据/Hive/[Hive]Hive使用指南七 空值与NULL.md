### 1. NULL(null)值

创建一个临时表tmp_null_empty_test，并插入一些NULL数据:
```
CREATE  TABLE IF NOT EXISTS tmp_null_empty_test(
  uid string
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
LINES TERMINATED BY '\n'
STORED AS TEXTFILE;

INSERT OVERWRITE TABLE tmp_null_empty_test select NULL from test WHERE dt = '20171016';
```
我们看一下从Hive中取出来的数据:
```
hive> select * from tmp_null_empty_test;
OK
NULL
NULL
NULL
NULL
NULL
NULL
NULL
NULL
NULL
NULL
NULL
NULL
...
```
我们再看一下这张表在HDFS上究竟是如何存储的?
```
hadoop fs -text /user/hive/warehouse/test.db/tmp_null_empty_test/* | less
\N
\N
\N
\N
\N
\N
\N
\N
\N
\N
\N
...
```
发现Hive将NULL值存储为'\N'。

Hive在底层数据中如何保存和标识NULL，是由`serialization.null.format`参数控制的，默认为`serialization.null.format'='\\N`。我们可以更改这一参数使之NULL值存储为其他形式，例如下面我们更改为"null":
```
ALTER TABLE tmp_null_empty_test SET SERDEPROPERTIES('serialization.null.format' = 'null');
```
我们删除数据重新倒入一遍数据:
```
hadoop fs -text /user/hive/warehouse/test.db/tmp_null_empty_test/* |less
null
null
null
null
null
null
null
null
null
...
```
这样的设计存在一个问题是如果实际想存储'\N'，那么实际查询出来的也是NULL而不是'\N' 。所以如果真的想存储'\N'时，可以更改配置参数为其他格式即可。

那我们在Hive中如何查询等于'NULL'的那些值呢?可以使用如下命令查询:
```
hive> select * from tmp_null_empty_test where uid is null;
OK
NULL
NULL
NULL
NULL
NULL
...
```

### 2. 空字符串

我们再看一下存储空字符串的情况，删除之前的数据，重新导入一些空字符串:
```
INSERT OVERWRITE TABLE tmp_null_empty_test select "" from test WHERE dt = '20171016';
```
我们看一下从Hive中取出来的数据:
```
hive> select * from tmp_null_empty_test;
OK

...

Time taken: 0.047 seconds, Fetched: 36 row(s)
```
我们再看一下在HDFS上究竟是如何存储的?
```
hadoop fs -text /user/hive/warehouse/test.db/tmp_null_empty_test/* |less

...

```
对于空字符串我们使用如下命令查询:
```
hive> select count(*) from tmp_null_empty_test where uid = "";
OK
36
```
但是不能使用`is null`来判断:
```
hive> select count(*) from tmp_null_empty_test where uid is null;
OK
0
```
### 3. 数据类型与NULL

INT与STRING的存储，NULL默认的存储都是'\N'。如果数据类型为String的数据为""，存储才是""。如果往Int类型的字段插入""数据，存储为'\N'。
查询的时候，对于Int类型数据可以使用`IS NULL`来判断NULL值；对于String数据类型的数据采用`IS NULL`来查询NULL值，采用`=""`来查询空字符串。

我们遇到的一个Case，在查询String数据类型uid缺失的数据时，我们不得不使用`IS NULL`和`=""`两个判断条件进行过滤，我们其实想遵循SQL规范使用`IS NULL`一个判断条件判断即可，但是在Hive中与传统的数据库又不一样，在于NULL的解读不同。如果想延续传统数据库中对于空值为NULL，可以通过alter语句来修改hive表的信息，保证解析时是按照空值来解析NULL值:
```
ALTER TABLE tmp_null_empty_test SET SERDEPROPERTIES('serialization.null.format' = '');
```
Example：
```
hive> INSERT OVERWRITE TABLE tmp_null_empty_test select "" from test WHERE dt = '20171016';
hive> select count(*) from tmp_null_empty_test where uid is null;
OK
36
```



备注:
```
Hive版本为2.1.1
```
