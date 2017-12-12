## 1.需求

我们项目中需要复制一个大文件，最开始使用的是hadoop cp命令，但是随着文件越来越大，拷贝的时间也水涨船高。下面进行hadoop cp与hadoop distcp拷贝时间上的一个对比。我们将11.9G的文件从data_group/adv/day=20170116下所有文件复制到tmp/data_group/adv/day=20170116/文件下
#### 1.1 查看文件大小
```
hadoop fs -du -s -h data_group/adv/day=20170116
11.9 G  data_group/adv/day=20170116
```
#### 1.2 复制 
```
hadoop distcp data_group/adv/day=20170116 \
tmp/data_group/adv/day=20170116 

hadoop fs -cp data_group/adv/day=20170116 \
tmp/data_group/adv/day=2010116
```

#### 1.3 对比

使用distcp命令 仅耗时1分钟；而hadoop cp命令耗时14分钟


## 2. 概述
DistCp（分布式拷贝）是用于大规模集群内部和集群之间拷贝的工具。 它使用Map/Reduce实现文件分发，错误处理和恢复，以及报告生成。 它把文件和目录的列表作为map任务的输入，每个任务会完成源列表中部分文件的拷贝。 由于使用了Map/Reduce方法，这个工具在语义和执行上都会有特殊的地方。

#### 2.1 distcp命令 输出
```
17/01/19 14:30:07 INFO tools.DistCp: Input Options: DistCpOptions{atomicCommit=false, syncFolder=false, deleteMissing=false, ignoreFailures=false, maxMaps=20, sslConfigurationFile='null', copyStrategy='uniformsize', sourceFileListing=null, sourcePaths=[data_group/adv/day=20170116], targetPath=tmp/data_group/adv/day=20170116}
...
17/01/19 14:30:17 INFO mapreduce.Job:  map 0% reduce 0%
17/01/19 14:30:29 INFO mapreduce.Job:  map 6% reduce 0%
17/01/19 14:30:34 INFO mapreduce.Job:  map 41% reduce 0%
17/01/19 14:30:35 INFO mapreduce.Job:  map 94% reduce 0%
17/01/19 14:30:36 INFO mapreduce.Job:  map 100% reduce 0%
17/01/19 14:31:34 INFO mapreduce.Job: Job job_1472052053889_8081193 completed successfully
17/01/19 14:31:34 INFO mapreduce.Job: Counters: 30
        File System Counters
                FILE: Number of bytes read=0
                FILE: Number of bytes written=1501420
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=12753350866
                HDFS: Number of bytes written=12753339159
                HDFS: Number of read operations=321
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=69
        Job Counters 
                Launched map tasks=17
                Other local map tasks=17
                Total time spent by all maps in occupied slots (ms)=485825
                Total time spent by all reduces in occupied slots (ms)=0
        Map-Reduce Framework
                Map input records=17
                Map output records=0
                Input split bytes=2414
                Spilled Records=0
                Failed Shuffles=0
                Merged Map outputs=0
                GC time elapsed (ms)=1634
                CPU time spent (ms)=156620
                Physical memory (bytes) snapshot=5716221952
                Virtual memory (bytes) snapshot=32341671936
                Total committed heap usage (bytes)=12159811584
        File Input Format Counters 
                Bytes Read=9293
        File Output Format Counters 
                Bytes Written=0
        org.apache.hadoop.tools.mapred.CopyMapper$Counter
                BYTESCOPIED=12753339159
                BYTESEXPECTED=12753339159
                COPY=17
```
是不是很熟悉，这就是我们经常看见到的MapReduce任务的输出信息，这从侧面说明了Distcp使用Map/Reduce实现。
## 3. 使用方法

#### 3.1 集群间拷贝
DistCp最常用在集群之间的拷贝：

```
hadoop distcp hdfs://nn1:8020/foo/bar \
    hdfs://nn2:8020/bar/foo
```
这条命令会把nn1集群的/foo/bar目录下的所有文件以及bar本身目录（默认情况下）存储到一个临时文件中，这些文件内容的拷贝工作被分配给多个map任务， 然后每个TaskTracker分别执行从nn1到nn2的拷贝操作。注意DistCp使用绝对路径进行操作。

#### 3.2 集群内部拷贝
DistCp也可以集群内部之间的拷贝：
```
hadoop distcp tmp/data_group/test/a 
    tmp/data_group/test/target
```
这条命令会把本集群tmp/data_group/test/a目录本身以及a目录下的所有文件拷贝到target目录下，原理同集群之间的拷贝一样。

*备注*

```
hadoop distcp tmp/data_group/test/a 
    tmp/data_group/test/target
```
上述命令默认情况下是复制a目录以及a目录下所有文件到target目录下：
```
hadoop fs -ls -R tmp/data_group/test/target/

tmp/data_group/test/target/aa
tmp/data_group/test/target/aa/aa.txt
tmp/data_group/test/target/aa/ab.txt
```
而有时我们的需求是复制a目录下的所有文件而不包含a目录，这时可以后面介绍的-update参数：
```
hadoop distcp -update tmp/data_group/test/aa tmp/data_group/test/target
...
hadoop fs -ls -R tmp/data_group/test/target/

tmp/data_group/test/target/aa/aa.txt
tmp/data_group/test/target/aa/ab.txt
```

#### 3.3 多源目录
命令行中可以指定多个源目录：
```
hadoop distcp hdfs://nn1:8020/foo/a 
    hdfs://nn1:8020/foo/b 
    hdfs://nn2:8020/bar/foo
```

或者使用-f选项，从文件里获得多个源：

```
hadoop distcp -f hdfs://nn1:8020/srclist \
    hdfs://nn2:8020/bar/foo 
```
其中srclist 的内容是
```
hdfs://nn1:8020/foo/a 
hdfs://nn1:8020/foo/b
```

*备注*

当从多个源拷贝时，如果两个源冲突，DistCp会停止拷贝并提示出错信息

Example：

```
hadoop distcp tmp/data_group/tes
t/source_new/aa.txt tmp/data_group/test/source_old/aa.txt tmp/data_group/test/target
```
我们分别复制source_new和source_old目录下的aa.txt文件到target文件夹下，报错如下：
```
17/01/21 15:15:05 ERROR tools.DistCp: Duplicate files in input path: 
org.apache.hadoop.tools.CopyListing$DuplicateFileException: File hdfs://XXX/user/XXX/tmp/data_group/test/source_new/aa.txt and \
hdfs://XXX/user/XXX/tmp/data_group/test/source_old/aa.txt would cause duplicates. Aborting
        at org.apache.hadoop.tools.CopyListing.checkForDuplicates(CopyListing.java:151)
        at org.apache.hadoop.tools.CopyListing.buildListing(CopyListing.java:87)
        at org.apache.hadoop.tools.GlobbedCopyListing.doBuildListing(GlobbedCopyListing.java:90)
        at org.apache.hadoop.tools.CopyListing.buildListing(CopyListing.java:80)
        at org.apache.hadoop.tools.DistCp.createInputFileListing(DistCp.java:327)
        at org.apache.hadoop.tools.DistCp.execute(DistCp.java:151)
```

如果在目的位置发生冲突，会根据选项设置解决。 默认情况会跳过已经存在的目标文件（比如不用源文件做替换操作）。每次操作结束时 都会报告跳过的文件数目，但是如果某些拷贝操作失败了，但在之后的尝试成功了， 那么报告的信息可能不够精确（请参考附录）。

每个TaskTracker必须都能够与源端和目的端文件系统进行访问和交互。 对于HDFS来说，源和目的端要运行相同版本的协议或者使用向下兼容的协议。 （请参考不同版本间的拷贝 ）。

拷贝完成后，建议生成源端和目的端文件的列表，并交叉检查，来确认拷贝真正成功。 因为DistCp使用Map/Reduce和文件系统API进行操作，所以这三者或它们之间有任何问题 都会影响拷贝操作。一些Distcp命令的成功执行可以通过再次执行带-update参数的该命令来完成， 但用户在如此操作之前应该对该命令的语法很熟悉。

值得注意的是，当另一个客户端同时在向源文件写入时，拷贝很有可能会失败。 尝试覆盖HDFS上正在被写入的文件的操作也会失败。 如果一个源文件在拷贝之前被移动或删除了，拷贝失败同时输出异常 FileNotFoundException。

## 4. 选项


#### 4.1 -i 忽略失败
这个选项会比默认情况提供关于拷贝的更精确的统计， 同时它还将保留失败拷贝操作的日志，这些日志信息可以用于调试。最后，如果一个map失败了，但并没完成所有分块任务的尝试，这不会导致整个作业的失败。


#### 4.2 -log <logdir> 记录日志
DistCp为每个文件的每次尝试拷贝操作都记录日志，并把日志作为map的输出。 如果一个map失败了，当重新执行时这个日志不会被保留。

#### 4.3 -overwrite 覆盖目标
如果一个map失败并且没有使用-i选项，不仅仅那些拷贝失败的文件，这个分块任务中的所有文件都会被重新拷贝。 就像下面提到的，它会改变生成目标路径的语义，所以 用户要小心使用这个选项。

#### 4.4 -update 源和目标的大小不一样则进行覆盖
这不是"同步"操作。 执行覆盖的唯一标准是源文件和目标文件大小是否相同；如果不同，则源文件替换目标文件。 像 下面提到的，它也改变生成目标路径的语义， 用户使用要小心。

## 5. 更新与覆盖
这里给出一些 -update和 -overwrite的例子。 考虑一个从/foo/a 和 /foo/b 到 /bar/foo的拷贝，源路径包括：
```
hdfs://nn1:8020/foo/a 
hdfs://nn1:8020/foo/a/aa 
hdfs://nn1:8020/foo/a/ab 
hdfs://nn1:8020/foo/b 
hdfs://nn1:8020/foo/b/ba 
hdfs://nn1:8020/foo/b/ab
```
如果没设置-update或 -overwrite选项， 那么两个源都会映射到目标端的 /bar/foo/ab。 如果设置了这两个选项，每个源目录的内容都会和目标目录的 内容 做比较。DistCp碰到这类冲突的情况会终止操作并退出。

默认情况下，/bar/foo/a 和 /bar/foo/b 目录都会被创建，所以并不会有冲突。

现在考虑一个使用-update合法的操作:
```
hadoop distcp -update tmp/data_group/test/source/ \ 
               tmp/data_group/test/target/
```
其中源路径/大小:
```
11 tmp/data_group/test/source/aa.txt
23 tmp/data_group/test/source/ab.txt
34 tmp/data_group/test/source/ba.txt
```
和目的路径/大小:
```
11 tmp/data_group/test/target/aa.txt
9  tmp/data_group/test/target/ba.txt
```
会产生:
```
11 tmp/data_group/test/target/aa.txt
23 tmp/data_group/test/target/ab.txt
34 tmp/data_group/test/target/ba.txt
```
只有target目录的aa.txt文件没有被覆盖。上面以及提到过，update不是"同步"操作。执行覆盖的唯一标准是源文件和目标文件大小是否相同；如果不同，则源文件替换目标文件。source中的aa.txt大小与target中的aa.txt大小一样，所以不会覆盖。

*==备注==*

实际测试的过程中，target目录下的aa.txt也会被覆盖，不得其解。求解......

如果指定了 -overwrite选项，所有文件都会被覆盖。

## 6. Map数目
DistCp会尝试着均分需要拷贝的内容，这样每个map拷贝差不多相等大小的内容。 但因为文件是最小的拷贝粒度，所以配置增加同时拷贝（如map）的数目不一定会增加实际同时拷贝的数目以及总吞吐量。

如果没使用-m选项，DistCp会尝试在调度工作时指定map的数目 为 min (total_bytes / bytes.per.map, 20 * num_task_trackers)， 其中bytes.per.map默认是256MB。

建议对于长时间运行或定期运行的作业，根据源和目标集群大小、拷贝数量大小以及带宽调整map的数目。


## 7.不同HDFS版本间的拷贝
对于不同Hadoop版本间的拷贝，用户应该使用HftpFileSystem。 这是一个只读文件系统，所以DistCp必须运行在目标端集群上（更确切的说是在能够写入目标集群的TaskTracker上）。 源的格式是 hftp://<dfs.http.address>/<path> （默认情况dfs.http.address是 <namenode>:50070）。

## 8. Map/Reduce和副效应
像前面提到的，map拷贝输入文件失败时，会带来一些副效应。

- 除非使用了-i，任务产生的日志会被新的尝试替换掉。
- 除非使用了-overwrite，文件被之前的map成功拷贝后当又一次执行拷贝时会被标记为 "被忽略"。
- 如果map失败了mapred.map.max.attempts次，剩下的map任务会被终止（除非使用了-i)。
- 如果mapred.speculative.execution被设置为 final和true，则拷贝的结果是未定义的。

