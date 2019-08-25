
`ZooKeeper` API 的核心部分是 `ZooKeeper` 类。在构造函数中提供一些参数来连接 `ZooKeeper`，并提供如下方法:
- `connect` − 连接 `ZooKeeper` 服务器。
- `create` − 创建一个 `ZNode` 节点。
- `exists` − 检查指定的节点是否存在。
- `getData` − 从指定的节点获取数据。
- `setData` − 为指定的节点设置数据。
- `getChildren` − 获取指定节点的所有子节点。
- `delete` − 删除指定节点以及子节点。
- `close` − 关闭连接。

### 1. 开发环境

在工程的 `pom.xml` 文件中加入下面的依赖：
```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.5.5</version>
</dependency>
```
> 截止目前最新版本为 3.5.5 版本。

### 2. 连接ZooKeeper服务器

`ZooKeeper` 通过构造函数连接服务器。构造函数如下所示:
```java
ZooKeeper(String connectionString, int sessionTimeout, Watcher watcher)
```
`ZooKeeper` 构造函数一共有三个参数：
- `connectionString`: 第一个参数是 `ZooKeeper` 服务器地址（可以指定端口，默认端口号为2181）
- `sessionTimeout`: 第二个参数是以毫秒为单位的会话超时时间。表示 `ZooKeeper` 等待客户端通信的最长时间，之后会声明会话结束。例如，我们设置为5000，即5s，这就是说如果 `ZooKeeper` 与客户端有5s的时间无法进行通信，`ZooKeeper` 就会终止客户端的会话。`ZooKeeper` 会话一般设置超时时间为5-10s。
- `watcher`: 第三个参数是 `Watcher` 对象的实例。`Watcher` 对象接收来自于 `ZooKeeper` 的回调，以获得各种事件的通知。这个对象需要我们自己创建，因为 `Watcher` 定义为接口，所以我们需要自己实现一个类，然后初始化这个类的实例并传入 `ZooKeeper` 的构造函数中。客户端使用 Watcher 接口来监控与 ZooKeeper 之间会话的健康情况。与 ZooKeeper 服务器之间建立或者断开连接时会产生事件。




当一个 ZooKeeper 的实例被创建时，会启动一个线程连接到 ZooKeeper 服务。由于对构造函数的调用是立即返回的，因此在使用新建的 ZooKeeper 对象前一定要等待其与 ZooKeeper 服务之间成功建立连接。我们使用 Java 的 CountDownLatch 类（位于 java.util.concurrent 包中）来阻止使用新建的 ZooKeeper 对象，直到这个 ZooKeeper 对象已经准备就绪。Watcher 类用于获取 ZooKeeper 对象是否准备就绪的信息，在它的接口中只有一个方法：
```java

```




















原文：https://segmentfault.com/a/1190000012262940
