
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
当数据发生变化时，将向客户端发送一个监视事件。例如，如果客户端执行 `getData("/znode1"，true)`，后面对 `/znode1` 的更改或删除，客户端都会获得 `/znode1` 的监视事件。如果 `/znode1` 再次更改，如果客户端没有执行新一次设置新监视点的读取，是不会发送监视事件的。









英译对照:
 - `Watch`: 监视点






...
