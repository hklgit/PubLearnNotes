---
layout: post
author: sjf0115
title: 在Zeppelin中如何使用Hive
date: 2020-08-31 21:43:01
tags:
  - Zeppelin

categories: Zeppelin
permalink: how-to-use-hive-in-zeppelin
---

### 1. 准备工作

我们来看看强大的 Zeppelin 能够给 Hive 带来什么吧。首先需要安装 Hive 和 Zeppelin。具体请参考如下两篇文章：
- [Zeppelin 安装与初体验](http://smartsi.club/zeppelin-install-and-config.html)
- [Hive 安装与配置](http://smartsi.club/hive-install-and-config.html)

完成以上步骤我们才能进行下一步。

### 2. Hive 元数据服务

Zeppelin 在访问 Hive 中的数据时需要得到 Hive 中的所有元数据信息，因此需要部署一个 HiveMetaStore 服务提供 Hive 的元数据信息。启动 HiveMetaStore 服务的命令如下：
```
nohup hive --service metastore -p 9083 > /opt/hive/logs/metastore.log 2>&1 &
```
上述表示在后台启动 Hive 的 MetaStore 服务，MetaStore 服务监听 9083 端口，并将错误和常规日志输出到日志文件 `/opt/hive/logs/metastore.log` 中。

### 3. 配置Hive解释器

解释器（Interpreter）是 Zeppelin 里最重要的概念，每一种解释器都对应一个引擎。需要注意的是Hive解释器被弃用并合并到 JDBC 解释器中。可以通过使用具有相同功能的 JDBC Interpreter 来使用 Hive Interpreter。Zeppelin 是通过 Hive 的 Jdbc 接口来运行 Hive SQL。

接下来我们可以在 Zeppelin 的 Interpreter 页面配置 Jdbc Interpreter 来启用 Hive。Jdbc Interpreter 可以支持所有 Jdbc 协议的数据库，包括 Hive。同时 Jdbc Interpreter 默认是连接 Postgresql。启动 Hive，我们可以有2种选择
- 修改默认 Jdbc Interpreter 的配置项：这种配置下，在Note里用hive可以直接 %jdbc 开头）
- 创建一个新的 Jdbc interpreter 并命名为 Hive： 这种配置下，在Note里用hive可以直接 %hive 开头）

这里我建议选用第2种方法，针对每一种引擎，单独创建一个解释器。这里我会创建一个新的 Hive Interprete。在解释器页面点击创建按钮，创建一个名为 hive 的解释器，解释器组选择为 jdbc：

![](1)



| 配置项     | 配置值     |
| :------------- | :------------- |
| hive.driver       | org.apache.hive.jdbc.HiveDriver |
| hive.url | jdbc:hive2://localhost:10000 |
| hive.user | 可选 |
| hive.password | 可选 |



属性
默认
描述
default.driver	org.apache.hive.jdbc.HiveDriver	JDBC驱动程序的类路径
default.url	jdbc:hive2://localhost:10000	网址连接
default.user	 	（可选）连接的用户名
default.password	 	（可选）连接密码
default.xxx	 	（可选）驱动程序使用的其他属性
${prefix}.driver	 	驱动程序类路径 %hive(${prefix})
${prefix}.url	 	的网址 %hive(${prefix})
${prefix}.user	 	（可选）连接的用户名%hive(${prefix})
${prefix}.password	 	（可选）连接密码%hive(${prefix})
${prefix}.xxx	 	（可选）驱动程序使用的其他属性%hive(${prefix})






参考：
- [Hive Interpreter for Apache Zeppelin](http://zeppelin.apache.org/docs/0.8.2/interpreter/hive.html)
- [如何在Zeppelin里玩转Hive](https://mp.weixin.qq.com/s/TzTrgR-eJ45kppuCabSovA)
- [Apache Zeppelin 中 Hive 解释器](https://cloud.tencent.com/developer/article/1014859)
