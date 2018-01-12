---
layout: post
author: sjf0115
title: Stream 批处理之外的流式世界(2)
date: 2018-01-09 17:54:01
tags:
  - Stream

categories: Stream
---

### 1. 总结和路线图

在[Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)中，我们首先澄清了一些术语。然后我开始区分有限数据和无限数据。有限数据源的大小是有限的，通常被称为`batch`数据。无限数据源可能具有无限大小，并且通常被称为`streaming`数据。我尽量避免使用`batch`和`streaming`术语来指代数据源，因为这些名称带有一些误导并经常受到限制。

然后，我继续定义了批处理引擎和流引擎之间的区别：批处理引擎是那些仅为有限数据而设计的引擎，而流引擎是设计时考虑到了无限数据。我的目标是在引用执行引擎时只使用`batch`和`streaming`术语。

在术语之后，我介绍了与处理无限数据有关的两个重要概念。我首先确定了`事件时间`(事件发生的时间)和`处理时间`(处理期间观察到时间)之间的关键区别。这为[Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)提出的主要论点之一提供了基础：如果你关心事件实际发生的正确性和上下文，则必须分析与固有事件时间相关的数据，而不是分析它们的在处理过程中遇到处理时间。

然后，我介绍了窗口的概念(即，将数据集按时间界线划分)，这是一种常见的方法，用于处理无限数据源在技术上永远不会结束的事实。窗口化策略中比较简单的是固定窗口的和滑动窗口，也还有更复杂的窗口类型，例如会话窗口(其中窗口由数据本身的特征定义，例如捕获每个用户的活动会话，紧跟不活跃的间隙)应用也比较广泛。

除了这两个概念之外，我们现在要仔细研究一下三个概念：
- `Watermarks`是关于事件时间输入完整性的概念。具有时间X值的`watermark`表示：`已经观察到所有输入数据并且事件时间小于X`。因此，当不知道无限数据源什么时候结束时，`watermark`就用作进度的度量。
- `Triggers`是一种机制，用于声明窗口输出何时应相对于某个外部信号实现。触发器在选择何时发送输出方面提供了灵活性。它们还可以随着时间的变化多次观察窗口的输出(observe the output for a window multiple times as it evolves)。这为随着时间的推移而修改结果提供了可能，这又开启了随时间改进结果的大门，这允许在数据到达时提供推测结果，并且随时间处理上游数据的变化或相对于`watermark`迟到的数据(例如，移动场景 ，其中某个人的电话在该人离线时记录各种动作和他们的事件时间，然后在重新获得连接时继续上传这些事件进行处理。)
- `Accumulation`

最后，因为我认为理解这些概念之间的关系比较容易，我们将重新回顾以前的问题，并在回答下面四个问题中探索新的问题，所有这些问题对于每一个无限数据处理问题都是至关重要的：

(1) `What`计算出什么样的结果？这个问题由流水线内的`转换类型`来回答。这包括诸如计算总和，构造直方图，训练机器学习模型等。这也是经典批处理回答的问题。

(2) `Where`事件发生的时间是在哪里计算的？ 这个问题将由管道内的`基于事件时间的窗口`来回答。这包括[Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)介绍的窗口常见示例(固定，滑动和会话窗口)，不使用窗口概念的用例(例如，[Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)中描述的与时间无关的处理；经典的批处理也通常属于这种类别)以及其他更复杂的窗口类型，例如有时间限制的拍卖。还要注意，也可以包含处理时间窗口，如果当记录到达系统时将摄入时间作为事件时间时。

(3) `When`处理时间什么时候被物化的？这个问题通过使用`watermark`和`触发器`来回答。在这个主题上有无限的变化，但是最常见的模式是使用`watermark`来描述给定窗口的输入是否完成，使用触发器允许指定早期结果(在窗口完成之前发送推测的部分结果)和后期结果(`watermark`仅仅是对完整性的估计，在`watermark`声明给定窗口的输入完成之后可能到达更多的输入数据的情况)。

(4) `How`如何使结果更加精致？这个问题由所使用的累积(`accumulation`)类型来回答：丢弃(结果都是独立的和不同的)，累积(后来的结果建立在先前的结果之上)，或者累积和撤回(累积值加上撤回先前发送的已被触发值)(the accumulating value plus a retraction for the previously triggered value(s) are emitted)。

在这篇文章的其余内容，我们将更详细地讨论这些问题。

### 2. Streaming 101 总结

首先，让我们回顾一下在[Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)中提出的一些概念，但是这次还有一些详细的例子，这些例子将有助于我们更好的理解这些概念。

#### 2.1 What: transformations

在经典的批处理应用中的转换操作回答了这个问题：`计算出什么样的结果？`，即使你们中的很多人可能对经典的批处理已经很熟悉了，我们还是从那里开始，因为它是我们添加所有其他的概念的基础。

在本节中，我们将看到一个简单的例子：在由10个值组成的简单数据集上计算键控整数和。如果你想要一个更实际一点的话，你可以把它想象成一个个人团队通过把自己的独立分数结合在一起来玩某种手机游戏的整体得分。 您可以想象它对于计费和使用情况监控用例同样适用。

对于每个示例，我将包含一个`Dataflow Java SDK`伪代码的简短片段，以更好的了解管道的定义。从某种意义上说，这是伪代码，有时我会略作修改以使示例更清晰，更详细(比如使用具体的I/O源)，或者简化名称(`Java`中当前的触发器名称非常冗长；为了清晰，我将使用更简单的名称)。除了那些小的东西(其中大部分我在`Postscript`中明确列举)之外，基本上都是真实的`Dataflow SDK`代码。我还会在后面为那些对类似例子(可以编译和运行)感兴趣的人提供一个实际代码演练的链接。

如果你熟悉`Spark Streaming`或`Flink`等类似的工具，那么对于理解`Dataflow`代码是比较容易的。为了给你一个崩溃的过程，在数据流中有两个基本的术语：
- `PCollections`表示执行并行转换操作的数据集(可能是大数据集)(因此名称以`p`开头)。
- `PTransforms`，将其应用于`PCollections`来创建新的`PCollections`。`PTransforms`可以执行元素转换，可以将多个元素聚合在一起，也可以是其他`PTransforms`的复合组合。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Other/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C-12.jpg?raw=true)

就我们的例子而言，我们首先假定认为`PCollection< KV<String，Integer> >`为`输入`(即由`Strings`和`Integer`的键/值对组成的`PCollection`，其中的`Strings`像是团队名称，`Integer`是对应团队的任意人的分数)。在现实世界的流水线中，我们将通过从`I/O`数据源读取原始数据(例如，日志记录)的`PCollection`来获取输入，然后通过将日志记录解析为适当的键/值对将其转换为`PCollection< KV<String，Integer> >`。 为了清楚起见，在第一个例子中，我将包含这些所有步骤的伪代码，但在随后的例子中，我忽略了`I/O`和解析部分。

因此，管道会简单地从`I/O`源读取数据，解析出团队/分数键值对，并计算每个团队的分数总和:
```
PCollection<String> raw = IO.read(...);
PCollection<KV<String, Integer>> input = raw.apply(ParDo.of(new ParseFn());
PCollection<KV<String, Integer>> scores = input.apply(Sum.integersPerKey());
```
上述代码从`I/O`源读取键/值对数据，其中以`String`(例如，团队名称)作为键和`Integer`(例如，团队每个成员分数)作为值。然后将每个键对应的值相加在一起以在输出集合中产生键对应数据的总和(例如一个团队的总得分)。

对于所有的例子来说，在看到描述管道的代码片段之后，我们将看看在具体数据集上执行该管道的动画渲染。更具体地说，我们将看到在的单个key的10个输入数据(唯一的一个key对应10个值)上执行管道的过程。在一个真正的管道中，你可以想象类似的操作将会在多台机器上并行执行，但是在我们的例子中尽量简单化。

每个动画在两个维度上绘制输入和输出：`事件时间`(在X轴上)和`处理时间`(在Y轴上)。因此，如粗的上升白线所示，管道从下到上的进度可实时观察。输入是圆圈，圆圈内的数字代表特定记录的值。当管道观察到它们时，它们开始改变之前灰色而变成白色。

当管道观察到值时，将它们累加在其状态中，并最终将聚合结果输出。状态和输出由矩形表示，聚合值靠近顶部，矩形覆盖的区域表示部分事件时间和处理时间累加到结果中。对于下图中的管道，在经典的批处理引擎上执行时看起来就像这样：

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-1.png?raw=true)

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-2.png?raw=true)

由于这是一个批处理管道，因此它会累积状态，直到看到所有输入(顶部的绿色虚线出现时表示看到所有的输入了)，此时它将产生单一输出`51`。在此示例中，我们是在所有事件时间上计算的总和，因为我们没有使用任何指定的窗口转换操作；因此状态和输出的矩形覆盖整个X轴。但是，如果我们想处理一个无限的数据源，那么经典的批处理是不够的。我们不能等待输入结束，因为它永远不会结束。我们需要的一个概念就是窗口化，我们在上篇文章中引入了这个概念。因此，在第二个问题的上下文中：`事件发生的时间是在哪里计算的？`，现在我们简要回顾一下窗口。

### 2.2 Where: windowin

正如上次讨论的那样，窗口化是沿着时间界线分割数据源的过程。常见的窗口策略包括固定窗口，滑动窗口和会话窗口：

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-3.jpg?raw=true)

为了更好地在实践中理解在窗口，让我们以整数求和管道为例，并将它窗口化为2分钟的固定窗口。使用`Dataflow SDK`比较简单，添加`Window.into`转换操作即可：
```
PCollection<KV<String, Integer>> scores = input
  .apply(Window.into(FixedWindows.of(Duration.standardMinutes(2))))
  .apply(Sum.integersPerKey());
```
回想一下，`Dataflow`提供了一个统一的模型，可以在批处理和流式处理中使用，因为语义上批处理只是流式处理的一个特殊情况。因此，我们首先在批处理引擎上执行这个管道；机制比较简单，可以与切换到的流处理引擎直接进行比较。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-3.png?raw=true)

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-4.png?raw=true)

和以前一样，输入在状态下累积，直到完全消费掉，最后输出。然而，在这种情况下，我们得到四个输出，而不是一个输出：四个相关的两分钟事件时间窗口对应一个输出。

此刻，我们回顾了[Streaming 101]()中引入的两个主要概念：`事件时间`和`处理时间`之间的关系，以及窗口化。如果我们想继续探讨，我们需要开始添加本节涉及的新概念：水位线`watermark`，触发器`triggers`和累积器`accumulation`。

### 3. Streaming 102

我们刚才观察到在批处理引擎上执行一个窗口的管道。但理想情况下，我们希望结果具有较低的延迟。切换到流式引擎是朝着正确的方向迈出了一步，但对于批处理引擎我们都知道，每个窗口的输入都是完整性的(即一旦有限输入源中的所有数据都已被消费)，但是我们目前缺乏对于无限数据源确定其完整性的实际方法。

#### 3.1 When: watermarks

`watermark`是问题"处理时间什么时候被物化的？"答案的前半部分。`watermark`是输入数据完整性的一个事件时间概念。换一种说法，它是系统相对于在事件流中正处理事件的事件时间进行衡量进度和完整性的方法(they are the way the system measures progress and completeness relative to the event times of the records being processed in a stream of events)(无论是有限还是无限数据，尽管在无限数据中作用更明显)。

回想一下`Streaming 101`中这个图，在这里稍作修改，这里我将大多数现实世界分布式数据处理系统中事件时间和处理时间之间的偏差描述为不断变化的函数。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-5.png?raw=true)

代表现实世界的那个弯弯曲曲的红线，实际上就是`watermark`；随着处理时间的推移能够获取事件时间完整性的进展(it captures the progress of event time completeness as processing time progresses.)。从概念上将，可以`watermark`看作为一个函数，`F（P） - > E`，输入一个处理时间点输出一个事件时间点。(更准确地说，函数的输入实际上是某个时间点在管道中观察到`watermark`的上游的所有东西的当前状态：输入源，缓冲的数据，正在处理的数据等；但从概念上讲，可以简单的理解为将处理时间到事件时间的映射)(More accurately, the input to the function is really the current state of everything upstream of the point in the pipeline where the watermark is being observed: the input source, buffered data, data actively being processed, etc.; but conceptually, it’s simpler to think of it as a mapping from processing time to event time.)。事件时间点E是表示事件时间小于E的那些所有输入数据都已经被看到了。换句话说，我们断言不会再看到事件时间小于E的其他数据了。根据`watermark`的类型，完美或启发式，上述断言分别是一个严格保证的或一个与依据的猜测：
- `Perfect watermarks:`：在对所有输入数据充分了解的情况下，可以构建`Perfect watermarks`；在这种情况下，没有延迟的数据；所有数据要不提前，要不准时。
- `Heuristic watermarks`：对于许多分布式输入源，充分了解输入数据是不切实际的，在这种情况下，下一个最佳选择是`Heuristic watermarks`。`Heuristic watermarks`使用任何可以获取到的输入信息(分区，分区内的排序(如果有的话)，文件的增长率等)来提供尽可能准确的进度估计。在许多情况下，这样的`watermark`在预测中是非常准确的。即使如此，`Heuristic watermarks`的使用意味着有时可能是错误的，这将导致延迟数据。我们将在下面的触发器部分中了解如何处理延迟数据。

`watermark`是一个非常吸引人并且复杂的话题。现在，为了更好地理解`watermark`的作用以及它的一些缺点，我们来看看两个仅使用`watermark`的流引擎的例子，确定在执上述代码中的窗口化管道时何时实现输出(materialize output)。左边的例子使用的是`Perfect watermarks`; 右边使用的是`Heuristic watermarks`。

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-6.png?raw=true)

![](https://github.com/sjf0115/PubLearnNotes/blob/master/image/Stream/%E6%89%B9%E5%A4%84%E7%90%86%E4%B9%8B%E5%A4%96%E7%9A%84%E6%B5%81%E5%BC%8F%E4%B8%96%E7%95%8C%E4%B9%8B%E4%BA%8C-7.png?raw=true)

在这两种情况下，随着`watermark`到达窗口的末端，窗口被实现(windows are materialized)。两次执行的主要区别在于右侧的`watermark`计算使用的是启发式算法没有考虑到`9`这个值，这很大程度上改变了`watermark`的形状。这些例子突出了`watermark`(以及其他的完整性概念)的两个缺点，具体来说可以是：

(1)太慢：当任何类型的`watermark`由于已知未处理的数据(例如，由于网络带宽限制而缓慢增长的输入日志)明确地延迟时，如果`watermark`的前进是唯一影响结果的因素，那么直接转换为在输出中的延迟(When a watermark of any type is correctly delayed due to known unprocessed data that translates directly into delays in output if advancement of the watermark is the only thing you depend on for stimulating results.)。

这在左图中是最明显的，其中延迟到达的数据9在所有后续窗口中都保持`watermark`，即使这些窗口的输入数据更早完成。这对于第二个窗口`[12：02,12：04]`尤其明显，从窗口中第一个值出现开始，直到我们看到窗口的所有结果为止，花费将近7分钟的时间。这个例子中的`Heuristic watermarks`不会遇到相同的问题(到输出5分钟)，但不要认为`Heuristic watermarks`永远不会受到`watermark`延迟的影响；这实际上只是我在这个具体例子中选择从启发式水印中省略的记录的结果。

这里非常重要的一点是：虽然`watermark`提供了一个非常有用的完整性概念，但是从延迟角度来看，根据完整性生成输出往往不是很理想的。设想一个仪表板，其中包含有价值的指标，按小时或天划分窗口。你不太可能要等一小时或一天才能看到当前窗口的结果；这是相比于此系统使用经典批处理系统的难点之一。相反，随着时间推移，窗口根据输入进行更改，最终变的完整，这种方式更好一些。

(2)太快：当`Heuristic watermarks`错误的比原本的提前了，在`watermark`之前的带有事件时间的数据可能延迟到达，并产生延迟数据。在右边的例子就发生了这样的情况：在观察到该窗口的所有输入数据之前，`watermark`就提前到达第一个窗口末端，导致错误的输出值`5`而不是`14`。这个缺点是`Heuristic watermarks`的一个严重问题；它们的启发特性意味着它们有时会出错。因此，如果你关心正确性，只依靠它们来确定何时输出是不够的。

在`Streaming 101`中，我对完整性的概念做了一些强调性的描述，它不足以应对无限数据流的无序处理。上述两个缺点，`watermark`太慢或太快，是这个论点的基础。你不能寄希望系统只依赖完整性就能获得低延迟和正确性。触发器就是为了解决这些缺点而引入的。

#### 3.2 When: Triggers

触发器(`Triggers`)是"处理时间什么时候被物化的？"问题答案的后半部分。触发器声明基于处理时间窗口什么时候输出(尽管触发器本身可以根据其他时间概念作出上述决策，例如基于事件时间的`watermark`处理)。窗口的每个特定输出都称为窗口的窗格(pane)。

用于触发的信号示例如下：
- `watermark`进度(即事件时间进度)，这是我们在上图中已经看到的隐式版本，其中当`watermark`到达窗口末尾时输出被触发。另一个用例是当一个窗口的生命周期结束时会触发垃圾回收，我们稍后会看到一个例子。
- 处理时间进度，这对于提供有规律与周期性的更新是有用的，因为处理时间(不像事件时间)基本一致并没有延迟。
- 元素计数，当一定数量的元素到达在窗口时会触发。
- 标点符号或其他依赖于数据的触发器，某些记录或者记录的特征(例如，EOF元素或刷新事件)指示应该生成输出。

除了基于具体信号触发的简单触发器之外，还有复合触发器，允许创建更复杂的触发逻辑。 复合触发器示例如下：
- 重复`Repetitions`，这在与触发器和处理时间触发器(conjunction with processing time triggers)时特别有用，提供定期周期性的更新。
- 与(逻辑AND)`Conjunctions`，只有当所有子触发器触发时(例如，在`watermark`到达窗口的末端并且我们观察到终止标点符记录之后)才触发。
- 或(逻辑OR)`Disjunctions`，在任何一个子触发器触发时(例如，在`watermark`到达窗口的末端或者我们观察到终止标点符记录之后)才触发。
- 序列`Sequences`，按照预先定义的顺序触发一系列的子触发器。

为了使触发器的概念更加具体一些(并且通过一些东西来建立)，让我们继续：
```
PCollection<KV<String, Integer>> scores = input
  .apply(Window.into(FixedWindows.of(Duration.standardMinutes(2)))
               .triggering(AtWatermark()))
  .apply(Sum.integersPerKey());
```
显式的默认触发器

记住这些，并且对触发器提供什么有基本了解，现在我们可以考虑解决`watermark`太慢或太快的问题。在这两种情况下，我们基本上都希望为给定的窗口提供某种定期周期性的更新，无论是在`watermark`超过窗口末端之前还是之后(除此之外，我们还在`watermark`通过窗口的末端这个阈值时接收到更新)(in addition to the update we’ll receive at the threshold of the watermark passing the end of the window)。所以，我们需要某种重复触发器。那么问题就变成了：我们重复什么？

在太慢的情况下(即提早提供推测结果)，我们应该假定对于任何给定的窗口有稳定的输入数据量，因为我们知道(根据在早期阶段定义的窗口)，我们观察到的窗口输入是远远不完整的。因此，当处理时间提前(例如，每分钟一次)时周期性地触发可能是明智的，因为触发器触发的数量不取决于窗口内观察到的实际数据量；在最坏的情况下，我们会得到稳定的周期性触发器触发规律。

在太快的情况下(即由于`Heuristic watermarks`而提供响应于晚期数据的更新结果)，假设我们的水印基于相对准确的启发（通常是合理安全的假设）。 在这种情况下，我们不希望经常看到迟到的数据，但是当我们这样做的时候，很快就会修改我们的结果。 在观察元素数为1之后触发将使我们快速更新我们的结果（即，每当我们看到晚期数据时），但是由于后期数据的预期频率不足而不可能压倒系统。




















































原文:https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-102
