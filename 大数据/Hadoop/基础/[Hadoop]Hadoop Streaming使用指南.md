---
layout: post
author: sjf0115
title: Hadoop Streaming使用指南
date: 2018-05-10 13:15:00
tags:
  - Hadoop
  - Hadoop 基础

categories: Hadoop
permalink: hadoop-streaming-use-guide
---

### 1. 概述

Hadoop Streaming 是Hadoop发行版附带的实用工具。该工具允许你使用任何可执行文件或脚本作为 mapper 或 reducer 来创建和运行 Map/Reduce 作业。 例如：
```
mapred streaming \
  -input myInputDirs \
  -output myOutputDir \
  -mapper /bin/cat \
  -reducer /usr/bin/wc
```

### 2. 如何工作


























> Hadoop版本:3.1.0

原文：http://hadoop.apache.org/docs/r3.1.0/hadoop-streaming/HadoopStreaming.html
