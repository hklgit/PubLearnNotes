---
layout: post
author: smartsi
title: Kafka 监控工具之CMAK
date: 2020-10-10 18:16:01
tags:
  - Kafka

categories: Kafka
permalink: cmak-managing-apache-kafka-cluster
---

### 1. 概述

### 2. 下载

环境要求：
- Kafka 0.8+
- Java 11+

由于我机器上只安装了 JDK 8，所以需要再安装一个 [JDK 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)：
```
sudo tar -zxvf jdk-11.0.8_osx-x64_bin.tar.gz -C /opt/
ln -s jdk-11.0.8.jdk/ jdk-11
```
> JDK11 只为CMAK使用，其他还是使用 JDK 8。

这里 CMAK 以 3.0.0.5 版本为例：
```
wget https://github.com/yahoo/CMAK/releases/download/3.0.0.5/cmak-3.0.0.5.zip
```
解压安装包：
```
unzip cmak-3.0.0.5.zip
```
创建软连接便于升级：
```
ln -s cmak-3.0.0.5/ cmak
```

### 3. 配置

修改 `/etc/profile` 配置环境变量，添加如下配置：
```
# kafka manager
export CMAK_HOME=/opt/cmak
export PATH=${CMAK_HOME}/bin:$PATH
```
运行命令 `source /etc/profile` 使环境变量生效。

按如下方式修改配置文件 application.conf，修改ZooKeeper服务器地址：
```
cmak.zkhosts="127.0.0.1:2181,127.0.0.1:2181:2182,127.0.0.1:2181:2183"
```
> 由于我们的ZooKeeper集群是伪分布式模式，通过不同的端口号来模拟不同的服务器。如果是正常集群模式应为 cmak.zkhosts="host1:2181,host2:2181,host3:2181...."。

### 4. 启动

默认使用 9000 端口，如果端口占用，可以通过参数指定端口：
```
cmak -Dconfig.file=/opt/cmak/conf/application.conf -Dhttp.port=9000 -java-home /opt/jdk-11/Contents/Home
```
参数解释：
- -Dconfig.file：指明 CMAK 配置文件路径
- -Dhttp.port：Web监听端口，默认9000端口
- -java-home：指定 JDK 路径，也可以不指定。这里由于需要用 JDK11，而我这台服务器上也安装了 JDK8，所以需要指定 JDK11 的路径。

启动 CMAK 服务后，通过 `http://localhost:9000/` 地址进入 WEB UI 界面：
![](1)




。。。
