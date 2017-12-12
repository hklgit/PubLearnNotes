
根据各种参数(如数据大小或集群中的机器数量)，Flink的优化器自动会为你的程序选择一个执行策略。在很多情况下，准确的知道Flink如何执行你的程序是很有帮助的。

### 1. 执行计划可视化工具

Flink内置一个执行计划的可视化工具。包含可视化工具的HTML文档位于`tools/planVisualizer.html`下。用JSON表示作业执行计划，并将其可视化为执行策略具有完整注释的图(visualizes it as a graph with complete annotations of execution strategies)。

备注:
```
打开可视化工具的方式有所改变:由本地文件 tools/planVisualizer.html 改为 url http://flink.apache.org/visualizer/index.html
```

以下代码显示了如何从程序中打印执行计划的JSON：

Java版本:
```java
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
...
System.out.println(env.getExecutionPlan());
```

Scala版本:
```
val env = ExecutionEnvironment.getExecutionEnvironment
...
println(env.getExecutionPlan())
```

要可视化执行计划，请执行以下操作：

(1) 使用浏览器打开`planVisualizer.html`(或者直接在浏览器中输入http://flink.apache.org/visualizer/index.html 网址)

![](http://img.blog.csdn.net/20171117100332547?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3VubnlZb29uYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

(2) 将JSON字符串粘贴到文本框中

(3) 点击Draw按钮

完成上面这些步骤后，将会显示详细的执行计划。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/plan_visualizer.png)

### 2. Web界面

Flink提供了一个用于提交和执行作业的Web界面。这个界面是JobManager Web监控界面的一部分，默认情况下在端口8081上运行。通过这个界面提交作业需要你在`flink-conf.yaml`中设置`jobmanager.web.submit.enable：true`。

你可以在作业执行之前指定程序参数。执行计划可视化器使你能够在执行Flink作业之前查看执行计划。


原文:https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/execution_plans.html
