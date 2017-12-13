
检查点(`Checkpointing`)是在`HDFS`中维护和保存文件系统元数据的重要组成部分。这对快速的恢复和重启`NameNode`是至关重要的，也是集群整体健康状况的重要指标。

在这篇文章中，将解释`HDFS`中检查点的用途，在不同集群配置下检查点工作原理的技术细节，然后介绍一系列关于此功能的操作问题和重要的bug修复。




























原文:http://blog.cloudera.com/blog/2014/03/a-guide-to-checkpointing-in-hadoop/
