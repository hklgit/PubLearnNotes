

在启动 Flink 组件（如Client，JobManager或TaskManager）之前，需要使用环境变量 HADOOP_CLASSPATH 来配置 classpath。大多数 Hadoop 发行版和云环境在默认情况下都不会设置此变量，因此如果 Flink 选择 Hadoop 类路径，则必须在运行Flink组件的所有计算机上导出环境变量。

在 YARN 上运行时，这通常不是问题，因为在 YARN 中运行的组件将使用 Hadoop 类路径启动，但是在向 YARN 提交作业时，Hadoop 依赖项必须位于类路径中。为此，通常就足够了。
```
export HADOOP_CLASSPATH=`hadoop classpath`
```

when 'high_active_user' then '重度' when 'mid_active_user' then '中度' when 'low_active_user' then '轻度' when 'edge_user' then '边缘用户'


case when [流失前用户活跃分层类型]='miss_user' then '流失用户' else '其他' end
