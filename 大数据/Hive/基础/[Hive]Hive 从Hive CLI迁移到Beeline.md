---
layout: post
author: sjf0115
title: Hive 从Hive CLI迁移到Beeline
date: 2018-07-24 13:01:01
tags:
  - Hive

categories: Hive
permalink: hive-base-migrating-from-hive-cli-to-beeline
---

之前，Apache Hive 是一个重量级命令行工具，接受查询SQL并转换为 MapReduce 执行。后来，该工具分为 `client-server` 模式，server 为 `HiveServer1` （负责编译和监控 MapReduce 作业），Hive CLI 是命令行界面（将SQL发送到 server）。

Hive社区推出了 HiveServer2，这是一个增强的Hive服务器，专为多客户端并发和改进身份验证而设计，也为通过 JDBC 和 ODBC 连接的客户端提供了更好的支持。现在，推荐使用 HiveServer2，Beeline 作为命令行界面；不推荐使用 HiveServer1 和Hive CLI，后者甚至都不能与 HiveServer2 一起使用。

Beeline 专门开发用来与新服务器进行交互。与基于 Apache Thrift 的 Hive CLI 客户端不同，Beeline 是基于 SQLLine CLI 的 JDBC 客户端 - 尽管 JDBC 驱动程序使用 HiveServer2 的 Thrift API 与 HiveServer2 进行通信。

随着Hive开发从最初的 Hive 服务器（HiveServer1）转移到新服务器（HiveServer2），用户和开发人员因此需要切换到新的客户端工具。但是，这个过程不仅仅简单地将可执行文件名从 hive 切换为 beeline。

在这篇文章中，我们将学习如何顺利地进行迁移，并了解两个客户端之间的差异和相似之处。虽然Beeline提供了一些非必要的选项，例如着色，但这篇文章我们主要关注如何使用Beeline实现我们以前使用Hive CLI做的事情。

### 1. 使用用法：Hive CLI VS Beeline

以下部分重点介绍 Hive CLI/HiveServer1 的常见用法以及如何在各种情况下迁移到 Beeline/HiveServer2。

#### 1.1 连接服务器

Hive CLI 使用 Thrift 协议连接到远程 HiveServer1 实例。要连接到服务器，需要指定远程服务器的主机名和可选的端口号：
```
> hive -h <hostname> -p <port>
```
相反，Beeline 使用 JDBC 连接到远程 HiveServer2 实例。因此，连接参数是在基于 JDBC 的客户端中常见的 JDBC URL：
```
> beeline -u  <url> -n <username> -p <password>
```
以下是一些　URL 示例：
```
jdbc:hive2://ubuntu:11000/db2?hive.cli.conf.printheader=true;hive.exec.mode.local.auto.inputbytes.max=9999#stab=salesTable;icol=customerID
jdbc:hive2://?hive.cli.conf.printheader=true;hive.exec.mode.local.auto.inputbytes.max=9999#stab=salesTable;icol=customerID
jdbc:hive2://ubuntu:11000/db2;user=foo;password=bar
jdbc:hive2://server:10001/db;user=foo;password=bar?hive.server2.transport.mode=http;hive.server2.thrift.http.path=hs2
```
#### 1.2 查询执行

在 Beeline 中执行查询与 Hive CLI 中的查询非常相似。在Hive CLI中：
```
> hive -e <query in quotes>
> hive -f <query file name>
```
在 Beeline 中:
```
> beeline -e <query in quotes>
> beeline -f <query file name>
```
在任何一种情况下，如果没有给出 `-e` 或 `-f` 选项，则两个客户端工具都将进入交互模式，你可以在其中逐行提供和执行查询或命令。

#### 1.3 嵌入模式

使用嵌入式服务器运行Hive客户端工具是测试查询或调试问题的便捷方式。虽然 Hive CLI 和 Beeline 都可以嵌入 Hive 服务器实例，但你可以采用稍微不同的方式在嵌入模式下启动它们。

要以嵌入模式启动 Hive CLI，只需启动客户端而不提供任何连接参数：
```
> hive
```
要在嵌入模式下启动 Beeline，需要做更多的工作。基本上，需要指定 `jdbc:hive2://` 的连接URL：
```
> beeline -u jdbc:hive2://
```
这时，Beeline 进入交互模式，在该模式下可以执行针对嵌入式 HiveServer2 实例的查询和命令。

#### 1.4 变量

也许客户端之间最有趣的区别在于Hive变量的使用。变量有四个名称空间：
- hiveconf：hive配置变量
- system：系统变量
- env：环境变量
- hivevar：Hive变量（[HIVE-1096](https://issues.apache.org/jira/browse/HIVE-1096)）








原文：http://blog.cloudera.com/blog/2014/02/migrating-from-hive-cli-to-beeline-a-primer/
