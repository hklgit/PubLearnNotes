
Watcher可以算是ZooKeeper中比较难以理解的部分，但又是十分重要的部分。我看了很多博文，发现博文上都会说一句“ZooKeeper的Watcher机制是一次性的，再次使用时需要重新注册”，而对怎样再次注册没有过多提过。（即使它简单！！）在查阅了官方的文档深入理解后，我发现这句“Watcher是一次性”都不是特别准确。

注意：

1. Watcher主要分为两种：Default Watcher 和Not Default Watcher

  （a）Default Watcher 是指我们在创建ZooKeeper时传入的参数


ZooKeeper zk = new ZooKeeper(String connectString, int sessionTimeout, Watcher watcher) ;
Default Watcher 的生命周期是整个Session对话，换句话说，它并不是一次性的。其次defaultWatcher并不与某一个节点路径相互关联。
 (b) Not Default Watcher是在调用getData()、getChildren()、exists()时传入的Watcher对象


public byte[] getData(String path,Watcher watcher, Stat stat)throws KeeperException,InterruptedException
Not Default Watcher不具有Defaul tWatcher会监听session的生命周期，比如session创建成功了、失效了等这个职责。


`ZooKeeper` 中的所有读取操作 -  `getData()`，`getChildren()` 以及 `exists()` - 都可以设置监视点。下面是对 `ZooKeeper` 监视点的定义：监视点事件是只能触发一次。当监控的数据发生变化时就会发发送到设置监视点的客户端上。对监视点定义中需要注意三个要点：

(1) 触发一次(`One-time trigger`)
当数据发生变化时，将向客户端发送一个监控事件。例如，如果客户端执行 `getData("/znode1"，true)`，后面对 `/znode1` 的更改或删除，客户端都会获得 `/znode1` 的监控事件。如果 `/znode1` 再次更改，如果客户端没有执行新一次设置新监视点的读取，是不会发送监视事件的。

(2) Sent to the client
监视点是以异步方式发送给 `Watcher`。ZooKeeper提供了顺序保证: 客户端在第一次看到监视事件之前，永远不会看到设置监视的更改。网络延迟或其他因素可能会导致不同的客户端会在不同时间点看到更新的监视点以及返回代码。关键点在于，不同客户看到的所有内容都会有一致的顺序。

(3) 数据监视点和子节点监视点
可以认为 `ZooKeeper` 维护两个监视点列表: 数据监视点和子节点监视点。`getData()` 和 `exists()` 设置数据监视点。`getChildren()` 设置子节点监视点。或者，可以根据返回的数据类型来思考设置的监视点。`getData()` 和 `exists()` 返回有关节点数据的信息，而  `getChildren()` 返回子节点列表。因此，`setData()` 会触发正在设置的 `Znode` 的数据监视点。`create()` 会触发正在创建 `Znode` 的数据监视点以及父 `Znode` 的子节点监视点。`delete()` 会同时触发正要删除 `Znode` 的数据监视点以及子节点监视点，以及父 `Znode` 的子节点监视点。







英译对照:
 - `Watch`: 监视点






...
