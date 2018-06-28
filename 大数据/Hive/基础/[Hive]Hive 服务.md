---
layout: post
author: sjf0115
title: Hive 服务
date: 2018-06-17 15:30:01
tags:
  - Hive

categories: Hive
permalink: hive-base-introduce-service
---

### 1. Hive 服务

Hive 的 Shell　环境只是 Hive 命令提供的其中一项服务。我们可以在运行时使用 service 选项指明要使用的哪种服务，如下所示:
```
[sjf@ying ~]$ hive --service help
Usage ./hive <parameters> --service serviceName <service parameters>
Service List: beeline cleardanglingscratchdir cli hbaseimport hbaseschematool help hiveburninclient hiveserver2 hplsql hwi jar lineage llapdump llap llapstatus metastore metatool orcfiledump rcfilecat schemaTool version
Parameters parsed:
  --auxpath : Auxillary jars
  --config : Hive configuration directory
  --service : Starts specific service/component. cli is default
Parameters used:
  HADOOP_HOME or HADOOP_PREFIX : Hadoop install directory
  HIVE_OPT : Hive options
For help on a particular service:
  ./hive --service serviceName --help
Debug help:  ./hive --debug --help
```
下面介绍一下比较有用的服务：
- CLI：Hive 的命令行接口(Shell环境)。这是默认的服务。
- HiveServer2：让 Hive 以提供 Thrift 服务的服务器形式运行，允许用不同语言编写的客户端进行范文。HiveServer2 在支持认证和多用户并发方面比原始的 HiveServer2 有很大的改进。使用 Thrift ，JDBC 和 ODBC 连接器的客户端需要运行 Hive 服务器来和 Hive　进行通信。通过设置 hive.server2.thrift.port 配置属性来执行服务器所监听的端口号（默认为10000）。
- beeline：以嵌入方式工作的 Hive 命令行接口（类似常规的CLI），或者使用 JDBC 连接到一个 HiveServer2 进程。
- metastore：默认情况下，metastore 和 Hive 服务运行在同一个进程里使用这个服务，可以让 metastore 作为一个单独的（远程）进程运行。通过设置 METASTORE_PORT 环境变量（或者使用-p命令行选项）可以指定服务器监听的端口号（默认为9083）。

### 2. Hive 客户端

如果以服务器方式运行 Hive （hive --service hiveserver2），可以在应用程序中以不同机制连接到服务器。Hive 客户端和服务之间的联系如下图所示。
- Thrift 客户端：Hive 服务器提供 Thrift 服务运行，因此任何支持 Thrift 的编程语言都可以与之交互。
- JDBC 驱动：Hive 提供了 Type4（纯Java）的 JDBC 驱动，定义在 org.apache.hadoop.hive.jdbc.HiveDriver 类中。在以 jdbc:hive2//host:port/dbname 形式配置 JDBC URI 以后，Java应用程序可以在指定的主机和端口号连接到在另一个进程中运行的 Hive 服务器。驱动使用 Java 的 Thrift 绑定来调用由 Hive Thrift 客户端实现的接口。

![]()






















....
