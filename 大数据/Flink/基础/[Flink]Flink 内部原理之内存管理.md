---
layout: post
author: sjf0115
title: Flink1.4 内部原理之内存管理
date: 2018-02-02 11:40:01
tags:
  - Flink
  - Flink内部原理

categories: Flink
permalink: flink-batch-internals-memory-management
---

### 1. 概述

Flink中的内存管理用于控制特定运行时操作使用的内存量。内存管理用于所有累积（可能很大）了一定数量记录的操作。
这种操作的典型例子是：
- 排序 - 排序用于对分组，连接后的记录进行排序以及生成排序结果。
- 哈希表 - 哈希表

### 2. Flink的内存管理

### 3. 对垃圾回收的影响

### 4. 内存管理算法

































原文: https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=53741525
