---
layout: post
author: smartsi
title: 如何启动HiveServer2
date: 2020-09-05 15:15:01
tags:
  - Hive

categories: Hive
permalink: how-to-config-and-start-hiveserver2
---

HiveServer2 是一种可选的 Hive 内置服务，可以允许远程客户端使用不同编程语言向 Hive 提交请求并返回结果。HiveServer2 是 HiveServer1 的改进版，主要解决了无法处理来自多个客户端的并发请求以及身份验证问题。具体可以参阅 [一起了解一下HiveServer2](http://smartsi.club/hiveserver2-overview.html)。下面我们具体看一下如何配置 HiveServer2。

### 1. 配置

假设我们已经成功安装了 Hive，如果没有安装，可以参阅 [Hive 安装与配置](http://smartsi.club/hive-install-and-config.html)。在启动 HiveServer2 之前，我们需要先进行一些配置：
```xml
<property>
  <name>hive.server2.transport.mode</name>
  <value>binary</value>
  <description>
    Expects one of [binary, http]. Transport mode of HiveServer2.
  </description>
</property>

<property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
    <description>Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'binary'.</description>
</property>
```
默认情况下，HiveServer2 启动使用的是默认配置。这些配置主要是服务启动的 Host 和端口号以及客户端和后台操作运行的线程数。我们可以重写 hive-site.xml 配置文件中的配置项来修改 HiveServer2 的默认配置：
| 配置项   | 默认值     | 说明 |
| :------------- | :------------- | :------------- |
| hive.server2.transport.mode | binary | HiveServer2 的传输模式，binary或者http |
| hive.server2.thrift.port | 10000 | HiveServer2 传输模式设置为 binary 时，Thrift 接口的端口号 |
| hive.server2.thrift.http.port | 10001 | HiveServer2 传输模式设置为 http 时，Thrift 接口的端口号 |
| hive.server2.thrift.bind.host | localhost | Thrift服务绑定的主机 |
| hive.server2.thrift.min.worker.threads | 5 | Thrift最小工作线程数 |
| hive.server2.thrift.max.worker.threads | 500 | Thrift最大工作线程数 |
| hive.server2.authentication | NONE | 客户端认证类型，NONE、LDAP、KERBEROS、CUSTOM、PAM、NOSASL |
| hive.server2.thrift.client.user | anonymous | Thrift 客户端用户名 |
| hive.server2.thrift.client.password | anonymous | Thrift 客户端密码 |

### 2. 启动

启动 HiveServer2 非常简单，我们需要做的只是运行如下命令即可：
```
$HIVE_HOME/bin/hiveserver2 &
```
或者
```
$HIVE_HOME/bin/hive --service hiveserver2 &
```
检查 HiveServer2 是否启动成功的最快捷的办法就是使用 netstat 命令查看 10000 端口是否打开并监听连接：
```
netstat -nl | grep 10000
```

### 3. 验证



### 4. Web UI

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
