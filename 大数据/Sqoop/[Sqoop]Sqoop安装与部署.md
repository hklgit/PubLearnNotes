## 1. 下载
```
http://www.apache.org/dyn/closer.lua/sqoop/1.4.6
```
## 2. 解压
```
xiaosi@Qunar:~$ sudo tar -zxvf sqoop-1.4.6.bin__hadoop-2.0.4-alpha.tar.gz -C /opt
```
进行重命名：
```
xiaosi@Qunar:/opt$ sudo mv sqoop-1.4.6.bin__hadoop-2.0.4-alpha sqoop-1.4.6
```
## 3. 配置环境变量
```
# sqoop
export SQOOP_HOME=/opt/sqoop-1.4.6
export PATH=${SQOOP_HOME}/bin:$PATH
```
## 4. 配置文件
```
xiaosi@Qunar:/opt/sqoop-1.4.6/conf$ sudo mv  sqoop-env-template.sh  sqoop-env.sh
```
进行如下修改：
```
# Set Hadoop-specific environment variables here.
#Set path to where bin/hadoop is available
export HADOOP_COMMON_HOME=/opt/hadoop-2.7.2
#Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/opt/hadoop-2.7.2
#set the path to where bin/hbase is available
#export HBASE_HOME=
#Set the path to where bin/hive is available
export HIVE_HOME=/opt/apache-hive-2.0.0-bin
#Set the path for where zookeper config dir is
#export ZOOCFGDIR=/opt/zookeeper-3.4.8
```
如果数据读取不涉及hbase和hive，那么相关hbase和hive的配置可以不用配置；如果集群有独立的zookeeper集群，那么配置zookeeper，反之，不用配置。

## 5. Copy Jar包

所需的包：hadoop-core包、mysql的jdbc包（或Oracle的jdbc包等）
```
xiaosi@Qunar:~$ sudo mv hadoop-core-1.2.1.jar /opt/sqoop-1.4.6/lib/
xiaosi@Qunar:~$ sudo mv mysql-connector-java-5.1.38.jar /opt/sqoop-1.4.6/lib/
```
## 6. 测试验证
```
xiaosi@Qunar:/opt/sqoop-1.4.6/bin$ sqoop list-databases --connect jdbc:mysql://localhost:3306 --username root -password root
Warning: /opt/sqoop-1.4.6/../hbase does not exist! HBase imports will fail.
Please set $HBASE_HOME to the root of your HBase installation.
Warning: /opt/sqoop-1.4.6/../hcatalog does not exist! HCatalog jobs will fail.
Please set $HCAT_HOME to the root of your HCatalog installation.
Warning: /opt/sqoop-1.4.6/../accumulo does not exist! Accumulo imports will fail.
Please set $ACCUMULO_HOME to the root of your Accumulo installation.
16/10/08 15:43:03 INFO sqoop.Sqoop: Running Sqoop version: 1.4.6
16/10/08 15:43:03 WARN tool.BaseSqoopTool: Setting your password on the command-line is insecure. Consider using -P instead.
16/10/08 15:43:03 INFO manager.MySQLManager: Preparing to use a MySQL streaming resultset.
information_schema
hive_db
mysql
performance_schema
phpmyadmin
test
`








