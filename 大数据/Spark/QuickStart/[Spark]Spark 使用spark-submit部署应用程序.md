---
layout: post
author: sjf0115
title: Spark 使用spark-submit部署应用程序
date: 2018-04-06 11:28:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-launching-applications-with-spark-submit
---

### 1. 简介

Spark的bin目录中的spark-submit脚本用于启动集群上的应用程序。 可以通过统一的接口使用Spark所有支持的集群管理器，因此不必为每个集群管理器专门配置你的应用程序（It can use all of Spark’s supported cluster managers through a uniform interface so you don’t have to configure your application specially for each one）。

### 2. 语法

```
xiaosi@yoona:~/opt/spark-2.1.0-bin-hadoop2.7$ spark-submit --help
Usage: spark-submit [options] <app jar | python file> [app arguments]
Usage: spark-submit --kill [submission ID] --master [spark://...]
Usage: spark-submit --status [submission ID] --master [spark://...]
Usage: spark-submit run-example [options] example-class [example args]

Options:
  --master MASTER_URL         spark://host:port, mesos://host:port, yarn, or local.
  --deploy-mode DEPLOY_MODE   Whether to launch the driver program locally ("client") or
                              on one of the worker machines inside the cluster ("cluster")
                              (Default: client).
  --class CLASS_NAME          Your application's main class (for Java / Scala apps).
  --name NAME                 A name of your application.
  --jars JARS                 Comma-separated list of local jars to include on the driver
                              and executor classpaths.
  --packages                  Comma-separated list of maven coordinates of jars to include
                              on the driver and executor classpaths. Will search the local
                              maven repo, then maven central and any additional remote
                              repositories given by --repositories. The format for the
                              coordinates should be groupId:artifactId:version.
  --exclude-packages          Comma-separated list of groupId:artifactId, to exclude while
                              resolving the dependencies provided in --packages to avoid
                              dependency conflicts.
  --repositories              Comma-separated list of additional remote repositories to
                              search for the maven coordinates given with --packages.
  --py-files PY_FILES         Comma-separated list of .zip, .egg, or .py files to place
                              on the PYTHONPATH for Python apps.
  --files FILES               Comma-separated list of files to be placed in the working
                              directory of each executor.

  --conf PROP=VALUE           Arbitrary Spark configuration property.
  --properties-file FILE      Path to a file from which to load extra properties. If not
                              specified, this will look for conf/spark-defaults.conf.

  --driver-memory MEM         Memory for driver (e.g. 1000M, 2G) (Default: 1024M).
  --driver-java-options       Extra Java options to pass to the driver.
  --driver-library-path       Extra library path entries to pass to the driver.
  --driver-class-path         Extra class path entries to pass to the driver. Note that
                              jars added with --jars are automatically included in the
                              classpath.

  --executor-memory MEM       Memory per executor (e.g. 1000M, 2G) (Default: 1G).

  --proxy-user NAME           User to impersonate when submitting the application.
                              This argument does not work with --principal / --keytab.

  --help, -h                  Show this help message and exit.
  --verbose, -v               Print additional debug output.
  --version,                  Print the version of current Spark.

 Spark standalone with cluster deploy mode only:
  --driver-cores NUM          Cores for driver (Default: 1).

 Spark standalone or Mesos with cluster deploy mode only:
  --supervise                 If given, restarts the driver on failure.
  --kill SUBMISSION_ID        If given, kills the driver specified.
  --status SUBMISSION_ID      If given, requests the status of the driver specified.

 Spark standalone and Mesos only:
  --total-executor-cores NUM  Total cores for all executors.

 Spark standalone and YARN only:
  --executor-cores NUM        Number of cores per executor. (Default: 1 in YARN mode,
                              or all available cores on the worker in standalone mode)

 YARN-only:
  --driver-cores NUM          Number of cores used by the driver, only in cluster mode
                              (Default: 1).
  --queue QUEUE_NAME          The YARN queue to submit to (Default: "default").
  --num-executors NUM         Number of executors to launch (Default: 2).
                              If dynamic allocation is enabled, the initial number of
                              executors will be at least NUM.
  --archives ARCHIVES         Comma separated list of archives to be extracted into the
                              working directory of each executor.
  --principal PRINCIPAL       Principal to be used to login to KDC, while running on
                              secure HDFS.
  --keytab KEYTAB             The full path to the file that contains the keytab for the
                              principal specified above. This keytab will be copied to
                              the node running the Application Master via the Secure
                              Distributed Cache, for renewing the login tickets and the
                              delegation tokens periodically.
```

### 3. 捆绑应用程序的依赖关系
如果你的代码依赖于其他项目，则需要将它们与应用程序一起打包，以便将代码分发到Spark集群上。为此，请创建一个包含代码及其依赖关系的程序集jar（或 Uber jar）。sbt和Maven都有装配插件。创建jar时，将Spark和Hadoop列出作为需要提供的依赖关系; 这些不需要捆绑，因为它们在运行时由集群管理器提供。一旦你有一个jar，你可以调用bin/ spark-submit脚本，如下所示，同时传递你的jar作为参数。

对于Python，您可以使用spark-submit的--py-files参数来添加.py，.zip或.egg文件以与应用程序一起分发。如果你依赖于多个Python文件，我们建议将它们打包成一个.zip或.egg文件。

### 4. 使用spark-submit启动应用程序

一旦用户应用程序打包成功后，可以使用bin/spark-submit脚本启动应用程序。此脚本负责设置Spark的 classpath 及其依赖关系，并且可以支持不同集群管理器和部署模式（Spark所支持的）：

```
./bin/spark-submit \
  --class <main-class> \
  --master <master-url> \
  --deploy-mode <deploy-mode> \
  --conf <key>=<value> \
  ... # other options
  <application-jar> \
  [application-arguments]
```

一些常用的选项：
- --class 应用程序入口 (例如：com.sjf.open.spark.Java.JavaWordCount 包含包名的全路径名称)
- --master 集群的主URL (例如：spark://23.195.26.187:7077)
- --deploy-mode 部署driver运行的地方，client或者cluster
- application-jar 包含应用程序和所有依赖关系的jar路径。 URL必须在集群内部全局可见，例如，所有节点上存在的hdfs：//路径或file：//路径。
- application-arguments 传递给主类main方法的参数（如果有的话）

Example:
```
bin/spark-submit --class com.sjf.open.spark.Java.JavaWordCount --master local common-tool-jar-with-dependencies.jar /home/xiaosi/click_uv.txt
```

如果你提交应用程序的机器远离工作节点机器（例如在笔记本电脑本地提交），则通常使用集群模式来最小化drivers和executors之间的网络延迟。 目前，对于Python应用程序而言，在独立模式上不支持集群模式。

对于Python应用程序，只需在<application-jar>位置传递一个.py文件来代替JAR，然后使用--py-files参数ca将Python .zip，.egg或.py文件添加到搜索路径。

有几个可用选项是特定用于集群管理器。例如，对于具有集群部署模式的Spark独立集群，可以指定--supervise参数以确保如果driver以非零退出而失败，则自动重新启动。如果要列举spark-submit所有可用选项，可以使用spark-submit --help命令来查看。以下是常见选项的几个示例：


```
# 在本地运行 8 核
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master local[8] \
  /path/to/examples.jar \
  100

# 以客户端部署模式在Spark独立集群上运行
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# 在集群部署模式下使用supervise在Spark独立集群上运行
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000

# 在 YARN 集群上运行
export HADOOP_CONF_DIR=XXX
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode cluster \  # can be client for client mode
  --executor-memory 20G \
  --num-executors 50 \
  /path/to/examples.jar \
  1000

# 在 Spark 独立集群上运行Python程序
./bin/spark-submit \
  --master spark://207.184.161.138:7077 \
  examples/src/main/python/pi.py \
  1000

# 在集群部署模式下使用supervise在Mesos集群上运行
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000
```

### 5. Master Urls
传递给Spark的master url 可以采用如下格式：

Master URL | 描述
---|---
local | 本地运行模式，使用单核
local[K] | 本地运行模式，，使用K个核心
`local[*]` | 本地运行模式，使用尽可能多的核心
spark://HOST:PORT| 连接到给定的Spark独立集群主机。 端口必须是主机配置可使用的端口，默认情况下为7077。
mesos://HOST:PORT | 连接到给定的Mesos集群。 端口必须是主机配置可使用的端口，默认为5050。 或者，对于使用ZooKeeper的Mesos集群，请使用mesos://zk:// .... 要使用--deploy-mode cluster 提交。
yarn | 以客户端模式还是以集群模式连接到YARN群集具体取决于--deploy-mode的值。 可以根据HADOOP_CONF_DIR或YARN_CONF_DIR变量找到集群位置

### 6. 从文件加载配置

spark-submit脚本可以从properties文件加载默认Spark配置选项，并将它们传递到应用程序。默认情况下，spark 从spark目录下的conf/spark-defaults.conf配置文件中读取配置选项。有关更多详细信息，请阅读[加载默认配置](http://spark.apache.org/docs/latest/configuration.html#loading-default-configurations)。

以这种方式加载默认Spark配置可以避免在spark-submit上添加配置选项。例如，如果默认配置文件中设置了spark.master属性，则可以安全地从spark-submit中省略--master参数。一般来说，在SparkConf上显式设置的配置选项拥有最高优先级，然后是传递到spark-submit的配置选项，然后是默认配置文件中的配置选项。

如果不清楚配置选项来自哪里，可以通过使用--verbose选项运行spark-submit打印出细粒度的调试信息。

### 7. 高级依赖管理

使用spark-submit时，应用程序jar以及包含在-jars选项中的jar将自动传输到集群。在--jars之后提供的URL列表必须用逗号分隔。 该列表会包含在driver和 executor 的classpath中。 目录扩展不能与--jars一起使用。

Spark使用如下URL方案以不同策略传播传送jar：
- file ： 绝对路径和file:/ URI 由driver 的HTTP文件服务器提供，每个executor从driver HTTP服务器拉取文件。
- hdfs :, http :, https :, ftp： 正如你希望的一样，这些从URI拉取文件和JAR
- local： 以local:/开头的URI应该作为每个工作节点上的本地文件存在。 这意味着不会产生网络IO，适用于推送大文件/ JAR到每个工作线程或通过NFS，GlusterFS等方式共享这些大文件/jar。

请注意，JAR和文件被复制到执行器节点上每个SparkContext的工作目录（Note that JARs and files are copied to the working directory for each SparkContext on the executor nodes）。随着时间的推移，这可能会占用大量的空间，需要定时清理。使用YARN，清理会自动执行；使用Spark独立集群，可以使用spark.worker.cleanup.appDataTtl属性配置自动清理。

用户还可以通过用--packages提供逗号分隔的maven坐标列表来包含任何其他依赖项。使用此命令时将处理所有传递依赖性。可以使用配置选项--repositories以逗号分隔的方式添加其他存储库（或SBT中的解析器）。pyspark，spark-shell和spark-submit都可以使用这些命令来包含Spark Packages。

对于Python，等价的--py-files选项可用于将.egg，.zip和.py库分发给执行程序。
