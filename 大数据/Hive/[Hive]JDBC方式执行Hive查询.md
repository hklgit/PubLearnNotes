

### 1. 启动hiveServer2服务

在使用JDBC执行Hive查询时, 必须首先开启Hive的远程服务

#### 1.1 检查服务是否启动

hiveServer2的thrift的默认端口为10000:
```
<property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
    <description>Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'binary'.</description>
</property>
```
通过端口来判断hiveServer2是否启动:
```
xiaosi@Qunar:~$ sudo netstat -anp | grep 10000
```
#### 1.2 启动服务

如果没有启动hiveServer2服务，通过如下命令启动：
```
xiaosi@Qunar:~$ hive --service hiveserver2 >/dev/null 2>/dev/null &
[1] 11978
```

### 2. 
