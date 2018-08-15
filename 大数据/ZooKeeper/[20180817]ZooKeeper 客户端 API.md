
### 1. 开发环境

zookeeper.jar中包含了zookeeper提供的java api，为了在项目中引入zookeeper.jar，在工程的pom.xml文件中加入下面的依赖：
```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.12</version>
</dependency>
```

### 2. 连接zk服务器

构造 ZooKeeper 类对象的过程就是与ZK服务器建立连接的过程。这个类是客户端API中的主要类，用于维护客户端和 ZooKeeper 服务之间的连接：
```java
ZooKeeper zooKeeper = new ZooKeeper("192.168.1.108:2181", 5000, watcher);
```
ZooKeeper 类的构造函数一共有三个参数：第一个参数是 ZooKeeper 服务器的地址（可以指定端口，默认端口号为2181），第二个参数是以毫秒为单位的会话超时时间（这里我们设置为5秒），第三个参数是 Watcher 对象的实例。Watcher 对象接收来自于 Zookeeper 的回调，以获得各种事件的通知。

当一个 ZooKeeper 的实例被创建时，会启动一个线程连接到 ZooKeeper 服务。由于对构造函数的调用是立即返回的，因此在使用新建的 ZooKeeper 对象前一定要等待其与 ZooKeeper 服务之间成功建立连接。我们使用 Java 的 CountDownLatch 类（位于 java.util.concurrent 包中）来阻止使用新建的 ZooKeeper 对象，直到这个 ZooKeeper 对象已经准备就绪。Watcher 类用于获取 ZooKeeper 对象是否准备就绪的信息，在它的接口中只有一个方法：
```java

```




















...
