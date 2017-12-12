### 1. 什么是Hadoop archives?

Hadoop archives是特殊的档案格式。一个Hadoop archive对应一个文件系统目录。 Hadoop archive的扩展名是*.har。Hadoop archive包含元数据（形式是_index和_masterindx）和数据（part）文件。index文件包含了档案中的文件的文件名和位置信息。

### 2. 如何创建archive?

#### 2.1 格式
```
hadoop archive -archiveName name -p <parent> <src>* <dest>
```
#### 2.2 参数

（1）由-archiveName选项指定你要创建的archive的名字（name）。比如user_order.har。archive的名字的扩展名应该是*.har

（2）-p参数指定文件存档文件（src）的相对路径，举个例子：

```
-p /foo/bar a/b/c e/f/g
```
这里的/foo/bar是a/b/c与e/f/g的父路径，所以完整路径为/foor/bar/a/b/c与/foo/bar/e/f/g
（3）src 是输入归档文件的目录

（4）dest 是目标目录，创建的archive会保存到该目录下

#### 2.3 Example
```
 hadoop archive -archiveName user_order.har -p /user/xiaosi user/user_active_new_user order/entrance_order test/archive
```
在上面的例子中，
```
/user/xiaosi/user/user_active_new_user 和 /user/xiaosi/order/entrance_order
```
会被归档到test/aarchive/user_order.har。当创建archive时，源文件不会被更改或删除。
注意创建archives是一个Map/Reduce job。你应该在map reduce集群上运行这个命令：

```
xiaosi@yoona:~/opt/hadoop-2.7.3$ hadoop archive -archiveName user_order.har -p /user/xiaosi user/user_active_new_user order/entrance_order test/archive
16/12/26 20:45:36 INFO Configuration.deprecation: session.id is deprecated. Instead, use dfs.metrics.session-id
16/12/26 20:45:36 INFO jvm.JvmMetrics: Initializing JVM Metrics with processName=JobTracker, sessionId=
16/12/26 20:45:37 INFO jvm.JvmMetrics: Cannot initialize JVM Metrics with processName=JobTracker, sessionId= - already initialized
16/12/26 20:45:37 INFO jvm.JvmMetrics: Cannot initialize JVM Metrics with processName=JobTracker, sessionId= - already initialized
16/12/26 20:45:37 INFO mapreduce.JobSubmitter: number of splits:1
16/12/26 20:45:37 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_local133687258_0001
16/12/26 20:45:37 INFO mapreduce.Job: The url to track the job: http://localhost:8080/
16/12/26 20:45:37 INFO mapreduce.Job: Running job: job_local133687258_0001
...
16/12/26 20:45:38 INFO mapred.LocalJobRunner: reduce task executor complete.
16/12/26 20:45:39 INFO mapreduce.Job:  map 100% reduce 100%
16/12/26 20:45:39 INFO mapreduce.Job: Job job_local133687258_0001 completed successfully
16/12/26 20:45:39 INFO mapreduce.Job: Counters: 35
	File System Counters
		FILE: Number of bytes read=95398
		FILE: Number of bytes written=678069
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=974540
		HDFS: Number of bytes written=975292
		HDFS: Number of read operations=55
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=11
	Map-Reduce Framework
		Map input records=8
		Map output records=8
		Map output bytes=761
		Map output materialized bytes=783
		Input split bytes=147
		Combine input records=0
		Combine output records=0
		Reduce input groups=8
		Reduce shuffle bytes=783
		Reduce input records=8
		Reduce output records=0
		Spilled Records=16
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=0
		Total committed heap usage (bytes)=593494016
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters
		Bytes Read=689
	File Output Format Counters
		Bytes Written=0
```
### 3. 如何查看archives中的文件?

archive作为文件系统层暴露给外界。所以所有的fs shell命令都能在archive上运行，但是要使用不同的URI。 另外，archive是不可改变的。所以重命名，删除和创建都会返回错误。

Hadoop Archives 的URI是
```
har://scheme-hostname:port/archivepath/fileinarchive
```
如果没提供scheme-hostname，它会使用默认的文件系统。这种情况下URI是这种形式
har:///archivepath/fileinarchive
获得创建的archive中的文件列表，使用命令
hadoop dfs -ls har:///user/xiaosi/test/archive/user_order.har
Example：
```
xiaosi@yoona:~/opt/hadoop-2.7.3$ hadoop dfs -ls har:///user/xiaosi/test/archive/user_order.har
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.
Found 2 items
drwxr-xr-x   - xiaosi supergroup          0 2016-12-13 13:39 har:///user/xiaosi/test/archive/user_order.har/order
drwxr-xr-x   - xiaosi supergroup          0 2016-12-24 15:51 har:///user/xiaosi/test/archive/user_order.har/user
```
查看archive中的entrance_order.txt文件的命令：
```
hadoop dfs -cat har:///user/xiaosi/test/archive/user_order.har/order/entrance_order/entrance_order.txt
```
Example：
```
xiaosi@yoona:~/opt/hadoop-2.7.3$ hadoop dfs -cat har:///user/xiaosi/test/archive/user_order.har/order/entrance_order/entrance_order.txt
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.
{"clickTime":"20161210 14:47:35.000","entrance":"306","actionTime":"20161210 14:48:14.000","orderId":"qgpww161210144814ea9","businessType":"TRAIN","gid":"0005B9C5-3B24-A63A-02C2-B4F1B369BF1D","uid":"866184028471741","vid":"60001151","income":105.5,"status":140}
{"clickTime":"20161210 14:47:35.000","entrance":"306","actionTime":"20161210 14:48:18.000","orderId":"huany161210144818e46","businessType":"TRAIN","gid":"0005B9C5-3B24-A63A-02C2-B4F1B369BF1D","uid":"866184028471741","vid":"60001151","income":105.5,"status":140}
```
### 4.archives 缺点

- 创建archives文件后，无法更新文件来添加或删除文件。 换句话说，har文件是不可变的。

- archives文件是所有原始文件的副本，所以一旦一个.har文件创建之后，它将会比原始文件占据更大的空间。不要误以为.har文件是原始文件的压缩文件。

- 当.har文件作为MapReduce作业的输入文件时，.har文件中的小文件将由单独的Mapper处理，这不是一种有效的方法。
