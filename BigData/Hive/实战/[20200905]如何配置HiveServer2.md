---
layout: post
author: sjf0115
title: 如何配置HiveServer2
date: 2020-09-05 15:15:01
tags:
  - Hive

categories: Hive
permalink: hive-base-how-to-use-bucket
---

HiveServer2(HS2)是一个服务器接口，能使远程客户端执行Hive查询，并且可以检索结果。HiveServer2是HiveServer1的改进版，HiveServer1已经被废弃。HiveServer2可以支持多客户端并发和身份认证。旨在为开放API客户端（如JDBC和ODBC）提供更好的支持。

这篇文章将介绍如何配置服务器端。如何使用客户端与此服务器端交互将在下篇文章中介绍。

==备注==

Hive 0.11版本引入. See [HIVE-2935](https://issues.apache.org/jira/browse/HIVE-2935).


### 1. 配置

#### 1.1 hive-site.xml中配置

```
hive.server2.thrift.min.worker.threads – 最小工作线程, 默认为 5.
hive.server2.thrift.max.worker.threads – 最大工作线程, 默认为 500.
hive.server2.thrift.port – 监听的TCP端口号, 默认为 10000.
hive.server2.thrift.bind.host – 绑定的TCP接口.
```
其他的选项可以参考　[HiveServer2 in the Configuration Properties document ](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-HiveServer2)

#### 1.2 可选环境设置

```
HIVE_SERVER2_THRIFT_BIND_HOST – 绑定到的TCP host接口(可选)。覆盖配置文件设置。
HIVE_SERVER2_THRIFT_PORT – 要监听的TCP端口号(可选)，默认为10000.覆盖配置文件设置。
```

#### 1.3 HTTP模式运行

HiveServer2支持通过HTTP传输发送Thrift RPC消息（Hive 0.13版本开始，参见[HIVE-4752](https://issues.apache.org/jira/browse/HIVE-4752)）。这对于支持客户端和服务器之间需要代理时非常有用（例如，为了负载均衡或安全原因）。目前，可以在TCP模式或HTTP模式下运行HiveServer2，但不能同时运行HiveServer2。对于相应的JDBC URL，请参考：[HiveServer2客户端 - JDBC连接URL](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients#HiveServer2Clients-JDBC)。 使用以下设置启用和配置HTTP模式：

设置|默认值|描述
---|---|---|
hive.server2.transport.mode|binary|设置为http以启用HTTP传输模式
hive.server2.thrift.http.port|10001|需要监听的HTTP端口
hive.server2.thrift.http.max.worker.threads|500|最大工作线程
hive.server2.thrift.http.min.worker.threads|5|最小工作线程
hive.server2.thrift.http.path|cliservice|服务端点

#### 1.4 可选的全局初始化文件

全局初始化文件可以放置在`hive.server2.global.init.file.location`在配置的位置（Hive 0.14开始版本，参见[HIVE-5160](https://issues.apache.org/jira/browse/HIVE-5160)，[HIVE-7497](https://issues.apache.org/jira/browse/HIVE-7497)和[HIVE-8138](https://issues.apache.org/jira/browse/HIVE-8138)）。 这可以是初始化文件本身的路径，也可以是一个名为`.hiverc`的初始化文件目录。

初始化文件列出了将为此HiveServer2实例的用户运行的一组命令，例如注册一组标准的jar和函数。

#### 1.5 日志记录配置

Beeline客户端可以获取HiveServer2操作日志（Hive 0.14开始版本）。配置日志记录一些参数如下：
```
hive.server2.logging.operation.enabled 默认为true，表示HiveServer2将为客户端保存操作日志
hive.server2.logging.operation.log.location　如果启用此功能，则存储操作日志到顶级目录中。
hive.server2.logging.operation.verbose (Hive 0.14 到 1.1)如果为true，则可以为客户端提供HiveServer2操作日志。 在Hive 1.2.0中替换为hive.server2.logging.operation.level。
hive.server2.logging.operation.level (Hive 1.2 开始版本) 可以设置HiveServer2操作日志级别
```
### 2. 如何开始

```
$HIVE_HOME/bin/hiveserver2
```
或者
```
$HIVE_HOME/bin/hive --service hiveserver2
```
### 2.1 使用信息

-H或--help选项显示使用消息，例如：
```
$HIVE_HOME/bin/hive --service hiveserver2 -H
Starting HiveServer2
usage: hiveserver2
 -H,--help                        Print help information
    --hiveconf <property=value>   Use value for given property
```

### 3. Web UI

==备注==

Hive 2.0.0版本引入．

HiveServer2的Web用户界面（UI）提供配置，日志记录，度量(metrics)和活动会话信息。 默认情况下，Web UI可以在端口10002（127.0.0.1:10002）上使用。

- 可以在`hive-site.xml`中自定义Web UI的配置属性，其中包括`hive.server2.webui.host`，`hive.server2.webui.port`，`hive.server2.webui.max.threads`等。
- Hive Metrics可以通过使用`Metrics Dump`选项卡查看。
- 可以使用``本地日志``选项卡查看日志。

例如如下配置：
```xml
<property>
  <name>hive.server2.webui.host</name>
  <value>127.0.0.1</value>
</property>

<property>
  <name>hive.server2.webui.port</name>
  <value>10002</value>
</property>
```

该接口目前正在[HIVE-12338](https://issues.apache.org/jira/browse/HIVE-12338)上开发。

![image](https://cwiki.apache.org/confluence/download/attachments/30758712/hs2-webui.png?version=1&modificationDate=1452895731000&api=v2)

### 4. Python 客户端驱动程序

HiveServer2的Python客户端驱动程序可在 https://github.com/BradRuderman/pyhs2 上获得（谢谢Brad）。它包括所有必需的软件包，如SASL和Thrift包装器(wrappers)。

该驱动程序已被认证可用于Python 2.6及更高版本。

要使用[pyhs2](https://github.com/BradRuderman/pyhs2)驱动程序：
```
pip install pyhs2
```
然后：
```Python

import pyhs2

with pyhs2.connect(host='localhost',
                   port=10000,
                   authMechanism="PLAIN",
                   user='root',
                   password='test',
                   database='default') as conn:
    with conn.cursor() as cur:
        #Show databases
        print cur.getDatabases()

        #Execute query
        cur.execute("select * from table")

        #Return column info from query
        print cur.getSchema()

        #Fetch table results
        for i in cur.fetch():
            print i
```

原文：https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2
