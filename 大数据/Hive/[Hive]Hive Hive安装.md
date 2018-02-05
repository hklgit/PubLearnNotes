---
layout: post
author: sjf0115
title: Hive Hive安装与启动
date: 2018-02-02 12:01:01
tags:
  - Hive

categories: Hive
permalink: hive-install-and-config
---

### 1. 下载

可以从http://hive.apache.org/downloads.html下载你想要的版本，在这我们使用的是2.1.0版本

### 2. 解压

把下载好的文件解压到~/opt目录下：
```
xiaosi@yoona:~$ tar -zxvf apache-hive-2.1.0-bin.tar.gz -C opt/
```
### 3. 配置

根据模板创建配置文件
```
xiaosi@yoona:~/opt/hive-2.1.0/conf$ cp hive-default.xml.template hive-site.xml
xiaosi@yoona:~/opt/hive-2.1.0/conf$ cp hive-log4j2.properties.template hive-log4j2.properties
```
修改配置文件
```
<property>
   <name>javax.jdo.option.ConnectionURL</name>
   <value>jdbc:mysql://localhost:3306/hive_meta?createDatabaseIfNotExist=true</value>
   <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
   </description>
</property>
<property>
   <name>javax.jdo.option.ConnectionDriverName</name>
   <value>com.mysql.jdbc.Driver</value>
   <description>Driver class name for a JDBC metastore</description>
</property>
<property>
   <name>javax.jdo.option.ConnectionUserName</name>
   <value>root</value>
   <description>Username to use against metastore database</description>
</property>
<property>
   <name>javax.jdo.option.ConnectionPassword</name>
   <value>root</value>
   <description>password to use against metastore database</description>
</property>
<property>
   <name>hive.metastore.warehouse.dir</name>
   <value>/user/hive/warehouse</value>
   <description>location of default database for the warehouse</description>
</property>
```
### 4. Hive元数据

创建存储Hive元数据的数据库，这里我们使用mysql存储：
```
mysql> create database hive_meta;
Query OK, 1 row affected (0.00 sec)
```
### 5. 设置环境变量

在/etc/profile文件下添加如下配置：
```
# hive path
export HIVE_HOME=/home/xiaosi/opt/hive-2.1.0
export PATH=${HIVE_HOME}/bin:$PATH
```
### 6. 启动

运行bin/hive命令，即可启动：
```
xiaosi@yoona:~$ cd $HIVE_HOME
xiaosi@yoona:~/opt/hive-2.1.0$ ./bin/hive
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/home/xiaosi/opt/hive-2.1.0/lib/log4j-slf4j-impl-2.4.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/xiaosi/opt/hadoop-2.7.3/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Logging initialized using configuration in file:/home/xiaosi/opt/hive-2.1.0/conf/hive-log4j2.properties Async: true
Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
hive>
```
如果遇到问题，可以查看[Hive]那些年踩过的Hive坑：http://blog.csdn.net/sunnyyoona/article/details/51648871
