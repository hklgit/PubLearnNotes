### 1. 简介
Hive有很多任务，FetchTask是最有效率的任务之一。 它直接找到文件并给出结果，而不是启动MapReduce作业进行查询。对于简单的查询，如带有`LIMIT`语句的`SELECT * `查询，它非常快(单位数秒级)。在这种情况下，Hive可以通过执行hdfs操作来返回结果。

Example:
```
hive>  SELECT * FROM tmp_client_behavior LIMIT 1;
OK
2017-08-16      22:24:54   ...
Time taken: 0.924 seconds, Fetched: 1 row(s)
```
如果我们只想得到几列怎么办？
```
hive> select vid, gid, os from tmp_client_behavior;
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
Query ID = wirelessdev_20170817203931_02392cd8-5df7-42e6-87ea-aaa1418e000c
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_1472052053889_23316306, Tracking URL = xxx
Kill Command = /home/q/hadoop/hadoop-2.2.0/bin/hadoop job  -kill job_1472052053889_23316306
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
2017-08-17 20:39:39,886 Stage-1 map = 0%,  reduce = 0%
2017-08-17 20:39:46,000 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 1.32 sec
MapReduce Total cumulative CPU time: 1 seconds 320 msec
Ended Job = job_1472052053889_23316306
MapReduce Jobs Launched:
Stage-Stage-1: Map: 1   Cumulative CPU: 1.32 sec   HDFS Read: 581021 HDFS Write: 757 SUCCESS
Total MapReduce CPU Time Spent: 1 seconds 320 msec
OK
...

```
从日志中我们可以看到MR任务已经启动，1个mapper和0个reducer。有没有方法可以让我们避免启动上面的MapReduce作业？这就需要设置`hive.fetch.task.conversion`配置：
```
<property>
  <name>hive.fetch.task.conversion</name>
  <value>none|minimal|more</value>
</property>
```
Hive已经做过优化了，从Hive 0.10.0版本开始，对于简单的不需要聚合去重的查询语句，可以不需要运行MapReduce任务，直接通过FetchTask获取数据:
```
hive> select vid, gid, os from tmp_client_behavior limit 10;
OK
60001 A34D4B08788A adr
...
```

### 2. 配置

#### 2.1 hive.fetch.task.conversion

```hive
<property>
  <name>hive.fetch.task.conversion</name>
  <value>none|minimal|more</value>
</property>
```
可支持的选项为`none`,`minimal`和`more`，从Hive 0.10.0版本到Hive 0.13.1版本起，默认值为`minimal`，Hive0.14.0版本以及更高版本默认值改为`more`:
(1) none: 禁用`hive.fetch.task.conversion`（在Hive 0.14.0版本中引入）
(2) minimal: 这意味着当使用LIMIT执行`SELECT *`时，可以转换为FetchTask．
(3) more: 如果我们查询某个具体的列，使用`more`也可以转换为FetchTask。.`more`可以应用在SELECT子句中任何表达式上，包括UDF。（UDTF和`lateral views`尚不支持）

还有一些其他要求：单个数据源(意味着一个表或一个分区)，没有子查询，没有聚合或去重；不适用于视图或JOIN。这意味着如果我们执行下面的查询，仍然会使用FetchTask：
```
SELECT col1 as `alias1`, col2 FROM table WHERE partitionkey='somePartitionValue'
```

#### 2.2 hive.fetch.task.conversion.threshold

```
<property>
  <name>hive.fetch.task.conversion.threshold</name>
  <value>1073741824</value>
</property>
```
从Hive 0.13.0版本到Hive 0.13.1版本起，默认值为`-1`，Hive 0.14.0版本以及更高版本默认值改为`1073741824`(1G)．
此优化可获取指定分区中所有文件的长度，并与阈值进行比较，以确定是否应使用Fetch任务。在本例配置中，如果表大小大于1G，则使用Mapreduce而不是Fetch任务．

Example:
```
hive> set hive.fetch.task.conversion.threshold=100000000;
hive> select * from passwords limit 1;
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks is set to 0 since there's no reduce operator
Starting Job = job_201501081639_0046, Tracking URL = http://n1a.mycluster2.com:50030/jobdetails.jsp?jobid=job_201501081639_0046
Kill Command = /opt/mapr/hadoop/hadoop-0.20.2/bin/../bin/hadoop job  -kill job_201501081639_0046
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 0
2015-01-15 12:19:06,474 Stage-1 map = 0%,  reduce = 0%
2015-01-15 12:19:11,496 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 0.85 sec
MapReduce Total cumulative CPU time: 850 msec
Ended Job = job_201501081639_0046
MapReduce Jobs Launched:
Job 0: Map: 1   Cumulative CPU: 0.85 sec   MAPRFS Read: 0 MAPRFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 850 msec
OK
root x 0 0 root /root /bin/bash
Time taken: 6.698 seconds, Fetched: 1 row(s)
```
Else, it will only use fetch task:
```
hive> set hive.fetch.task.conversion.threshold=600000000;
hive> select * from passwords limit 1;
OK
root x 0 0 root /root /bin/bash
Time taken: 0.325 seconds, Fetched: 1 row(s)
```

**备注**
```
此参数根据表大小而不是结果集大小来计算或估计。
```

#### 2.3 hive.fetch.task.aggr

```
<property>
  <name>hive.fetch.task.aggr</name>
  <value>false</value>
</property>  
```
没有group-by子句的聚合查询(例如, `select count(*) from src`)．在单个reduce任务中执行最终的聚合。如果设置为真，Hive将最终聚合阶段委托给Fetch任务，可能会减少查询时间。

### 3. 设置Fetch任务

(1) 直接在命令行中使用`set`命令进行设置:
```
hive> set hive.fetch.task.conversion=more;
```
(2) 使用`hiveconf`进行设置
```
bin/hive --hiveconf hive.fetch.task.conversion=more
```
(3) 上面的两种方法都可以开启了Fetch Task，但是都是临时起作用的；如果你想一直启用这个功能，可以在${HIVE_HOME}/conf/hive-site.xml里面修改配置：
```
<property>
  <name>hive.fetch.task.conversion</name>
  <value>more</value>
</property>
```
