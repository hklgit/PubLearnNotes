Flink提供了一个命令行接口（CLI）用来运行打成JAR包的程序，并且可以控制程序的运行。命令行接口在Flink安装完之后即可拥有，本地单节点或是分布式的部署安装都会有命令行接口。命令行接口启动脚本是 $FLINK_HOME/bin目录下的flink脚本， 默认情况下会连接运行中的Flink master(JobManager)，JobManager的启动脚本与CLI在同一安装目录下。

使用命令行接口的先决条件是JobManager已经被启动或是在Flink YARN环境下。JobManager可以通过如下命令启动:
```
$FLINK_HOME/bin/start-local.sh
或
$FLINK_HOME/bin/start-cluster.sh
```
### 1. Example

(1) 运行示例程序，不传参数：
```
./bin/flink run ./examples/batch/WordCount.jar
```
(2) 运行示例程序，带输入和输出文件参数：
```
./bin/flink run ./examples/batch/WordCount.jar --input file:///home/xiaosi/a.txt --output file:///home/xiaosi/result.txt
```
(3) 运行示例程序，带输入和输出文件参数,并设置16个并发度：
```
./bin/flink run -p 16 ./examples/batch/WordCount.jar --input file:///home/xiaosi/a.txt --output file:///home/xiaosi/result.txt
```
(4) 运行示例程序，并禁止Flink输出日志
```
./bin/flink run -q ./examples/batch/WordCount.jar
```
(5) 以独立(detached)模式运行示例程序
```
./bin/flink run -d ./examples/batch/WordCount.jar
```
(6) 在指定JobManager上运行示例程序
```
./bin/flink run -m myJMHost:6123 ./examples/batch/WordCount.jar --input file:///home/xiaosi/a.txt --output file:///home/xiaosi/result.txt
```
(7) 运行示例程序，指定程序入口类(Main方法所在类)：
```
./bin/flink run -c org.apache.flink.examples.java.wordcount.WordCount ./examples/batch/WordCount.jar --input file:///home/xiaosi/a.txt --output file:///home/xiaosi/result.txt
```
(8) 运行示例程序，使用[per-job YARN 集群](https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/yarn_setup.html#run-a-single-flink-job-on-hadoop-yarn)启动 2 个TaskManager
```
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar --input hdfs:///xiaosi/a.txt --output hdfs:///xiaosi/result.txt
```
(9) 以JSON格式输出 WordCount示例程序优化执行计划：
```
./bin/flink info ./examples/batch/WordCount.jar --input file:///home/xiaosi/a.txt --output file:///home/xiaosi/result.txt
```
(10) 列出已经调度的和正在运行的Job(包含Job ID信息)
```
./bin/flink list
```
(11) 列出已经调度的Job(包含Job ID信息)
```
./bin/flink list -s
```
(13) 列出正在运行的Job(包含Job ID信息)
```
./bin/flink list -r
```
(14) 列出在Flink YARN中运行Job
```
./bin/flink list -m yarn-cluster -yid <yarnApplicationID> -r
```
(15) 取消一个Job
```
./bin/flink cancel <jobID>
```
(16) 取消一个带有保存点(savepoint)的Job
```
./bin/flink cancel -s [targetDirectory] <jobID>
```
(17) 停止一个Job(只适用于流计算Job)
```
./bin/flink stop <jobID>
```

备注:
```
取消和停止Job区别如下：
调用取消Job时，作业中的operator立即收到一个调用cancel()方法的指令以尽快取消它们。如果operator在调用取消操作后没有停止，Flink将定期开启中断线程来取消作业直到作业停止。
调用停止Job是一种停止正在运行的流作业的更加优雅的方法。停止仅适用于使用实现`StoppableFunction`接口的源的那些作业。当用户请求停止作业时，所有源将收到调用stop()方法指令。但是Job还是会持续运行，直到所有来源已经正确关闭。这允许作业完成处理所有正在传输的数据(inflight data)。
```

### 2. 保存点

保存点通过命令行客户端进行控制：

#### 2.1 触发保存点

```
./bin/flink savepoint <jobID> [savepointDirectory]
```
返回创建的保存点的路径。你需要此路径来还原和处理保存点。

触发保存点时，可以选择是否指定`savepointDirectory`。如果在此处未指定，则需要为Flink安装配置默认的保存点目录(请参阅[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/savepoints.html#configuration)）。

#### 2.2 根据保存点取消Job

你可以自动触发保存点并取消一个Job:
```
./bin/flink cancel -s  [savepointDirectory] <jobID>
```
如果没有指定保存点目录，则需要为Flink安装配置默认的保存点目录(请参阅[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/savepoints.html#configuration)）。如果保存点触发成功，该作业将被取消

#### 2.3 恢复保存点

```
./bin/flink run -s <savepointPath> ...
```
这个`run`命令提交Job时带有一个保存点标记，这使得程序可以从保存点中恢复状态。保存点路径是通过保存点触发命令得到的。

默认情况下，我们尝试将所有保存点状态与正在提交的作业相匹配。 如果要允许跳过那些无法使用它恢复新作业的保存点状态(allow to skip savepoint state that cannot be restored with the new job)，则可以设置`allowNonRestoredState`标志。如果当保存点触发时，从你程序中删除了作为程序一部分的operator，但是仍然要使用保存点，则需要允许这一点(You need to allow this if you removed an operator from your program that was part of the program when the savepoint was triggered and you still want to use the savepoint.)。

```
./bin/flink run -s <savepointPath> -n ...
```
如果你的程序删除了作为保存点一部分的operator,这时会非常有用(This is useful if your program dropped an operator that was part of the savepoint.)。

#### 2.4 销毁保存点

```
./bin/flink savepoint -d <savepointPath>
```
销毁一个保存点同样需要一个路径。这个保存点路径是通过保存点触发命令得到的。

### 3. 用法

下面是Flink命令行接口的用法:
```
xiaosi@yoona:~/qunar/company/opt/flink-1.3.2$ ./bin/flink
./flink <ACTION> [OPTIONS] [ARGUMENTS]

The following actions are available:

Action "run" compiles and runs a program.

  Syntax: run [OPTIONS] <jar-file> <arguments>
  "run" action options:
     -c,--class <classname>                         Class with the program entry
                                                    point ("main" method or
                                                    "getPlan()" method. Only
                                                    needed if the JAR file does
                                                    not specify the class in its
                                                    manifest.
     -C,--classpath <url>                           Adds a URL to each user code
                                                    classloader  on all nodes in
                                                    the cluster. The paths must
                                                    specify a protocol (e.g.
                                                    file://) and be accessible
                                                    on all nodes (e.g. by means
                                                    of a NFS share). You can use
                                                    this option multiple times
                                                    for specifying more than one
                                                    URL. The protocol must be
                                                    supported by the {@link
                                                    java.net.URLClassLoader}.
     -d,--detached                                  If present, runs the job in
                                                    detached mode
     -m,--jobmanager <host:port>                    Address of the JobManager
                                                    (master) to which to
                                                    connect. Use this flag to
                                                    connect to a different
                                                    JobManager than the one
                                                    specified in the
                                                    configuration.
     -n,--allowNonRestoredState                     Allow to skip savepoint
                                                    state that cannot be
                                                    restored. You need to allow
                                                    this if you removed an
                                                    operator from your program
                                                    that was part of the program
                                                    when the savepoint was
                                                    triggered.
     -p,--parallelism <parallelism>                 The parallelism with which
                                                    to run the program. Optional
                                                    flag to override the default
                                                    value specified in the
                                                    configuration.
     -q,--sysoutLogging                             If present, suppress logging
                                                    output to standard out.
     -s,--fromSavepoint <savepointPath>             Path to a savepoint to
                                                    restore the job from (for
                                                    example
                                                    hdfs:///flink/savepoint-1537
                                                    ).
     -z,--zookeeperNamespace <zookeeperNamespace>   Namespace to create the
                                                    Zookeeper sub-paths for high
                                                    availability mode
  Options for yarn-cluster mode:
     -yD <arg>                            Dynamic properties
     -yd,--yarndetached                   Start detached
     -yid,--yarnapplicationId <arg>       Attach to running YARN session
     -yj,--yarnjar <arg>                  Path to Flink jar file
     -yjm,--yarnjobManagerMemory <arg>    Memory for JobManager Container [in
                                          MB]
     -yn,--yarncontainer <arg>            Number of YARN container to allocate
                                          (=Number of Task Managers)
     -ynm,--yarnname <arg>                Set a custom name for the application
                                          on YARN
     -yq,--yarnquery                      Display available YARN resources
                                          (memory, cores)
     -yqu,--yarnqueue <arg>               Specify YARN queue.
     -ys,--yarnslots <arg>                Number of slots per TaskManager
     -yst,--yarnstreaming                 Start Flink in streaming mode
     -yt,--yarnship <arg>                 Ship files in the specified directory
                                          (t for transfer)
     -ytm,--yarntaskManagerMemory <arg>   Memory per TaskManager Container [in
                                          MB]
     -yz,--yarnzookeeperNamespace <arg>   Namespace to create the Zookeeper
                                          sub-paths for high availability mode

  Options for yarn mode:
     -ya,--yarnattached                   Start attached
     -yD <arg>                            Dynamic properties
     -yj,--yarnjar <arg>                  Path to Flink jar file
     -yjm,--yarnjobManagerMemory <arg>    Memory for JobManager Container [in
                                          MB]
     -yqu,--yarnqueue <arg>               Specify YARN queue.
     -yt,--yarnship <arg>                 Ship files in the specified directory
                                          (t for transfer)
     -yz,--yarnzookeeperNamespace <arg>   Namespace to create the Zookeeper
                                          sub-paths for high availability mode



Action "info" shows the optimized execution plan of the program (JSON).

  Syntax: info [OPTIONS] <jar-file> <arguments>
  "info" action options:
     -c,--class <classname>           Class with the program entry point ("main"
                                      method or "getPlan()" method. Only needed
                                      if the JAR file does not specify the class
                                      in its manifest.
     -p,--parallelism <parallelism>   The parallelism with which to run the
                                      program. Optional flag to override the
                                      default value specified in the
                                      configuration.
  Options for yarn-cluster mode:
     -yid,--yarnapplicationId <arg>   Attach to running YARN session

  Options for yarn mode:




Action "list" lists running and scheduled programs.

  Syntax: list [OPTIONS]
  "list" action options:
     -m,--jobmanager <host:port>   Address of the JobManager (master) to which
                                   to connect. Use this flag to connect to a
                                   different JobManager than the one specified
                                   in the configuration.
     -r,--running                  Show only running programs and their JobIDs
     -s,--scheduled                Show only scheduled programs and their JobIDs
  Options for yarn-cluster mode:
     -yid,--yarnapplicationId <arg>   Attach to running YARN session

  Options for yarn mode:




Action "stop" stops a running program (streaming jobs only).

  Syntax: stop [OPTIONS] <Job ID>
  "stop" action options:
     -m,--jobmanager <host:port>   Address of the JobManager (master) to which
                                   to connect. Use this flag to connect to a
                                   different JobManager than the one specified
                                   in the configuration.
  Options for yarn-cluster mode:
     -yid,--yarnapplicationId <arg>   Attach to running YARN session

  Options for yarn mode:




Action "cancel" cancels a running program.

  Syntax: cancel [OPTIONS] <Job ID>
  "cancel" action options:
     -m,--jobmanager <host:port>            Address of the JobManager (master)
                                            to which to connect. Use this flag
                                            to connect to a different JobManager
                                            than the one specified in the
                                            configuration.
     -s,--withSavepoint <targetDirectory>   Trigger savepoint and cancel job.
                                            The target directory is optional. If
                                            no directory is specified, the
                                            configured default directory
                                            (state.savepoints.dir) is used.
  Options for yarn-cluster mode:
     -yid,--yarnapplicationId <arg>   Attach to running YARN session

  Options for yarn mode:




Action "savepoint" triggers savepoints for a running job or disposes existing ones.

  Syntax: savepoint [OPTIONS] <Job ID> [<target directory>]
  "savepoint" action options:
     -d,--dispose <arg>            Path of savepoint to dispose.
     -j,--jarfile <jarfile>        Flink program JAR file.
     -m,--jobmanager <host:port>   Address of the JobManager (master) to which
                                   to connect. Use this flag to connect to a
                                   different JobManager than the one specified
                                   in the configuration.
  Options for yarn-cluster mode:
     -yid,--yarnapplicationId <arg>   Attach to running YARN session

  Options for yarn mode:

  Please specify an action.
```


原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/cli.html

备注:
```
Flink版本:1.3
由于翻译者水平有限，如果有问题，欢迎指正。在阅读时可以参考原文阅读。
```
