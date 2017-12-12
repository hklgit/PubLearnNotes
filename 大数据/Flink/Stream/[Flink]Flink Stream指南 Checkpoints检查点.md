
### 1. 概述

检查点通过允许状态和相应的流位置被恢复来使Flink容错状态成为可能，从而为应用程序提供与无故障执行相同的语义。

请参阅检查点以了解如何为您的程序启用和配置检查点。

### 2. 外部检查点

检查点默认情况下不会在外部持续存在，只能用于从故障中恢复作业。当一个程序被取消时它们被删除。但是，你可以将定期检查点配置为与保存点类似地在外部持久保存。这些外部持久化的检查点将其元数据写入持久性存储中，即使在作业失败时也不会自动清理。这样，如果你的作业失败时，你会有一个检查点用于恢复作业。

```java
CheckpointConfig config = env.getCheckpointConfig();
config.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```
`ExternalizedCheckpointCleanup`模式配置当你取消作业时外部化检查点如何操作：

(1) `ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION`：作业取消时保留外部化检查点。请注意，在这种情况下，你必须手动清除取消后的检查点状态。

(2) `ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION`: 作业取消时，删除外部化的检查点。检查点状态只有在作业失败时才可用。

#### 2.1 目录结构

与保存点类似，外部化检查点由元数据文件组成，并根据状态终端包含一些其他数据文件。外部化检查点元数据的目标目录是由配置属性`state.checkpoints.dir`确定的，目前它只能通过配置文件来设置。

```
state.checkpoints.dir: hdfs:///checkpoints/
```

该目录包含恢复检查点所需的检查点元数据。对于`MemoryStateBackend`，这个元数据文件已经包含了，不需要其他的文件。

#### 2.2 与保存点的区别

#### 2.3 从外部化检查点恢复

































原文:https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/state/checkpoints.html
