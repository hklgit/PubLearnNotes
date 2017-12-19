`Secondary Namenode`是`Hadoop`中命名不当组件其中之一。不当命名很容易造成歧义，通过`Secondary Namenode`这个名字，我们很容易理解为是一个备份`NameNode`，但实际上它不是。很多`Hadoop`的初学者对`Secondary Namenode`究竟做了什么以及为什么存在于`HDFS`中感到困惑。因此，在这篇博文中，我试图解释`HDFS`中`Secondary Namenode`的作用。

通过它的名字，你可能会认为它和`Namenode`有关，它确实是`NameNode`相关。所以在我们深入研究`Secondary Namenode`之前，让我们看看`Namenode`究竟做了什么。


`Namenode`保存HDFS的元数据，如名称空间信息，块信息等。使用时，所有这些信息都存储在主存储器中。 但是这些信息也存储在磁盘中用于持久性存储。


























原文:http://blog.madhukaraphatak.com/secondary-namenode---what-it-really-do/
