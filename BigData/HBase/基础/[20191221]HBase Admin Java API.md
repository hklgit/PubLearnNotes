
### 1. 连接集群

Java 连接 HBase 需要两个类：
- `HBaseConfiguration`
- `ConnectionFactory`

要连接 HBase，我们首先需要通过 HBaseConfiguration 创建 Configuration 对象。HBaseConfiguration 用于读取指定路径下 hbase-site.xml 和 hbase-default.xml 的配置信息：
```java
// 使用create()静态方法就可以得到Configuration对象
Configuration config = HBaseConfiguration.create();
```
然后通过 ConnectionFactory 获取到 Connection 对象：
```java
Connection connection = ConnectionFactory.createConnection(config);
```
使用这两个步骤就能完成连接 HBase 了。

### 2. 
