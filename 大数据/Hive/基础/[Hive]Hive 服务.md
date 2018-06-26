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



























....
