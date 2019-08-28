
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

#### 2.1 语法

`ZooKeeper` 通过构造函数连接服务器。构造函数如下所示:
```java
ZooKeeper(String connectionString, int sessionTimeout, Watcher watcher)
```
`ZooKeeper` 构造函数一共有三个参数：
- `connectionString`: 第一个参数是 `ZooKeeper` 服务器地址（可以指定端口，默认端口号为2181）
- `sessionTimeout`: 第二个参数是以毫秒为单位的会话超时时间。表示 `ZooKeeper` 等待客户端通信的最长时间，之后会声明会话结束。例如，我们设置为5000，即5s，这就是说如果 `ZooKeeper` 与客户端有5s的时间无法进行通信，`ZooKeeper` 就会终止客户端的会话。`ZooKeeper` 会话一般设置超时时间为5-10s。
- `watcher`: 第三个参数是 `Watcher` 对象的实例。`Watcher` 对象接收来自于 `ZooKeeper` 的回调，以获得各种事件的通知。这个对象需要我们自己创建，因为 `Watcher` 定义为接口，所以我们需要自己实现一个类，然后初始化这个类的实例并传入 `ZooKeeper` 的构造函数中。客户端使用 Watcher 接口来监控与 ZooKeeper 之间会话的健康情况。与 ZooKeeper 服务器之间建立或者断开连接时会产生事件。

#### 2.2 Example

下面会创建一个 `ZooKeeperConnection` 类并实现一个 `connect` 方法。 `connect` 方法创建一个 `ZooKeeper` 对象，连接到 `ZooKeeper` 集群，然后返回该对象。`CountDownLatch` 阻塞主进程，直到客户端连接到 `ZooKeeper` 集群。

`ZooKeeper` 通过 `Watcher` 回调返回连接状态。一旦客户端与 `ZooKeeper` 集群连接，`Watcher` 回调函数会被调用，`Watcher` 回调调用 `CountDownLatch`的 `countDown` 方法释放锁。
```java
// 连接ZooKeeper服务器
public class ZooKeeperConnection {

    // ZooKeeper实例
    private ZooKeeper zoo;
    private final CountDownLatch connectedSignal = new CountDownLatch(1);
    // 连接服务器
    public ZooKeeper connect(String host) throws IOException,InterruptedException {
        zoo = new ZooKeeper(host,5000, new Watcher() {
            public void process(WatchedEvent event) {
                Event.KeeperState state = event.getState();
                // 连接成功
                if (state == Event.KeeperState.SyncConnected) {
                    System.out.println("连接ZooKeeper服务器成功");
                    connectedSignal.countDown();
                }
                // 连接失败
                else if (state == Event.KeeperState.Disconnected){
                    System.out.println("连接ZooKeeper服务器失败");
                }
            }
        });

        connectedSignal.await();
        return zoo;
    }

    // 断开与服务器的连接
    public void close() throws InterruptedException {
        zoo.close();
    }
}
```
### 3. 创建Znode

下面要介绍的每种 API 操作都可以分为两种类型:同步和异步。同步操作一般会有返回值，并且会抛出相应的异常。异步操作没有返回值，也不会抛出异常。此外异步方法参数在同步方法参数的基础上，会增加 Callback 和 context 两个参数。

#### 3.1 语法

`ZooKeeper` 类提供了 `create` 方法，用于在 `ZooKeeper` 集群中创建新的 `znode`。`create` 方法如下所示:
```java
// 同步方式
create(String path, byte[] data, List<ACL> acl, CreateMode createMode)
// 异步方式
create(String path, byte[] data, List<ACL> acl, CreateMode createMode, StringCallback cb, Object ctx)
```  

同步版的 `create()` 方法一共有4个参数:
- `path`: 第一个参数是要创建的节点路径。
- `data`: 第二个参数是存储在指定创建节点中的数据。参数类型是字节数组。
- `acl`: 第三个参数是创建节点的访问权限列表。`ZooKeeper` API提供了一个静态接口 `ZooDefs.Ids` 来获取一些基本的 `acl` 列表。
- `createMode`: 第四个参数是创建节点的类型，可以是临时节点，也可以是顺序节点。`CreateMode` 是一个枚举，它有如下取值: `PERSISTENT`、`PERSISTENT_SEQUENTIAL`、`EPHEMERAL`、`EPHEMERAL_SEQUENTIAL`、`CONTAINER`、`PERSISTENT_WITH_TTL` 以及 `PERSISTENT_SEQUENTIAL_WITH_TTL`。

除了同步 `create` 方法中的四个参数以外，异步模式的 `create` 方法还增加了 `cb` 和 `ctx` 两个参数。

#### 3.2 Example

同步操作会返回创建节点的路径，并且会抛出相应的异常。异步操作没有返回值，也不会抛出异常。此外异步方法参数在同步方法参数的基础上，会增加 `cb` 和 `ctx` 两个参数。`StringCallback` 接口中的 `processResult` 方法会在节点创建好之后被调用，它有四个参数:
- `resultCode`: 第一个参数为创建节点的结果码，成功创建节点时，`resultCode` 的值为0。
- `path`: 第二个参数是创建节点的路径。
- `ctx`: 第三个参数是上下文名称，当一个 `StringCallback` 对象作为多个 `create` 方法的参数时，这个参数就很有用了。
- `name`: 第四个参数是创建节点的名字，其实与path参数相同。

```java
package com.zookeeper.example;
import org.apache.zookeeper.*;
import java.io.IOException;

// 创建Znode节点
public class ZKCreate {
    // 创建ZooKeeper实例
    private static ZooKeeper zk;

    // 创建ZooKeeperConnection实例
    private static ZooKeeperConnection conn;

    // 同步方式创建Znode节点
    public static void createNodeSync(String path, byte[] data) throws KeeperException,InterruptedException {
        String nodePath = zk.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        System.out.println("NodePath: " + nodePath);
    }

    // 异步方式创建Znode节点
    public static void createNodeAsync(String path, byte[] data) {
        zk.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT,
                new AsyncCallback.StringCallback() {
                    public void processResult(int resultCode, String path, Object ctx, String name) {
                        System.out.println("ResultCode: " + resultCode);
                        System.out.println("Path: " + path);
                        System.out.println("Ctx: " + ctx);
                        System.out.println("Name: " + name);
                    }
                },
                "create_node_async"
        );
    }

    public static void main(String[] args) throws IOException, InterruptedException, KeeperException {
        // 路径
        String path1 = "/demo/node1";
        String path2 = "/demo/node2";
        // 数据
        byte[] data1 = "zookeeper node1".getBytes();
        byte[] data2 = "zookeeper node2".getBytes();
        // 建立连接
        conn = new ZooKeeperConnection();
        zk = conn.connect("localhost");
        // 同步方式创建节点
        createNodeSync(path1, data1);
        // 异步方式创建节点
        createNodeAsync(path2, data2);
        // 断开连接
        conn.close();
    }
}
```
编译并执行应用程序后，将在 `ZooKeeper` 集群中创建具有指定数据的 `Znode`。你可以使用 `zkCli.sh`进行检查:
```
[zk: 127.0.0.1:2181(CONNECTED) 10] ls /demo
[node2, node1]
```

### 4. 判断Znode是否存在

#### 4.1 语法

`ZooKeeper` 类提供了检查 `Znode` 是否存在的 `exists` 方法。如果指定的 `Znode` 存在，则返回 `Znode` 的元数据。`exists` 方法如下所示:
```java
// 同步方式
Stat exists(String path, boolean watch) throws KeeperException, InterruptedException
// 异步方式
void exists(String path, boolean watch, StatCallback cb, Object ctx)
```
同步 `exists` 方法一共有2个参数:
- `path`: `Znode` 路径。
- `watch`: 指定是否监控指定 `Znode` 的布尔值。

除了同步 `exists` 方法中的两个参数以外，异步模式的 `exists` 方法还增加了 `cb` 和 `ctx` 两个参数。

#### 4.2 Example

`exists` 方法的 `watch` 参数比较特别，如果将其指定为 `true`，那么代表你对该节点的创建、删除以及数据内容变更都感兴趣，所以会响应三种事件类型。
```java
package com.zookeeper.example;
import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import java.io.IOException;

// 判断指定路径节点是否存在
public class ZKExists {
    private static ZooKeeper zk;
    private static ZooKeeperConnection conn;

    // 同步方式判断指定路径的节点是否存在
    public static Stat existSync(String path) throws KeeperException,InterruptedException {
        return zk.exists(path, true);
    }

    // 异步方式判断指定路径的节点是否存在
    public static void existAsync(String path) {
        zk.exists(path, true,
            new AsyncCallback.StatCallback() {
                public void processResult(int resultCode, String path, Object ctx, Stat stat) {
                    System.out.println("ResultCode:" + resultCode);
                    System.out.println("Path: " + path);
                    System.out.println("Ctx: " + ctx);
                    System.out.println("Stat: " + stat);
                }
            }, "exist_async");
    }

    public static void main(String[] args) throws InterruptedException, KeeperException, IOException {
        String path1 = "/demo/node1";
        String path2 = "/demo/node2";
        conn = new ZooKeeperConnection();
        zk = conn.connect("localhost");
        // 同步方式判断指定路径的节点是否存在
        Stat stat = existSync(path1);
        if(stat != null) {
            System.out.println("Node exists and the node version is " + stat.getVersion());
        } else {
            System.out.println("Node does not exists");
        }
        // 同步方式判断指定路径的节点是否存在
        existAsync(path2);
        // 断开连接
        zk.close();
    }
}
```
### 5. 获取节点的数据值

#### 5.1 语法

`ZooKeeper` 类提供了 `getData` 方法来获取指定 `Znode` 中的数据以及状态。`getData` 方法的如下所示:
```java
// 同步方式
byte[] getData(String path, Watcher watcher, Stat stat) throws KeeperException, InterruptedException
// 异步方式
void getData(String path, Watcher watcher, DataCallback cb, Object ctx)
```
同步 `getData` 方法返回值就是节点中存储的数据值，它有三个参数:
- `path`: 第一个参数是节点的路径，用于表示要获取哪个节点中的数据。
- `watcher`: 第二个参数是一个布尔类型的 `watch`。当 `watch` 为 `true` 时，表示我们想要监控这个节点的数据变化。`ZooKeeper` 集群会通过 `Watcher` 回调通知。
- `stat`: 第三个参数用于存储节点的状态信息。

除了同步 `getData` 方法中的三个参数以外，异步模式的 `getData` 方法还增加了 `cb` 和 `ctx` 两个参数。

#### 5.2 Example

在调用 `getData` 方法前，会先构造一个空的 `Stat` 类型对象作为参数传给 `getData` 方法，当 `getData` 方法调用返回后，节点的状态信息会被填充到 `stat` 对象中。

值得注意的是，`zooKeeper` 设置的监听只生效一次，如果在接收到事件后还想继续对该节点的数据内容改变进行监听，需要在事件处理逻辑中重新调用 `getData` 方法并将 `watch` 参数设置为 `true`。
```java
package com.zookeeper.example;
import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import java.io.IOException;

// 获取指定节点中的数据
public class ZKGetData {

    private static ZooKeeper zk;
    private static ZooKeeperConnection conn;

    // 同步方式获取数据
    public static void getDataSync(String path) throws KeeperException, InterruptedException {
        Stat stat = new Stat();
        byte[] data = zk.getData(path, true, stat);
        System.out.println("Data: " + new String(data));
        System.out.println("Stat: " + stat);
    }

    // 异步方式获取数据
    public static void getDataAsync(String path){
        zk.getData(path, true, new AsyncCallback.DataCallback(){
            public void processResult(int resultCode, String path, Object ctx, byte[] data, Stat stat) {
                System.out.println("ResultCode: " + resultCode);
                System.out.println("Path: " + path);
                System.out.println("Ctx: " + ctx);
                System.out.println("Data: " + new String(data));
                System.out.println("Stat: " + stat);
            }
        }, "get_data_async");
    }

    public static void main(String[] args) throws InterruptedException, KeeperException, IOException {
        String path1 = "/demo/node1";
        String path2 = "/demo/node2";
        // 连接服务器
        conn = new ZooKeeperConnection();
        zk = conn.connect("localhost");
        // 同步方式获取数据
        getDataSync(path1);
        // 异步方式获取数据
        getDataAsync(path2);
        // 断开连接
        conn.close();
    }
}
```
### 6. 修改节点的数据内容

#### 6.1 语法

`ZooKeeper` 类提供了 `setData` 方法来修改指定 `Znode`中的数据。 `setData` 方法如下所示:
```java
Stat setData(String path, byte[] data, int version) throws KeeperException, InterruptedException
void getData(String path, boolean watch, DataCallback cb, Object ctx)
```
同步 `setData` 方法有三个参数:
- `path`: 第一个参数是节点的路径
- `data`: 第二个参数是修改的数据值
- `version`: 最后一个参数当前 `Znode` 版本。每当数据发生变化时，`ZooKeeper` 都会更新 `Znode` 的版本号。

除了同步 `setData` 方法中的三个参数以外，异步模式的 `setData` 方法还增加了 `cb` 和 `ctx` 两个参数。

#### 6.2 Example

在调用 `setData` 方法修改节点数据内容时，只有当 `version` 参数的值与节点状态信息中的 `dataVersion` 值相等时，数据才能修改，否则会抛出 `BadVersion` 异常。主要是为了防止丢失数据的更新，在 `ZooKeeper` 提供的API中，所有的写操作都必有 `version` 参数。
```java
package com.zookeeper.example;
import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import java.io.IOException;

// 修改节点的数据内容
public class ZKSetData {
    private static ZooKeeper zk;
    private static ZooKeeperConnection conn;

    // 同步方式修改节点数据内容
    public static void setDataSync(String path, byte[] data) throws KeeperException,InterruptedException {
        zk.setData(path, data, zk.exists(path,true).getVersion());
    }

    // 异步方式修改节点数据内容
    public static void setDataAsync(String path, byte[] data) throws KeeperException, InterruptedException {
        zk.setData(path, data, zk.exists(path,true).getVersion(), new AsyncCallback.StatCallback() {
            public void processResult(int resultCode, String path, Object ctx, Stat stat) {
                System.out.println("ResultCode: " + resultCode);
                System.out.println("Path: " + path);
                System.out.println("Ctx: " + ctx);
                System.out.println("Stat: " + stat);
            }
        }, "set_data_async");
    }

    public static void main(String[] args) throws InterruptedException, KeeperException, IOException {
        String path1 = "/demo/node1";
        String path2 = "/demo/node2";

        byte[] data1 = "zookeeper node1 update success".getBytes();
        byte[] data2 = "zookeeper node2 update success".getBytes();
        // 连接服务器
        conn = new ZooKeeperConnection();
        zk = conn.connect("localhost");
        // 同步方式修改节点数据内容
        setDataSync(path1, data1);
        // 异步方式修改节点数据内容
        setDataAsync(path2, data2);
        // 断开连接
        conn.close();
    }
}
```
### 7. 获取节点的子节点列表

#### 7.1 语法

`ZooKeeper` 类提供 `getChildren` 方法来获取指定 `Znode` 下的所有子节点。`getChildren` 方法如下所示:
```java
// 同步方式
List<String> getChildren(String path, boolean watch) throws KeeperException, InterruptedException
// 异步方式
void getChildren(String path, boolean watch, Children2Callback cb, Object ctx)
```
同步 `getChildren` 方法有两个参数:
- `path`: 第一个参数是节点路径。
- `watch`: 第二个参数是一个布尔类型的 `watch`。当 `watch` 为 `true` 时，表示我们想要监控这个节点的删除操作以及子节点的创建与删除操作。`ZooKeeper` 集群会通过 `Watcher` 回调通知。

除了同步 `getChildren` 方法中的三个参数以外，异步模式的 `getChildren` 方法还增加了 `cb` 和 `ctx` 两个参数。

#### 7.2 Example

```java
package com.zookeeper.example;
import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import java.io.IOException;
import java.util.List;

// 获取节点的子节点列表
public class ZKGetChildren {
    private static ZooKeeper zk;
    private static ZooKeeperConnection conn;

    // 同步方式获取节点的子节点列表
    public static void getChildrenSync(String path) throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(path, true);
        for(int i = 0; i < children.size(); i++) {
            System.out.println("Child: " + children.get(i));
        }
    }

    // 异步方式获取节点的子节点列表
    public static void getChildrenAsync(String path) {
        zk.getChildren(path, true, new AsyncCallback.Children2Callback() {
            public void processResult(int resultCode, String path, Object ctx, List<String> children, Stat stat) {
                System.out.println("ResultCode: " + resultCode);
                System.out.println("Path: " + path);
                System.out.println("Ctx: " + ctx);
                System.out.println("Stat: " + stat);
                for(String child : children) {
                    System.out.println("Child: " + child);
                }
            }
        }, "get_children_async");
    }

    public static void main(String[] args) throws InterruptedException, KeeperException, IOException {
        String path = "/demo";
        // 连接服务器
        conn = new ZooKeeperConnection();
        zk = conn.connect("localhost");
        // 同步方式获取节点的子节点列表
        getChildrenSync(path);
        // 异步方式获取节点的子节点列表
        getChildrenAsync(path);
        // 断开连接
        conn.close();
    }
}
```
### 8. 删除节点

#### 8.1 语法

`ZooKeeper` 类提供了 `delete` 方法来删除指定的 `Znode`。 `delete` 方法如下所示:
```java
// 同步方式
void delete(String path, int version) throws InterruptedException, KeeperException
// 异步方式
void delete(String path, int version, VoidCallback cb, Object ctx)
```
同步 `delete` 方法有两个方法:
- `path`: 节点路径。
- `version`: 最后一个参数当前 `Znode` 版本。每当数据发生变化时，`ZooKeeper` 都会更新 `Znode` 的版本号。

除了同步 `delete` 方法中的两个参数以外，异步模式的 `delete` 方法还增加了 `cb` 和 `ctx` 两个参数。

#### 8.2 Example
```java
package com.zookeeper.example;
import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.ZooKeeper;
import java.io.IOException;

// 删除节点
public class ZKDelete {
    private static ZooKeeper zk;
    private static ZooKeeperConnection conn;

    // 同步方式删除节点
    public static void deleteSync(String path) throws KeeperException,InterruptedException {
        zk.delete(path, zk.exists(path,true).getVersion());
    }

    // 异步方式删除节点
    private static void deleteAsync(String path) throws KeeperException, InterruptedException {
        zk.delete(path, zk.exists(path,true).getVersion(), new AsyncCallback.VoidCallback() {
            public void processResult(int resultCode, String path, Object ctx) {
                System.out.println("ResultCode: " + resultCode);
                System.out.println("Path: " + path);
                System.out.println("Ctx: " + ctx);
            }
        }, "delete_async");
    }

    public static void main(String[] args) throws InterruptedException, KeeperException, IOException {
        String path1 = "/test/node1";
        String path2 = "/test/node2";
        // 连接服务器
        conn = new ZooKeeperConnection();
        zk = conn.connect("localhost");
        // 同步方式删除节点
        deleteSync(path1);
        // 异步方式删除节点
        deleteAsync(path2);
        // 断开连接
        conn.close();
    }
}
```

原文：https://segmentfault.com/a/1190000012262940
