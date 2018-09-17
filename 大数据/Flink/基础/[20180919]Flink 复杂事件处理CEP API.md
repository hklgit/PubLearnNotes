---
layout: post
author: sjf0115
title: Flink 复杂事件处理CEP API
date: 2018-09-19 15:21:01
tags:
  - Flink
  - Flink 基础

categories: Flink
permalink: api-of-complex-event-processing-with-flink
---

Flink CEP 是在Flink之上实现复杂事件处理（CEP）的库。可以允许你在无穷无尽的事件流中检测事件模式。

这篇文章介绍了Flink CEP中有哪些可用的API。在这里我们首先介绍[Pattern API]()，它允许你在描述如何检测以及匹配事件序列并对其进行操作之前指定需要在流中检测的模式。 然后，我们将介绍CEP库在按事件时间处理延迟时所做的假设，以及如何将你的作业从老的Flink版本迁移到Flink-1.3版本。

### 1. 开始

如果要使用CEP库，需要将FlinkCEP依赖添加到pom.xml中：
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```

> FlinkCEP不是二进制发行包的一部分。

现在，你可以使用 `Pattern API` 编写第一个CEP程序。

> 注意，如果想使用模式去匹配 DataStream 中的事件，事件必须实现正确的 `equals（）`和 `hashCode（）` 方法，因为 FlinkCEP 使用它们来比较和匹配事件。

Java版本：
```java
DataStream<Event> input = ...

Pattern<Event, ?> pattern = Pattern.<Event>begin("start").where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getId() == 42;
            }
        }
    ).next("middle").subtype(SubEvent.class).where(
        new SimpleCondition<SubEvent>() {
            @Override
            public boolean filter(SubEvent subEvent) {
                return subEvent.getVolume() >= 10.0;
            }
        }
    ).followedBy("end").where(
         new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getName().equals("end");
            }
         }
    );

PatternStream<Event> patternStream = CEP.pattern(input, pattern);

DataStream<Alert> result = patternStream.select(
    new PatternSelectFunction<Event, Alert>() {
        @Override
        public Alert select(Map<String, List<Event>> pattern) throws Exception {
            return createAlertFrom(pattern);
        }
    }
});
```
Scala版本：
```scala
val input: DataStream[Event] = ...

val pattern = Pattern.begin[Event]("start").where(_.getId == 42)
  .next("middle").subtype(classOf[SubEvent]).where(_.getVolume >= 10.0).followedBy("end").where(_.getName == "end")

val patternStream = CEP.pattern(input, pattern)

val result: DataStream[Alert] = patternStream.select(createAlert(_))
```
### 2. Pattern　API

Pattern API 允许你定义要从输入流中提取的复杂模式序列。

每个复杂模式序列由多个简单模式组成，即寻找具有相同属性的单个事件的模式。从现在开始，我们将调用这些简单的模式 `patterns`，以及我们要在流中搜索的最终复杂模式序列，即模式序列。你可以将模式序列视为模式的一个图，根据用户指定的发生条件，从一个模式转换到下一个模式，例如，`event.getName（）.equals（"end"）`。匹配是指一系列输入事件满足复杂模式图上的所有模式，并能通过一系列有效的模式转换。

> 每个模式都必须具有唯一的名称，后面你可以使用该名称来标识匹配的事件。模式名称中不能包含字符`：`。

在下面的内容中，我们将介绍如何定义单模式，然后如何将各个模式组合到复杂模式中。

#### 2.1 单模式

`Pattern` 可以是单例也可以是循环模式。单例模式接受单个事件，而循环模式可以接受多个事件。在模式匹配符中，模式 `a b + c？d`（可以理解为：`a` 后跟一个或多个 `b`，`b` 后可选地跟一个 `c`，最后跟一个 `d`），`a`，`c？`，和 `d` 是单例模式，而 `b+` 是循环模式。默认情况下，模式是单例模式，你可以使用 `Quantifiers`（量词） 将其转换为循环模式。每个模式可以有一个或多个条件。

##### 2.1.1 量词

在 FlinkCEP 中，你可以使用如下方法指定循环模式。

使用如下方法来期望指定事件发生一次或多次（例如之前提到的`b+`）：
```
pattern.oneOrMore()
```

使用如下方法用于期望给定类型事件的出现次数的模式（例如，4个`a`）：
```
pattern.times（#ofTimes）
```

使用如下方法用于期望给定类型事件最小出现次数和最大出现次数的模式（例如，2~4个`a`）：
```
pattern.times（#fromTimes，＃toTimes）
```

使用如下方法用于期望给定类型事件出现尽可能多的次数的模式：
```
pattern.greedy（）
```
> 但不能应用到组模式中。

使用如下方法用于指定模式是否使用：
```
pattern.optional（）。
```

Java版本：
```java
// expecting 4 occurrences
start.times(4);

// expecting 0 or 4 occurrences
start.times(4).optional();

// expecting 2, 3 or 4 occurrences
start.times(2, 4);

// expecting 2, 3 or 4 occurrences and repeating as many as possible
start.times(2, 4).greedy();

// expecting 0, 2, 3 or 4 occurrences
start.times(2, 4).optional();

// expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
start.times(2, 4).optional().greedy();

// expecting 1 or more occurrences
start.oneOrMore();

// expecting 1 or more occurrences and repeating as many as possible
start.oneOrMore().greedy();

// expecting 0 or more occurrences
start.oneOrMore().optional();

// expecting 0 or more occurrences and repeating as many as possible
start.oneOrMore().optional().greedy();

// expecting 2 or more occurrences
start.timesOrMore(2);

// expecting 2 or more occurrences and repeating as many as possible
start.timesOrMore(2).greedy();

// expecting 0, 2 or more occurrences and repeating as many as possible
start.timesOrMore(2).optional().greedy();
```
Scala版本：
```scala
// expecting 4 occurrences
start.times(4)

// expecting 0 or 4 occurrences
start.times(4).optional()

// expecting 2, 3 or 4 occurrences
start.times(2, 4)

// expecting 2, 3 or 4 occurrences and repeating as many as possible
start.times(2, 4).greedy()

// expecting 0, 2, 3 or 4 occurrences
start.times(2, 4).optional()

// expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
start.times(2, 4).optional().greedy()

// expecting 1 or more occurrences
start.oneOrMore()

// expecting 1 or more occurrences and repeating as many as possible
start.oneOrMore().greedy()

// expecting 0 or more occurrences
start.oneOrMore().optional()

// expecting 0 or more occurrences and repeating as many as possible
start.oneOrMore().optional().greedy()

// expecting 2 or more occurrences
start.timesOrMore(2)

// expecting 2 or more occurrences and repeating as many as possible
start.timesOrMore(2).greedy()

// expecting 0, 2 or more occurrences
start.timesOrMore(2).optional()

// expecting 0, 2 or more occurrences and repeating as many as possible
start.timesOrMore(2).optional().greedy()
```
##### 2.1.2 条件

对于每个模式，你可以指定传入事件必须满足的条件，以便被模式接受，例如 其值应大于5，或大于先前接受的事件的平均值。你可以通过 `pattern.where（）`， `pattern.or（）`或 `pattern.until（）` 方法指定事件属性的条件。条件可以是 `IterativeConditions` 或 `SimpleConditions`。

###### 2.1.2.1 IterativeConditions

这是最常见的一种条件。根据先前接受的事件的属性或其子集的统计信息来接受后续事件。

下面是一个`IterativeConditions`的示例代码，如果事件名称以 `foo` 开头，并且该模式先前接受的事件的价格总和加上当前事件的价格不会超过5.0，则会接受并为`middle`模式接受下一个事件。

`IterativeConditions`功能是非常强大的，尤其是与循环模式组合，例如，`oneOrMore()`。

Java版本:
```java
middle.oneOrMore()
.subtype(SubEvent.class)
.where(new IterativeCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value, Context<SubEvent> ctx) throws Exception {
        if (!value.getName().startsWith("foo")) {
            return false;
        }
        double sum = value.getPrice();
        for (Event event : ctx.getEventsForPattern("middle")) {
            sum += event.getPrice();
        }
        return Double.compare(sum, 5.0) < 0;
    }
});
```
Scala版本：
```scala
middle.oneOrMore()
.subtype(classOf[SubEvent])
.where(
    (value, ctx) => {
        lazy val sum = ctx.getEventsForPattern("middle").map(_.getPrice).sum
        value.getName.startsWith("foo") && sum + value.getPrice < 5.0
    }
)
```
调用 `ctx.getEventsForPattern（...）` 用来查找所有先前接受的事件。此操作的代价可能会有所不同，因此在实现你的条件时，请尽量减少它的使用。

###### 2.1.2.2 SimpleConditions

这种类型的条件扩展了前面提到的 `IterativeCondition` 类，并且仅根据事件本身的属性决定是否接受事件。

Java版本：
```java
start.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return value.getName().startsWith("foo");
    }
});
```
Scala版本：
```
start.where(event => event.getName.startsWith("foo"))
```
最后，你还可以通过 `pattern.subtype（subClass）` 方法将接受事件的类型限制为初始事件类型（此处为Event）的子类型。

Java版本：
```java
start.subtype(SubEvent.class)
.where(new SimpleCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value) {
        return ... // some condition
    }
});
```
Scala版本：
```
start.subtype(classOf[SubEvent]).where(subEvent => ... /* some condition */)
```

###### 2.1.2.3 CombiningConditions

如上所示，你可以将子类型条件与其他条件组合使用。这适用于所有条件。你可以通过顺序调用 `where()` 来任意组合条件。最终结果将是各个条件的结果的逻辑AND。如果要使用OR组合条件，可以使用`or()`方法，如下所示。

Java版本：
```java
pattern.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // some condition
    }
}).or(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // or condition
    }
});
```
Scala版本：
```
pattern.where(event => ... /* some condition */).or(event => ... /* or condition */)
```
###### 2.1.2.4 Stopcondition

在循环模式（`oneOrMore()` 和 `oneOrMore().optional()`）下，你还可以指定停止条件，例如： 接受值大于5的事件，直到值的总和小于50。为了更好地理解它，请看下面的示例：
- `(a+ until b)` 模式（一个或者多个 `a` 直到遇到 `b`）
- 输入序列，`a1` ,`c` ,`a2` ,`b` ,`a3`
- 输出结果：{a1 a2} {a1} {a2} {a3}

如上所见，由于停止条件，没有返回 `{a1 a2 a3}` 或 `{a2 a3}`。

##### 2.1.3 常用量词与条件

(1) where(condition)

定义当前模式的条件。为了匹配模式，事件必须满足一定的条件。多个连续的 `where()` 子句表示只有在全部满足这些条件时，事件才能匹配该模式。

Java版本：
```java
pattern.where(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // some condition
    }
});
```
Scala版本：
```
pattern.where(event => ... /* some condition */)
```

(2) or(condition)

添加与现有条件进行逻辑或OR运算的新条件。只有在至少通过其中一个条件时，事件才能匹配该模式。

Java版本：
```java
pattern.where(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // some condition
    }
}).or(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // alternative condition
    }
});
```
Scala版本：
```
pattern.where(event => ... /* some condition */)
    .or(event => ... /* alternative condition */)
```
(3) until(condition)

指定循环模式的停止条件。意味着如果匹配给定条件的事件发生，则该模式不再接受其他事件。仅适用于`oneOrMore()`。

Java版本：
```java
pattern.oneOrMore().until(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // alternative condition
    }
});
```
Scala版本：
```
pattern.oneOrMore().until(event => ... /* some condition */)
```

(4) subtype(subClass)

定义当前模式的子类型条件。如果事件属于此子类型，则事件才能匹配该模式。

Java版本：
```java
pattern.subtype(SubEvent.class);
```
Scala版本：
```
pattern.subtype(classOf[SubEvent])
```

(5) oneOrMore()

指定此模式至少发生一次匹配事件。

> 建议使用until（）或within（）来启用状态清除

Java版本：
```java
pattern.oneOrMore();
```
Scala版本：
```
```

(6) timesOrMore(#times)

指定此模式至少发生`#times`次匹配事件。

Java版本：
```java
pattern.timesOrMore(2);
```
Scala版本:
```

```

(7) times(#ofTimes)

(8) times(#fromTimes, #toTimes)

(9) optional()

(10) greedy()



#### 2.2 组合模式

#### 2.3 模式组

#### 2.4 匹配跳过策略

### 3. 检测模式

#### 3.1 从模式中选择

#### 3.2 处理部分超时模式

### 4. 处理基于事件时间的延迟

### 5. Example








> Flink 版本：１.6

原文：https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/cep.html
